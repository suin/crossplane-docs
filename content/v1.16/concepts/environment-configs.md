---
title: 環境設定
weight: 75
state: alpha
alphaVersion: "1.11"
description: "環境設定または EnvironmentConfigs は、Composition のパッチ処理に使用されるメモリ内データストアです"
---

<!--
TODO: ポリシーを追加
-->


Crossplane の EnvironmentConfig は、クラスター スコープの 
[ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/)-のような 
リソースで、Composition によって使用されます。Composition は、個々のリソースからの情報を保存するためや、[パッチ]({{<ref "patch-and-transform">}})を適用するために環境を使用できます。

Crossplane は複数の EnvironmentConfigs をサポートしており、それぞれがユニークなデータストアとして機能します。

Crossplane が複合リソースを作成すると、Crossplane は関連する Composition で参照されているすべての 
EnvironmentConfigs をマージし、その複合リソースのためのユニークなメモリ内環境を作成します。

複合リソースは、そのユニークなメモリ内環境にデータを読み書きできます。

{{<hint "important" >}}
メモリ内環境は各複合リソースに固有です。  
複合リソースは、他の複合リソースの環境内のデータを読み取ることはできません。 
{{< /hint >}}

## EnvironmentConfigs を有効にする
EnvironmentConfigs はアルファ機能です。アルファ機能はデフォルトでは有効になっていません。

[Crossplane ポッド設定を変更する]({{<ref "./pods#change-pod-settings">}})ことで 
EnvironmentConfig サポートを有効にし、  
{{<hover label="deployment" line="12">}}--enable-environment-configs{{</hover>}} 
引数を有効にします。

```yaml {label="deployment",copy-lines="12"}
$ kubectl edit deployment crossplane --namespace crossplane-system
apiVersion: apps/v1
kind: Deployment
spec:
# Removed for brevity
  template:
    spec:
      containers:
      - args:
        - core
        - start
        - --enable-environment-configs
```

{{<hint "tip" >}}

[Crossplane インストールガイド]({{<ref "../software/install#feature-flags">}}) 
では、Helm を使用して 
{{<hover label="deployment" line="12">}}--enable-environment-configs{{</hover>}} 
のような機能フラグを有効にする方法が説明されています。
{{< /hint >}}

<!-- vale Google.Headings = NO -->
## EnvironmentConfig を作成する
<!-- vale Google.Headings = YES -->

{{<hover label="env1" line="2">}}EnvironmentConfig{{</hover>}} には、単一の
オブジェクトフィールド、 
{{<hover label="env1" line="5">}}data{{</hover>}} があります。

EnvironmentConfig は、 
{{<hover label="env1" line="5">}}data{{</hover>}} フィールド内の任意のデータをサポートします。

ここに例があります 
{{<hover label="env1" line="2">}}EnvironmentConfig{{</hover>}}。

```yaml {label="env1"}
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
  key3:
    - item1
    - item2
```

<!-- vale Google.Headings = NO -->
## 環境構成を選択
<!-- vale Google.Headings = YES -->

Compositionの 
{{<hover label="comp" line="6">}}environment{{</hover>}} フィールドで使用する
EnvironmentConfigsを選択します。

{{<hover label="comp" line="7">}}environmentConfigs{{</hover>}} フィールドは、このCompositionが使用できる環境のリストです。

{{<hover label="comp" line="8">}}Reference{{</hover>}} または 
{{<hover label="comp" line="11">}}Selector{{</hover>}}によって環境を選択します。

{{<hover label="comp" line="8">}}Reference{{</hover>}}は、 
{{<hover label="comp" line="10">}}name{{</hover>}}によって環境を選択します。  
{{<hover label="comp" line="11">}}Selector{{</hover>}}は、環境に適用された 
{{<hover label="comp" line="13">}}Labels{{</hover>}}に基づいて環境を選択します。

```yaml {label="comp",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
    - type: Reference
      ref:
        name: example-environment
    - type: Selector
      selector:
        matchLabels:
      # Removed for brevity
```

Compositionが複数の 
{{<hover label="comp" line="7">}}environmentConfigs{{</hover>}}を使用する場合、Crossplaneはそれらをリストに表示された順序で統合します。

{{<hint "note" >}}
複数の 
{{<hover label="comp" line="7">}}environmentConfigs{{</hover>}}が同じキーを使用する場合、Compositionはリストの最後に表示された環境の値を使用します。
{{</hint >}}

### 名前で選択

{{<hover label="byName" line="8">}}type: Reference{{</hover>}}を使用して名前で環境を選択します。

{{<hover label="byName" line="9">}}ref{{</hover>}}オブジェクトと、環境の正確な名前に一致する 
{{<hover label="byName" line="10">}}name{{</hover>}}を定義します。

例えば、次の 
{{<hover label="byName" line="7">}}environmentConfig{{</hover>}}を選択します
{{<hover label="byName" line="10">}}example-environment{{</hover>}}という名前の。

```yaml {label="byName",copy-lines="all"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
    - type: Reference
      ref:
        name: example-environment
```

### ラベルで選択

{{<hover label="byLabel" line="8">}}type: Selector{{</hover>}}を使用してラベルで環境を選択します。

{{<hover label="byLabel" line="9">}}selector{{</hover>}}オブジェクトを定義します。


{{<hover label="byLabel" line="10">}}matchLabels{{</hover>}} オブジェクトには、一致させるラベルのリストが含まれています。 

ラベルを選択するには、ラベルの 
{{<hover label="byLabel" line="11">}}key{{</hover>}} 
とキーの値の両方を一致させる必要があります。 

ラベルの値を一致させる際には、 
{{<hover label="byLabel" line="12">}}type: Value{{</hover>}} を指定し、 
{{<hover label="byLabel" line="13">}}value{{</hover>}} フィールドに一致させる値を提供します。

Crossplane は、複合リソース内の入力に基づいてラベルの値を一致させることもできます。 
{{<hover label="byLabel" line="15">}}type: FromCompositeFieldPath{{</hover>}} を使用し、 
{{<hover label="byLabel" line="16">}}valueFromFieldPath{{</hover>}} フィールドに一致させるフィールドを提供します。

```yaml {label="byLabel",copy-lines="all"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
    - type: Selector
      selector: 
        matchLabels:
          - key: my-label-key
            type: Value
            value: my-label-value
          - key: my-label-key
            type: FromCompositeFieldPath
            valueFromFieldPath: spec.parameters.deploy
  resources:
  # Removed for brevity
```

#### セレクター結果の管理

ラベルによって環境を選択すると、複数の環境が返される場合があります。  
Composition は、環境の名前で結果をすべてソートし、ソートされたリストの最初の環境のみを使用します。 

{{<hover label="selectResults" line="10">}}mode{{</hover>}} を 
{{<hover label="selectResults" line="10">}}mode: Multiple{{</hover>}} に設定すると、 
一致したすべての環境が返されます。 
{{<hover label="selectResults" line="19">}}mode: Single{{</hover>}} を使用すると、 
単一の環境が返されます。

{{<hint "note" >}}
ソートと選択 
{{<hover label="selectResults" line="10">}}mode{{</hover>}} は、 
単一の 
{{<hover label="selectResults" line="8">}}type: Selector{{</hover>}} にのみ適用されます。 

これは、Compositions が複数の 
{{<hover label="selectResults" line="7">}}environmentConfigs{{</hover>}} をマージする方法を変更するものではありません。
{{< /hint >}}


```yaml {label="selectResults"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
    - type: Selector
      selector:
        mode: Multiple
        matchLabels:
          - key: my-label-key
            type: Value
            value: my-label-value
          - key: my-label-key
            type: FromCompositeFieldPath
            valueFromFieldPath: spec.parameters.deploy
    - type: Selector
      selector:
        mode: Single
        matchLabels:
          - key: my-other-label-key
            type: Value
            value: my-other-label-value
          - key: my-other-label-key
            type: FromCompositeFieldPath
            valueFromFieldPath: spec.parameters.deploy
```

{{<hover label="maxMatch" line="10">}}mode: Multiple{{</hover>}} を使用する場合、 
{{<hover label="maxMatch" line="11">}}maxMatch{{</hover>}} を使用して返される環境の数を制限し、 
返される環境の最大数を定義します。 

`minMatch` を使用して、返される環境の最小数を定義します。


Compositionは、返された環境を名前でアルファベット順にソートします。 
{{<hover label="maxMatch" line="12">}}sortByFieldPath{{</hover>}}を使用して
異なるフィールドで環境をソートし、ソートするフィールドを定義します。 


```yaml {label="maxMatch"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
    - type: Selector
      selector:
        mode: Multiple
        maxMatch: 4
        sortByFieldPath: metadata.annotations[sort.by/weight]
        matchLabels:
          - key: my-label-key
            type: Value
            value: my-label-value
          - key: my-label-key
            type: FromCompositeFieldPath
            valueFromFieldPath: spec.parameters.deploy
```

{{<hover label="maxMatch" line="18">}}matchLabels{{</hover>}}によって選択された環境は、
{{<hover label="maxMatch" line="7">}}environmentConfigs{{</hover>}}にリストされている
他の環境にマージされます。

#### オプションのセレクタラベル
デフォルトでは、Crossplaneは
{{<hover label="byLabelOptional" line="16">}}valueFromFieldPath{{</hover>}}
フィールドが合成リソースに存在しない場合、エラーを発行します。  

フィールドが存在しない場合に無視するために、
{{<hover label="byLabelOptional" line="17">}}fromFieldPathPolicy{{</hover>}}を
{{<hover label="byLabelOptional" line="17">}}Optional{{</hover>}}として追加します。

```yaml {label="byLabelOptional",copy-lines="all"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
      - type: Selector
        selector:
          matchLabels:
            - key: my-first-label-key
              type: Value
              value: my-first-label-value
            - key: my-second-label-key
              type: FromCompositeFieldPath
              valueFromFieldPath: spec.parameters.deploy
              fromFieldPathPolicy: Optional
  resources:
  # Removed for brevity
```


オプションのラベルのデフォルト値を設定するには、まず
{{<hover label="byLabelOptionalDefault" line="15">}}value{{</hover>}}のデフォルトを設定し、
次に
{{<hover label="byLabelOptionalDefault" line="20">}}Optional{{</hover>}}ラベルを定義します。

例えば、このCompositionは
{{<hover label="byLabelOptionalDefault" line="16">}}value: my-default-value{{</hover>}}
をキー{{<hover label="byLabelOptionalDefault" line="14">}}my-second-label-key{{</hover>}}に対して定義します。
ラベル
{{<hover label="byLabelOptionalDefault" line="17">}}my-second-label-key{{</hover>}}
が存在する場合、Crossplaneはそのラベルから値を使用します。

```yaml {label="byLabelOptionalDefault",copy-lines="all"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example-composition
spec:
  environment:
    environmentConfigs:
      - type: Selector
        selector:
          matchLabels:
            - key: my-first-label-key
              type: Value
              value: my-label-value
            - key: my-second-label-key
              type: Value
              value: my-default-value
            - key: my-second-label-key
              type: FromCompositeFieldPath
              valueFromFieldPath: spec.parameters.deploy
              fromFieldPathPolicy: Optional
  resources:
  # Removed for brevity
```

{{<hint "warning" >}}
Crossplaneは値を順番に適用します。定義された最後のキーの値が常に優先されます。

ラベルの後にデフォルト値を定義すると、常にラベルの値が上書きされます。
{{< /hint >}}

## EnvironmentConfigsによるパッチ適用

Crossplaneが合成リソースを作成または更新する際、Crossplaneは
指定されたすべてのEnvironmentConfigsをメモリ内の環境にマージします。


複合リソースは、EnvironmentConfig と複合リソースの間、または EnvironmentConfig と複合リソース内で定義された個々のリソースの間でデータを読み書きできます。

{{<hint "tip" >}}
[Patch and Transform]({{<ref "./patch-and-transform">}}) ドキュメントで EnvironmentConfig パッチタイプについて読んでください。
{{< /hint >}}

<!-- これらの2つのセクションは、異なるヘッダーの深さで構成ドキュメントに重複しています --> 

### 複合リソースをパッチする
複合リソースをパッチするには、 
{{< hover label="xrpatch" line="7">}}patches{{</hover>}} を 
{{< hover label="xrpatch" line="5">}}environment{{</hover>}} 内で使用します。

{{< hover label="xrpatch" line="5">}}ToCompositeFieldPath{{</hover>}} を使用して、メモリ内の環境から複合リソースにデータをコピーします。  
{{< hover label="xrpatch" line="5">}}FromCompositeFieldPath{{</hover>}} を使用して、複合リソースからメモリ内の環境にデータをコピーします。

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

個々のリソースは、メモリ内の環境に書き込まれたデータを使用できます。

### 個々のリソースをパッチする
個々のリソースをパッチするには、リソースの 
{{<hover label="envpatch" line="16">}}patches{{</hover>}} 内で、 
{{<hover label="envpatch" line="17">}}ToEnvironmentFieldPath{{</hover>}} を使用して、リソースからメモリ内の環境にデータをコピーします。  
{{<hover label="envpatch" line="20">}}FromEnvironmentFieldPath{{</hover>}} を使用して、メモリ内の環境からリソースにデータをコピーします。

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

[Patch and Transform]({{<ref "./patch-and-transform">}}) ドキュメントには、個々のリソースをパッチするための詳細情報があります。

<!-- 重複コンテンツの終了 -->
