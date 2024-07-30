---
title: コンポジットリソース定義
weight: 40
description: "コンポジットリソース定義またはXRDはカスタムAPIスキーマを定義します"
---

コンポジットリソース定義（`XRD`）はカスタムAPIのスキーマを定義します。  
ユーザーは`XRD`で定義されたAPIスキーマを使用してコンポジットリソース（`XR`）とクレーム（`XC`）を作成します。


{{< hint "note" >}}

コンポジットリソースに関する詳細は[コンポジットリソース]({{<ref "./composite-resources">}})ページをお読みください。

クレームに関する詳細は[クレーム]({{<ref "./claims">}})ページをお読みください。
{{</hint >}}


{{<expand "コンポジション、XRD、XR、およびクレームについて混乱していますか？" >}}
Crossplaneには、ユーザーが一般的に混同する4つのコアコンポーネントがあります：

* [コンポジション]({{<ref "./compositions" >}}) - リソースを作成する方法を定義するテンプレート。
* コンポジットリソース定義（`XRD`） - このページ。カスタムAPI仕様。 
* [コンポジットリソース]({{<ref "./composite-resources">}})（`XR`） - コンポジットリソース定義で定義されたカスタムAPIを使用して作成されます。XRはコンポジションテンプレートを使用して新しい管理リソースを作成します。 
* [クレーム]({{<ref "./claims" >}})（`XRC`） - コンポジットリソースのようですが、名前空間スコープがあります。 
{{</expand >}}

CrossplaneのXRDは、[Kubernetesカスタムリソース定義](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)のようなものです。 
XRDは必要なフィールドが少なく、クレームや接続シークレットなどCrossplaneに関連するオプションを追加します。 

## コンポジットリソース定義の作成

コンポジットリソース定義の作成は以下から成ります：
* [カスタムAPIグループの定義](#xrd-groups)。
* [カスタムAPI名の定義](#xrd-names)。
* [カスタムAPIスキーマとバージョンの定義](#xrd-versions)。
  
オプションとして、コンポジットリソース定義は以下もサポートします：
* [クレームの提供](#enable-claims)。
* [接続シークレットの定義](#manage-connection-secrets)。
* [コンポジットリソースのデフォルト設定](#set-composite-resource-defaults)。
 
コンポジットリソース定義（`XRD`）はKubernetesクラスター内に新しいAPIエンドポイントを作成します。 

新しいAPIを作成するには、API 
{{<hover label="xrd1" line="6">}}グループ{{</hover>}},
{{<hover label="xrd1" line="7">}}名前{{</hover>}}および
{{<hover label="xrd1" line="10">}}バージョン{{</hover>}}を定義する必要があります。

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
  versions:
  - name: v1alpha1
  # Removed for brevity
```

XRDを適用すると、Crossplaneは定義されたAPIに一致する新しいKubernetesカスタムリソース定義を作成します。

例えば、XRD 
{{<hover label="xrd1" line="4">}}xmydatabases.example.org{{</hover >}} 
はカスタムリソース定義 
{{<hover label="kubeapi" line="2">}}xmydatabases.example.org{{</hover >}}を作成します。  

```shell {label="kubeapi",copy-lines="3"}
kubectl api-resources
NAME                              SHORTNAMES   APIVERSION          NAMESPACED   KIND
xmydatabases.example.org                       v1alpha1            false        xmydatabases
# Removed for brevity
```

{{<hint "warning">}}
XRDの
{{<hover label="xrd1" line="6">}}group{{</hover>}}や
{{<hover label="xrd1" line="7">}}names{{</hover>}}を変更することはできません。  
{{<hover label="xrd1" line="6">}}group{{</hover>}}や
{{<hover label="xrd1" line="7">}}names{{</hover>}}を変更するには、XRDを削除して再作成する必要があります。
{{</hint >}}

### XRDグループ

グループは関連するAPIエンドポイントのコレクションを定義します。 `group`は任意の値を使用できますが、一般的な慣習は完全修飾ドメイン名にマッピングすることです。

<!-- vale write-good.Weasel = NO -->
多くのXRDが同じ`group`を使用してAPIの論理コレクションを作成することがあります。  
<!-- vale write-good.Weasel = YES -->
例えば、`database`グループには`relational`と`nosql`の種類があるかもしれません。

{{<hint "tip" >}}
グループ名はクラスターのスコープです。プロバイダーと競合しないグループ名を選択してください。  
グループ内でプロバイダー名を避けてください。
{{< /hint >}}

### XRD名

`names`フィールドは、この特定のXRDを参照する方法を定義します。  
必要な名前フィールドは次のとおりです：

* `kind` - このAPIを呼び出すときに使用する`kind`値。kindは
  [UpperCamelCased](https://kubernetes.io/docs/contribute/style/style-guide/#use-upper-camel-case-for-api-objects)である必要があります。
  Crossplaneは、XRDの`kind`を`X`で始めることを推奨しており、これはカスタムCrossplane API定義であることを示します。 
* `plural` - API URLに使用される複数形の名前。複数形の名前は
  小文字である必要があります。

{{<hint "important" >}}
XRD 
{{<hover label="xrdName" line="4">}}metadata.name{{</hover>}}は 
{{<hover label="xrdName" line="9">}}plural{{</hover>}}名、`.`（ドット文字）、
{{<hover label="xrdName" line="6">}}group{{</hover>}}である必要があります。

例えば、{{<hover label="xrdName" line="4">}}xmydatabases.example.org{{</hover>}}は{{<hover label="xrdName" line="9">}}plural{{</hover>}}名{{<hover label="xrdName" line="9">}}xmydatabases{{</hover>}}、 `.` {{<hover label="xrdName" line="6">}}group{{</hover>}}名、{{<hover label="xrdName" line="6">}}example.org{{</hover>}}に一致します。

```yaml {label="xrdName",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata: 
  name: xmydatabases.example.org
spec:
  group: example.org
  names:
    kind: XMyDatabase
    plural: xmydatabases
    # Removed for brevity
```
{{</hint >}}

### XRD バージョン

<!-- vale gitlab.SentenceLength = NO -->
XRD `version` は、 
[Kubernetes によって使用される API バージョニング](https://kubernetes.io/docs/reference/using-api/#api-versioning)のようなものです。
バージョンは、API がどれだけ成熟しているか、または安定しているかを示し、API のフィールドを変更、追加、または削除する際に増加します。
<!-- vale gitlab.SentenceLength = YES -->

Crossplane は特定のバージョンや特定のバージョン命名規則を必要としませんが、 
[Kubernetes API バージョニングガイドライン](https://kubernetes.io/docs/reference/using-api/#api-versioning)に従うことが強く推奨されます。

* `v1alpha1` - いつでも変更される可能性のある新しい API。
* `v1beta1` - 安定していると見なされる既存の API。破壊的変更は強く推奨されません。
* `v1` - 破壊的変更がない安定した API。

#### スキーマの定義

<!-- vale write-good.Passive = NO -->
<!-- vale write-good.TooWordy = NO -->
`schema` は、パラメータの名前、パラメータのデータ型、およびどのパラメータが必須またはオプションであるかを定義します。
<!-- vale write-good.Passive = YES -->
<!-- vale write-good.TooWordy = YES -->

{{<hint "note" >}}
すべての `schemas` は、Kubernetes カスタムリソース定義の 
[OpenAPIv3 構造スキーマ](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#specifying-a-structural-schema)に従います。
{{< /hint >}}

API の各{{<hover label="schema" line="11">}}version{{</hover>}}には、ユニークな{{<hover label="schema" line="12">}}schema{{</hover>}}があります。

すべての XRD {{<hover label="schema" line="12">}}schemas{{</hover>}}は、{{<hover label="schema" line="13">}}openAPIV3Schema{{</hover>}}に対して検証されます。スキーマは、OpenAPIの{{<hover label="schema" line="14">}}object{{</hover>}}であり、{{<hover label="schema" line="15">}}properties{{</hover>}}は{{<hover label="schema" line="16">}}spec{{</hover>}}の{{<hover label="schema" line="17">}}object{{</hover>}}です。


{{<hover label="schema" line="18">}}spec.properties{{</hover>}}の中にはカスタム
API定義があります。

この例では、キー{{<hover label="schema" line="19">}}region{{</hover>}}
は{{<hover label="schema" line="20">}}string{{</hover>}}です。

```yaml {label="schema",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  group: custom-api.example.org
  names:
    kind: xDatabase
    plural: xdatabases
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              region:
                type: string
    # Removed for brevity
```

このAPIを使用するコンポジットリソースは、 
{{<hover label="xr" line="1">}}group/version{{</hover>}}と 
{{<hover label="xr" line="2">}}kind{{</hover>}}を参照します。 
{{<hover label="xr" line="5">}}spec{{</hover>}}には、 
{{<hover label="xr" line="6">}}region{{</hover>}}キーが文字列値で含まれています。 

```yaml {label="xr"}
apiVersion: custom-api.example.org/v1alpha1
kind: xDatabase
metadata:
  name: my-composite-resource
spec: 
  region: "US"
```


{{<hint "tip" >}}
{{<hover label="schema" line="18">}}spec.properties{{</hover>}}の中に定義されたカスタムAPIはOpenAPIv3
仕様です。Swaggerドキュメントの 
[data models page](https://swagger.io/docs/specification/data-models/) では、データ型や入力
制限を使用した例のリストが提供されています。

Kubernetesドキュメントでは、 
[OpenAPIv3カスタムAPIが使用できる特別な制限のセット](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#validation)
がリストされています。
{{< /hint >}}

{{<hint "important" >}}

XRDスキーマを変更または拡張するには、[Crossplane pod]({{<ref "./pods#crossplane-pod">}})を再起動する必要があります。
{{< /hint >}}

##### 必須フィールド

デフォルトでは、スキーマ内のすべてのフィールドはオプションです。 
{{< hover label="required" line="25">}}required{{</hover>}}属性を使用してパラメータを必須として定義します。

この例では、XRDは 
{{< hover label="required" line="19">}}region{{</hover>}}と 
{{< hover label="required" line="21">}}size{{</hover>}}を必要としますが、
{{< hover label="required" line="23">}}name{{</hover>}}はオプションです。 
```yaml {label="required",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  group: custom-api.example.org
  names:
    kind: xDatabase
    plural: xdatabases
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              region:
                type: string 
              size:
                type: string  
              name:
                type: string  
            required: 
              - region 
              - size
    # Removed for brevity
```

OpenAPIv3仕様によれば、`required`フィールドはオブジェクトごとです。 
スキーマに複数のオブジェクトが含まれている場合、スキーマには複数の`required`
フィールドが必要になることがあります。

このXRDは2つのオブジェクトを定義します：
 1. トップレベルの {{<hover label="required2" line="7">}}spec{{</hover>}} オブジェクト
 2. 2番目の {{<hover label="required2" line="14">}}location{{</hover>}} オブジェクト

{{<hover label="required2" line="7">}}spec{{</hover>}} オブジェクトは 
{{<hover label="required2" line="23">}}requires{{</hover>}} 
{{<hover label="required2" line="10">}}size{{</hover>}} と 
{{<hover label="required2" line="14">}}location{{</hover>}} を必要としますが、 
{{<hover label="required2" line="12">}}name{{</hover>}} はオプションです。

必須の {{<hover label="required2" line="14">}}location{{</hover>}} 
オブジェクト内では、
{{<hover label="required2" line="17">}}country{{</hover>}} は 
{{<hover label="required2" line="21">}}required{{</hover>}} であり、
{{<hover label="required2" line="19">}}zone{{</hover>}} はオプションです。

```yaml {copy-lines="none",label="required2"}
# Removed for brevity
- name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size:
                type: string  
              name:
                type: string 
              location:
                type: object
                properties:
                  country: 
                    type: string 
                  zone:
                    type: string
                required:
                  - country
            required:  
              - size
              - location
```

Swaggerの "[Describing Parameters](https://swagger.io/docs/specification/describing-parameters/)"
ドキュメントには、さらに多くの例があります。

##### Crossplaneの予約フィールド

Crossplaneはスキーマ内で以下のフィールドを許可しません：
* `spec.resourceRef`
* `spec.resourceRefs`
* `spec.claimRef`
* `spec.writeConnectionSecretToRef`
* `status.conditions`
* `status.connectionDetails`

Crossplaneは予約フィールドに一致するフィールドを無視します。

#### スキーマを提供し参照する

スキーマを使用するには、次のようにする必要があります：
{{<hover label="served" line="12" >}}served: true{{</hover >}}
および 
{{<hover label="served" line="13" >}}referenceable: true{{</hover>}}。

```yaml {label="served"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  group: custom-api.example.org
  names:
    kind: xDatabase
    plural: xdatabases
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              region:
                type: string            
```

複合リソースは、{{<hover label="served" line="12" >}}served: true{{</hover >}}として設定された任意のスキーマバージョンを使用できます。  
Kubernetesは、`served: false`として設定されたスキーマバージョンを使用する複合リソースを拒否します。

{{< hint "tip" >}}
スキーマバージョンを`served:false`として設定すると、古いスキーマを使用しているユーザーにエラーが発生します。これは、古いスキーマバージョンを削除する前にユーザーを特定し、アップグレードする効果的な方法となる可能性があります。 
{{< /hint >}}

{{<hover label="served" line="13" >}}referenceable: true{{</hover>}} 
フィールドは、Compositionが使用するスキーマのバージョンを示します。 
`referenceable`であることができるのは1つのバージョンのみです。


{{< hint "note" >}}
`referenceable:true` のバージョンを変更するには、その XRD を参照している任意の Composition の [compositeTypeRef.apiVersion]({{<ref "./compositions#enabling-composite-resources" >}}) を更新する必要があります。
{{< /hint >}}


#### 複数のスキーマバージョン

{{<hint "warning" >}}
Crossplane は複数の `versions` を定義することをサポートしていますが、各バージョンのスキーマは既存のフィールドを変更することはできず、これを「破壊的変更」と呼びます。

バージョン間の破壊的スキーマ変更には、[変換ウェブホック](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#webhook-conversion) の使用が必要です。

新しいバージョンは新しいオプションのパラメータを定義できますが、新しい必須フィールドは「破壊的変更」となります。

<!-- vale Crossplane.Spelling = NO -->
<!-- ignore to allow for CRDs -->
<!-- don't add to the spelling exceptions to catch when it's used instead of XRD -->
Crossplane XRD はバージョン管理のために Kubernetes カスタムリソース定義を使用します。 
バージョンと破壊的変更に関する詳細は、Kubernetes の 
[CustomResourceDefinitions におけるバージョン](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/) に関するドキュメントをお読みください。 
<!-- vale Crossplane.Spelling = YES -->

Crossplane は、破壊的スキーマ変更を新しい XRD として実装することを推奨します。
{{< /hint >}}

XRD の場合、API の新しいバージョンを作成するには、新しい 
{{<hover label="ver" line="21">}}name{{</hover>}} を 
{{<hover label="ver" line="10">}}versions{{</hover>}} リストに追加します。 

例えば、この XRD バージョン 
{{<hover label="ver" line="11">}}v1alpha1{{</hover>}} には 
{{<hover label="ver" line="19">}}region{{</hover>}} フィールドのみがあります。

2 番目のバージョン 
{{<hover label="ver" line="21">}}v1{{</hover>}} は API を拡張し、 
{{<hover label="ver" line="29">}}region{{</hover>}} と 
{{<hover label="ver" line="31">}}size{{</hover>}} の両方を持つようになります。

```yaml {label="ver",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  group: custom-api.example.org
  names:
    kind: xDatabase
    plural: xdatabases
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              region:
                type: string  
  - name: v1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              region:
                type: string 
              size:
                type: string            
```

{{<hint "important" >}}

XRD スキーマの変更または拡張には、[Crossplane pod]({{<ref "./pods#crossplane-pod">}}) を再起動する必要があります。
{{< /hint >}}

### クレームの有効化

オプションとして、XRDはクレームがXRD APIを使用することを許可できます。

{{<hint "note" >}}

クレームに関する詳細情報は[Claims]({{<ref "./claims">}})ページを参照してください。
{{</hint >}}

XRDはクレームに対して
{{<hover label="claim" line="10">}}claimNames{{</hover >}}オブジェクトを提供します。

{{<hover label="claim" line="10">}}claimNames{{</hover >}}は、XRDの
{{<hover label="claim" line="7">}}names{{</hover >}}オブジェクトのように
{{<hover label="claim" line="11">}}kind{{</hover >}}と
{{<hover label="claim" line="12">}}plural{{</hover >}}を定義します。   
また、XRDの
{{<hover label="claim" line="7">}}names{{</hover >}}と同様に、
{{<hover label="claim" line="11">}}kind{{</hover >}}にはUpperCamelCaseを使用し、
{{<hover label="claim" line="12">}}plural{{</hover >}}には小文字を使用します。

クレームの
{{<hover label="claim" line="11">}}kind{{</hover >}}と
{{<hover label="claim" line="12">}}plural{{</hover >}}は一意でなければなりません。
他のクレームや他のXRDの
{{<hover label="claim" line="8">}}kind{{</hover >}}と一致してはいけません。

{{<hint "tip" >}}
一般的なCrossplaneの慣例は、XRDの
{{<hover label="claim" line="7">}}names{{</hover >}}と一致する
{{<hover label="claim" line="10">}}claimNames{{</hover >}}を使用することですが、
先頭の"x."は除外します。
{{</hint >}}

```yaml {label="claim",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  group: custom-api.example.org
  names:
    kind: xDatabase
    plural: xdatabases
  claimNames:
    kind: Database
    plural: databases
  versions:
  # Removed for brevity
```

{{<hint "important" >}}
一度定義された
{{<hover label="claim" line="10">}}claimNames{{</hover >}}は変更できません。
{{<hover label="claim" line="10">}}claimNames{{</hover >}}を変更するには、XRDを削除して再作成する必要があります。
{{</hint >}}

### 接続シークレットの管理

複合リソースが管理リソースを作成する際、Crossplaneは複合リソースまたはクレームに対して
[接続シークレット]({{<ref "./managed-resources#writeconnectionsecrettoref">}})を提供します。
これには、複合リソースとクレームの作成者が管理リソースによって提供されるシークレットを知っている必要があります。
他の場合では、Crossplaneの管理者は生成された接続シークレットの一部またはすべてを公開したくないかもしれません。

XRDは、複合リソースまたはクレームに提供される内容を制限するために
{{<hover label="key" line="10">}}connectionSecretKeys{{</hover>}}のリストを定義できます。


Crossplaneは、このXRDを使用して合成リソースまたはクレームに対して、  
{{<hover label="key" line="10">}}connectionSecretKeys{{</hover>}}  
にリストされているキーのみを提供します。他の接続シークレットは合成リソースまたはクレームに渡されません。  

{{<hint "important" >}}  
{{<hover label="key" line="10">}}connectionSecretKeys{{</hover>}}  
にリストされているキーは、Compositionの`connectionDetails`にリストされているキー名と一致する必要があります。  

XRDは、管理リソースによって作成されていないキーを無視します。  

詳細については、  
[Composition documentation]({{<ref "./compositions#storing-connection-details">}})をお読みください。  
{{< /hint >}}  

例えば、XRDはキー  
{{<hover label="key" line="11">}}username{{</hover>}},  
{{<hover label="key" line="12">}}password{{</hover>}} および  
{{<hover label="key" line="13">}}address{{</hover>}}を渡します。  

合成リソースまたはクレームは、これらを`writeConnectionSecretToRef`フィールドで定義されたシークレットに保存します。  

```yaml {label="key",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  group: custom-api.example.org
  names:
    kind: xDatabase
    plural: xdatabases
  connectionSecretKeys:
    - username
    - password
    - address
  versions:
  # Removed for brevity
```

{{<hint "warning">}}  
XRDの`connectionSecretKeys`を変更することはできません。`connectionSecretKeys`を変更するには、XRDを削除して再作成する必要があります。  
{{</hint >}}  

接続シークレットに関する詳細は、  
[Connection Secrets knowledge base article]({{<ref "connection-details">}})をお読みください。  

### 合成リソースのデフォルトを設定  
XRDは、合成リソースとクレームのデフォルトパラメータを設定できます。  

<!-- vale off -->  
#### defaultCompositeDeletePolicy  
<!-- vale on -->  
`defaultCompositeDeletePolicy`は、ユーザーがクレームを作成する際に値を指定しない場合のクレームの`compositeDeletePolicy`プロパティのデフォルト値を定義します。クレームコントローラーは、関連する合成を削除する際に伝播ポリシーを指定するために`compositeDeletePolicy`プロパティを使用します。`compositeDeletePolicy`は、関連するクレームを持たないスタンドアロンの合成には適用されません。  

`defaultCompositeDeletePolicy: Background`ポリシーを使用すると、クレームのCRDは`compositeDeletePolicy`プロパティのデフォルト値`Background`を持つことになります。削除されたクレームの`compositeDeletePolicy`プロパティが`Background`に設定されている場合、クレームコントローラーは伝播ポリシー`background`を使用して合成リソースを削除し、残りの子オブジェクト（管理リソース、ネストされた合成、シークレットなど）を削除するためにKubernetesに依存します。


`defaultCompositeDeletePolicy: Foreground` を使用すると、クレームの CRD に `compositeDeletePolicy` のデフォルト値 `Foreground` が設定されます。削除されたクレームが `compositeDeletePolicy` プロパティを `Foreground` に設定している場合、コントローラーは関連するコンポジットを伝播ポリシー `foreground` を使用して削除します。これにより、Kubernetes はフォアグラウンドカスケード削除を使用し、親リソースを削除する前にすべての子リソースを削除します。クレームコントローラーは、削除が完了するまで待機します。

クレームを作成する際、ユーザーは `spec.compositeDeletePolicy` プロパティに `Background` または `Foreground` の値を含めることで `defaultCompositeDeletePolicy` をオーバーライドできます。

デフォルト値は `defaultCompositeDeletePolicy: Background` です。

{{<hover label="delete" line="6">}}defaultCompositeDeletePolicy: Foreground{{</hover>}} 
を設定して XRD 削除ポリシーを変更します。

```yaml {label="delete",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  defaultCompositeDeletePolicy: Foreground
  group: custom-api.example.org
  names:
  # Removed for brevity
  versions:
  # Removed for brevity
```

<!-- vale off -->
#### defaultCompositionRef
<!-- vale on -->
複数の [Compositions]({{<ref "./compositions">}}) が同じ XRD を参照することが可能です。複数の Composition が同じ XRD を参照する場合、コンポジットリソースまたはクレームはどの Composition を使用するかを選択する必要があります。

XRD は `defaultCompositionRef` 値を使用して、使用するデフォルトの Composition を定義できます。

{{<hover label="defaultComp" line="6">}}defaultCompositionRef{{</hover>}} 
を設定してデフォルトの Composition を設定します。

```yaml {label="defaultComp",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  defaultCompositionRef: 
    name: myComposition
  group: custom-api.example.org
  names:
  # Removed for brevity
  versions:
  # Removed for brevity
```

<!-- vale off -->
#### defaultCompositionUpdatePolicy
<!-- vale on -->

Composition の変更は新しい Composition リビジョンを生成します。デフォルトでは、すべてのコンポジットリソースとクレームは更新された Composition リビジョンを使用します。

XRD の `defaultCompositionUpdatePolicy` を `Manual` に設定して、コンポジットリソースとクレームが新しいリビジョンを自動的に使用しないようにします。

デフォルト値は `defaultCompositionUpdatePolicy: Automatic` です。

{{<hover label="compRev" line="6">}}defaultCompositionUpdatePolicy: Manual{{</hover>}} 
を設定して、この XRD を使用するコンポジットリソースとクレームのデフォルトの Composition 更新ポリシーを設定します。

```yaml {label="compRev",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  defaultCompositionUpdatePolicy: Manual
  group: custom-api.example.org
  names:
  # Removed for brevity
  versions:
  # Removed for brevity
```

<!-- vale off -->
#### enforcedCompositionRef
<!-- vale on -->
特定のCompositionを使用するようにすべての複合リソースまたはClaimを要求するには、
XRDの`enforcedCompositionRef`設定を使用します。

たとえば、このXRDを使用するすべての複合リソースとClaimが
Composition 
{{<hover label="enforceComp" line="6">}}myComposition{{</hover>}} 
を使用するように要求するには、 
{{<hover label="enforceComp" line="6">}}enforcedCompositionRef.name: myComposition{{</hover>}}を設定します。

```yaml {label="defaultComp",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.custom-api.example.org
spec:
  enforcedCompositionRef: 
    name: myComposition
  group: custom-api.example.org
  names:
  # Removed for brevity
  versions:
  # Removed for brevity
```

## CompositeResourceDefinitionの検証

`kubectl get compositeresourcedefinition`または短縮形の 
{{<hover label="getxrd" line="1">}}kubectl get xrd{{</hover>}}を使用してXRDを検証します。

```yaml {label="getxrd",copy-lines="1"}
kubectl get xrd                                
NAME                                ESTABLISHED   OFFERED   AGE
xdatabases.custom-api.example.org   True          True      22m
```

`ESTABLISHED`フィールドは、CrossplaneがこのXRDのためにKubernetesカスタム
リソース定義をインストールしたことを示します。

`OFFERED`フィールドは、このXRDがClaimを提供し、Crossplaneが
ClaimのためにKubernetesカスタムリソース定義をインストールしたことを示します。

### XRDの条件
CrossplaneはXRDのために標準の`Conditions`セットを使用します。  
`kubectl describe xrd`を使用して、XRDの`Status`の下にある条件を表示します。

```yaml {copy-lines="none"}
kubectl describe xrd
Name:         xpostgresqlinstances.database.starter.org
API Version:  apiextensions.crossplane.io/v1
Kind:         CompositeResourceDefinition
# Removed for brevity
Status:
  Conditions:
    Reason:                WatchingCompositeResource
    Status:                True
    Type:                  Established
    Reason:                WatchingCompositeResourceClaim
    Status:                True
    Type:                  Offered
# Removed for brevity
```

<!-- vale off -->
#### WatchingCompositeResource
<!-- vale on -->
`Reason: WatchingCompositeResource`は、Crossplaneが複合リソースに関連する新しい
Kubernetesカスタムリソース定義を定義し、新しい複合リソースの作成を監視していることを示します。

```yaml
Type: Established
Status: True
Reason: WatchingCompositeResource
```

<!-- vale off -->
#### TerminatingCompositeResource
<!-- vale on -->
`Reason: TerminatingCompositeResource`は、Crossplaneが複合リソースに関連する
カスタムリソース定義を削除しており、複合リソースコントローラーを終了していることを示します。

```yaml
Type: Established
Status: False
Reason: TerminatingCompositeResource
```

<!-- vale off -->
#### WatchingCompositeResourceClaim
<!-- vale on -->
`Reason: WatchingCompositeResourceClaim` は、Crossplane が提供されたクレームに関連する新しい
Kubernetes カスタムリソース定義を定義し、新しいクレームの作成を監視していることを示します。

```yaml
Type: Offered
Status: True
Reason: WatchingCompositeResourceClaim
```

<!-- vale off -->
#### TerminatingCompositeResourceClaim
<!-- vale on -->
`Reason: TerminatingCompositeResourceClaim` は、Crossplane が提供されたクレームに関連する
カスタムリソース定義を削除しており、クレームコントローラーを終了していることを示します。

```yaml
Type: Offered
Status: False
Reason: TerminatingCompositeResourceClaim
```
