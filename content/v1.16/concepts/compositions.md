---
title: コンポジション
weight: 30
aliases: 
  - composition
description: "コンポジションはCrossplaneリソースを作成するためのテンプレートです"
---

コンポジションは、複数の管理リソースを単一のオブジェクトとして作成するためのテンプレートです。

コンポジションは、個々の管理リソースを組み合わせて、より大きく再利用可能なソリューションを作成します。

例として、コンポジションは仮想マシン、ストレージリソース、およびネットワークポリシーを組み合わせることがあります。コンポジションテンプレートは、これらの個々のリソースをすべてリンクします。

{{<expand "コンポジション、XRD、XR、およびクレームについて混乱していますか？" >}}
Crossplaneには、ユーザーが混同しがちな4つのコアコンポーネントがあります：

* コンポジション - このページ。リソースを作成する方法を定義するためのテンプレート。
* [複合リソース定義]({{<ref "./composite-resource-definitions">}})
  (`XRD`) - カスタムAPI仕様。
* [複合リソース]({{<ref "./composite-resources">}}) (`XR`) - 複合リソース定義で定義されたカスタムAPIを使用して作成されます。XRは、コンポジションテンプレートを使用して新しい管理リソースを作成します。
* [クレーム]({{<ref "./claims" >}}) (`XRC`) - 複合リソースのようですが、名前空間スコープがあります。
{{</expand >}}

## コンポジションの作成

コンポジションの作成は以下で構成されます：
* [リソーステンプレート](#resource-templates)が作成するリソースを定義します。
* [複合リソースの有効化](#enabling-composite-resources)がこのコンポジションテンプレートを使用します。

<!-- vale Google.WordList = NO -->
オプションとして、コンポジションは以下もサポートします：
* [リソース設定の変更とパッチ適用](#changing-resource-fields)。
* 管理リソースによって生成された接続詳細とシークレットの[保存](#storing-connection-details)。
* カスタムプログラムを使用してリソースをテンプレート化するための[コンポジション関数](#use-composition-functions)の使用。
* リソースが準備完了であるかどうかを確認するための[カスタムチェックの作成](#resource-readiness-checks)。
<!-- vale Google.WordList = YES -->

### リソーステンプレート
コンポジションの{{<hover label="resources" line="4">}}resources{{</hover>}}フィールドは、複合リソースが作成するもののセットを定義します。


{{<hint "note" >}}
Composite Resources についての詳細は 
[Composite Resources ページ]({{<ref "./composite-resources" >}})を参照してください。 
{{< /hint >}}
  

例えば、Composition は仮想マシンを作成するためのテンプレートと、同時に関連するストレージバケットを定義できます。 

{{<hover label="resources" line="4">}}resources{{</hover>}} フィールドは、 
個々のリソースを 
{{<hover label="resources" line="5">}}name{{</hover>}} でリストします。  
この名前は、Composition 内でリソースを識別し、Provider で使用される外部名とは関係ありません。

#### 管理リソースのテンプレート
{{<hover label="resources" line="6">}}base{{</hover>}} の内容は、 
スタンドアロンの [managed resource]({{<ref "./managed-resources">}}) を作成するのと同じです。

この例では、 
[UpboundのProvider AWS](https://marketplace.upbound.io/providers/upbound/provider-aws/v0.35.0) を使用して、 
S3 ストレージ {{<hover label="resources" line="8">}}Bucket{{</hover>}} と 
EC2 コンピュート {{<hover label="resources" line="15">}}Instance{{</hover>}} を定義します。

{{<hover label="resources" line="7">}}apiVersion{{</hover>}} と 
{{<hover label="resources" line="8">}}kind{{</hover>}} を定義した後、 
リソースの設定を定義する 
{{<hover label="resources" line="10">}}spec.forProvider{{</hover>}} フィールドを定義します。

```yaml {copy-lines="none",label="resources"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - name: StorageBucket
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: "us-east-2"
    - name: VM
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            ami: ami-0d9858aa3c6322f73
            instanceType: t2.micro
            region: "us-east-2"
```

[Composite Resource]({{<ref "./composite-resources" >}}) がこの 
Composition テンプレートを使用すると、Composite Resource は提供された 
{{<hover label="resources" line="17">}}spec.forProvider{{</hover>}} 設定を持つ2つの新しい管理リソースを作成します。

{{<hover label="resources" line="16">}}spec{{</hover>}} は、 
管理リソースで使用される任意の設定をサポートし、 
`annotations` や `labels` を適用したり、特定の 
`providerConfigRef` を使用したりできます。

{{<hint "note" >}}
Compositions はリソースの 
{{<hover label="resources" line="16">}}spec{{</hover>}} に `metadata.name` を適用することを許可しますが、無視します。 
`metadata.name` フィールドは、Crossplane 内の管理リソースや 
Provider 内の外部リソースの名前に影響を与えません。

リソースに `crossplane.io/external-name` アノテーションを使用して、外部リソース名に影響を与えます。
{{< /hint >}}

#### ProviderConfigのテンプレート

コンポジションは、管理リソースを定義するのと同様にProviderConfigを定義できます。
ProviderConfigを生成することは、各デプロイメントにユニークな資格情報を提供するのに役立ちます。

```yaml {copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - name: my-aws-provider-config
      base:
        apiVersion: aws.upbound.io/v1beta1
        kind: ProviderConfig
        spec:
          source: Secret
          secretRef:
            namespace: crossplane-system
            name: aws-secret
            key: creds
```

#### 別の複合リソースのテンプレート

コンポジションは、他の複合リソースを使用して、より複雑なテンプレートを定義できます。

一般的なユースケースは、他のコンポジションを使用するコンポジションです。たとえば、他のコンポジションが参照する標準的なネットワークリソースのセットを作成するためのコンポジションを作成します。

{{< hint "note" >}}
両方のコンポジションは、対応するXRDを持っている必要があります。
{{< /hint >}}

この例のネットワーキングコンポジションは、新しいAWS仮想ネットワークを作成するために必要なリソースのセットを定義します。これには
{{<hover label="xnetwork" line="8">}}VPC{{</hover>}},
{{<hover label="xnetwork" line="13">}}InternetGateway{{</hover>}} および
{{<hover label="xnetwork" line="18">}}Subnet{{</hover>}}が含まれます。

```yaml {copy-lines="none",label="xnetwork"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - name: vpc-resource
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: VPC
        # Removed for Brevity
    - name: gateway-resource
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: InternetGateway
        # Removed for Brevity
    - name: subnet-resource
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: Subnet
        # Removed for Brevity
  compositeTypeRef:
    apiVersion: aws.platformref.upbound.io/v1alpha1
    kind: XNetwork
```

{{<hover label="xnetwork" line="20">}}compositeTypeRef{{</hover>}}は、このコンポジションに
{{<hover label="xnetwork" line="21">}}apiVersion{{</hover>}}と
{{<hover label="xnetwork" line="22">}}kind{{</hover>}}を与え、別のコンポジションで参照できるようにします。

{{<hint "note" >}}
[複合リソースの有効化](#enabling-composite-resources)セクションでは、
{{<hover label="xnetwork" line="20">}}compositeTypeRef{{</hover>}}フィールドについて説明しています。
{{< /hint >}}

別のコンポジションは、この例ではAWS Elastic Kubernetes Clusterを定義し、前の
{{<hover label="xnetwork" line="22">}}XNetwork{{</hover>}}を参照できます。

```yaml {copy-lines="none",label="xcluster"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - name: nested-network-composition
      base:
        apiVersion: aws.platformref.upbound.io/v1alpha1
        kind: XNetwork
        # Removed for brevity
    - name: eks-cluster-resource
      base:
        apiVersion: eks.aws.upbound.io/v1beta1
        kind: Cluster
        # Removed for brevity
```

複合リソースがこのコンポジションからすべての管理リソースを作成すると、 
{{<hover label="xcluster" line="8">}}XNetwork{{</hover>}}によって定義されたリソースがEKS 
{{<hover label="xcluster" line="13">}}クラスター{{</hover >}}とともに作成されます。


{{<hint "note" >}}
この省略された例は、Upboundの 
[AWS Reference Platform](https://github.com/upbound/platform-ref-aws)からのものです。

参照プラットフォームの完全なコンポジションは 
[パッケージディレクトリ](https://github.com/upbound/platform-ref-aws/blob/main/apis/cluster/composition.yaml)で確認できます。
{{</hint >}}

#### リソース間の参照

コンポジション内のいくつかのリソースは、他のリソースの識別子や名前を使用します。
例えば、新しい `network` を作成し、そのネットワーク識別子を
仮想マシンに適用することです。

コンポジション内のリソースは、ラベルや _コントローラー参照_ を一致させることで
他のリソースを相互参照できます。

{{<hint "note" >}}
プロバイダーは、リソースごとにラベルとコントローラー参照の一致を許可します。
特定のプロバイダーリソースのドキュメントを確認して、サポートされている内容を確認してください。

異なるプロバイダー間でのラベルとコントローラーの一致はサポートされていません。
{{< /hint >}}

##### リソースラベルの一致

リソースラベルを一致させるには、まず 
{{<hover label="matchlabel" line="11">}}label{{</hover>}}を一致させるリソースに適用し、
次のリソースで 
{{<hover label="matchlabel" line="19">}}matchLabels{{</hover>}}を使用します。

この例では、AWS 
{{<hover label="matchlabel" line="7">}}Role{{</hover>}}を作成し、 
{{<hover label="matchlabel" line="11">}}label{{</hover>}}を適用します。2つ目のリソースは 
{{<hover label="matchlabel" line="14">}}RolePolicyAttachment{{</hover>}}であり、既存の `Role` に
アタッチする必要があります。

リソースの 
{{<hover label="matchlabel" line="19">}}roleSelector.matchLabels{{</hover>}}を使用することで、
この 
{{<hover label="matchlabel" line="14">}}RolePolicyAttachment{{</hover>}}が一致する 
{{<hover label="matchlabel" line="7">}}Role{{</hover>}}を参照することが保証されます。たとえユニークなロール
識別子が知られていなくてもです。

```yaml {label="matchlabel",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: Role
        name: iamRole
        metadata:
          labels:
            role: controlplane
    - base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: RolePolicyAttachment
        name: iamPolicy
        spec:
          forProvider:
            roleSelector:
              matchLabels:
                role: controlplane
  # Removed for brevity
```

##### コントローラー参照の一致 

コントローラー参照を一致させることで、一致するリソースが
同じ複合リソース内にあることが保証されます。

コントローラー参照のみを一致させることで、ラベルやその他の情報を必要とせずに
一致プロセスが簡素化されます。

例えば、AWSの
{{<hover label="controller1" line="14">}}InternetGateway{{</hover>}}を作成するには、
{{<hover label="controller1" line="7">}}VPC{{</hover>}}が必要です。

{{<hover label="controller1" line="14">}}InternetGateway{{</hover>}}はラベルと一致する可能性がありますが、このCompositionによって作成されたすべてのVPCは同じラベルを共有します。

{{<hover label="controller1" line="19">}}matchControllerRef{{</hover>}}を使用すると、{{<hover label="controller1" line="14">}}InternetGateway{{</hover>}}を作成した同じ複合リソース内で作成されたVPCのみが一致します。

```yaml {label="controller1",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: VPC
        name: my-vpc
        spec:
          forProvider:
          # Removed for brevity
    - base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: InternetGateway
        name: my-gateway
        spec:
          forProvider:
            vpcIdSelector:
              matchControllerRef: true
# Removed for brevity
```

リソースは、特定のリソースを大きな複合リソース内で一致させるために、ラベルとコントローラー参照の両方に一致させることができます。

例えば、このCompositionは2つの
{{<hover label="controller2" line="17">}}VPC{{</hover>}}
リソースを作成しますが、
{{<hover label="controller2" line="27">}}InternetGateway{{</hover>}} 
は1つのものにのみ一致する必要があります。

2番目の{{<hover label="controller2" line="17">}}VPC{{</hover>}}に
{{<hover label="controller2" line="21">}}label{{</hover>}}を適用することで、
{{<hover label="controller2" line="27">}}InternetGateway{{</hover>}}は
ラベル
{{<hover label="controller2" line="34">}}type: internet{{</hover>}}と一致し、
{{<hover label="controller2" line="32">}}matchControllerRef{{</hover>}}を使用して
同じ複合リソース内のオブジェクトのみと一致します。

```yaml {label="controller2",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: VPC
        name: my-first-vpc
        metadata:
          labels:
            type: backend
        spec:
          forProvider:
          # Removed for brevity
    - base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: VPC
        name: my-second-vpc
        metadata:
          labels:
            type: internet
        spec:
          forProvider:
          # Removed for brevity
    - base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: InternetGateway
        name: my-gateway
        spec:
          forProvider:
            vpcIdSelector:
              matchControllerRef: true
              matchLabels:
                type: internet
# Removed for brevity
```
### 複合リソースの有効化

Compositionは、管理リソースを作成する方法を定義するテンプレートに過ぎません。Compositionは、このテンプレートを使用できる複合リソースを制限します。

Compositionの{{<hover label="typeref" line="6">}}compositeTypeRef{{</hover>}}は、
どの複合リソースタイプがこのCompositionを使用できるかを定義します。

{{<hint "note" >}}
複合リソースについての詳細は、
[Composite Resources page]({{<ref "./composite-resources" >}})をお読みください。
{{< /hint >}}

Compositionの
{{<hover label="typeref" line="5">}}spec{{</hover>}}内では、
複合リソースの
{{<hover label="typeref" line="7">}}apiVersion{{</hover>}}と
{{<hover label="typeref" line="8">}}kind{{</hover>}}
を定義し、Compositionがこのテンプレートを使用することを許可します。

```yaml {label="typeref",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: dynamodb-with-bucket
spec:
  compositeTypeRef:
    apiVersion: custom-api.example.org/v1alpha1
    kind: database
  # Removed for brevity
```

### リソースフィールドの変更

ほとんどのコンポジションでは、リソースのフィールドをカスタマイズする必要があります。これには、ユニークなパスワードの適用、リソースのデプロイ先の変更、またはラベルやアノテーションの適用が含まれます。

リソースを変更する主な方法は、リソースの 
[パッチと変換]({{<ref "./patch-and-transform" >}})を使用することです。パッチと変換は、特定の入力フィールドを一致させ、それを変更して管理対象リソースに適用することを可能にします。

{{<hint "重要" >}}
パッチと変換の作成およびそのオプションの詳細は、 
[パッチと変換ページ]({{<ref "./patch-and-transform" >}})にあります。

このセクションでは、コンポジションにパッチと変換を適用する方法について説明します。
{{< /hint >}}

個々の `resources` にパッチを適用するには、 
{{<hover label="patch" line="13">}}patches{{</hover>}}
フィールドを使用します。

例えば、クレームで提供された 
{{<hover label="patchClaim" line="6">}}location{{</hover>}} を取得し、管理対象リソースの 
{{<hover label="patch" line="12">}}region{{</hover>}} 値に適用します。

```yaml {copy-lines="none",label="patchClaim"}
apiVersion: example.org/v1alpha1
kind: ExampleClaim
metadata:
  name: my-example-claim
spec:
  location: "eu-north-1"
```

コンポジションパッチは、 
{{<hover label="patch" line="15">}}fromFieldPath{{</hover>}} を使用して、クレーム内の 
{{<hover label="patchClaim" line="6">}}location{{</hover>}} フィールドと一致させ、 
{{<hover label="patch" line="16">}}toFieldPath{{</hover>}} を使用して、コンポジション内で変更するフィールドを定義します。

```yaml {copy-lines="none",label="patch"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
    - name: s3Bucket
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: "us-east-2"
      patches:
      - type: FromCompositeFieldPath
        fromFieldPath: "spec.location"
        toFieldPath: "spec.forProvider.region"
```

#### パッチセット

一部のコンポジションには、同一のパッチを適用する必要があるリソースがあります。同じ `patches` フィールドを繰り返す代わりに、リソースは単一の `patchSet` を参照できます。

{{<hover label="patchset" line="5">}}patchSet{{</hover>}} を 
{{<hover label="patchset" line="6">}}name{{</hover>}} と 
{{<hover label="patchset" line="7">}}patch{{</hover>}} 操作で定義します。

次に、 
{{<hover label="patchset" line="5">}}patchSet{{</hover>}} を各リソースに適用し、 
{{<hover label="patchset" line="16">}}type: patchSet{{< /hover >}} を使用して、 
{{<hover label="patchset" line="6">}}name{{< /hover >}} を 
{{<hover label="patchset" line="17">}}patchSetName{{< /hover >}} フィールドで参照します。

```yaml {copy-lines="none",label="patchset"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  patchSets:
    - name: reusable-patch
      patches:
      - type: FromCompositeFieldPath
        fromFieldPath: "location"
        toFieldPath: "spec.forProvider.region"
  resources:
    - name: first-resource
      base:
      # Removed for Brevity
      patches:
        - type: PatchSet
          patchSetName: reusable-patch
    - name: second-resource
      base:
      # Removed for Brevity
      patches:
        - type: PatchSet
          patchSetName: reusable-patch
```

#### 環境設定によるパッチ

Crossplaneは、環境設定を使用してメモリ内データストアを作成します。コンポジションは、このデータストアから読み書きすることがパッチプロセスの一部として可能です。

{{<hint "重要" >}}
環境設定はアルファ機能です。アルファ機能はデフォルトでは有効になっていません。
{{< /hint >}}

環境設定は、コンポジションが使用できるデータを事前に定義したり、コンポジットリソースが他のリソースが読み取るためにメモリ内環境にデータを書き込んだりすることができます。

<!-- vale off -->
{{< hint "注" >}}
<!-- vale on -->
環境設定の使用に関する詳細は、[環境設定]({{<ref "./environment-configs" >}})ページを参照してください。
{{< /hint >}}

環境設定を使用してパッチを適用するには、まず使用する環境設定を
{{<hover label="envselect" line="6">}}environment.environmentConfigs{{</hover>}}で定義します。

<!-- vale Google.Quotes = NO -->
<!-- vale gitlab.SentenceLength = NO -->
<!-- ignore false positive -->
使用する環境設定を特定するには、[参照]({{<ref "./managed-resources#matching-by-name-reference" >}})または[セレクタ]({{<ref "./managed-resources#matching-by-selector" >}})を使用します。
<!-- vale Google.Quotes = YES -->

```yaml {label="envselect",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  environment:
    environmentConfigs:
      - ref:
          name: example-environment
  resources:
  # Removed for Brevity
```

<!-- これらの2つのセクションは環境設定のドキュメントに重複しています -->

##### コンポジットリソースのパッチ
コンポジットリソースとメモリ内環境の間でパッチを適用するには、
{{< hover label="xrpatch" line="7">}}patches{{</hover>}}を
{{< hover label="xrpatch" line="5">}}environment{{</hover>}}内で使用します。

メモリ内環境からコンポジットリソースにデータをコピーするには、
{{< hover label="xrpatch" line="5">}}ToCompositeFieldPath{{</hover>}}を使用します。
コンポジットリソースからメモリ内環境にデータをコピーするには、
{{< hover label="xrpatch" line="5">}}FromCompositeFieldPath{{</hover>}}を使用します。

```yaml {label="xrpatch",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  environment:
  # Removed for Brevity
      patches:
      - type: ToCompositeFieldPath
        fromFieldPath: tags
        toFieldPath: metadata.labels[envTag]
      - type: FromCompositeFieldPath
        fromFieldPath: metadata.name
        toFieldPath: newEnvironmentKey
```

個々のリソースは、メモリ内環境に書き込まれたデータを使用できます。

##### 個々のリソースをパッチする
個々のリソースをパッチするには、リソースの 
{{<hover label="envpatch" line="16">}}patches{{</hover>}} 内で、 
{{<hover label="envpatch" line="17">}}ToEnvironmentFieldPath{{</hover>}} を使用して
リソースからメモリ内環境にデータをコピーします。  
{{<hover label="envpatch" line="20">}}FromEnvironmentFieldPath{{</hover>}} を使用して
メモリ内環境からリソースにデータをコピーします。

```yaml {label="envpatch",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  environment:
  # Removed for Brevity
  resources:
  # Removed for Brevity
    - name: vpc
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: VPC
        spec:
          forProvider:
            cidrBlock: 172.16.0.0/16
      patches:
        - type: ToEnvironmentFieldPath
          fromFieldPath: status.atProvider.id
          toFieldPath: vpcId
        - type: FromEnvironmentFieldPath
          fromFieldPath: tags
          toFieldPath: spec.forProvider.tags
```

[EnvironmentConfigs]({{<ref "./environment-configs" >}}) ページには 
EnvironmentConfigs のオプションと使用法に関する詳細情報があります。

### コンポジション関数を使用する

コンポジション関数（略して関数）は、Crossplane リソースをテンプレート化するカスタムプログラムです。 
Go や Python のような汎用プログラミング言語を使用してリソースをテンプレート化する関数を書くことができます。 
汎用プログラミング言語を使用することで、関数はループや条件文のようなより高度なロジックを使用してリソースをテンプレート化できます。

{{<hint "important" >}}
コンポジション関数はベータ機能です。Crossplane はデフォルトでベータ関数を有効にします。 
[Composition Functions]({{<ref "./composition-functions#disable-composition-functions">}})
ページでは、コンポジション関数を無効にする方法を説明しています。
{{< /hint >}}

コンポジション関数を使用するには、Composition 
{{<hover label="xfn" line="6">}}mode{{</hover>}} を 
{{<hover label="xfn" line="6">}}Pipeline{{</hover>}} に設定します。

{{<hover label="xfn" line="7">}}pipeline{{</hover>}} を 
{{<hover label="xfn" line="8">}}steps{{</hover>}} の定義します。各 
{{<hover label="xfn" line="8">}}step{{</hover>}} は関数を呼び出します。  

各 {{<hover label="xfn" line="8">}}step{{</hover>}} は 
{{<hover label="xfn" line="9">}}functionRef{{</hover>}} を使用して
呼び出す関数の {{<hover label="xfn" line="10">}}name{{</hover>}} を参照します。 

一部の関数では、{{<hover label="xfn" line="11">}}input{{</hover>}} を指定することもできます。  
関数は {{<hover label="xfn" line="13">}}kind{{</hover>}} の入力を定義します。


{{<hint "important" >}}
{{<hover label="xfn" line="6">}}モード: パイプライン{{</hover>}}を使用するコンポジションは、`resources`フィールドでリソーステンプレートを指定できません。

リソーステンプレートを作成するには、「パッチと変換」関数を使用してください。
{{< /hint >}}

この例では、関数パッチと変換を使用しています。関数パッチと変換は
Crossplaneリソーステンプレートを実装する関数です。関数パッチと変換を使用して、他の
関数とともにパイプライン内でリソーステンプレートを指定できます。

```yaml {label="xfn",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  # Removed for Brevity
  mode: Pipeline
  pipeline:
  - step: patch-and-transform
    functionRef:
      name: function-patch-and-transform
    input:
      apiVersion: pt.fn.crossplane.io/v1beta1
      kind: Resources
      resources:
      - name: storage-bucket
        base:
          apiVersion: s3.aws.upbound.io/v1beta1
          kind: Bucket
          spec:
            forProvider:
              region: "us-east-2"
```

コンポジション関数の構築と使用に関する詳細は、[composition functions]({{<ref "./composition-functions">}})ページを参照してください。

### 接続詳細の保存

一部の管理リソースは、ユーザー名、パスワード、IPアドレス、ポート、またはその他の接続詳細のようなユニークな詳細を生成します。

コンポジション内のリソースが接続詳細を作成すると、Crossplaneは接続詳細を生成する各管理リソースのためにKubernetesシークレットオブジェクトを作成します。

{{<hint "note">}}
このセクションでは、Kubernetesシークレットの作成について説明します。  
Crossplaneは、[HashiCorp Vault](https://www.vaultproject.io/)のような外部シークレットストアの使用もサポートしています。

Crossplaneを外部シークレットストアと一緒に使用するための詳細については、[external secrets store guide]({{<ref "../guides/vault-as-secret-store">}})を参照してください。
{{</hint >}}

#### 複合リソースの結合シークレット
Crossplaneは、コンポジション内のリソースによって生成されたすべてのシークレットを単一のKubernetesシークレットに結合し、オプションで[Claims]({{<ref "./claims#claim-connection-secrets">}})のためにシークレットオブジェクトをコピーできます。

{{<hover label="writeConn" line="5">}}writeConnectionSecretsToNamespace{{</hover>}}の値を、Crossplaneが結合シークレットオブジェクトを保存すべき名前空間に設定します。

```yaml {copy-lines="none",label="writeConn"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  writeConnectionSecretsToNamespace: my-namespace
  resources:
  # Removed for brevity
```

#### 構成リソースのシークレット
接続詳細を生成する各リソースの{{<hover label="writeConnRes" line="10">}}spec{{</hover>}}内で、リソースのシークレットオブジェクトの{{<hover label="writeConnRes" line="13">}}writeConnectionSecretToRef{{</hover>}}を定義し、{{<hover label="writeConnRes" line="14">}}namespace{{</hover>}}と{{<hover label="writeConnRes" line="15">}}name{{</hover>}}を指定します。

もし
{{<hover label="writeConnRes" line="13">}}writeConnectionSecretToRef{{</hover>}}
が定義されていない場合、Crossplaneはシークレットにキーを書き込みません。

```yaml {label="writeConnRes"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  writeConnectionSecretsToNamespace: other-namespace
  resources:
    - name: key
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
          forProvider:
          # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
```

Crossplaneは、提供された
{{<hover label="viewComposedSec" line="3">}}name{{</hover>}}
でシークレットを保存します。

```shell {label="viewComposedSec"}
kubectl get secrets -n docs
NAME   TYPE                                DATA   AGE
key1   connection.crossplane.io/v1alpha1   4      4m30s
```

{{<hint "tip" >}}

Crossplaneは、各シークレットのユニークな名前を作成するために[Patch]({{<ref "./patch-and-transform">}})を使用することを推奨します。

例えば、リソースのユニーク識別子をキー名として追加する
{{<hover label="tipPatch" line="15">}}patch{{</hover>}}です。

```yaml {label="tipPatch",copy-lines="14-20"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  # Removed for brevity
  resources:
    - name: key
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
        # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
      patches:
        - fromFieldPath: "metadata.uid"
          toFieldPath: "spec.writeConnectionSecretToRef.name"
          transforms:
            - type: string
              string:
                fmt: "%s-secret"
```
{{< /hint >}}

#### シークレットキーの定義

Compositionは、リソースが
{{<hover label="conDeet" line="14">}}connectionDetails{{</hover>}}オブジェクトを使用して作成する特定のシークレットキーを定義する必要があります。

{{<table "table table-sm" >}}
| シークレットタイプ | 説明 | 
| --- | --- | 
| {{<hover label="conDeet" line="16">}}fromConnectionSecretKey{{</hover>}} | リソースによって生成されたシークレットのキーに一致するシークレットキーを作成します。 | 
| {{<hover label="conDeet" line="18">}}fromFieldPath{{</hover>}}  | リソースのフィールドパスに一致するシークレットキーを作成します。 |
| {{<hover label="conDeet" line="20">}}value{{</hover>}}  | 事前定義された値を持つシークレットキーを作成します。 |
{{< /table >}}

{{<hint "note">}}
{{<hover label="conDeet" line="20">}}value{{</hover>}}タイプは
文字列値を使用する必要があります。

{{<hover label="conDeet" line="20">}}value{{</hover>}}は、個々のリソースシークレットオブジェクトに追加されません。{{<hover label="conDeet" line="20">}}value{{</hover>}}は、結合されたコンポジットリソースシークレットにのみ表示されます。
{{< /hint >}}

```yaml {label="conDeet",copy-lines="none"}
kind: Composition
spec:
  writeConnectionSecretsToNamespace: other-namespace
  resources:
    - name: key
      base:
        # Removed for brevity
        spec:
          forProvider:
          # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
      connectionDetails:
        - name: myUsername
          fromConnectionSecretKey: username
        - name: myFieldSecret
          fromFieldPath: spec.forProvider.user
        - name: myStaticSecret
          value: "docs.crossplane.io"
```

リソース内の
{{<hover label="conDeet" line="14">}}connectionDetails{{</hover>}}は、
{{<hover label="conDeet" line="16">}}fromConnectionSecretKey{{</hover>}}を使用してリソースからシークレットを参照したり、  
{{<hover label="conDeet" line="18">}}fromFieldPath{{</hover>}}を使用してリソース内の別のフィールドから参照したり、  
{{<hover label="conDeet" line="20">}}value{{</hover>}}を使用して静的に定義された値を参照したりできます。


Crossplaneは、{{<hover label="conDeet" line="15">}}name{{</hover>}}の値に秘密鍵を設定します。 

秘密を記述して、秘密オブジェクト内の秘密鍵を表示します。

{{<hint "tip" >}}
同じ秘密鍵名で複数のリソースが秘密を生成する場合、
Crossplaneは1つの値のみを保存します。 

カスタム{{<hover label="conDeet" line="15">}}name{{</hover>}}を使用して
ユニークな秘密鍵を作成します。
{{< /hint >}}

{{<hint "important">}}
Crossplaneは、{{<hover label="conDeet" line="16">}}connectionDetails{{</hover>}}に
リストされている接続詳細のみを
結合された秘密オブジェクトに追加します。  
{{<hover label="conDeet" line="16">}}connectionDetails{{</hover>}}に定義されていない
管理リソース内の接続秘密は、結合された秘密オブジェクトに追加されません。   
{{< /hint >}}


```shell {copy-lines="1"}
kubectl describe secret
Name:         my-access-key-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  connection.crossplane.io/v1alpha1

Data
====
myUsername:      20 bytes
myFieldSecret:   24 bytes
myStaticSecret:  18 bytes
```

{{<hint "note" >}}
CompositeResourceDefinitionは、CompositeリソースからCrossplaneが保存する
鍵を制限することもできます。  
デフォルトでは、XRDは構成されたリソースの
`connectionDetails`にリストされているすべての秘密鍵を
結合された秘密オブジェクトに書き込みます。

秘密鍵の制限に関する詳細は、
[CompositeResourceDefinitionのドキュメント]({{<ref "composite-resource-definitions#manage-connection-secrets">}})を
お読みください。
{{< /hint >}}

接続秘密に関する詳細は、
[Connection Secretsの知識ベース記事]({{<ref "connection-details">}})をお読みください。

{{<hint "warning">}}
Compositionの{{<hover label="conDeet" line="16">}}connectionDetails{{</hover>}}を変更することはできません。  
{{<hover label="conDeet" line="16">}}connectionDetails{{</hover>}}を変更するには、
Compositionを削除して再作成する必要があります。
{{</hint >}}


#### 接続詳細を外部秘密ストアに保存する

Crossplaneは
[External Secret Stores]({{<ref "../guides/vault-as-secret-store" >}})を使用して、
HashiCorp Vaultのような外部秘密ストアに秘密と接続詳細を書き込みます。 

{{<hint "important" >}}
External Secret Storesはアルファ機能です。

本番環境での使用は推奨されていません。CrossplaneはデフォルトでExternal Secret
Storesを無効にしています。
{{< /hint >}}

使用する 
{{<hover label="gcp-storeconfig"
line="11">}}publishConnectionDetailsWithStoreConfigRef{{</hover>}}
の代わりに 
`writeConnectionSecretsToNamespace` を使用して、 
{{<hover label="gcp-storeconfig" line="2">}}StoreConfig{{</hover>}} 
に接続詳細を保存します。 

例えば、 
{{<hover label="gcp-storeconfig" line="2">}}StoreConfig{{</hover>}} を 
{{<hover label="gcp-storeconfig" line="4">}}name{{</hover>}} "vault" と共に使用し、 
{{<hover label="gcp-storeconfig" line="12">}}publishConnectionDetailsWithStoreConfigRef.name{{</hover>}}
が {{<hover label="gcp-storeconfig" line="4">}}StoreConfig.name{{</hover>}} 
に一致するようにします。この例では "vault" です。


```yaml {label="gcp-storeconfig",copy-lines="none"}
apiVersion: gcp.crossplane.io/v1alpha1
kind: StoreConfig
metadata:
  name: vault
# Removed for brevity.
---
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  publishConnectionDetailsWithStoreConfigRef: 
    name: vault
  resources:
  # Removed for brevity
```

詳細については、[External Secret Stores]({{<ref "../guides/vault-as-secret-store" >}}) 
統合ガイドをお読みください。

### リソースの準備チェック

デフォルトでは、Crossplane は Composite Resource または Claim を `READY` と見なします。
これは、作成されたすべてのリソースのステータスが `Type: Ready` および `Status: True` の場合です。

例えば、ProviderConfig のような一部のリソースは Kubernetes ステータスを持たず、
決して `Ready` と見なされません。

カスタム準備チェックを使用すると、Composition がリソースが `Ready` であるために満たすべきカスタム条件を定義できます。

{{< hint "tip" >}}
リソースが `Ready` であるために複数の条件を満たす必要がある場合は、複数の準備チェックを使用してください。
{{< /hint >}}

<!-- vale Google.WordList = NO -->
リソースの 
{{<hover label="check" line="10" >}}readinessChecks{{</hover>}} フィールドを使用して、カスタム準備チェックを定義します。
<!-- vale Google.WordList = YES -->

チェックには、リソースを一致させる方法を定義する 
{{<hover label="check" line="11" >}}type{{</hover>}} と、リソース内のどのフィールドを比較するかを示す 
{{<hover label="check" line="12" >}}fieldPath{{</hover>}} があります。

```yaml {label="check",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: <match type>
          fieldPath: <resource field>
```

Composition はリソースフィールドを一致させることをサポートしています：
 * [文字列一致](#match-a-string)
 * [整数一致](#match-an-integer)
 * [非空一致](#match-that-a-field-exists)
 * [常に準備完了](#always-consider-a-resource-ready)
 * [条件一致](#match-a-condition)
 * [ブール値一致](#match-a-boolean)

#### 文字列の一致

{{<hover label="matchstring" line="11">}}MatchString{{</hover>}} は、リソース内のフィールドの値が指定された文字列と一致する場合に、構成されたリソースが準備完了と見なします。

{{<hint "note" >}}
<!-- vale Google.WordList = NO -->
Crossplane は完全一致の文字列のみをサポートしています。部分文字列や正規表現は準備チェックではサポートされていません。
<!-- vale Google.WordList = YES -->
{{</hint >}}

例えば、リソースの 
{{<hover label="matchstring" line="12">}}status.atProvider.state{{</hover>}}
フィールドで文字列 
{{<hover label="matchstring" line="13">}}Online{{</hover>}}
と一致させることができます。 

```yaml {label="matchstring",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: MatchString
          fieldPath: status.atProvider.state
          matchString: "Online"
```

#### 整数の一致

{{<hover label="matchint" line="11">}}MatchInteger{{</hover>}} は、リソース内のフィールドの値が指定された整数と一致する場合に、構成されたリソースが準備完了と見なします。

{{<hint "note" >}}
<!-- vale Google.WordList = NO -->
Crossplane は `0` の一致をサポートしていません。 
<!-- vale Google.WordList = YES -->
{{</hint >}}

例えば、リソースの 
{{<hover label="matchint" line="12">}}status.atProvider.state{{</hover>}}
フィールドで数値 
{{<hover label="matchint" line="13">}}4{{</hover>}}
と一致させることができます。 

```yaml {label="matchint",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: MatchInteger
          fieldPath: status.atProvider.state
          matchInteger: 4
```

#### フィールドが存在することの一致
{{<hover label="NonEmpty" line="11">}}NonEmpty{{</hover>}} は、値を持つフィールドが存在する場合に、構成されたリソースが準備完了と見なします。 

{{<hint "note" >}}
<!-- vale Google.WordList = NO -->
Crossplane は `0` の値や空文字列を空として扱います。
{{</hint >}}

例えば、リソースの 
{{<hover label="NonEmpty" line="12">}}status.atProvider.state{{</hover>}}
フィールドが空でないことを確認します。 
<!-- vale Google.WordList = YES -->

```yaml {label="NonEmpty",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: NonEmpty
          fieldPath: status.atProvider.state
```

{{<hint "tip" >}}
{{<hover label="NonEmpty" line="11">}}NonEmpty{{</hover>}} をチェックするには、他のフィールドを設定する必要はありません。
{{< /hint >}} 

#### リソースを常に準備完了と見なす
{{<hover label="none" line="11">}}None{{</hover>}} は、リソースが作成されるとすぐに構成されたリソースを準備完了と見なします。Crossplane はリソースを準備完了と宣言する前に、他の条件を待ちません。

例えば、次のように考えてみてください 
{{<hover label="none" line="7">}}my-resource{{</hover>}}
作成されるとすぐに準備完了です。


```yaml {label="none",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: None
```

#### 条件に一致させる
{{<hover label="condition" line="11">}}Condition{{</hover>}} は、期待される条件タイプが見つかり、その `status.conditions` に期待されるステータスがあるときに、構成されたリソースが準備完了と見なします。

例えば、次のように考えてみてください 
{{<hover label="condition" line="7">}}my-resource{{</hover>}} は、タイプ 
{{<hover label="condition" line="13">}}MyType{{</hover>}} の条件があり、そのステータスが 
{{<hover label="condition" line="14">}}Success{{</hover>}} の場合に準備完了です。

```yaml {label="condition",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: MatchCondition
          matchCondition:
            type: MyType
            status: Success
```

#### ブール値に一致させる

ブールフィールドに一致させるための2種類のチェックがあります：
 * `MatchTrue`
 * `MatchFalse`

`MatchTrue` は、そのリソース内のフィールドの値が `true` のときに、構成されたリソースが準備完了と見なします。

`MatchFalse` は、そのリソース内のフィールドの値が `false` のときに、構成されたリソースが準備完了と見なします。

例えば、次のように考えてみてください 
{{<hover label="matchTrue" line="7">}}my-resource{{</hover>}} は、 
{{<hover label="matchTrue" line="12">}} status.atProvider.manifest.status.ready{{</hover>}}
が {{<hover label="matchTrue" line="11">}}true{{</hover>}} の場合に準備完了です。

```yaml {label="matchTrue",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: MatchTrue
          fieldPath: status.atProvider.manifest.status.ready
```
{{<hint "tip" >}}
{{<hover label="matchTrue" line="11">}}MatchTrue{{</hover>}} をチェックするには、他のフィールドを設定する必要はありません。
{{< /hint >}} 

`MatchFalse` は、値が `false` であることを示すフィールドに一致します。

例えば、次のように考えてみてください 
{{<hover label="matchFalse" line="7">}}my-resource{{</hover>}} は、 
{{<hover label="matchFalse" line="12">}} status.atProvider.manifest.status.pending{{</hover>}}
が {{<hover label="matchFalse" line="11">}}false{{</hover>}} の場合に準備完了です。

```yaml {label="matchFalse",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for Brevity
spec:
  resources:
  # Removed for Brevity
    - name: my-resource
      base:
        # Removed for brevity
      readinessChecks:
        - type: MatchFalse
          fieldPath: status.atProvider.manifest.status.pending
```

{{<hint "tip" >}}
{{<hover label="matchFalse" line="11">}}MatchFalse{{</hover>}} をチェックするには、他のフィールドを設定する必要はありません。
{{< /hint >}}

## コンポジションの検証

`kubectl get composition`を使用して、利用可能なすべてのコンポジションを表示します。

```shell {copy-lines="1"}
kubectl get composition
NAME                                       XR-KIND        XR-APIVERSION                         AGE
xapps.aws.platformref.upbound.io           XApp           aws.platformref.upbound.io/v1alpha1   123m
xclusters.aws.platformref.upbound.io       XCluster       aws.platformref.upbound.io/v1alpha1   123m
xeks.aws.platformref.upbound.io            XEKS           aws.platformref.upbound.io/v1alpha1   123m
xnetworks.aws.platformref.upbound.io       XNetwork       aws.platformref.upbound.io/v1alpha1   123m
xservices.aws.platformref.upbound.io       XServices      aws.platformref.upbound.io/v1alpha1   123m
xsqlinstances.aws.platformref.upbound.io   XSQLInstance   aws.platformref.upbound.io/v1alpha1   123m
```

`XR-KIND`は、コンポジションテンプレートを使用することが許可されている複合リソースの`kind`をリストします。  
`XR-APIVERSION`は、コンポジションテンプレートを使用することが許可されている複合リソースのAPIバージョンをリストします。

{{<hint "note" >}}
`kubectl get composition`の出力は`kubectl get composite`とは異なります。

`kubectl get composition`は、利用可能なすべてのコンポジションをリストします。

`kubectl get composite`は、作成されたすべての複合リソースとそれに関連するコンポジションをリストします。 
{{< /hint >}}

## コンポジションの検証

コンポジションを作成する際、Crossplaneはその整合性を自動的に検証し、コンポジションが正しく形成されているかを確認します。例えば：

`mode: Resources`を使用する場合：

* `resources`フィールドが空でないこと。
* すべてのリソースが`name`を使用するか、使用しないかのいずれかであること。コンポジションは、名前付きリソースと名前なしリソースの両方を使用できません。
* 重複するリソース名がないこと。
* パッチセットには名前が必要です。
* `fromFieldPath`値を必要とするパッチは、それを提供すること。
* `toFieldPath`値を必要とするパッチは、それを提供すること。
* `combine`フィールドを必要とするパッチは、それを提供すること。
* `matchString`を使用したレディネスチェックが空でないこと。
* `matchInteger`を使用したレディネスチェックが`0`でないこと。
* `fieldPath`値を必要とするレディネスチェックは、それを提供すること。

`mode: Pipeline`（コンポジション関数）を使用する場合：

* `pipeline`フィールドが空でないこと。
* 重複するステップ名がないこと。

### コンポジションスキーマに基づく検証

Crossplaneは、コンポジションのスキーマに基づく検証も行います。スキーマ検証は、`patches`、`readinessChecks`、および`connectionDetails`がリソーススキーマに従って有効であることを確認します。例えば、パッチのソースおよび宛先フィールドがソースおよび宛先リソーススキーマに従って有効であることを確認します。

{{<hint "note" >}}
コンポジションスキーマに基づく検証はベータ機能です。Crossplaneはデフォルトでベータ機能を有効にします。

スキーマに基づく検証を無効にするには、Crossplaneポッドで`--enable-composition-webhook-schema-validation=false`フラグを設定します。


[Crossplane Pods]({{<ref "./pods#edit-the-deployment">}}) ページには、Crossplane フラグを有効にするための詳細情報があります。
{{< /hint >}}

#### スキーマ対応の検証モード

Crossplane は、整合性エラーが発生した場合、常に Composition を拒否します。

スキーマ対応の検証モードを設定して、Crossplane がリソーススキーマの欠如やスキーマ対応の検証エラーをどのように処理するかを構成します。

{{<hint "note" >}}
リソーススキーマが欠如している場合、Crossplane はスキーマ対応の検証をスキップしますが、整合性エラーに対してはエラーを返し、欠如しているスキーマに対しては警告またはエラーを返します。
{{< /hint >}}

以下のモードが利用可能です：

{{< table "table table-sm table-striped" >}}
| モード     | 欠如スキーマ | スキーマ対応エラー | 整合性エラー |
| -------- | -------------- |--------------------|-----------------|
| `warn`   | 警告        | 警告            | エラー           |
| `loose`  | 警告        | エラー              | エラー           |
| `strict` | エラー          | エラー              | エラー           |
{{< /table >}}

Composition の検証モードを変更するには、 
{{<hover label="mode" line="5">}}crossplane.io/composition-schema-aware-validation-mode{{</hover>}} 
アノテーションを使用します。

指定しない場合、デフォルトモードは `warn` です。

たとえば、`loose` モードのチェックを有効にするには、アノテーションの値を 
{{<hover label="mode" line="5">}}loose{{</hover>}} に設定します。

```yaml {copy-lines="none",label="mode"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  annotations:
    crossplane.io/composition-schema-aware-validation-mode: loose
  # Removed for brevity
spec:
  # Removed for brevity
```

{{<hint "important" >}}
検証モードは、Configuration パッケージによって定義された Composition にも適用されます。

Composition に設定されたモードに応じて、スキーマ対応の検証問題は警告や Composition の拒否を引き起こす可能性があります。

検証警告については Crossplane のログを確認してください。

Crossplane は、検証エラーがある場合、Configuration を不健康として設定します。
特定のエラーを確認するには、`kubectl describe configuration` で Configuration の詳細を表示します。
{{< /hint >}}
