---
title: マルチテナント Crossplane
weight: 240
---

このガイドでは、Kubernetes プリミティブとクラウドネイティブエコシステムの互換性のあるポリシー強制プロジェクトを活用して、マルチテナント環境で Crossplane を効果的に使用する方法について説明します。

## TL;DR

マルチテナント Crossplane 環境のインフラストラクチャオペレーターは、通常、構成と Kubernetes RBAC を利用して、インフラストラクチャを要求する際に開発者に与えられるセルフサービスのレベルを定義する軽量で標準化されたポリシーを定義します。これは主に、名前空間スコープで抽象リソースタイプを公開し、その名前空間内のチームや個人のために `Roles` を定義し、基盤となる管理リソースの `spec.providerConfigRef` をパッチして、各名前空間からプロビジョニングされる際に特定の `ProviderConfig` と資格情報を使用するようにすることで達成されます。大規模な組織や、より複雑な環境を持つ組織は、サードパーティのポリシーエンジンを組み込むか、複数の Crossplane クラスターにスケールアップすることを選択する場合があります。以下のセクションでは、これらのシナリオのそれぞれについて詳しく説明します。

- [TL;DR](#tldr)
- [背景](#background)
  - [クラスター スコープの管理リソース](#cluster-scoped-managed-resources)
  - [名前空間スコープのクレーム](#namespace-scoped-claims)
- [単一クラスターのマルチテナンシー](#single-cluster-multi-tenancy)
  - [隔離メカニズムとしての構成](#composition-as-an-isolation-mechanism)
  - [隔離メカニズムとしての名前空間](#namespaces-as-an-isolation-mechanism)
  - [Open Policy Agentによるポリシー強制](#policy-enforcement-with-open-policy-agent)
- [マルチクラスターのマルチテナンシー](#multi-cluster-multi-tenancy)
  - [構成パッケージによる再現可能なプラットフォーム](#reproducible-platforms-with-configuration-packages)
  - [コントロールプレーンのコントロールプレーン](#control-plane-of-control-planes)

## 背景

Crossplane は、多くのチームがクラスター内のインフラストラクチャオペレーターによって提供されるサービスと抽象化を利用するマルチテナント環境で実行されるように設計されています。この機能は、Crossplane エコシステム内の 2 つの主要なデザインパターンによって促進されます。

### クラスター範囲の管理リソース

通常、外部APIを反映する詳細な [管理リソース] を提供するCrossplaneプロバイダーは、資格情報ソース（Kubernetesの`Secret`、`Pod`ファイルシステム、または環境変数など）を指す`ProviderConfig`オブジェクトを使用して認証します。その後、すべての管理リソースは、そのリソースタイプを管理するのに十分な権限を持つ資格情報を指す`ProviderConfig`を参照します。

たとえば、以下の`provider-aws`の`ProviderConfig`は、AWS資格情報を持つKubernetesの`Secret`を指しています。

```yaml
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: cool-aws-creds
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
```

ユーザーがこれらの資格情報を使用して`RDSInstance`をプロビジョニングしたい場合、オブジェクトマニフェストで`ProviderConfig`を参照します。

```yaml
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: rdsmysql
spec:
  forProvider:
    region: us-east-1
    dbInstanceClass: db.t3.medium
    masterUsername: masteruser
    allocatedStorage: 20
    engine: mysql
    engineVersion: "5.6.35"
    skipFinalSnapshotBeforeDeletion: true
  providerConfigRef:
    name: cool-aws-creds # name of ProviderConfig above
  writeConnectionSecretToRef:
    namespace: crossplane-system
    name: aws-rdsmysql-conn
```

`ProviderConfig`とすべての管理リソースがクラスター範囲であるため、`provider-aws`のRDSコントローラーはこの参照を解決するために`ProviderConfig`を取得し、それが指す資格情報を取得し、それらの資格情報を使用して`RDSInstance`を調整します。これは、`RDSInstance`オブジェクトを管理するために[RBAC]を与えられた誰でも、任意の資格情報を使用できることを意味します。実際には、Crossplaneはインフラストラクチャ管理者またはプラットフォームビルダーとして行動する人々だけがクラスター範囲のリソースと直接対話することを前提としています。

### ネームスペーススコープのクレーム

管理リソースはクラスター範囲に存在しますが、**CompositeResourceDefinition (XRD)**を使用して定義された複合リソースは、クラスター範囲またはネームスペース範囲のいずれかに存在する可能性があります。プラットフォームビルダーは、XRDのインスタンスの作成に応じてどの詳細な管理リソースを作成するかを指定するXRDと**Composition**を定義します。このアーキテクチャに関する詳細情報は、[Composition]ドキュメントにあります。

すべてのXRDはクラスター範囲で公開されますが、`spec.claimNames`が定義されているものだけがネームスペーススコープのバリアントを持ちます。

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xmysqlinstances.example.org
spec:
  group: example.org
  names:
    kind: XMySQLInstance
    plural: xmysqlinstances
  claimNames:
    kind: MySQLInstance
    plural: mysqlinstances
...
```

上記の例が作成されると、Crossplaneは2つの[CustomResourceDefinitions]を生成します：
1. `kind: XMySQLInstance`を持つクラスター範囲のタイプ。これは**Composite Resource (XR)**と呼ばれます。
2. `kind: MySQLInstance`を持つネームスペース範囲のタイプ。これは**Claim (XRC)**と呼ばれます。

プラットフォームビルダーは、これらのタイプにマッピングされる任意の数のコンポジションを定義することを選択できるため、特定のネームスペースで `MySQLInstance` を作成すると、クラスターのスコープで任意の管理リソースのセットが作成される可能性があります。たとえば、`MySQLInstance` を作成すると、上記で定義された `RDSInstance` の作成が行われる可能性があります。

## シングルクラスターのマルチテナンシー

組織の規模や範囲に応じて、プラットフォームチームは1つの中央のCrossplaneコントロールプレーンを運用するか、各チームやビジネスユニットのために多くの異なるものを運用することを選択できます。このセクションでは、組織内の他の多くのCrossplaneクラスターの1つであるかどうかにかかわらず、単一のクラスター内で複数のチームにサービスを提供することに焦点を当てます。

### 孤立メカニズムとしてのコンポジション

管理リソースは常に基盤となるプロバイダーAPIが公開するすべてのフィールドを反映しますが、XRDはプラットフォームビルダーが選択した任意のスキーマを持つことができます。XRDスキーマ内のフィールドは、コンポジションで定義された基盤となる管理リソースのフィールドにパッチを当てることができ、実質的にそれらのフィールドをXRまたはXRCの消費者に対して構成可能として公開します。

この機能は、消費者に対して基盤となるリソースをプラットフォームビルダーが望む範囲でカスタマイズする能力のみを与えることによって、軽量なポリシーメカニズムとして機能します。たとえば、上記の例では、プラットフォームビルダーは、`east` と `west` のオプションを持つ列挙型である `XMySQLInstance` のスキーマ内に `spec.location` フィールドを定義することを選択するかもしれません。コンポジション内では、これらのフィールドは `RDSInstance` の `spec.region` フィールドにマッピングされ、値は `us-east-1` または `us-west-1` になります。`RDSInstance` に対して他のパッチが定義されていない場合、ユーザーに `XMySQLInstance` / `MySQLInstance` を作成する能力を与えることは、非常に特定の構成の `RDSInstance` を作成する能力を与えることに等しく、ユーザーはそれが存在するリージョンを決定することができ、2つのオプションに制限されます。

このモデルは、エンドユーザーが抽象化からレンダリングされる基盤となるリソースを作成するためにプロバイダーの資格情報を持たなければならない多くのインフラストラクチャコードツールとは対照的です。Crossplaneは異なるアプローチを取り、クラスター内でさまざまな資格情報を定義し（`ProviderConfig`を使用）、プロバイダーコントローラーのみにその資格情報を利用してユーザーの代理でインフラストラクチャをプロビジョニングする能力を与えます。これにより、異なるIAMモデルを持つ多くのプロバイダーを使用しても、一貫した権限モデルが作成され、Kubernetes RBACに標準化されます。

### 名前空間を隔離メカニズムとして

抽象スキーマと具体的なリソースタイプへのパッチを定義する能力は強力ですが、名前空間スコープでクレームタイプを定義する能力は、名前空間制限を適用できるRBACを可能にすることで機能をさらに強化します。クラスター内のほとんどのユーザーは、KubernetesとCrossplaneの両方によってインフラストラクチャ管理者にのみ関連すると見なされるため、クラスター範囲のリソースにアクセスできません。

シンプルな `XMySQLInstance` / `MySQLInstance` の例を基に、プラットフォームビルダーは `Role` を使用して名前空間スコープで `MySQLInstance` に対する権限を定義することを選択できます。これにより、ユーザーは自分の名前空間内で `MySQLInstances` を作成および管理する能力を持ちますが、他の名前空間で定義されたものを見る能力は持ちません。

さらに、`metadata.namespace` がXRCのフィールドであるため、パッチを利用して対応するXRCが定義された名前空間に基づいて管理リソースを構成できます。これは、プラットフォームビルダーが特定の資格情報または特定の名前空間内のユーザーがXRCを使用してインフラストラクチャをプロビジョニングする際に利用できる資格情報のセットを指定したい場合に特に便利です。これは、`ProviderConfig` 名に名前空間の名前を含む1つ以上の `ProviderConfig` オブジェクトを作成することで実現できます。たとえば、`team-1` 名前空間で作成された任意の `MySQLInstance` が、プロバイダーコントローラーが基盤となる `RDSInstance` を作成する際に特定のAWS資格情報を使用する必要がある場合、プラットフォームビルダーは次のようにできます。

1. 名前 `team-1` の `ProviderConfig` を定義します。

```yaml
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: team-1
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: team-1-creds
      key: creds
```

2. XR内のクレーム参照の名前空間を `RDSInstance` の `providerConfigRef` にパッチする `Composition` を定義します。

```yaml
...
resources:
- base:
    apiVersion: database.aws.crossplane.io/v1beta1
    kind: RDSInstance
    spec:
      forProvider:
      ...
  patches:
  - fromFieldPath: spec.claimRef.namespace
    toFieldPath: spec.providerConfigRef.name
    policy:
      fromFieldPath: Required
```

これにより、`RDSInstance` は対応する `MySQLInstance` が作成された名前空間の `ProviderConfig` を使用することになります。

> このモデルは現在、名前空間ごとに単一の `ProviderConfig` のみを許可しています。ただし、将来のCrossplaneリリースでは、[Multiple Source Field patching] を使用して選択できる `ProviderConfig` のセットを定義できるようになるはずです。

### Open Policy Agentによるポリシーの強制

一部のCrossplaneデプロイメントモデルでは、ポリシーを定義するためにコンポジションとRBACのみを使用することは、十分な柔軟性を持たない場合があります。しかし、Crossplaneは外部インフラストラクチャの管理をKubernetes APIに統合するため、クラウドネイティブエコシステム内の他のプロジェクトと統合するのに適しています。より堅牢なポリシーエンジンが必要な組織や個人、またはポリシーを定義するためのより一般的な言語を好む場合は、[Open Policy Agent](OPA)に目を向けることがよくあります。OPAはプラットフォームビルダーが[Rego]というドメイン固有言語でカスタムロジックを書くことを可能にします。この方法でポリシーを書くことで、評価されている特定のリソースで利用可能な情報を取り入れるだけでなく、クラスター内で表現されている他の状態を使用することもできます。Crossplaneユーザーは通常、ポリシー管理をできるだけ効率的にするためにOPAの[Gatekeeper]をインストールします。

> OPAをCrossplaneと一緒に使用するライブデモは[こちら]で見ることができます。

## マルチクラスター マルチテナンシー

多くのクラスターにCrossplaneをデプロイする組織は、複数のコントロールプレーンを管理するのをはるかに簡単にする2つの主要な機能を活用することが一般的です。

### 設定パッケージによる再現可能なプラットフォーム

[設定パッケージ]は、プラットフォームビルダーが自分のXRDとコンポジションを[OCIイメージ]にパッケージ化し、任意のOCI準拠のイメージレジストリを介して配布できるようにします。これらのパッケージはプロバイダーへの依存関係を宣言することもでき、単一のパッケージがすべての詳細な管理リソース、これらを調整するためにデプロイする必要があるコントローラー、およびコンポジションを使用して基盤となるリソースを公開する抽象型を宣言できます。

多くのCrossplaneデプロイメントを持つ組織は、各クラスターでプラットフォームを再現するために設定パッケージを利用します。これは、Crossplaneをインストールする際に設定パッケージを自動的にインストールするフラグを付けるだけで済む場合もあります。

```
helm install crossplane --namespace crossplane-system crossplane-stable/crossplane --set configuration.packages='{"registry.upbound.io/xp/getting-started-with-aws:latest"}'
```

### コントロールプレーンのコントロールプレーン

マルチクラスター・マルチテナンシーモデルをさらに一歩進めて、一部の組織は単一の中央Crossplaneコントロールプレーンを使用して多くのCrossplaneクラスターを管理することを選択します。これには、中央クラスターのセットアップが必要で、その後、プロバイダーを使用して新しいクラスターを立ち上げます（例えば、[EKS Cluster]を[provider-aws]を使用して）、次に、[provider-helm]を使用して新しいリモートクラスターにCrossplaneをインストールし、上記の方法で共通のConfigurationパッケージを各インストールにバンドルすることができます。

この高度なパターンは、Crossplane自体を使用してCrossplaneクラスターを完全に管理することを可能にし、適切に実行されれば、単一の組織内の多くのテナントに専用のコントロールプレーンを提供するためのスケーラブルなソリューションとなります。


<!-- Named Links -->
[managed resources]: {{<ref "../../master/concepts/managed-resources" >}}
[RBAC]: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
[Composition]: {{<ref "../../master/concepts/compositions" >}}
[CustomResourceDefinitions]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
[Open Policy Agent]: https://www.openpolicyagent.org/
[Rego]: https://www.openpolicyagent.org/docs/latest/policy-language/
[Gatekeeper]: https://open-policy-agent.github.io/gatekeeper/website/docs/
[here]: https://youtu.be/TaF0_syejXc
[Multiple Source Field patching]: https://github.com/crossplane/crossplane/pull/2093
[Configuration packages]: {{<ref "../../master/concepts/packages" >}}
[OCI images]: https://github.com/opencontainers/image-spec
[EKS Cluster]: https://marketplace.upbound.io/providers/crossplane-contrib/provider-aws/latest/resources/eks.aws.crossplane.io/Cluster/v1beta1
[provider-aws]: https://marketplace.upbound.io/providers/crossplane-contrib/provider-aws
[provider-helm]: https://marketplace.upbound.io/providers/crossplane-contrib/provider-helm/
[Open Service Broker API]: https://github.com/openservicebrokerapi/servicebroker
[Crossplane Service Broker]: https://github.com/vshn/crossplane-service-broker
[Cloudfoundry]: https://www.cloudfoundry.org/
[Kubernetes Service Catalog]: https://github.com/kubernetes-sigs/service-catalog
[vshn/application-catalog-demo]: https://github.com/vshn/application-catalog-demo

It seems that there is no content provided for translation. Please paste the Markdown content you'd like me to translate into Japanese.
