---
title: パッチと変換
weight: 70
description: "Crossplaneのコンポジションは、管理リソースを作成する前に、クレームや複合リソースからの入力を変更するためにパッチと変換を使用します"
---

Crossplaneのコンポジションは「パッチと変換」操作を可能にします。パッチを使用すると、
コンポジションはコンポジションによって定義されたリソースに変更を適用できます。

ユーザーがクレームを作成すると、Crossplaneはクレーム内の設定を
関連する複合リソースに渡します。パッチはこれらの設定を使用して
関連する複合リソースや管理リソースを変更できます。

パッチと変換の使用例には以下が含まれます：
 * 外部リソースの名前を変更する
 * 「東」や「西」といった一般的な用語を特定のプロバイダーの場所にマッピングする
 * リソースフィールドにカスタムラベルや文字列を追加する


{{<hint "note" >}}
<!-- vale alex.Condescending = NO -->
Crossplaneはパッチと変換操作が単純な変更であることを期待しています。  
より複雑またはプログラム的な変更には、[Composition Functions]({{<ref "./composition-functions">}})を使用してください。
<!-- vale  alex.Condescending = YES -->
{{</hint >}}


コンポジションの[パッチ](#create-a-patch)はフィールドを変更するアクションです。  
コンポジションの[変換](#transform-a-patch)は、パッチを適用する前に
値を修正します。

## パッチを作成する

パッチは個々の 
{{<hover label="createComp" line="4">}}リソース{{</hover>}}の一部であり、
{{<hover label="createComp" line="2">}}コンポジション{{</hover>}}内にあります。

{{<hover label="createComp" line="8">}}patches{{</hover>}}フィールドは、
個々のリソースに適用するパッチのリストを取ります。

各パッチには{{<hover label="createComp" line="9">}}type{{</hover>}}があり、
これはCrossplaneが適用するパッチアクションの種類を定義します。

パッチは、パッチタイプに応じて複合リソースまたはコンポジション内のフィールドを
異なる方法で参照しますが、すべてのパッチは
{{<hover label="createComp" line="10">}}fromFieldPath{{</hover>}}と
{{<hover label="createComp" line="11">}}toFieldPath{{</hover>}}を参照します。

{{<hover label="createComp" line="10">}}fromFieldPath{{</hover>}}は
パッチの入力値を定義します。 
{{<hover label="createComp" line="11">}}toFieldPath{{</hover>}}は
パッチで変更するデータを定義します。

ここに、Composition内のリソースに適用されたパッチの例があります。  
```yaml {label="createComp",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - name: my-composed-resource
      base:
        # Removed for brevity
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.field1
          toFieldPath: metadata.labels["patchLabel"]
```

### フィールドの選択

Crossplaneは、合成リソースまたは管理リソース内のフィールドを
[JSONPathセレクタ](https://kubernetes.io/docs/reference/kubectl/jsonpath/)の
サブセットを使用して選択します。
これを「フィールドセレクタ」と呼びます。

フィールドセレクタは、合成リソースまたは管理リソースオブジェクト内の
任意のフィールドを選択できます。これには、`metadata`、`spec`、または`status`フィールドが含まれます。

フィールドセレクタは、フィールド名または配列インデックスを
ブラケット内で指定する文字列であることができます。フィールド名は、子要素を選択するために`。`文字を使用できます。

#### フィールドセレクタの例
合成リソースオブジェクトからのいくつかの例のセレクタを示します。  
{{<table "table" >}}  
| セレクタ | 選択された要素 |  
| --- | --- |  
| `kind` | {{<hover label="select" line="3">}}kind{{</hover>}} |  
| `metadata.labels['crossplane.io/claim-name']` | {{<hover label="select" line="7">}}my-example-claim{{</hover>}} |  
| `spec.desiredRegion` | {{<hover label="select" line="11">}}eu-north-1{{</hover>}} |  
| `spec.resourceRefs[0].name` | {{<hover label="select" line="16">}}my-example-claim-978mh-r6z64{{</hover>}} |  
{{</table >}}  

```yaml {label="select",copy-lines="none"}
$ kubectl get composite -o yaml
apiVersion: example.org/v1alpha1
kind: xExample
metadata:
  # Removed for brevity
  labels:
    crossplane.io/claim-name: my-example-claim
    crossplane.io/claim-namespace: default
    crossplane.io/composite: my-example-claim-978mh
spec:
  desiredRegion: eu-north-1
  field1: field1-text
  resourceRefs:
  - apiVersion: s3.aws.upbound.io/v1beta1
    kind: Bucket
    name: my-example-claim-978mh-r6z64
  - apiVersion: s3.aws.upbound.io/v1beta1
    kind: Bucket
    name: my-example-claim-978mh-cnlhj
  - apiVersion: s3.aws.upbound.io/v1beta1
    kind: Bucket
    name: my-example-claim-978mh-rv5nm
  # Removed for brevity
```

## パッチの再利用

Compositionは、複数のリソースに対してパッチオブジェクトを再利用することができます。
これをPatchSetと呼びます。

PatchSetを作成するには、Compositionの
{{<hover label="patchset" line="5">}}PatchSets{{</hover>}}オブジェクトを定義します。  

PatchSet内の各パッチには、  
{{<hover label="patchset" line="6">}}name{{</hover>}}と
{{<hover label="patchset" line="7">}}patches{{</hover>}}のリストがあります。  

{{<hint "note" >}}  
複数のPatchSetsを使用する場合は、単一の  
{{<hover label="patchset" line="5">}}PatchSets{{</hover>}}オブジェクトのみを使用してください。  

各ユニークなPatchSetをユニークな  
{{<hover label="patchset" line="6">}}name{{</hover>}}で識別します。  
{{</hint >}}  

PatchSetをリソースに適用するには、パッチ  
{{<hover label="patchset" line="16">}}type: PatchSet{{</hover>}}を使用します。  
{{<hover label="patchset" line="17">}}patchSetName{{</hover>}}をPatchSetの  
{{<hover label="patchset" line="6">}}name{{</hover>}}に設定します。

```yaml {label="patchset"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
spec:
  patchSets:
  - name: my-patchset
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.desiredRegion
      toFieldPath: spec.forProvider.region
  resources:
    - name: bucket1
      base:
        # Removed for brevity
      patches:
        - type: PatchSet
          patchSetName: my-patchset
    - name: bucket2
      base:
        # Removed for brevity
      patches:
        - type: PatchSet
          patchSetName: my-patchset  
```

{{<hint "important" >}}
PatchSetには他のPatchSetを含めることはできません。  

CrossplaneはPatchSet内の[transform](#transform-a-patch)や
[policies](#patch-policies)を無視します。
{{< /hint >}}

## リソース間のパッチ適用

Composition内のリソース間で直接パッチを適用することはできません。  
例えば、ネットワークリソースを生成し、そのリソース名を
コンピュートリソースにパッチ適用することです。

{{<hint "important">}}
[ToEnvironmentFieldPath](#toenvironmentfieldpath)パッチは
`Status`フィールドから読み取ることができません。
{{< /hint >}}

リソースは、合成リソース内のユーザー定義の
{{<hover label="xrdPatch" line="13">}}Status{{</hover>}}
フィールドにパッチを適用できます。

リソースはその
{{<hover label="xrdPatch" line="13">}}Status{{</hover>}} 
フィールドから読み取ってフィールドにパッチを適用できます。

まず、Composite Resource Definitionでカスタム
{{<hover label="xrdPatch" line="13">}}Status{{</hover>}}
とカスタムフィールドを定義します。例えば
{{<hover label="xrdPatch" line="16">}}secondResource{{</hover>}}

```yaml {label="xrdPatch",copy-lines="13-17"}
kind: CompositeResourceDefinition
# Removed for brevity.
spec:
  # Removed for brevity.
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            # Removed for brevity.
          status:
              type: object
              properties:
                secondResource:
                  type: string
```

Composition内で、ソースデータを持つリソースは
{{<hover label="patchBetween" line="10">}}ToCompositeFieldPath{{</hover>}}
パッチを使用して、合成リソース内の
{{<hover label="patchBetween" line="12">}}status.secondResource{{</hover>}} 
フィールドにデータを書き込みます。

宛先リソースは
{{<hover label="patchBetween" line="19">}}FromCompositeFieldPath{{</hover>}}
パッチを使用して、合成リソースの
{{<hover label="patchBetween" line="20">}}status.secondResource{{</hover>}} 
フィールドからデータを読み取り、それを
管理リソース内の{{<hover label="patchBetween" line="21">}}secondResource{{</hover>}}というラベルに書き込みます。

```yaml {label="patchBetween",copy-lines="9-11"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        # Removed for brevity
      patches:
        - type: ToCompositeFieldPath
          fromFieldPath: metadata.name
          toFieldPath: status.secondResource
    - name: bucket2
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        # Removed for brevity
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: status.secondResource
          toFieldPath: metadata.labels['secondResource']
```

合成リソースを記述して、 
{{<hover label="descCompPatch" line="5">}}resources{{</hover>}}と
{{<hover label="descCompPatch" line="11">}}status.secondResource{{</hover>}}
の値を表示します。

```yaml {label="descCompPatch",copy-lines="none"}
$ kubectl describe composite
Name:         my-example-claim-jp7rx
Spec:
  # Removed for brevity
  Resource Refs:
    Name:         my-example-claim-jp7rx-gfg4m
    # Removed for brevity
    Name:         my-example-claim-jp7rx-fttpj
Status:
  # Removed for brevity
  Second Resource:         my-example-claim-jp7rx-gfg4m
```

宛先の管理リソースを説明してラベルを確認します 
{{<hover label="bucketlabel" line="5">}}secondResource{{</hover>}}。

```yaml {label="bucketlabel",copy-lines="none"}
$ kubectl describe bucket
kubectl describe bucket my-example-claim-jp7rx-fttpj
Name:         my-example-claim-jp7rx-fttpj
Labels:       crossplane.io/composite=my-example-claim-jp7rx
              secondResource=my-example-claim-jp7rx-gfg4m
```

## パッチの種類
Crossplaneは複数のパッチタイプをサポートしており、それぞれ異なるデータソースを使用し、異なる場所にパッチを適用します。

{{<hint "important" >}}

このセクションでは、Composition内の個々のリソースに適用されるパッチについて説明します。

Compositionの`environment.patches`を使用して、全体の複合リソースにパッチを適用する方法については、 
[環境設定]({{<ref "environment-configs" >}})のドキュメントを参照してください。

{{< /hint >}}

Crossplaneパッチの概要
{{< table "table table-hover" >}}
| パッチタイプ | データソース | データ宛先 | 
| ---  | --- | --- | 
| [FromCompositeFieldPath](#fromcompositefieldpath) | 複合リソース内のフィールド。 | パッチを適用された管理リソース内のフィールド。 | 
| [ToCompositeFieldPath](#tocompositefieldpath) | パッチを適用された管理リソース内のフィールド。 | 複合リソース内のフィールド。 |  
| [CombineFromComposite](#combinefromcomposite) | 複合リソース内の複数のフィールド。 | パッチを適用された管理リソース内のフィールド。 | 
| [CombineToComposite](#combinetocomposite) | パッチを適用された管理リソース内の複数のフィールド。 | 複合リソース内のフィールド。 | 
| [FromEnvironmentFieldPath](#fromenvironmentfieldpath) | メモリ内のEnvironmentConfig環境のデータ | パッチを適用された管理リソース内のフィールド。 | 
| [ToEnvironmentFieldPath](#toenvironmentfieldpath) | パッチを適用された管理リソース内のフィールド。 | メモリ内のEnvironmentConfig環境。 | 
| [CombineFromEnvironment](#combinefromenvironment) | メモリ内のEnvironmentConfig環境の複数のフィールド。 | パッチを適用された管理リソース内のフィールド。 | 
| [CombineToEnvironment](#combinetoenvironment) | パッチを適用された管理リソース内の複数のフィールド。 | メモリ内のEnvironmentConfig環境のフィールド。 | 
{{< /table >}}


{{<hint "note" >}}
以下のすべての例は、同じセットのコンポジション、 
CompositeResourceDefinitions、Claims、およびEnvironmentConfigsを使用しています。  
例の間で変更されるのは適用されたパッチのみです。 

すべての例は、Upboundの
[provider-aws-s3](https://marketplace.upbound.io/providers/upbound/provider-aws-s3/)
を使用してリソースを作成します。

{{< expand "Reference Composition" >}}
```yaml {copy-lines="all"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  compositeTypeRef:
    apiVersion: example.org/v1alpha1
    kind: xExample
  environment:
    environmentConfigs:
    - ref:
        name: example-environment
  resources:
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
    - name: bucket2
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
```
{{< /expand >}}

{{<expand "Reference CompositeResourceDefinition" >}}
```yaml {copy-lines="all"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xexamples.example.org
spec:
  group: example.org
  names:
    kind: xExample
    plural: xexamples
  claimNames:
    kind: ExampleClaim
    plural: exampleclaims
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
              field1:
                type: string
              field2:
                type: string
              field3: 
                type: string
              desiredRegion: 
                type: string
              boolField:
                type: boolean
              numberField:
                type: integer
          status:
              type: object
              properties:
                url:
                  type: string
```
{{< /expand >}}


{{< expand "Reference Claim" >}}
```yaml {copy-lines="all"}
apiVersion: example.org/v1alpha1
kind: ExampleClaim
metadata:
  name: my-example-claim
spec:
  field1: "field1-text"
  field2: "field2-text"
  desiredRegion: "eu-north-1"
  boolField: false
  numberField: 10
```
{{< /expand >}}

{{< expand "Reference EnvironmentConfig" >}}
```yaml {copy-lines="all"}
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: EnvironmentConfig
metadata:
  name: example-environment
data:
  locations:
    us: us-east-2
    eu: eu-north-1
  key1: value1
  key2: value2

```
{{< /expand >}}
{{< /hint >}}

<!-- vale Google.Headings = NO -->
### FromCompositeFieldPath
<!-- vale Google.Headings = YES -->

{{<hover label="fromComposite" line="12">}}FromCompositeFieldPath{{</hover>}}
パッチは、コンポジットリソース内の値を取得し、それを
管理リソースのフィールドに適用します。 

{{< hint "tip" >}}
{{<hover label="fromComposite" line="12">}}FromCompositeFieldPath{{</hover>}}
パッチを使用して、ユーザーのClaimsから管理リソースの
`forProvider`設定にオプションを適用します。 
{{< /hint >}}

例えば、ユーザーがコンポジットリソースで提供した値
{{<hover label="fromComposite" line="13">}}desiredRegion{{</hover>}}を
管理リソースの
{{<hover label="fromComposite" line="10">}}region{{</hover>}}に使用します。 

{{<hover label="fromComposite" line="13">}}fromFieldPath{{</hover>}}の値は
コンポジットリソース内のフィールドです。 

{{<hover label="fromComposite" line="14">}}toFieldPath{{</hover>}}の値は
変更する管理リソースのフィールドです。 

```yaml {label="fromComposite",copy-lines="9-11"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.desiredRegion
          toFieldPath: spec.forProvider.region
```

管理リソースを表示して、更新された
{{<hover label="fromCompMR" line="6">}}region{{</hover>}}を確認します。

```yaml {label="fromCompMR",copy-lines="1"}
$ kubectl describe bucket
Name:         my-example-claim-qlr68-29nqf
# Removed for brevity
Spec:
  For Provider:
    Region:  eu-north-1
```

<!-- vale Google.Headings = NO -->
### ToCompositeFieldPath
<!-- vale Google.Headings = YES -->

{{<hover label="toComposite" line="12">}}ToCompositeFieldPath{{</hover>}} は、個々の管理リソースからデータを取得し、それを作成した複合リソースに書き込みます。

{{< hint "tip" >}}
{{<hover label="toComposite" line="12">}}ToCompositeFieldPath{{</hover>}} パッチを使用して、Composition内の1つの管理リソースからデータを取得し、同じComposition内の2つ目の管理リソースで使用します。
{{< /hint >}}

例えば、Crossplaneが新しい管理リソースを作成した後、値 {{<hover label="toComposite" line="13">}}hostedZoneID{{</hover>}} を取得し、それを複合リソースの {{<hover label="toComposite" line="14">}}label{{</hover>}} として適用します。

```yaml {label="toComposite",copy-lines="9-11"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.hostedZoneId
          toFieldPath: metadata.labels['ZoneID']
```

作成された管理リソースを表示して、 {{<hover label="toCompMR" line="6">}}Hosted Zone Id{{</hover>}} フィールドを確認します。
```yaml {label="toCompMR",copy-lines="none"}
$ kubectl describe bucket
Name:         my-example-claim-p5pxf-5vnp8
# Removed for brevity
Status:
  At Provider:
    Hosted Zone Id:       Z2O1EMRO9K5GLX
    # Removed for brevity
```

次に、複合リソースを表示し、パッチが {{<hover label="toCompositeXR" line="3">}}label{{</hover>}} に適用されたことを確認します。
```yaml {label="toCompositeXR",copy-lines="none"}
$ kubectl describe composite
Name:         my-example-claim-p5pxf
Labels:       ZoneID=Z2O1EMRO9K5GLX
```

{{<hint "important">}}
Crossplaneは、管理リソースを作成した後、次のリコンシリエーションループまで複合リソースにパッチを適用しません。これにより、管理リソースがReadyになるのとパッチが適用されるのとの間に遅延が生じます。
{{< /hint >}}


<!-- vale Google.Headings = NO -->
### CombineFromComposite
<!-- vale Google.Headings = YES -->

{{<hover label="combineFromComp" line="12">}}CombineFromComposite{{</hover>}} パッチは、複合リソースから値を取得し、それらを組み合わせて管理リソースに適用します。

{{< hint "tip" >}}
{{<hover label="combineFromComp" line="12">}}CombineFromComposite{{</hover>}} パッチを使用して、セキュリティポリシーのような複雑な文字列を作成し、それを管理リソースに適用します。
{{< /hint >}}

例えば、Claim値を使用します 
{{<hover label="combineFromComp" line="15">}}desiredRegion{{</hover>}} と 
{{<hover label="combineFromComp" line="16">}}field2{{</hover>}} を使用して、管理リソースの
{{<hover label="combineFromComp" line="20">}}name{{</hover>}} を生成します。

{{<hover label="combineFromComp" line="12">}}CombineFromComposite{{</hover>}}
パッチは、{{<hover label="combineFromComp" line="13">}}combine{{</hover>}} オプションのみをサポートしています。

{{<hover label="combineFromComp" line="14">}}variables{{</hover>}} は、結合するための
コンポジットリソースからの 
{{<hover label="combineFromComp" line="15">}}fromFieldPath{{</hover>}} 値のリストです。

サポートされている唯一の 
{{<hover label="combineFromComp" line="17">}}strategy{{</hover>}} は 
{{<hover label="combineFromComp" line="17">}}strategy: string{{</hover>}} です。

オプションで、文字列を結合する方法を指定するために、 
[Go文字列フォーマット](https://pkg.go.dev/fmt) に基づいて 
{{<hover label="combineFromComp" line="19">}}string.fmt{{</hover>}} を適用できます。

{{<hover label="combineFromComp" line="20">}}toFieldPath{{</hover>}} は、管理リソースに新しい文字列を適用するフィールドです。

```yaml {label="combineFromComp",copy-lines="11-20"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: CombineFromComposite
          combine:
            variables:
              - fromFieldPath: spec.desiredRegion
              - fromFieldPath: spec.field2
            strategy: string
            string:
              fmt: "my-resource-%s-%s"
          toFieldPath: metadata.name
```

適用されたパッチを確認するために管理リソースを記述します。

```yaml {label="describeCombineFromComp",copy-lines="none"}
$ kubectl describe bucket
Name:         my-resource-eu-north-1-field2-text
```

<!-- vale Google.Headings = NO -->
### CombineToComposite
<!-- vale Google.Headings = YES -->

{{<hover label="combineToComposite" line="12">}}CombineToComposite{{</hover>}}
パッチは、管理リソースから値を取得し、それらを結合してコンポジットリソースに適用します。

{{<hint "tip" >}}
{{<hover label="combineToComposite" line="12">}}CombineToComposite{{</hover>}} 
パッチを使用して、管理リソース内の複数のフィールドからURLのような単一のフィールドを作成します。 
{{< /hint >}}

例えば、管理リソースの 
{{<hover label="combineToComposite" line="15">}}name{{</hover>}} と 
{{<hover label="combineToComposite" line="16">}}region{{</hover>}} を使用して、カスタム 
{{<hover label="combineToComposite" line="20">}}url{{</hover>}} フィールドを生成します。

```markdown
{{< hint "重要" >}}
合成リソースのステータスフィールドにカスタムフィールドを書くには、
最初にCompositeResourceDefinitionでカスタムフィールドを定義する必要があります。 

{{< /hint >}}

{{<hover label="combineToComposite" line="12">}}CombineToComposite{{</hover>}}
パッチはのみ
{{<hover label="combineToComposite" line="13">}}combine{{</hover>}}オプションをサポートしています。 

{{<hover label="combineToComposite" line="14">}}variables{{</hover>}}は
結合する管理リソースの
{{<hover label="combineToComposite" line="15">}}fromFieldPath{{</hover>}}のリストです。 

サポートされている唯一の
{{<hover label="combineToComposite" line="17">}}strategy{{</hover>}}は
{{<hover label="combineToComposite" line="17">}}strategy: string{{</hover>}}です。

オプションで、文字列を結合する方法を指定するために
[Go string formatting](https://pkg.go.dev/fmt)に基づいて
{{<hover label="combineToComposite" line="19">}}string.fmt{{</hover>}}を適用できます。

{{<hover label="combineToComposite" line="20">}}toFieldPath{{</hover>}}は
新しい文字列を適用する合成リソースのフィールドです。 

```yaml {label="combineToComposite",copy-lines="9-11"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: CombineToComposite
          combine:
            variables:
              - fromFieldPath: metadata.name
              - fromFieldPath: spec.forProvider.region
            strategy: string
            string:
              fmt: "https://%s.%s.com"
          toFieldPath: status.url
```

適用されたパッチを確認するために合成リソースを表示します。

```yaml {copy-lines="none"}
$ kubectl describe composite
Name:         my-example-claim-bjdjw
API Version:  example.org/v1alpha1
Kind:         xExample
# Removed for brevity
Status:
  # Removed for brevity
  URL:                     https://my-example-claim-bjdjw-r6ncd.us-east-2.com
```

<!-- vale Google.Headings = NO -->
### FromEnvironmentFieldPath
<!-- vale Google.Headings = YES -->

{{<hint "重要" >}}
EnvironmentConfigsはアルファ機能です。デフォルトでは有効になっていません。  

EnvironmentConfigの使用に関する詳細は、
[EnvironmentConfigs]({{<ref "./environment-configs">}})ドキュメントをお読みください。
{{< /hint >}}

{{<hover label="fromEnvField" line="12">}}FromEnvironmentFieldPath{{</hover>}}
パッチはメモリ内のEnvironmentConfig環境から値を取得し、
それらを管理リソースに適用します。

{{<hint "ヒント" >}}
現在の環境に基づいてカスタム管理リソース設定を適用するには、
{{<hover label="fromEnvField" line="12">}}FromEnvironmentFieldPath{{</hover>}}を使用してください。  
{{< /hint >}}

例えば、環境の
{{<hover label="fromEnvField" line="13">}}locations.eu{{</hover>}}の値を使用し、
それを
{{<hover label="fromEnvField" line="14">}}region{{</hover>}}として適用します。
```

```yaml {label="fromEnvField",copy-lines="9-11"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
        patches:
        - type: FromEnvironmentFieldPath
          fromFieldPath: locations.eu
          toFieldPath: spec.forProvider.region
```

適用されたパッチを確認するために、管理リソースを検証します。 

```yaml {copy-lines="none"}
kubectl describe bucket
Name:         my-example-claim-8vrvc-xx5sr
Labels:       crossplane.io/claim-name=my-example-claim
# Removed for brevity
Spec:
  For Provider:
    Region:  eu-north-1
  # Removed for brevity
```

<!-- vale Google.Headings = NO -->
### ToEnvironmentFieldPath
<!-- vale Google.Headings = YES -->

{{<hint "important" >}}
EnvironmentConfigsはアルファ機能です。デフォルトでは有効になっていません。  

EnvironmentConfigの使用に関する詳細は、 
[EnvironmentConfigs]({{<ref "./environment-configs">}}) ドキュメントを参照してください。
{{< /hint >}}

{{<hover label="toEnvField" line="12">}}ToEnvironmentFieldPath{{</hover>}}
パッチは、管理リソースから値を取得し、それをメモリ内の 
EnvironmentConfig環境に適用します。

{{<hint "tip" >}}
{{<hover label="toEnvField" line="12">}}ToEnvironmentFieldPath{{</hover>}}
を使用して、任意のFromEnvironmentFieldPath
パッチがアクセスできる環境にデータを書き込みます。 
{{< /hint >}}

例えば、希望する
{{<hover label="toEnvField" line="13">}}region{{</hover>}} 値を使用し、
それを環境の
{{<hover label="toEnvField" line="14">}}key1{{</hover>}}として適用します。


```yaml {label="toEnvField",copy-lines="9-11"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
        patches:
        - type: ToEnvironmentFieldPath
          fromFieldPath: spec.forProvider.region
          toFieldPath: key1
```

環境はメモリ内にあるため、パッチが値を環境に書き込んだことを確認するコマンドはありません。


<!-- vale Google.Headings = NO -->
### CombineFromEnvironment
<!-- vale Google.Headings = YES -->

{{<hint "important" >}}
EnvironmentConfigsはアルファ機能です。デフォルトでは有効になっていません。  

EnvironmentConfigの使用に関する詳細は、 
[EnvironmentConfigs]({{<ref "./environment-configs">}}) ドキュメントを参照してください。
{{< /hint >}}

{{<hover label="combineFromEnv" line="12">}}CombineFromEnvironment{{</hover>}}
パッチは、メモリ内のEnvironmentConfig環境から複数の値を結合し、
それを管理リソースに適用します。

{{<hint "tip" >}}
{{<hover label="combineFromEnv" line="12">}}CombineFromEnvironment{{</hover>}}
パッチを使用して、セキュリティポリシーのような複雑な文字列を作成し、
それを管理リソースに適用します。 
{{< /hint >}}

例えば、環境内の複数のフィールドを組み合わせてユニークな 
{{<hover label="combineFromEnv" line="20">}}アノテーション{{</hover>}}
を作成します。 

{{<hover label="combineFromEnv" line="12">}}CombineFromEnvironment{{</hover>}}
パッチは 
{{<hover label="combineFromEnv" line="13">}}combine{{</hover>}} オプションのみをサポートしています。 

サポートされている唯一の 
{{<hover label="combineFromEnv" line="14">}}戦略{{</hover>}}は 
{{<hover label="combineFromEnv" line="14">}}strategy: string{{</hover>}} です。

{{<hover label="combineFromEnv" line="15">}}変数{{</hover>}}は、結合するための
メモリ内環境からの 
{{<hover label="combineFromEnv" line="16">}}fromFieldPath{{</hover>}} 値のリストです。 

オプションで、 
{{<hover label="combineFromEnv" line="19">}}string.fmt{{</hover>}} を適用して、 
[Goの文字列フォーマット](https://pkg.go.dev/fmt) に基づいて文字列を結合する方法を指定できます。

{{<hover label="combineFromEnv" line="20">}}toFieldPath{{</hover>}} は、 
新しい文字列を適用するための管理リソース内のフィールドです。 


```yaml {label="combineFromEnv",copy-lines="11-20"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: CombineFromEnvironment
          combine:
            strategy: string
            variables:
            - fromFieldPath: key1
            - fromFieldPath: key2
            string: 
              fmt: "%s-%s"
          toFieldPath: metadata.annotations[EnvironmentPatch]
```

管理リソースを記述して、新しい 
{{<hover label="combineFromEnvDesc" line="4">}}アノテーション{{</hover>}} を確認します。

```yaml {copy-lines="none",label="combineFromEnvDesc"}
$ kubectl describe bucket
Name:         my-example-claim-zmxdg-grl6p
# Removed for brevity
Annotations:  EnvironmentPatch: value1-value2
# Removed for brevity
```

<!-- vale Google.Headings = NO -->
### CombineToEnvironment
<!-- vale Google.Headings = YES -->

{{<hint "important" >}}
EnvironmentConfigs はアルファ機能です。デフォルトでは有効になっていません。  

EnvironmentConfig の使用に関する詳細は、 
[EnvironmentConfigs]({{<ref "./environment-configs">}}) ドキュメントを参照してください。
{{< /hint >}}

{{<hover label="combineToEnv" line="12">}}CombineToEnvironment{{</hover>}}
パッチは、管理リソースからの複数の値を結合し、それらをメモリ内の EnvironmentConfig 環境に適用します。

{{<hint "tip" >}}
{{<hover label="combineToEnv" line="12">}}CombineToEnvironment{{</hover>}}
パッチを使用して、他の管理リソースで使用するセキュリティポリシーのような複雑な文字列を作成します。 
{{< /hint >}}

例えば、管理リソース内の複数のフィールドを組み合わせて一意の
文字列を作成し、それを環境の
{{<hover label="combineToEnv" line="20">}}key2{{</hover>}} 値に保存します。 

この文字列は
管理リソースの 
{{<hover label="combineToEnv" line="16">}}Kind{{</hover>}} と 
{{<hover label="combineToEnv" line="17">}}region{{</hover>}} を組み合わせたものです。

{{<hover label="combineToEnv" line="12">}}CombineToEnvironment{{</hover>}}
パッチは 
{{<hover label="combineToEnv" line="13">}}combine{{</hover>}} オプションのみをサポートします。 

サポートされている唯一の 
{{<hover label="combineToEnv" line="14">}}strategy{{</hover>}} は 
{{<hover label="combineToEnv" line="14">}}strategy: string{{</hover>}} です。

{{<hover label="combineToEnv" line="15">}}variables{{</hover>}} は
管理リソース内の 
{{<hover label="combineToEnv" line="16">}}fromFieldPath{{</hover>}} 
値のリストです。 

オプションで、 
{{<hover label="combineToEnv" line="19">}}string.fmt{{</hover>}} を適用して、 
[Goの文字列フォーマット](https://pkg.go.dev/fmt) に基づいて文字列を組み合わせる方法を指定できます。

{{<hover label="combineToEnv" line="20">}}toFieldPath{{</hover>}} は
新しい文字列を書き込むための環境内のキーです。 

```yaml {label="combineToEnv",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: CombineToEnvironment
          combine:
            strategy: string
            variables:
            - fromFieldPath: kind
            - fromFieldPath: spec.forProvider.region
            string:
              fmt: "%s.%s"
          toFieldPath: key2
```

環境はメモリ内にあるため、パッチが値を環境に書き込んだことを確認するコマンドはありません。

## パッチの変換

パッチを適用する際、Crossplaneはデータをパッチとして適用する前に変更することをサポートしています。Crossplaneはこれを「変換」操作と呼びます。 

Crossplaneの変換の概要。
{{< table "table table-hover" >}}
| 変換タイプ | アクション |
| ---  | --- |
| [convert](#convert-transforms) | 入力データ型を別の型に変換します。「キャスティング」とも呼ばれます。 | 
| [map](#map-transforms) | 特定の入力に基づいて特定の出力を選択します。 | 
| [match](#match-transform) | 文字列または正規表現に基づいて特定の出力を選択します。 | 
| [math](#math-transforms) | 入力に対して数学的操作を適用します。 | 
| [string](#string-transforms) | [Goの文字列フォーマット](https://pkg.go.dev/fmt) を使用して入力文字列を変更します。 | 
{{< /table >}}


個々のパッチに直接変換を適用するには、 
{{<hover label="transform1" line="15">}}transforms{{</hover>}} フィールドを使用します。 

{{<hover label="transform1" line="15">}}transform{{</hover>}} 
には、実行する変換アクションを示す 
{{<hover label="transform1" line="16">}}type{{</hover>}} が必要です。 

他の変換フィールドは、 
{{<hover label="transform1" line="16">}}type{{</hover>}} と同じで、この例では 
{{<hover label="transform1" line="17">}}map{{</hover>}} です。

他のフィールドは、使用されるパッチタイプによって異なります。 

この例では、 
{{<hover label="transform1" line="16">}}type: map{{</hover>}} 変換を使用し、 
入力の 
{{<hover label="transform1" line="13">}}spec.desiredRegion{{</hover>}} を取得し、 
それを 
{{<hover label="transform1" line="18">}}us{{</hover>}} または 
{{<hover label="transform1" line="19">}}eu{{</hover>}} に一致させ、 
対応する AWS リージョンを 
{{<hover label="transform1" line="14">}}spec.forProvider.region{{</hover>}} 
値として返します。 

```yaml {label="transform1",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
# Removed for brevity
    - name: bucket1
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        spec:
          forProvider:
            region: us-east-2
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.desiredRegion
          toFieldPath: spec.forProvider.region
          transforms:
            - type: map
              map:
                us: us-east-2
                eu: eu-north-1
```

### 変換の変換

{{<hover label="convert" line="6">}}convert{{</hover>}} 変換タイプは、 
入力データタイプを別のデータタイプに変更します。

{{< hint "tip" >}}
一部のプロバイダー API では、フィールドが文字列である必要があります。 
{{<hover label="convert" line="7">}}convert{{</hover>}} タイプを使用して、 
任意のブール値または整数フィールドを文字列に変更します。 
{{< /hint >}}

{{<hover label="convert" line="6">}}convert{{</hover>}} 
変換には、出力データタイプを定義する 
{{<hover label="convert" line="8">}}toType{{</hover>}} が必要です。 

```yaml {label="convert",copy-lines="none"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.numberField
    toFieldPath: metadata.label["numberToString"]
    transforms:
      - type: convert
        convert:
          toType: string
```

サポートされている `toType` 値：
{{< table "table table-sm table-hover" >}}
| `toType` 値 | 説明 | 
| -- | -- |
| `bool` | `true` または `false` のブール値。 | 
| `float64` | 64 ビット浮動小数点値。 | 
| `int` | 32 ビット整数値。 | 
| `int64` | 64 ビット整数値。 | 
| `string` | 文字列値。 | 
| `object` | オブジェクト。 |
| `array` | 配列。 |
{{< /table >}}

#### 文字列をブール値に変換する
文字列から `bool` への変換時に、Crossplane は文字列値  
`1`、`t`、`T`、`TRUE`、`True` および `true`  
をブール値 `True` と等しいと見なします。  

文字列  
`0`、`f`、`F`、`FALSE`、`False` および `false`  
はブール値 `False` と等しいです。  

#### 数値をブール値に変換する
Crossplane は整数 `1` および浮動小数点数 `1.0` をブール値
`True` と等しいと見なします。  
その他の整数または浮動小数点数の値は `False` です。  

#### ブール値を数値に変換する
Crossplane はブール値 `True` を整数 `1` または浮動小数点数 `1.0` に変換します。  

値 `False` は整数 `0` または浮動小数点数 `0.0` に変換されます。  

#### 文字列を float64 に変換する
`string` から 
{{<hover label="format" line="3">}}float64{{</hover>}} への変換時に、Crossplane は 
オプションの  
{{<hover label="format" line="4">}}format: quantity{{</hover>}} フィールドをサポートしています。  

{{<hover label="format" line="4">}}format: quantity{{</hover>}} を使用すると、  
サイズの接尾辞 `M`（メガバイト）や `Mi`（メガビット）を正しい float64
値に変換します。  

{{<hint "note" >}}
サポートされている接尾辞の完全なリストについては、[Go 言語のドキュメント](https://pkg.go.dev/k8s.io/apimachinery/pkg/api/resource#Quantity)を参照してください。
{{</hint >}}

{{<hover label="format" line="4">}}format: quantity{{</hover>}} を 
{{<hover label="format" line="1">}}convert{{</hover>}} オブジェクトに追加して、数量接尾辞のサポートを有効にします。  

```yaml {label="format",copy-lines="all"}
- type: convert
  convert:
   toType: float64
   format: quantity
```

#### 文字列をオブジェクトに変換する

Crossplane は JSON 文字列をオブジェクトに変換します。  

{{<hover label="object" line="4">}}format: json{{</hover>}} を 
{{<hover label="object" line="1">}}convert{{</hover>}} オブジェクトに追加してください。これはこの変換に対して唯一サポートされている文字列形式です。  

```yaml {label="object",copy-lines="all"}
- type: convert
  convert:
   toType: object
   format: json
```

{{< hint "tip" >}}
この変換はオブジェクト内のキーをパッチするのに便利です。
{{< /hint >}}

次の例は、{{<hover label="patch-key" line="8">}}カスタマイズされたキー{{</hover>}}を持つリソースにタグを追加します。

```yaml {label="patch-key",copy-lines="all"}
    - type: FromCompositeFieldPath
      fromFieldPath: spec.clusterName
      toFieldPath: spec.forProvider.tags
      transforms:
      - type: string
        string:
          type: Format
          fmt: '{"kubernetes.io/cluster/%s": "true"}'
      - type: convert
        convert:
          toType: object
          format: json
```

#### 文字列を配列に変換する

CrossplaneはJSON文字列を配列に変換します。

{{<hover label="array" line="4">}}format: json{{</hover>}}を
{{<hover label="array" line="1">}}convert{{</hover>}}オブジェクトに追加します。
これはこの変換に対して唯一サポートされている文字列形式です。

```yaml {label="array",copy-lines="all"}
- type: convert
  convert:
   toType: array
   format: json
```

### マップ変換
{{<hover label="map" line="6">}}map{{</hover>}}変換タイプは
入力値を出力値に_マッピング_します。

{{< hint "tip" >}}
{{<hover label="map" line="6">}}map{{</hover>}}変換は、`US`や`EU`のような一般的な地域名をプロバイダー特有の地域名に翻訳するのに便利です。
{{< /hint >}}

{{<hover label="map" line="6">}}map{{</hover>}}変換は、{{<hover label="map" line="3">}}fromFieldPath{{</hover>}}からの値を
{{<hover label="map" line="6">}}map{{</hover>}}にリストされているオプションと比較します。

Crossplaneが値を見つけると、Crossplaneは
マッピングされた値を{{<hover label="map" line="4">}}toFieldPath{{</hover>}}に置きます。

{{<hint "note" >}}
Crossplaneは、値が見つからない場合、パッチに対してエラーをスローします。
{{< /hint >}}

{{<hover label="map" line="3">}}spec.field1{{</hover>}}が文字列
{{<hover label="map" line="8">}}"field1-text"{{</hover>}}の場合、Crossplaneは
{{<hover label="map" line="8">}}firstField{{</hover>}}という文字列を
{{<hover label="map" line="4">}}annotation{{</hover>}}に使用します。

もし
{{<hover label="map" line="3">}}spec.field1{{</hover>}}が文字列
{{<hover label="map" line="8">}}"field2-text"{{</hover>}}の場合、Crossplaneは
{{<hover label="map" line="8">}}secondField{{</hover>}}という文字列を
{{<hover label="map" line="4">}}annotation{{</hover>}}に使用します。

```yaml {label="map",copy-lines="none"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["myAnnotation"]
    transforms:
      - type: map
        map:
          "field1-text": "firstField"
          "field2-text": "secondField"
```
この例では、{{<hover label="map" line="3">}}spec.field1{{</hover>}}の値は
{{<hover label="comositeMap" line="5">}}field1-text{{</hover>}}です。

```yaml {label="comositeMap",copy-lines="none"}
$ kubectl describe composite
Name:         my-example-claim-twx7n
Spec:
  # Removed for brevity
  field1:         field1-text
```

管理リソースに適用されるアノテーションは 
{{<hover label="mrMap" line="4">}}firstField{{</hover>}}です。

```yaml {label="mrMap",copy-lines="none"}
$ kubectl describe bucket
Name:         my-example-claim-twx7n-ndb2f
Annotations:  crossplane.io/composition-resource-name: bucket1
              myAnnotation: firstField
# Removed for brevity.
```

### マッチ変換
{{<hover label="match" line="6">}}match{{</hover>}} 変換は 
`map` 変換のようなものです。  

{{<hover label="match" line="6">}}match{{</hover>}} 
変換は、正確な文字列に加えて正規表現をサポートし、一致しない場合にはデフォルト値を提供できます。

{{<hover label="match" line="7">}}match{{</hover>}} オブジェクトには 
{{<hover label="match" line="8">}}patterns{{</hover>}} オブジェクトが必要です。

{{<hover label="match" line="8">}}patterns{{</hover>}} は、入力値に対して一致を試みる1つ以上のパターンのリストです。

```yaml {label="match",copy-lines="1-8"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["myAnnotation"]
    transforms:
      - type: match
        match:
          patterns:
            - type: literal
              # Removed for brevity
            - type: regexp
              # Removed for brevity
```

マッチ {{<hover label="match" line="8">}}patterns{{</hover>}} は、 
{{<hover label="match" line="9">}}type: literal{{</hover>}} を使用して
正確な文字列と一致させるか、 
{{<hover label="match" line="11">}}type: regexp{{</hover>}} を使用して
正規表現と一致させることができます。

{{<hint "note" >}}
Crossplane は最初のパターン一致の後にマッチ処理を停止します。
{{< /hint >}}

#### 正確な文字列と一致させる
{{<hover label="matchLiteral" line="8">}}pattern{{</hover>}} を 
{{<hover label="matchLiteral" line="9">}}type: literal{{</hover>}} とともに使用して
正確な文字列と一致させます。

一致が成功すると、Crossplane は 
{{<hover label="matchLiteral" line="11">}}result:{{</hover>}} を
パッチ {{<hover label="matchLiteral" line="4">}}toFieldPath{{</hover>}} に提供します。

```yaml {label="matchLiteral"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["myAnnotation"]
    transforms:
      - type: match
        match:
          patterns:
            - type: literal
              literal: "field1-text"
              result: "matchedLiteral"
```

#### 正規表現と一致させる
{{<hover label="matchRegex" line="8">}}pattern{{</hover>}} を 
{{<hover label="matchRegex" line="9">}}type: regexp{{</hover>}} とともに使用して
正規表現と一致させます。  
一致させる正規表現の値を持つ 
{{<hover label="matchRegex" line="10">}}regexp{{</hover>}} キーを定義します。

一致が成功すると、Crossplane は 
{{<hover label="matchRegex" line="11">}}result:{{</hover>}} を
パッチ {{<hover label="matchRegex" line="4">}}toFieldPath{{</hover>}} に提供します。

```yaml {label="matchRegex"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["myAnnotation"]
    transforms:
      - type: match
        match:
          patterns:
            - type: regexp
              regexp: '^field1.*'
              result: "foundField1"
```

#### デフォルト値の使用

オプションで、一致するパターンがない場合に使用するデフォルト値を提供できます。  

デフォルト値は、元の入力値または定義されたデフォルト値のいずれかであることができます。 

一致が見つからない場合は、{{<hover label="defaultValue" line="12">}}fallbackTo: Value{{</hover>}}を使用してデフォルト値を提供します。

例えば、文字列{{<hover label="defaultValue" line="10">}}unknownString{{</hover>}}が一致しない場合、Crossplaneは{{<hover label="defaultValue" line="12">}}Value{{</hover>}} 
{{<hover label="defaultValue" line="13">}}StringNotFound{{</hover>}}を{{<hover label="defaultValue" line="4">}}toFieldPath{{</hover>}}に提供します。 

```yaml {label="defaultValue"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["myAnnotation"]
    transforms:
      - type: match
        match:
          patterns:
            - type: literal
              literal: "UnknownString"
              result: "foundField1"
          fallbackTo: Value
          fallbackValue: "StringNotFound"
```

元の入力をフォールバック値として使用するには、{{<hover label="defaultInput" line="12">}}fallbackTo: Input{{</hover>}}を使用します。

Crossplaneは、元の{{<hover label="defaultInput" line="3">}}fromFieldPath{{</hover>}}入力を{{<hover label="defaultInput" line="4">}}toFieldPath{{</hover>}}値に使用します。
```yaml {label="defaultInput"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["myAnnotation"]
    transforms:
      - type: match
        match:
          patterns:
            - type: literal
              literal: "UnknownString"
              result: "foundField1"
          fallbackTo: Input
```

### 数学変換

入力を乗算したり、最小値または最大値を適用するには、{{<hover label="math" line="6">}}math{{</hover>}}変換を使用します。 

{{<hint "important">}}
{{<hover label="math" line="6">}}math{{</hover>}}変換は整数入力のみをサポートします。 
{{< /hint >}}

```yaml {label="math",copy-lines="1-7"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.numberField
    toFieldPath: metadata.annotations["mathAnnotation"]
    transforms:
      - type: math
        math:
          ...
```

<!-- vale Google.Headings = NO -->
#### clampMin
<!-- vale Google.Headings = YES -->

{{<hover label="clampMin" line="8">}}type: clampMin{{</hover>}}は、入力が{{<hover label="clampMin" line="8">}}type: clampMin{{</hover>}}値より大きい場合に定義された最小値を使用します。

例えば、この{{<hover label="clampMin" line="8">}}type: clampMin{{</hover>}}は、入力が{{<hover label="clampMin" line="9">}}20{{</hover>}}より大きいことを要求します。

入力が{{<hover label="clampMin" line="9">}}20{{</hover>}}より低い場合、Crossplaneは{{<hover label="clampMin" line="9">}}clampMin{{</hover>}}値を{{<hover label="clampMin" line="4">}}toFieldPath{{</hover>}}に使用します。

```yaml {label="clampMin"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.numberField
    toFieldPath: metadata.annotations["mathAnnotation"]
    transforms:
      - type: math
        math:
          type: clampMin
          clampMin: 20
```

<!-- vale Google.Headings = NO -->
#### clampMax
<!-- vale Google.Headings = YES -->

{{<hover label="clampMax" line="8">}}type: clampMax{{</hover>}}は、入力が
{{<hover label="clampMax" line="8">}}type: clampMax{{</hover>}}の値よりも大きい場合に
定義された最小値を使用します。

例えば、この
{{<hover label="clampMax" line="8">}}type: clampMax{{</hover>}}は、入力が
{{<hover label="clampMax" line="9">}}5{{</hover>}}未満であることを要求します。

入力が
{{<hover label="clampMax" line="9">}}5{{</hover>}}よりも高い場合、Crossplaneは
{{<hover label="clampMax" line="9">}}clampMax{{</hover>}}の値を
{{<hover label="clampMax" line="4">}}toFieldPath{{</hover>}}に使用します。

```yaml {label="clampMax"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.numberField
    toFieldPath: metadata.annotations["mathAnnotation"]
    transforms:
      - type: math
        math:
          type: clampMax
          clampMax: 5
```

<!-- vale Google.Headings = NO -->
#### Multiply
<!-- vale Google.Headings = YES -->

{{<hover label="multiply" line="8">}}type: multiply{{</hover>}}は、入力を
{{<hover label="multiply" line="9">}}multiply{{</hover>}}の値で
乗算します。

例えば、この
{{<hover label="multiply" line="8">}}type: multiply{{</hover>}}は、
{{<hover label="multiply" line="3">}}fromFieldPath{{</hover>}}の値を
{{<hover label="multiply" line="9">}}2{{</hover>}}で乗算します。

```yaml {label="multiply"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.numberField
    toFieldPath: metadata.annotations["mathAnnotation"]
    transforms:
      - type: math
        math:
          type: multiply
          multiply: 2
```

{{<hint "note" >}}
{{<hover label="multiply" line="9">}}multiply{{</hover>}}の値は整数のみを
サポートします。
{{< /hint >}}

### String transforms

{{<hover label="string" line="6">}}string{{</hover>}}変換は、文字列入力に対して
文字列のフォーマットや操作を適用します。

```yaml {label="string"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["stringAnnotation"]
    transforms:
      - type: string
        string:
          type: ...
```

文字列変換は以下の
{{<hover label="string" line="7">}}types{{</hover>}}をサポートします。

* [Convert](#string-convert)
* [Format](#string-format)
* [Join](#join)
* [Regexp](#regular-expression-type)
* [TrimPrefix](#trim-prefix)
* [TrimSuffix](#trim-suffix)

#### String convert

{{<hover label="stringConvert" line="9">}}type: convert{{</hover>}}は、以下の
変換タイプのいずれかに基づいて入力を変換します：
* `ToUpper` - 文字列をすべて大文字に変更します。
* `ToLower` - 文字列をすべて小文字に変更します。
* `ToBase64` - 入力から新しいbase64文字列を作成します。
* `FromBase64` - base64入力から新しいテキスト文字列を作成します。
* `ToJson` - 入力文字列を有効なJSONに変換します。
* `ToSha1` - 入力文字列のSHA-1ハッシュを作成します。
* `ToSha256` - 入力文字列のSHA-256ハッシュを作成します。
* `ToSha512` - 入力文字列のSHA-512ハッシュを作成します。
* `ToAdler32` - 入力文字列のAdler32ハッシュを作成します。

```yaml {label="stringConvert"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["FIELD1-TEXT"]
    transforms:
      - type: string
        string:
          type: Convert
          convert: "ToUpper"
```

#### 文字列形式
{{<hover label="typeFormat" line="9">}}type: format{{</hover>}}は、入力に[Go文字列フォーマット](https://pkg.go.dev/fmt)を適用します。

```yaml {label="typeFormat"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.field1
    toFieldPath: metadata.annotations["stringAnnotation"]
    transforms:
      - type: string
        string:
          type: Format
          fmt: "the-field-%s"
```

#### 結合

{{<hover label="typeJoin" line="8">}}type: Join{{</hover>}}は、指定された区切り文字を使用して、入力配列内のすべての値を文字列に結合します。

この変換は配列入力でのみ機能します。

```yaml {label="typeJoin"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.parameters.inputList
    toFieldPath: spec.targetJoined
    transforms:
      - type: string
        string:
          type: Join
          join:
            separator: ","
```

#### 正規表現タイプ
{{<hover label="typeRegex" line="8">}}type: Regexp{{</hover>}}は、正規表現に一致する入力の部分を抽出します。

オプションで、{{<hover label="typeRegex" line="11">}}group{{</hover>}}を使用して、正規表現キャプチャグループに一致させることができます。  
デフォルトでは、Crossplaneは正規表現全体に一致します。

```yaml {label="typeRegex"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.desiredRegion
    toFieldPath: metadata.annotations["euRegion"]
    transforms:
      - type: string
        string:
          type: Regexp
          regexp:
            match: '^eu-(.*)-'
            group: 1
```

#### プレフィックスのトリム

{{<hover label="typeTrimP" line="8">}}type: TrimPrefix{{</hover>}}は、Goの[TrimPrefix](https://pkg.go.dev/strings#TrimPrefix)を使用して、行の先頭から文字を削除します。

```yaml {label="typeTrimP"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.desiredRegion
    toFieldPath: metadata.annotations["north-1"]
    transforms:
      - type: string
        string:
          type: TrimPrefix
          trim: `eu-
```

#### サフィックスのトリム

{{<hover label="typeTrimS" line="8">}}type: TrimSuffix{{</hover>}}は、Goの[TrimSuffix](https://pkg.go.dev/strings#TrimSuffix)を使用して、行の末尾から文字を削除します。

```yaml {label="typeTrimS"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.desiredRegion
    toFieldPath: metadata.annotations["eu"]
    transforms:
      - type: string
        string:
          type: TrimSuffix
          trim: `-north-1'
```

## パッチポリシー

Crossplaneは2種類のパッチポリシーをサポートしています：
* `fromFieldPath`
* `mergeOptions`

<!-- vale Google.Headings = NO -->
### fromFieldPathポリシー
<!-- vale Google.Headings = YES -->

パッチに`fromFieldPath: Required`ポリシーを使用する場合、`fromFieldPath`は合成リソースに存在する必要があります。

{{<hint "tip" >}}
リソースパッチが機能しない場合、`fromFieldPath: Required`ポリシーを適用すると、合成リソースにエラーが発生し、トラブルシューティングに役立つ場合があります。 
{{< /hint >}}

デフォルトでは、Crossplaneは`fromFieldPath: Optional`ポリシーを適用します。`fromFieldPath: Optional`では、`fromFieldPath`が存在しない場合、Crossplaneはパッチを無視します。


`{{<hover label="required" line="6">}}fromFieldPath: Required{{</hover>}}`を使用すると、コンポジットリソースは`{{<hover label="required" line="6">}}fromFieldPath{{</hover>}}`が存在しない場合にエラーを生成します。

```yaml {label="required"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.desiredRegion
    toFieldPath: metadata.annotations["eu"]
    policy:
      fromFieldPath: Required
```

### マージオプション

デフォルトでは、パッチを適用する際に宛先データが上書きされます。`{{<hover label="merge" line="6">}}mergeOptions{{</hover>}}`を使用して、パッチが配列やオブジェクトを上書きすることなくマージできるようにします。

配列入力の場合、`{{<hover label="merge" line="7">}}appendSlice: true{{</hover>}}`を使用して、配列データを既存の配列の末尾に追加します。

オブジェクトの場合、`{{<hover label="merge" line="8">}}keepMapValues: true{{</hover>}}`を使用して、既存のオブジェクトキーをそのままにします。パッチは、入力データと宛先データの間で一致するキーを更新します。

```yaml {label="merge"}
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.desiredRegion
    toFieldPath: metadata.annotations["eu"]
    policy:
      mergeOptions:
        appendSlice: true
        keepMapValues: true
```
