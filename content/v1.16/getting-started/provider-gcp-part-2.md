---
title: GCP クイックスタート パート 2
weight: 120
tocHidden: true
aliases:
  - /master/getting-started/provider-azure-part-3
---

{{< hint "重要" >}}
このガイドはシリーズのパート 2 です。  

[**パート 1**]({{<ref "provider-gcp" >}}) では
Crossplane のインストールと Kubernetes クラスターを GCP に接続する方法について説明します。

{{< /hint >}}

このガイドでは、Crossplane を使用してカスタム API を構築し、アクセスする方法を説明します。

## 前提条件
* Kubernetes を GCP に接続する [クイックスタート パート 1]({{<ref "provider-gcp" >}}) を完了してください。
* GCP 
  [ストレージ バケット](https://cloud.google.com/storage) と 
  [Pub/Sub トピック](https://cloud.google.com/pubsub) を作成する権限を持つ GCP アカウント。

{{<expand "パート 1 をスキップしてすぐに始める" >}}
1. Crossplane Helm リポジトリを追加し、Crossplane をインストールします。
```shell
helm repo add \
crossplane-stable https://charts.crossplane.io/stable
helm repo update
&&
helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace
```

2. Crossplane ポッドのインストールが完了し、準備が整ったら、GCP 
プロバイダーを適用します。
   
```yaml {label="provider",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp-storage
spec:
  package: xpkg.upbound.io/upbound/provider-gcp-storage:v0.41.0
EOF
```

3. GCP サービス アカウントの JSON ファイルを使用して `gcp-credentials.json` という名前のファイルを作成します。

{{< hint "ヒント" >}}
[GCP ドキュメント](https://cloud.google.com/iam/docs/creating-managing-service-account-keys) 
には、サービス アカウントの JSON ファイルを生成する方法が記載されています。
{{< /hint >}}

4. GCP JSON ファイルから Kubernetes シークレットを作成します。
```shell {label="kube-create-secret",copy-lines="all"}
kubectl create secret \
generic gcp-secret \
-n crossplane-system \
--from-file=creds=./gcp-credentials.json
```

5. _ProviderConfig_ を作成します。
_ProviderConfig_ 設定に 
{{< hover label="providerconfig" line="7" >}}GCP プロジェクト ID{{< /hover >}} を含めます。

{{< hint type="tip" >}}
`gcp-credentials.json` ファイルの `project_id` フィールドから GCP プロジェクト ID を見つけてください。
{{< /hint >}}

{{< editCode >}}
```yaml {label="providerconfig",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: $@<PROJECT_ID>$@
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-secret
      key: creds
EOF
```
{{< /editCode >}}

{{</expand >}}

## PubSub プロバイダーのインストール

パート 1 では GCP ストレージ プロバイダーのみがインストールされました。このセクションでは、GCP ストレージ バケットと共に PubSub トピックをデプロイします。  
まず、GCP PubSub プロバイダーをインストールします。

新しいプロバイダーをクラスターに追加します。

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp-pubsub
spec:
  package: xpkg.upbound.io/upbound/provider-gcp-pubsub:v0.41.0
EOF
```

新しいPubSubプロバイダーを表示するには、`kubectl get providers`を使用します。

```shell {copy-lines="1"}
kubectl get providers
NAME                          INSTALLED   HEALTHY   PACKAGE                                                AGE
provider-gcp-pubsub           True        True      xpkg.upbound.io/upbound/provider-gcp-pubsub:v0.41.0    39s
provider-gcp-storage          True        True      xpkg.upbound.io/upbound/provider-gcp-storage:v0.41.0   13m
upbound-provider-family-gcp   True        True      xpkg.upbound.io/upbound/provider-family-gcp:v0.41.0    12m
```


## カスタムAPIの作成

<!-- vale alex.Condescending = NO -->
Crossplaneを使用すると、クラウドプロバイダーやそのリソースに関する詳細を抽象化し、ユーザーのために独自のカスタムAPIを構築できます。APIは、あなたの望むように複雑でもシンプルでも構いません。 
<!-- vale alex.Condescending = YES -->

カスタムAPIはKubernetesオブジェクトです。  
以下はカスタムAPIの例です。

```yaml {label="exAPI"}
apiVersion: database.example.com/v1alpha1
kind: NoSQL
metadata:
  name: my-nosql-database
spec: 
  location: "US"
```

他のKubernetesオブジェクトと同様に、APIには 
{{<hover label="exAPI" line="1">}}version{{</hover>}}, 
{{<hover label="exAPI" line="2">}}kind{{</hover>}} および 
{{<hover label="exAPI" line="5">}}spec{{</hover>}}があります。

### グループとバージョンの定義
独自のAPIを作成するには、まず 
[APIグループ](https://kubernetes.io/docs/reference/using-api/#api-groups) と 
[バージョン](https://kubernetes.io/docs/reference/using-api/#api-versioning)を定義します。  

_グループ_は任意の値を使用できますが、一般的な慣習として完全修飾ドメイン名にマッピングすることが推奨されます。

<!-- vale gitlab.SentenceLength = NO -->
バージョンはAPIの成熟度や安定性を示し、API内のフィールドを変更、追加、または削除する際にインクリメントされます。
<!-- vale gitlab.SentenceLength = YES -->

Crossplaneは特定のバージョンや特定のバージョン命名規則を必要としませんが、 
[Kubernetes APIバージョニングガイドライン](https://kubernetes.io/docs/reference/using-api/#api-versioning)に従うことを強く推奨します。 

* `v1alpha1` - いつでも変更される可能性のある新しいAPI。
* `v1beta1` - 安定していると見なされる既存のAPI。破壊的変更は強く推奨されません。
* `v1` - 破壊的変更がない安定したAPI。 

このガイドでは、グループ 
{{<hover label="version" line="1">}}database.example.com{{</hover>}}を使用します。

これはAPIの最初のバージョンであるため、このガイドではバージョン
{{<hover label="version" line="1">}}v1alpha1{{</hover>}}を使用します。

```yaml {label="version",copy-lines="none"}
apiVersion: database.example.com/v1alpha1
```

### 種類を定義する

API グループは、関連する API の論理的なコレクションです。グループ内には、異なるリソースを表す個々の種類があります。

例えば、`queue` グループには `PubSub` と `CloudTask` の種類があるかもしれません。

`kind` は何でも構いませんが、[UpperCamelCased](https://kubernetes.io/docs/contribute/style/style-guide/#use-upper-camel-case-for-api-objects) である必要があります。

この API の種類は 
{{<hover label="kind" line="2">}}PubSub{{</hover>}}

```yaml {label="kind",copy-lines="none"}
apiVersion: queue.example.com/v1alpha1
kind: PubSub
```

### スペックを定義する

API の最も重要な部分はスキーマです。スキーマは、ユーザーから受け入れられる入力を定義します。

この API は、ユーザーがクラウドリソースを実行する場所の 
{{<hover label="spec" line="4">}}location{{</hover>}} を提供することを許可します。

他のすべてのリソース設定はユーザーによって構成できません。これにより、Crossplane はユーザーエラーを心配することなく、ポリシーや基準を強制することができます。

```yaml {label="spec",copy-lines="none"}
apiVersion: queue.example.com/v1alpha1
kind: PubSub
spec: 
  location: "US"
```

### API を適用する

Crossplane は、Kubernetes にカスタム API をインストールするために 
{{<hover label="xrd" line="3">}}Composite Resource Definitions{{</hover>}} 
（`XRD` とも呼ばれます）を使用します。

XRD の {{<hover label="xrd" line="6">}}spec{{</hover>}} には、API に関するすべての情報が含まれています。これには、 
{{<hover label="xrd" line="7">}}group{{</hover>}}、 
{{<hover label="xrd" line="12">}}version{{</hover>}}、 
{{<hover label="xrd" line="9">}}kind{{</hover>}} および 
{{<hover label="xrd" line="13">}}schema{{</hover>}} が含まれます。

XRD の {{<hover label="xrd" line="5">}}name{{</hover>}} は、 
{{<hover label="xrd" line="9">}}plural{{</hover>}} と 
{{<hover label="xrd" line="7">}}group{{</hover>}} の組み合わせでなければなりません。

{{<hover label="xrd" line="13">}}schema{{</hover>}} は、API {{<hover label="xrd" line="17">}}spec{{</hover>}} を定義するために 
{{<hover label="xrd" line="14">}}OpenAPIv3{{</hover>}} 仕様を使用します。

API は、{{<hover label="xrd" line="20">}}location{{</hover>}} を定義しており、これは 
{{<hover label="xrd" line="22">}}oneOf{{</hover>}} で、 
{{<hover label="xrd" line="23">}}EU{{</hover>}} または 
{{<hover label="xrd" line="24">}}US{{</hover>}} のいずれかでなければなりません。

このXRDを適用して、KubernetesクラスターにカスタムAPIを作成します。

```yaml {label="xrd",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: pubsubs.queue.example.com
spec:
  group: queue.example.com
  names:
    kind: PubSub
    plural: pubsubs
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              location:
                type: string
                oneOf:
                  - pattern: '^EU$'
                  - pattern: '^US$'
            required:
              - location
    served: true
    referenceable: true
  claimNames:
    kind: PubSubClaim
    plural: pubsubclaims
EOF
```

{{<hover label="xrd" line="29">}}claimNames{{</hover>}}を追加することで、ユーザーはこのAPIにアクセスできるようになります。クラスター全体での 
{{<hover label="xrd" line="9">}}pubsub{{</hover>}} エンドポイントまたは、名前空間内での 
{{<hover label="xrd" line="29">}}pubsubclaim{{</hover>}} エンドポイントを使用します。

名前空間スコープのAPIは、Crossplaneの _Claim_ です。

{{<hint "tip" >}}
Composite Resource Definitionsのフィールドとオプションの詳細については、 
[XRD documentation]({{<ref "../concepts/composite-resource-definitions">}})をお読みください。 
{{< /hint >}}

インストールされたXRDを表示するには、`kubectl get xrd`を実行します。

```shell {copy-lines="1"}
kubectl get xrd
NAME                        ESTABLISHED   OFFERED   AGE
pubsubs.queue.example.com   True          True      7s
```

新しいカスタムAPIエンドポイントを表示するには、`kubectl api-resources | grep pubsub`を実行します。

```shell {copy-lines="1",label="apiRes"}
kubectl api-resources | grep queue.example
pubsubclaims                 queue.example.com/v1alpha1             true         PubSubClaim
pubsubs                      queue.example.com/v1alpha1             false        PubSub
```

## デプロイメントテンプレートの作成

ユーザーがカスタムAPIにアクセスすると、Crossplaneはその入力を受け取り、デプロイするインフラストラクチャを説明するテンプレートと組み合わせます。Crossplaneはこのテンプレートを _Composition_ と呼びます。

{{<hover label="comp" line="3">}}Composition{{</hover>}} は、デプロイするすべてのクラウドリソースを定義します。
テンプレート内の各エントリは、リソース設定やメタデータ（ラベルやアノテーションなど）を定義する完全なリソース定義です。

このテンプレートは、GCPの 
{{<hover label="comp" line="10">}}Storage{{</hover>}} 
{{<hover label="comp" line="11">}}Bucket{{</hover>}} と 
{{<hover label="comp" line="25">}}PubSub{{</hover>}} 
{{<hover label="comp" line="26">}}Topic{{</hover>}} を作成します。

Crossplaneは、ユーザーの入力をリソーステンプレートに適用するために 
{{<hover label="comp" line="15">}}patches{{</hover>}} を使用します。  
このCompositionは、ユーザーの 
{{<hover label="comp" line="16">}}location{{</hover>}} 入力を受け取り、それを個々のリソースで使用される 
{{<hover label="comp" line="14">}}location{{</hover>}} として使用します。


このCompositionをクラスターに適用します。 

```yaml {label="comp",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: topic-with-bucket
spec:
  resources:
    - name: crossplane-quickstart-bucket
      base:
        apiVersion: storage.gcp.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            location: "US"
      patches:
        - fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.location"
          transforms:
            - type: map
              map: 
                EU: "EU"
                US: "US"
    - name: crossplane-quickstart-topic
      base:
        apiVersion: pubsub.gcp.upbound.io/v1beta1
        kind: Topic
        spec:
          forProvider:
            messageStoragePolicy:
              - allowedPersistenceRegions: 
                - "us-central1"
      patches:
        - fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.messageStoragePolicy[0].allowedPersistenceRegions[0]"
          transforms:
            - type: map
              map: 
                EU: "europe-central2"
                US: "us-central1"
  compositeTypeRef:
    apiVersion: queue.example.com/v1alpha1
    kind: PubSub
EOF
```

{{<hover label="comp" line="40">}}compositeTypeRef{{</hover >}}は
このテンプレートを使用してリソースを作成できるカスタムAPIを定義します。

{{<hint "tip" >}}
[Compositionのドキュメント]({{<ref "../concepts/compositions">}})を読んで
Compositionの構成や利用可能なオプションについての詳細を確認してください。

[Patch and Transformのドキュメント]({{<ref "../concepts/patch-and-transform">}})を読んで
Crossplaneがパッチを使用してユーザー入力をCompositionリソーステンプレートにマッピングする方法についての詳細を確認してください。
{{< /hint >}}

`kubectl get composition`でCompositionを表示します。

```shell {copy-lines="1"}
kubectl get composition
NAME                XR-KIND   XR-APIVERSION       AGE
topic-with-bucket   PubSub    queue.example.com   3s
```

## カスタムAPIにアクセスする

カスタムAPI（XRD）がインストールされ、リソーステンプレート（Composition）に関連付けられると、ユーザーはAPIにアクセスしてリソースを作成できます。

{{<hover label="xr" line="2">}}PubSub{{</hover>}}オブジェクトを作成して
クラウドリソースを作成します。

```yaml {copy-lines="all",label="xr"}
cat <<EOF | kubectl apply -f -
apiVersion: queue.example.com/v1alpha1
kind: PubSub
metadata:
  name: my-pubsub-queue
spec: 
  location: "US"
EOF
```

`kubectl get pubsub`でリソースを表示します。

```shell {copy-lines="1"}
kubectl get pubsub
NAME              SYNCED   READY   COMPOSITION         AGE
my-pubsub-queue   True     True    topic-with-bucket   2m12s
```

このオブジェクトはCrossplaneの_コンポジットリソース_（`XR`とも呼ばれます）です。  
これは、Compositionテンプレートから作成されたリソースのコレクションを表す単一のオブジェクトです。 

`kubectl get managed`で個々のリソースを表示します。

```shell {copy-lines="1"}
kubectl get managed
NAME                                                READY   SYNCED   EXTERNAL-NAME           AGE
topic.pubsub.gcp.upbound.io/my-pubsub-queue-cjswx   True    True     my-pubsub-queue-cjswx   3m4s

NAME                                                  READY   SYNCED   EXTERNAL-NAME           AGE
bucket.storage.gcp.upbound.io/my-pubsub-queue-vljg9   True    True     my-pubsub-queue-vljg9   3m4s
```

`kubectl delete pubsub`でリソースを削除します。

```shell {copy-lines="1"}
kubectl delete pubsub my-pubsub-queue
pubsub.queue.example.com "my-pubsub-queue" deleted
```

`kubectl get managed`でCrossplaneがリソースを削除したことを確認します。

{{<hint "note" >}}
リソースを削除するのに最大5分かかる場合があります。
{{< /hint >}}

```shell {copy-lines="1"}
kubectl get managed
No resources found
```

## 名前空間を使用したAPIの利用

API `pubsub` へのアクセスはクラスターのスコープで行われます。  
ほとんどの組織は
ユーザーを名前空間に隔離します。  

Crossplaneの _Claim_ は名前空間内のカスタムAPIです。

_Claim_ を作成することは、カスタムAPIエンドポイントにアクセスするのと同じですが、カスタムAPIの `claimNames` からの
{{<hover label="claim" line="3">}}kind{{</hover>}} を使用します。

Claimを作成するための新しい名前空間を作成します。

```shell
kubectl create namespace crossplane-test
```

次に、`crossplane-test` 名前空間にClaimを作成します。

```yaml {label="claim",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: queue.example.com/v1alpha1
kind: PubSubClaim
metadata:  
  name: my-pubsub-queue
  namespace: crossplane-test
spec: 
  location: "US"
EOF
```
`kubectl get claim -n crossplane-test` でClaimを表示します。

```shell {copy-lines="1"}
kubectl get claim -n crossplane-test
NAME                SYNCED   READY   CONNECTION-SECRET   AGE
my-pubsub-queue   True     True                        2m10s
```

Claimは自動的に複合リソースを作成し、それが管理リソースを作成します。

`kubectl get composite` でCrossplaneが作成した複合リソースを表示します。

```shell {copy-lines="1"}
kubectl get composite
NAME                    SYNCED   READY   COMPOSITION         AGE
my-pubsub-queue-7bm9n   True     True    topic-with-bucket   3m10s
```

再度、`kubectl get managed` で管理リソースを表示します。

```shell {copy-lines="1"}
kubectl get managed
NAME                                                      READY   SYNCED   EXTERNAL-NAME                 AGE
topic.pubsub.gcp.upbound.io/my-pubsub-queue-7bm9n-6kdq4   True    True     my-pubsub-queue-7bm9n-6kdq4   3m22s

NAME                                                        READY   SYNCED   EXTERNAL-NAME                 AGE
bucket.storage.gcp.upbound.io/my-pubsub-queue-7bm9n-hhwx8   True    True     my-pubsub-queue-7bm9n-hhwx8   3m22s
```

Claimを削除すると、すべてのCrossplane生成リソースが削除されます。

`kubectl delete claim -n crossplane-test my-pubsub-queue`

```shell {copy-lines="1"}
kubectl delete pubsubclaim my-pubsub-queue -n crossplane-test
pubsubclaim.queue.example.com "my-pubsub-queue" deleted
```

{{<hint "note" >}}
リソースの削除には最大で5分かかる場合があります。
{{< /hint >}}

`kubectl get composite` でCrossplaneが複合リソースを削除したことを確認します。

```shell {copy-lines="1"}
kubectl get composite
No resources found
```

`kubectl get managed` でCrossplaneが管理リソースを削除したことを確認します。

```shell {copy-lines="1"}
kubectl get managed
No resources found
```

## 次のステップ
* Crossplaneが構成できるAWSリソースを 
  [Provider CRDリファレンス](https://marketplace.upbound.io/providers/upbound/provider-family-aws/) で探ります。
* [Crossplane Slack](https://slack.crossplane.io/) に参加し、 
  Crossplaneのユーザーや貢献者とつながります。
* Crossplaneでできることをさらに知るために、[Crossplaneの概念]({{<ref "../concepts">}})についてもっと読む。
