---
title: クレーム
weight: 60
description: "クレームは、名前空間スコープを持つCrossplaneリソースを消費する方法です"
---

クレームは、名前空間内の単一のKubernetesオブジェクトとして管理されたリソースのセットを表します。

ユーザーは、CompositeResourceDefinitionで定義されたカスタムAPIにアクセスする際にクレームを作成します。

{{< hint "tip" >}}

クレームは[複合リソース]({{<ref "./composite-resources">}})のようなものです。クレームと複合リソースの違いは、Crossplaneが名前空間内にクレームを作成できるのに対し、複合リソースはクラスター全体にスコープされることです。
{{< /hint >}}

{{<expand "構成、XRD、XR、およびクレームについて混乱していますか？" >}}
Crossplaneには、ユーザーが一般的に混同する4つのコアコンポーネントがあります：

* [構成]({{<ref "./compositions">}}) - リソースを作成する方法を定義するテンプレート。
* [複合リソース定義]({{<ref "./composite-resource-definitions">}}) 
  (`XRD`) - カスタムAPI仕様。
* [複合リソース]({{<ref "./composite-resources">}}) (`XR`) - 複合リソース定義で定義されたカスタムAPIを使用して作成されます。XRは、構成テンプレートを使用して新しい管理リソースを作成します。
* クレーム (`XRC`) - このページ。複合リソースのようですが、名前空間スコープがあります。
{{</expand >}}

## クレームの作成

クレームを作成するには、 
[構成]({{<ref "./compositions">}})と 
[複合リソース定義]({{<ref "./composite-resource-definitions">}}) 
(`XRD`)がすでにインストールされている必要があります。

{{<hint "note" >}}
XRDは 
[クレームを有効にする]({{<ref "./composite-resource-definitions#enable-claims">}})必要があります。
{{< /hint >}}

構成は、作成するリソースのセットを定義します。  
XRDは、ユーザーがリソースのセットを要求するために呼び出すカスタムAPIを定義します。

![Crossplaneコンポーネントの関係の図](/media/composition-how-it-works.svg)

例えば、この{{<hover label="xrd1" line="2">}}複合リソース定義{{</hover>}}は、複合リソースAPIエンドポイント{{<hover label="xrd1" line="4">}}xmydatabases.example.org{{</hover>}}を作成し、クレームAPIエンドポイント{{<hover label="xrd1" line="11">}}database.example.org{{</hover>}}を有効にします。

```yaml {label="xrd1",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata: 
  name: xmydatabases.example.org
spec:
  group: example.org
  names:
    kind: XMyDatabase
    plural: xmydatabases
  claimNames:
    kind: Database
    plural: databases
  # Removed for brevity
```

ClaimはXRDの 
{{<hover label="xrd1" line="11">}}kind{{</hover>}} APIエンドポイントを使用して 
リソースをリクエストします。

Claimの{{<hover label="xrd1" line="1">}}apiVersion{{</hover>}}は
XRDの{{<hover label="xrd1" line="6">}}group{{</hover>}}と一致し、 
{{<hover label="claim1" line="2">}}kind{{</hover>}}はXRDの
{{<hover label="xrd1" line="11">}}claimNames.kind{{</hover>}}と一致します。

```yaml {label="claim1",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: database
metadata:
  name: my-claimed-database
spec:
  # Removed for brevity
```

ユーザーが名前空間にClaimを作成すると、Crossplaneはコンポジット
リソースも作成します。

Claimに対して{{<hover label="claimcomp" line="1">}}kubectl describe{{</hover>}}を使用して、関連するコンポジットリソースを表示します。

{{<hover label="claimcomp" line="6">}}Resource Ref{{</hover>}}は
このClaimのためにCrossplaneが作成したコンポジットリソースです。

```shell {label="claimcomp",copy-lines="1"}
kubectl describe database.example.org/my-claimed-database
Name:         my-claimed-database
API Version:  example.org/v1alpha1
Kind:         database
Spec:
  Resource Ref:
    API Version:  example.org/v1alpha1
    Kind:         XMyDatabase
    Name:         my-claimed-database-rr4ll
# Removed for brevity.
```

コンポジットリソースに対して{{<hover label="getcomp" line="1">}}kubectl describe{{</hover>}}を使用して、 
コンポジットリソースを元のClaimにリンクする{{<hover label="getcomp" line="6">}}Claim Ref{{</hover>}}を表示します。

```shell {label="getcomp",copy-lines="1"}
kubectl describe xmydatabase.example.org/my-claimed-database-rr4ll
Name:         my-claimed-database-rr4ll
API Version:  example.org/v1alpha1
Kind:         XMyDatabase
Spec:
  Claim Ref:
    API Version:  example.org/v1alpha1
    Kind:         database
    Name:         my-claimed-database
    Namespace:    default
```

{{<hint "note" >}}
Crossplaneはコンポジットリソースを直接作成することをサポートしています。Claimは
カスタムAPIを利用するユーザーのために名前空間のスコープと隔離を提供します。

Kubernetesのデプロイメントで名前空間を使用しない場合、Claimは必要ありません。
{{< /hint >}}

### 既存のコンポジットリソースの請求

デフォルトでは、Claimを作成すると新しいコンポジットリソースが作成されます。Claimは既存のコンポジットリソースにリンクすることもできます。

既存のコンポジットリソースを請求するユースケースは、リソースのプロビジョニングが遅い場合です。コンポジットリソースは事前にプロビジョニングされ、Claimはそれらのリソースを作成を待たずに使用できます。

Claimの{{<hover label="resourceref" line="6">}}resourceRef{{</hover>}}を設定し、既存のコンポジットリソースの
{{<hover label="resourceref" line="9">}}name{{</hover>}}と一致させます。

```yaml {label="resourceref",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: database
metadata:
  name: my-claimed-database
spec:
  resourceRef:
    apiVersion: example.org/v1alpha1
    kind: XMyDatabase
    name: my-pre-created-xr
```

クレームが存在しない 
{{<hover label="resourceref" line="6">}}resourceRef{{</hover>}}を指定した場合、Crossplaneは複合リソースを作成しません。

{{<hint "note" >}}
すべてのクレームには 
{{<hover label="resourceref" line="6">}}resourceRef{{</hover>}}があります。手動で 
{{<hover label="resourceref" line="6">}}resourceRef{{</hover>}}を定義する必要はありません。Crossplaneはクレームのために作成された複合リソースからの情報で 
{{<hover label="resourceref" line="6">}}resourceRef{{</hover>}}を埋めます。
{{< /hint >}}

## クレーム接続シークレット

クレームが接続シークレットを期待する場合、クレームは 
{{<hover label="claimSec" line="6">}}writeConnectionSecretToRef{{</hover>}}オブジェクトを定義する必要があります。

{{<hover label="claimSec" line="6">}}writeConnectionSecretToRef{{</hover>}}オブジェクトは、Crossplaneが接続の詳細を保存するKubernetesシークレットオブジェクトの名前を定義します。

{{<hint "note" >}}
Crossplaneはクレームと同じ名前空間にシークレットオブジェクトを作成します。
{{< /hint >}}

たとえば、新しいシークレットオブジェクトの名前を 
{{<hover label="claimSec" line="7">}}my-claim-secret{{</hover>}}にするには、 
{{<hover label="claimSec" line="6">}}writeConnectionSecretToRef{{</hover>}}を使用して 
{{<hover label="claimSec" line="7">}}name: my-claim-secret{{</hover>}}を指定します。
```yaml {label="claimSec"}
apiVersion: example.org/v1alpha1
kind: database
metadata:
  name: my-claimed-database
spec:
  writeConnectionSecretToRef:
    name: my-claim-secret
```

接続シークレットに関する詳細は、[接続シークレットのナレッジベース記事]({{<ref "connection-details">}})をお読みください。
