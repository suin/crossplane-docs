---
title: コンポジットリソース
weight: 50
description: "コンポジットリソース、XRまたはXRsは、関連するクラウドリソースのコレクションを表します。"
---

コンポジットリソースは、管理されたリソースのセットを単一の
Kubernetesオブジェクトとして表します。Crossplaneは、ユーザーが
CompositeResourceDefinitionで定義されたカスタムAPIにアクセスするときに
コンポジットリソースを作成します。

{{<hint "tip" >}}
コンポジットリソースは、管理されたリソースの_コンポジット_です。  
_コンポジション_は、管理されたリソースをどのように_構成_するかを定義します。
{{< /hint >}}

{{<expand "コンポジション、XRD、XR、およびクレームについて混乱していますか？" >}}
Crossplaneには、ユーザーが一般的に混同する4つのコアコンポーネントがあります：

* [コンポジション]({{<ref "./compositions">}}) - リソースを作成する方法を定義するテンプレート。
* [コンポジットリソース定義]({{<ref "./composite-resource-definitions">}})
  (`XRD`) - カスタムAPI仕様。 
* コンポジットリソース (`XR`) - このページ。 
  コンポジットリソース定義で定義されたカスタムAPIを使用して作成されます。 
  XRsは、コンポジションテンプレートを使用して新しい管理リソースを作成します。 
* [クレーム]({{<ref "./claims" >}}) (`XRC`) - コンポジットリソースのようですが、
  名前空間スコープがあります。 
{{</expand >}}

## コンポジットリソースの作成

コンポジットリソースを作成するには、 
[コンポジション]({{<ref "./compositions">}})と 
[コンポジットリソース定義]({{<ref "./composite-resource-definitions">}}) 
(`XRD`)が必要です。  
コンポジションは、作成するリソースのセットを定義します。  
XRDは、ユーザーがリソースのセットを要求するために呼び出すカスタムAPIを定義します。

![Crossplaneコンポーネントの関係の図](/media/composition-how-it-works.svg)

XRDは、コンポジットリソースを作成するために使用されるAPIを定義します。  
例えば、この {{<hover label="xrd1" line="2">}}コンポジットリソース定義{{</hover>}}は、 
カスタムAPIエンドポイント 
{{<hover label="xrd1" line="4">}}xmydatabases.example.org{{</hover>}}を作成します。

```yaml {label="xrd1",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata: 
  name: xmydatabases.example.org
spec:
  group: example.org
  names:
    kind: xMyDatabase
    plural: xmydatabases
  # Removed for brevity
```

ユーザーがカスタムAPI 
{{<hover label="xrd1" line="4">}}xmydatabases.example.org{{</hover>}}を呼び出すと、 
Crossplaneは、コンポジションの 
{{<hover label="typeref" line="6">}}compositeTypeRef{{</hover>}}に基づいて使用するコンポジションを選択します。

```yaml {label="typeref",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: my-composition
spec:
  compositeTypeRef:
    apiVersion: example.org/v1alpha1
    kind: xMyDatabase
  # Removed for brevity
```

Composition
{{<hover label="typeref" line="6">}}compositeTypeRef{{</hover>}} は 
XRD {{<hover label="xrd1" line="6">}}group{{</hover>}} と 
{{<hover label="xrd1" line="9">}}kind{{</hover>}} に一致します。

Crossplane は一致する Composition で定義されたリソースを作成し、
それらを単一の `composite` リソースとして表現します。 

```shell{copy-lines="1"}
kubectl get composite
NAME                    SYNCED   READY   COMPOSITION         AGE
my-composite-resource   True     True    my-composition      4s
```

### 外部リソースの命名
デフォルトでは、コンポジットリソースによって作成された管理リソースは
コンポジットリソースの名前にランダムなサフィックスが付加された名前を持ちます。

<!-- vale Google.FirstPerson = NO -->
<!-- vale Crossplane.Spelling = NO -->
例えば、「my-composite-resource」という名前のコンポジットリソースは
「my-composite-resource-fqvkw」という名前の外部リソースを作成します。 
<!-- vale Google.FirstPerson = YES -->
<!-- vale Crossplane.Spelling = YES  -->

リソース名は、コンポジットリソースに 
{{<hover label="annotation" line="5">}}annotation{{</hover>}} を適用することで
決定論的にすることができます。 

```yaml {label="annotation",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
  annotations: 
    crossplane.io/external-name: my-custom-name
# Removed for brevity
```

Composition 内では、リソースに外部名を適用するために 
{{<hover label="comp" line="10">}}patch{{</hover>}} を使用します。 

{{<hover label="comp" line="11">}}fromFieldPath{{</hover>}} パッチは、
コンポジットリソースから 
{{<hover label="comp" line="11">}}metadata.annotations{{</hover>}} フィールドを
管理リソース内の 
{{<hover label="comp" line="12">}}metadata.annotations{{</hover>}} にコピーします。 

{{<hint "note" >}}
管理リソースに `crossplane.io/external-name` アノテーションがある場合、
Crossplane はアノテーションの値を使用して外部リソースの名前を付けます。
{{</hint >}}

```yaml {label="comp",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: my-composition
spec:
  resources:
    - name: database
      base:
        # Removed for brevity
      patches:
      - fromFieldPath: metadata.annotations
        toFieldPath: metadata.annotations
```

リソースのパッチに関する詳細は、[Patch and Transform]({{<ref "./patch-and-transform">}}) ドキュメントを参照してください。

### コンポジションの選択

特定のコンポジションを選択して、コンポジットリソースで使用します 
{{<hover label="compref" line="6">}}compositionRef{{</hover>}}

{{<hint "important">}}
選択したコンポジションは、コンポジットリソースが
`compositeTypeRef`を使用できるようにする必要があります。 `compositeTypeRef`フィールドについては、コンポジションの
[コンポジットリソースの有効化]({{<ref "./compositions#enabling-composite-resources">}})
セクションを参照してください。 
{{< /hint >}}

```yaml {label="compref",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  compositionRef:
    name: my-other-composition
  # Removed for brevity
```

コンポジットリソースは、正確な名前の代わりにラベルに基づいてコンポジションを選択することもできます 
{{<hover label="complabel" line="6">}}compositionSelector{{</hover>}}を使用して。

{{<hover label="complabel" line="7">}}matchLabels{{</hover>}}セクション内で
一致させる1つ以上のコンポジションラベルを提供します。

```yaml {label="complabel",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  compositionSelector:
    matchLabels:
      environment: production
  # Removed for brevity
```

### コンポジションのリビジョンポリシー

Crossplaneは、コンポジションの変更を 
[コンポジションリビジョン]({{<ref "composition-revisions">}}) として追跡します。 

コンポジットリソースは、 
{{<hover label="comprev" line="6">}}compositionUpdatePolicy{{</hover>}}を使用して
手動または自動で新しいコンポジションリビジョンを参照できます。

デフォルトの 
{{<hover label="comprev" line="6">}}compositionUpdatePolicy{{</hover>}}は 
「自動」です。コンポジットリソースは自動的に最新のコンポジション
リビジョンを使用します。 

ポリシーを 
{{<hover label="comprev" line="6">}}手動{{</hover>}}に変更すると、コンポジット
リソースが自動的にアップグレードされるのを防ぐことができます。

```yaml {label="comprev",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  compositionUpdatePolicy: Manual
  # Removed for brevity
```

### コンポジションリビジョンの選択

Crossplaneは、コンポジションの変更を 
[コンポジションリビジョン]({{<ref "composition-revisions">}}) として記録します。    
コンポジットリソースは、特定のコンポジションリビジョンを
選択できます。

{{<hover label="comprevref" line="6">}}compositionRevisionRef{{</hover>}}を使用して
特定のコンポジションリビジョンを名前で選択します。

たとえば、特定のコンポジションリビジョンを選択するには、
希望するコンポジションリビジョンの名前を使用します。

```yaml {label="comprevref",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  compositionUpdatePolicy: Manual
  compositionRevisionRef:
    name: my-composition-b5aa1eb
  # Removed for brevity
```

{{<hint "note" >}}
Compositionのリビジョン名を見つけるには、 
{{<hover label="getcomprev" line="1">}}kubectl get compositionrevision{{</hover>}}を使用します。

```shell {label="getcomprev",copy-lines="1"}
kubectl get compositionrevision
NAME                         REVISION   XR-KIND        XR-APIVERSION            AGE
my-composition-5c976ad       1          xmydatabases   example.org/v1alpha1     65m
my-composition-b5aa1eb       2          xmydatabases   example.org/v1alpha1     64m
```
{{< /hint >}}

Compositeリソースは、正確な名前の代わりにラベルに基づいてCompositionリビジョンを選択することもできます。
{{<hover label="comprevsel" line="6">}}compositionRevisionSelector{{</hover>}}を使用します。

{{<hover label="comprevsel" line="7">}}matchLabels{{</hover>}} 
セクション内で、1つ以上のCompositionリビジョンラベルを提供して一致させます。

```yaml {label="comprevsel",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  compositionRevisionSelector:
    matchLabels:
      channel: dev
  # Removed for brevity
```

### 接続シークレットの管理

Compositeリソースがリソースを作成すると、Crossplaneは
[接続シークレット]({{<ref "./managed-resources#writeconnectionsecrettoref">}})
をCompositeリソースに提供します。

{{<hint "important" >}}

リソースは、XRDによって許可された接続シークレットのみをアクセスできます。デフォルトでは、XRDは管理リソースによって生成されたすべての接続シークレットへのアクセスを提供します。  
XRDドキュメントで[接続シークレットの管理]({{<ref "./composite-resource-definitions#manage-connection-secrets">}})について詳しく読むことができます。
{{< /hint >}}

{{<hover label="writesecret" line="6">}}writeConnectionSecretToRef{{</hover>}} 
を使用して、Compositeリソースが接続シークレットを書き込む場所を指定します。

例えば、このCompositeリソースは接続シークレットを
{{<hover label="writesecret" line="7">}}my-secret{{</hover>}}という名前のKubernetesシークレットオブジェクトに、 
{{<hover label="writesecret" line="8">}}crossplane-system{{</hover>}}という名前空間に保存します。

```yaml {label="writesecret",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  writeConnectionSecretToRef:
    name: my-secret
    namespace: crossplane-system
  # Removed for brevity
```

Compositeリソースは、HashiCorp Vaultのような
[外部シークレットストア]({{<ref "../guides/vault-as-secret-store">}})に接続シークレットを書き込むことができます。

{{<hint "important" >}}
外部シークレットストアはアルファ機能です。アルファ機能はデフォルトでは有効になっていません。 
{{< /hint >}}

{{<hover label="publishsecret"
line="6">}}publishConnectionDetailsTo{{</hover>}} フィールドを使用して、接続シークレットを外部シークレットストアに保存します。

```yaml {label="publishsecret",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
spec:
  publishConnectionDetailsTo:
    name: my-external-secret-store
  # Removed for brevity
```

外部シークレットストアの使用に関する詳細は、[External Secrets Store]({{<ref "../guides/vault-as-secret-store">}}) ドキュメントを参照してください。

接続シークレットに関する詳細は、[Connection Secrets knowledge base article]({{<ref "connection-details">}}) を参照してください。

### 複合リソースの一時停止

<!-- vale Google.WordList = NO -->
Crossplane は複合リソースの一時停止をサポートしています。一時停止された複合リソースは、その外部リソースをチェックしたり変更したりしません。
<!-- vale Google.WordList = YES -->

複合リソースを一時停止するには、 
{{<hover label="pause" line="4">}}crossplane.io/paused{{</hover>}} アノテーションを適用します。

```yaml {label="pause",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: xMyDatabase
metadata:
  name: my-composite-resource
  annotations:
    crossplane.io/paused: "true"
spec:
  # Removed for brevity
```

## 複合リソースの確認
すべての複合リソースを表示するには、 
{{<hover label="getcomposite" line="1">}}kubectl get composite{{</hover>}} を使用します。

```shell{copy-lines="1",label="getcomposite"}
kubectl get composite
NAME                    SYNCED   READY   COMPOSITION         AGE
my-composite-resource   True     True    my-composition      4s
```

特定のカスタム API エンドポイントのリソースのみを表示するには、`kubectl get` を使用します。

```shell {copy-lines="1"}
kubectl get xMyDatabase.example.org
NAME                    SYNCED   READY   COMPOSITION        AGE
my-composite-resource   True     True    my-composition     12m
```

リンクされた 
{{<hover label="desccomposite" line="16">}}Composition Ref{{</hover>}} と、 
{{<hover label="desccomposite" line="22">}}Resource Refs{{</hover>}} に作成されたユニークな管理リソースを表示するには、 
{{<hover label="desccomposite" line="1">}}kubectl describe composite{{</hover>}} を使用します。

```yaml {copy-lines="1",label="desccomposite"}
kubectl describe composite my-composite-resource
Name:         my-composite-resource
API Version:  example.org/v1alpha1
Kind:         xMyDatabase
Spec:
  Composition Ref:
    Name:  my-composition
  Composition Revision Ref:
    Name:                     my-composition-cf2d3a7
  Composition Update Policy:  Automatic
  Resource Refs:
    API Version:  s3.aws.upbound.io/v1beta1
    Kind:         Bucket
    Name:         my-composite-resource-fmrks
    API Version:  dynamodb.aws.upbound.io/v1beta1
    Kind:         Table
    Name:         my-composite-resource-wnr9t
# Removed for brevity
```

### 複合リソースの条件

複合リソースの条件は、その管理リソースの条件と一致します。

以下の内容を参照してください。
[条件セクション]({{<ref "./managed-resources#conditions">}})の
管理リソースのドキュメントの詳細については。

## 複合リソースのラベル

Crossplaneは、複合リソースにラベルを追加して、他のCrossplaneコンポーネントとの関係を示します。

### 複合ラベル
Crossplaneは、すべての複合リソースに 
{{<hover label="complabel" line="4">}} crossplane.io/composite{{</hover>}} ラベルを追加します。このラベルは、複合の名前と一致します。
Crossplaneは、複合によって作成された任意の管理リソースに複合ラベルを適用し、管理リソースと所有する複合リソースとの間に参照を作成します。

```shell {label="claimname",copy-lines="1"}
kubectl describe xmydatabase.example.org/my-claimed-database-x9rx9
Name:         my-claimed-database2-x9rx9
Namespace:
Labels:       crossplane.io/composite=my-claimed-database-x9rx9
```

### クレーム名ラベル
Crossplaneは、クレームから作成された複合リソースに 
{{<hover label="claimname" line="4">}}crossplane.io/claim-name{{</hover>}} 
ラベルを追加します。このラベルは、この複合リソースにリンクされたクレームの名前を示します。

```shell {label="claimname",copy-lines="1"}
kubectl describe xmydatabase.example.org/my-claimed-database-x9rx9
Name:         my-claimed-database2-x9rx9
Namespace:
Labels:       crossplane.io/claim-name=my-claimed-database
```

クレームを使用せずに直接作成された複合リソースには、 
{{<hover label="claimname" line="4">}}crossplane.io/claim-name{{</hover>}} 
ラベルはありません。

### クレームネームスペースラベル
Crossplaneは、クレームから作成された複合リソースに 
{{<hover label="claimname" line="4">}}crossplane.io/claim-namespace{{</hover>}} 
ラベルを追加します。このラベルは、この複合リソースにリンクされたクレームのネームスペースを示します。

```shell {label="claimname",copy-lines="1"}
kubectl describe xmydatabase.example.org/my-claimed-database-x9rx9
Name:         my-claimed-database2-x9rx9
Namespace:
Labels:       crossplane.io/claim-namespace=default
```

クレームを使用せずに直接作成された複合リソースには、 
{{<hover label="claimname" line="4">}}crossplane.io/claim-namespace{{</hover>}} 
ラベルはありません。
