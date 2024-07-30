---
title: AWS クイックスタート パート 2
weight: 120
tocHidden: true
aliases:
  - /master/getting-started/provider-aws-part-3
---

{{< hint "重要" >}}
このガイドはシリーズのパート 2 です。  

[**パート 1**]({{<ref "provider-aws" >}}) では
Crossplane のインストールと Kubernetes クラスターを AWS に接続する方法について説明しています。

{{< /hint >}}

このガイドでは、Crossplane を使用してカスタム API を構築し、アクセスする方法を説明します。

## 前提条件
* [クイックスタート パート 1]({{<ref "provider-aws" >}}) を完了し、Kubernetes を
  AWS に接続します。
* AWS S3 ストレージバケットと DynamoDB インスタンスを作成する権限を持つ
  AWS アカウント

{{<expand "パート 1 をスキップしてすぐに始める" >}}
1. Crossplane Helm リポジトリを追加し、Crossplane をインストールします
```shell
helm repo add \
crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace
```

2. Crossplane ポッドのインストールが完了し、準備が整ったら、AWS プロバイダーを適用します
   
```yaml {label="provider",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1.1.0
EOF
```

3. AWS キーを含むファイルを作成します
```ini
[default]
aws_access_key_id = <aws_access_key>
aws_secret_access_key = <aws_secret_key>
```

4. AWS キーから Kubernetes シークレットを作成します
```shell {label="kube-create-secret",copy-lines="all"}
kubectl create secret \
generic aws-secret \
-n crossplane-system \
--from-file=creds=./aws-credentials.txt
```

5. _ProviderConfig_ を作成します
```yaml {label="providerconfig",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
EOF
```
{{</expand >}}

## DynamoDB プロバイダーをインストールする

パート 1 では AWS S3 プロバイダーのみがインストールされました。このセクションでは、DynamoDB テーブルとともに S3 バケットをデプロイします。  
DynamoDB テーブルをデプロイするには、DynamoDB プロバイダーも必要です。

新しいプロバイダーをクラスターに追加します。

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-dynamodb
spec:
  package: xpkg.upbound.io/upbound/provider-aws-dynamodb:v1.1.0
EOF
```

`kubectl get providers` を使用して新しい DynamoDB プロバイダーを表示します。

```shell {copy-lines="1"}
kubectl get providers
NAME                          INSTALLED   HEALTHY   PACKAGE                                                 AGE
provider-aws-dynamodb         True        True      xpkg.upbound.io/upbound/provider-aws-dynamodb:v1.1.0     3m55s
provider-aws-s3               True        True      xpkg.upbound.io/upbound/provider-aws-s3:v1.1.0           13m
upbound-provider-family-aws   True        True      xpkg.upbound.io/upbound/provider-family-aws:v1.1.0       13m
```

## カスタム API を作成する

<!-- vale alex.Condescending = NO -->
Crossplane を使用すると、ユーザーのために独自のカスタム API を構築でき、クラウドプロバイダーやそのリソースに関する詳細を抽象化できます。API を複雑にしたりシンプルにしたりすることができます。 
<!-- vale alex.Condescending = YES -->

カスタム API は Kubernetes オブジェクトです。  
以下はカスタム API の例です。

```yaml {label="exAPI"}
apiVersion: database.example.com/v1alpha1
kind: NoSQL
metadata:
  name: my-nosql-database
spec: 
  location: "US"
```

Kubernetesオブジェクトと同様に、APIには 
{{<hover label="exAPI" line="1">}}version{{</hover>}}、 
{{<hover label="exAPI" line="2">}}kind{{</hover>}}、および 
{{<hover label="exAPI" line="5">}}spec{{</hover>}}があります。

### グループとバージョンの定義
独自のAPIを作成するには、まず 
[APIグループ](https://kubernetes.io/docs/reference/using-api/#api-groups) と 
[バージョン](https://kubernetes.io/docs/reference/using-api/#api-versioning) を定義します。  

_グループ_ は任意の値を使用できますが、一般的な慣習として完全修飾ドメイン名にマッピングされます。 

<!-- vale gitlab.SentenceLength = NO -->
バージョンはAPIの成熟度や安定性を示し、API内のフィールドを変更、追加、または削除する際にインクリメントされます。
<!-- vale gitlab.SentenceLength = YES -->

Crossplaneは特定のバージョンや特定のバージョン命名規則を必要としませんが、 
[Kubernetes APIバージョン管理ガイドライン](https://kubernetes.io/docs/reference/using-api/#api-versioning) に従うことを強く推奨します。 

* `v1alpha1` - いつでも変更される可能性のある新しいAPI。
* `v1beta1` - 安定と見なされる既存のAPI。破壊的変更は強く推奨されません。
* `v1` - 破壊的変更のない安定したAPI。 

このガイドでは、グループ 
{{<hover label="version" line="1">}}database.example.com{{</hover>}} を使用します。

これはAPIの最初のバージョンであるため、このガイドではバージョン 
{{<hover label="version" line="1">}}v1alpha1{{</hover>}} を使用します。

```yaml {label="version",copy-lines="none"}
apiVersion: database.example.com/v1alpha1
```

### kindの定義

APIグループは、関連するAPIの論理的なコレクションです。グループ内には、異なるリソースを表す個々のkindがあります。

たとえば、`database`グループには`Relational`および`NoSQL`のkindがあるかもしれません。

`kind`は何でも可能ですが、 
[UpperCamelCased](https://kubernetes.io/docs/contribute/style/style-guide/#use-upper-camel-case-for-api-objects) である必要があります。

このAPIのkindは 
{{<hover label="kind" line="2">}}NoSQL{{</hover>}} です。

```yaml {label="kind",copy-lines="none"}
apiVersion: database.example.com/v1alpha1
kind: NoSQL
```

### スペックを定義する

APIの最も重要な部分はスキーマです。スキーマはユーザーから受け入れられる入力を定義します。

このAPIでは、ユーザーがクラウドリソースを実行する場所の 
{{<hover label="spec" line="4">}}location{{</hover>}} を提供できます。

他のすべてのリソース設定はユーザーによって構成できません。これにより、Crossplaneはユーザーエラーを心配することなく、ポリシーや基準を強制できます。

```yaml {label="spec",copy-lines="none"}
apiVersion: database.example.com/v1alpha1
kind: NoSQL
spec: 
  location: "US"
```

### APIを適用する

Crossplaneは 
{{<hover label="xrd" line="3">}}Composite Resource Definitions{{</hover>}} 
（`XRD`とも呼ばれます）を使用して、KubernetesにカスタムAPIをインストールします。

XRDの {{<hover label="xrd" line="6">}}spec{{</hover>}} には、APIに関するすべての情報が含まれています。これには 
{{<hover label="xrd" line="7">}}group{{</hover>}},
{{<hover label="xrd" line="12">}}version{{</hover>}},
{{<hover label="xrd" line="9">}}kind{{</hover>}} および 
{{<hover label="xrd" line="13">}}schema{{</hover>}} が含まれます。

XRDの {{<hover label="xrd" line="5">}}name{{</hover>}} は、{{<hover label="xrd" line="9">}}plural{{</hover>}} と 
{{<hover label="xrd" line="7">}}group{{</hover>}} の組み合わせでなければなりません。

{{<hover label="xrd" line="13">}}schema{{</hover>}} は、API {{<hover label="xrd" line="17">}}spec{{</hover>}} を定義するために 
{{<hover label="xrd" line="14">}}OpenAPIv3{{</hover>}} 仕様を使用します。

APIは、{{<hover label="xrd" line="20">}}location{{</hover>}} を定義し、これは 
{{<hover label="xrd" line="22">}}oneOf{{</hover>}} である必要があります。すなわち、 
{{<hover label="xrd" line="23">}}EU{{</hover>}} または 
{{<hover label="xrd" line="24">}}US{{</hover>}} のいずれかです。

このXRDを適用して、KubernetesクラスターにカスタムAPIを作成します。

```yaml {label="xrd",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: nosqls.database.example.com
spec:
  group: database.example.com
  names:
    kind: NoSQL
    plural: nosqls
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
    kind: NoSQLClaim
    plural: nosqlclaim
EOF
```

{{<hover label="xrd" line="29">}}claimNames{{</hover>}} を追加することで、ユーザーはこのAPIにアクセスできます。クラスター レベルでは 
{{<hover label="xrd" line="9">}}nosql{{</hover>}} エンドポイントを使用し、名前空間では 
{{<hover label="xrd" line="29">}}nosqlclaim{{</hover>}} エンドポイントを使用します。


名前空間スコープのAPIは、Crossplaneの_Claim_です。

{{<hint "tip" >}}
Composite Resource Definitionsのフィールドとオプションの詳細については、
[XRDドキュメント]({{<ref "../concepts/composite-resource-definitions">}})をお読みください。 
{{< /hint >}}

インストールされたXRDを`kubectl get xrd`で表示します。  

```shell {copy-lines="1"}
kubectl get xrd
NAME                          ESTABLISHED   OFFERED   AGE
nosqls.database.example.com   True          True      2s
```

新しいカスタムAPIエンドポイントを`kubectl api-resources | grep nosql`で表示します。

```shell {copy-lines="1",label="apiRes"}
kubectl api-resources | grep nosql
nosqlclaim                                     database.example.com/v1alpha1          true         NoSQLClaim
nosqls                                         database.example.com/v1alpha1          false        NoSQL
```

## デプロイメントテンプレートの作成

ユーザーがカスタムAPIにアクセスすると、Crossplaneはその入力を取得し、
デプロイするインフラストラクチャを説明するテンプレートと組み合わせます。Crossplaneはこの
テンプレートを_Composition_と呼びます。

{{<hover label="comp" line="3">}}Composition{{</hover>}}は、デプロイするすべての
クラウドリソースを定義します。
テンプレート内の各エントリは、リソース設定やメタデータ
（ラベルやアノテーションなど）を定義する完全なリソース定義です。

このテンプレートは、AWSの 
{{<hover label="comp" line="13">}}S3{{</hover>}}
{{<hover label="comp" line="14">}}Bucket{{</hover>}}と 
{{<hover label="comp" line="33">}}DynamoDB{{</hover>}}
{{<hover label="comp" line="34">}}Table{{</hover>}}を作成します。

Crossplaneは、{{<hover label="comp" line="19">}}patches{{</hover>}}を使用して
ユーザーの入力をリソーステンプレートに適用します。  
このCompositionは、ユーザーの
{{<hover label="comp" line="21">}}location{{</hover>}}入力を取得し、それを
{{<hover label="comp" line="16">}}region{{</hover>}}として使用します。
個々のリソースで使用されます。

このCompositionをクラスターに適用します。 

```yaml {label="comp",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: dynamo-with-bucket
spec:
  resources:
    - name: s3Bucket
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        metadata:
          name: crossplane-quickstart-bucket
        spec:
          forProvider:
            region: us-east-2
          providerConfigRef:
            name: default
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.region"
          transforms:
            - type: map
              map: 
                EU: "eu-north-1"
                US: "us-east-2"
    - name: dynamoDB
      base:
        apiVersion: dynamodb.aws.upbound.io/v1beta1
        kind: Table
        metadata:
          name: crossplane-quickstart-database
        spec:
          forProvider:
            region: "us-east-2"
            writeCapacity: 1
            readCapacity: 1
            attribute:
              - name: S3ID
                type: S
            hashKey: S3ID
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.region"
          transforms:
            - type: map
              map: 
                EU: "eu-north-1"
                US: "us-east-2"
  compositeTypeRef:
    apiVersion: database.example.com/v1alpha1
    kind: NoSQL
EOF
```

{{<hover label="comp" line="52">}}compositeTypeRef{{</hover >}}は、
このテンプレートを使用してリソースを作成できるカスタムAPIを定義します。

{{<hint "tip" >}}
[Composition ドキュメント]({{<ref "../concepts/compositions">}})を読んで、 
Composition の構成や利用可能なオプションについての詳細を確認してください。

[Patch and Transform ドキュメント]({{<ref "../concepts/patch-and-transform">}})を読んで、 
Crossplane がパッチを使用してユーザー入力を Composition リソーステンプレートにマッピングする方法についての詳細を確認してください。
{{< /hint >}}

`kubectl get composition` で Composition を表示します。

```shell {copy-lines="1"}
kubectl get composition
NAME                 XR-KIND   XR-APIVERSION                   AGE
dynamo-with-bucket   NoSQL     database.example.com/v1alpha1   3s
```

## カスタム API へのアクセス

カスタム API (XRD) がインストールされ、リソーステンプレート (Composition) に関連付けられると、ユーザーはリソースを作成するために API にアクセスできます。

{{<hover label="xr" line="2">}}NoSQL{{</hover>}} オブジェクトを作成して、 
クラウドリソースを作成します。

```yaml {copy-lines="all",label="xr"}
cat <<EOF | kubectl apply -f -
apiVersion: database.example.com/v1alpha1
kind: NoSQL
metadata:
  name: my-nosql-database
spec: 
  location: "US"
EOF
```

`kubectl get nosql` でリソースを表示します。

```shell {copy-lines="1"}
kubectl get nosql
NAME                SYNCED   READY   COMPOSITION          AGE
my-nosql-database   True     True    dynamo-with-bucket   14s
```

このオブジェクトは Crossplane の _コンポジットリソース_ (XR とも呼ばれます) です。  
これは、Composition テンプレートから作成されたリソースのコレクションを表す単一のオブジェクトです。

`kubectl get managed` で個々のリソースを表示します。

```shell {copy-lines="1"}
kubectl get managed
NAME                                                    READY   SYNCED   EXTERNAL-NAME             AGE
table.dynamodb.aws.upbound.io/my-nosql-database-t5wtx   True    True     my-nosql-database-t5wtx   33s

NAME                                               READY   SYNCED   EXTERNAL-NAME             AGE
bucket.s3.aws.upbound.io/my-nosql-database-xtzph   True    True     my-nosql-database-xtzph   33s
```

`kubectl delete nosql` でリソースを削除します。

```shell {copy-lines="1"}
kubectl delete nosql my-nosql-database
nosql.database.example.com "my-nosql-database" deleted
```

`kubectl get managed` で Crossplane がリソースを削除したことを確認します。

{{<hint "note" >}}
リソースを削除するのに最大で 5 分かかる場合があります。
{{< /hint >}}

```shell {copy-lines="1"}
kubectl get managed
No resources found
```

## 名前空間を使用した API

API `nosql` へのアクセスはクラスターのスコープで行われます。  
ほとんどの組織はユーザーを名前空間に隔離します。  

Crossplane の _Claim_ は名前空間内のカスタム API です。


_Claim_ を作成することは、カスタム API エンドポイントにアクセスするのと同じですが、カスタム API の `claimNames` からの 
{{<hover label="claim" line="3">}}kind{{</hover>}} 
を使用します。

Claim を作成するための新しい名前空間を作成します。

```shell
kubectl create namespace crossplane-test
```

次に、`crossplane-test` 名前空間に Claim を作成します。

```yaml {label="claim",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: database.example.com/v1alpha1
kind: NoSQLClaim
metadata:
  name: my-nosql-database
  namespace: crossplane-test
spec: 
  location: "US"
EOF
```
`kubectl get claim -n crossplane-test` を使用して Claim を表示します。

```shell {copy-lines="1"}
kubectl get claim -n crossplane-test
NAME                SYNCED   READY   CONNECTION-SECRET   AGE
my-nosql-database   True     True                        17s
```

Claim は自動的に複合リソースを作成し、それが管理リソースを作成します。

`kubectl get composite` を使用して Crossplane が作成した複合リソースを表示します。

```shell {copy-lines="1"}
kubectl get composite
NAME                      SYNCED   READY   COMPOSITION          AGE
my-nosql-database-t9qrw   True     True    dynamo-with-bucket   77s
```

再度、`kubectl get managed` を使用して管理リソースを表示します。

```shell {copy-lines="1"}
kubectl get managed
NAME                                                          READY   SYNCED   EXTERNAL-NAME                   AGE
table.dynamodb.aws.upbound.io/my-nosql-database-t9qrw-dcpwv   True    True     my-nosql-database-t9qrw-dcpwv   116s

NAME                                                     READY   SYNCED   EXTERNAL-NAME                   AGE
bucket.s3.aws.upbound.io/my-nosql-database-t9qrw-g98lv   True    True     my-nosql-database-t9qrw-g98lv   117s
```

Claim を削除すると、すべての Crossplane 生成リソースが削除されます。

`kubectl delete claim -n crossplane-test my-nosql-database`

```shell {copy-lines="1"}
kubectl delete claim -n crossplane-test my-nosql-database
nosqlclaim.database.example.com "my-nosql-database" deleted
```

{{<hint "note" >}}
リソースの削除には最大で 5 分かかる場合があります。
{{< /hint >}}

`kubectl get composite` を使用して Crossplane が複合リソースを削除したことを確認します。

```shell {copy-lines="1"}
kubectl get composite
No resources found
```

`kubectl get managed` を使用して Crossplane が管理リソースを削除したことを確認します。

```shell {copy-lines="1"}
kubectl get managed
No resources found
```

## 次のステップ
* Crossplane が構成できる AWS リソースを 
  [Provider CRD リファレンス](https://marketplace.upbound.io/providers/upbound/provider-family-aws/) で探ります。
* [Crossplane Slack](https://slack.crossplane.io/) に参加し、Crossplane ユーザーや貢献者とつながります。
* [Crossplane の概念]({{<ref "../concepts">}}) についてさらに読み、Crossplane でできることを見つけます。

It seems that there is no content provided for translation. Please paste the Markdown content you'd like me to translate into Japanese.
