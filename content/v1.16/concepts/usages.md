---
title: 使用法
weight: 95
state: alpha
alphaVersion: "1.14"
description: "Usageは、管理リソースまたはコンポジットの使用関係を定義します"
---

`Usage`は、管理リソースまたはコンポジットリソースの使用関係を定義するCrossplaneリソースです。Usageの主な使用ケースは以下の2つです。

1. リソースを誤って削除されるのから保護すること。
2. 依存リソースの削除前にリソースが削除されないようにすることで、削除の順序を管理すること。

最初の使用ケースについては[削除保護のためのUsage](#usage-for-deletion-protection)のセクションを、2つ目の使用ケースについては[削除順序のためのUsage](#usage-for-deletion-ordering)のセクションを参照してください。

## 使用法の有効化
Usageはアルファ機能です。アルファ機能はデフォルトでは有効になっていません。

[Crossplaneポッド設定を変更する]({{<ref "./pods#change-pod-settings">}})ことで`Usage`サポートを有効にし、  
{{<hover label="deployment" line="12">}}--enable-usages{{</hover>}}引数を有効にします。

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
        - --enable-usages
```

{{<hint "tip" >}}

[Crossplaneインストールガイド]({{<ref "../software/install#feature-flags">}})では、Helmを使用して
{{<hover label="deployment" line="12">}}\-\-enable-usages{{</hover>}}のような機能フラグを有効にする方法が説明されています。
{{< /hint >}}

<!-- vale Google.Headings = NO -->
## 使用法の作成
<!-- vale Google.Headings = YES -->

<!-- vale write-good.Passive = NO -->
{{<hover label="protect" line="2">}}Usage{{</hover>}}の{{<hover label="protect" line="5">}}spec{{</hover>}}には、使用中または保護されているリソースを定義するための必須の{{<hover label="protect" line="6">}}of{{</hover>}}フィールドがあります。 
{{<hover label="protect" line="11">}}reason{{</hover>}}フィールドは保護の理由を定義し、{{<hover label="order" line="11">}}by{{</hover>}}フィールドは使用しているリソースを定義します。両方のフィールドはオプションですが、少なくとも1つは提供する必要があります。
<!-- vale write-good.Passive = YES -->

{{<hint "important" >}}
<!-- vale write-good.Passive = NO -->
Usage関係は`Managed Resources`と`Composites`の間で定義できます。
<!-- vale write-good.TooWordy = NO -->
ただし、使用リソースとしての`Composite`（`spec.by`）は、`compositeDeletePolicy`が`Foreground`でない限り効果がなく、デフォルトの削除ポリシー`Background`では自身の削除前に子リソースの削除をブロックしないためです。
<!-- vale write-good.TooWordy = YES -->
<!-- vale write-good.Passive = YES -->
{{< /hint >}}

### 削除保護の使用法

以下の例では、{{<hover label="protect" line="10">}}my-database{{</hover>}}リソースの削除を防ぎ、
{{<hover label="protect" line="11">}}reason{{</hover>}}が定義された削除リクエストを拒否します。

```yaml {label="protect"}
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Usage
metadata:
  name: protect-production-database
spec:
  of:
    apiVersion: rds.aws.upbound.io/v1beta1
    kind: Instance
    resourceRef:
      name: my-database
  reason: "Production Database - should never be deleted!"
```

### 削除順序の使用法

以下の例では、{{<hover label="order" line="10">}}my-cluster{{</hover>}}リソースの削除を防ぎ、
{{<hover label="order" line="15">}}my-prometheus-chart{{</hover>}}リソースの削除前に
削除リクエストを拒否します。

```yaml {label="order"}
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Usage
metadata:
  name: release-uses-cluster
spec:
  of:
    apiVersion: eks.upbound.io/v1beta1
    kind: Cluster
    resourceRef:
      name: my-cluster
  by:
    apiVersion: helm.crossplane.io/v1beta1
    kind: Release
    resourceRef:
      name: my-prometheus-chart
```

### 使用法にセレクタを使用する

使用法は、{{<hover label="selectors" line="9">}}selectors{{</hover>}}を使用して
使用中のリソースまたは使用するリソースを定義できます。
これにより、リソース名を提供する代わりに、{{<hover label="selectors" line="12">}}labels{{</hover>}}や
{{<hover label="selectors" line="10">}}マッチングコントローラ参照{{</hover>}}を使用して
リソースを定義できます。

```yaml {label="selectors"}
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Usage
metadata:
  name: release-uses-cluster
spec:
  of:
    apiVersion: eks.upbound.io/v1beta1
    kind: Cluster
    resourceSelector:
      matchControllerRef: false # default, and could be omitted
      matchLabels:
        foo: bar
  by:
    apiVersion: helm.crossplane.io/v1beta1
    kind: Release
    resourceSelector:
       matchLabels:
          baz: qux
```

`Usage`コントローラがセレクタを解決した後、リソース名は
{{<hover label="selectors-resolved" line="10">}}resourceRef.name{{</hover>}}フィールドに
永続化されます。以下の例は、セレクタの解決後の`Usage`リソースを示しています。

{{<hint "important" >}}
<!-- vale write-good.Passive = NO -->
セレクタは一度だけ解決されます。マッチが複数ある場合は、マッチしたリソースのリストから
ランダムにリソースが選択されます。
<!-- vale write-good.Passive = YES -->
{{< /hint >}}

```yaml {label="selectors-resolved"}
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Usage
metadata:
  name: release-uses-cluster
spec:
  of:
    apiVersion: eks.upbound.io/v1beta1
    kind: Cluster
    resourceRef:
       name: my-cluster
    resourceSelector:
      matchLabels:
        foo: bar
  by:
    apiVersion: helm.crossplane.io/v1beta1
    kind: Release
    resourceRef:
       name: my-cluster
    resourceSelector:
       matchLabels:
          baz: qux
```

### ブロックされた削除試行の再実行

デフォルトでは、`Usage`リソースの削除は、`Usage`によってブロックされた削除試行があっても、
使用中のリソースの削除をトリガーしません。
ブロックされた削除を再実行するには、{{<hover label="replay" line="6">}}replayDeletion{{</hover>}}フィールドを`true`に設定します。

```yaml {label="replay"}
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Usage
metadata:
  name: release-uses-cluster
spec:
  replayDeletion: true
  of:
    apiVersion: eks.upbound.io/v1beta1
    kind: Cluster
    resourceRef:
      name: my-cluster
  by:
    apiVersion: helm.crossplane.io/v1beta1
    kind: Release
    resourceRef:
      name: my-prometheus-chart
```


{{<hint "tip" >}}

リプレイ削除は、使用されるリソースがコンポジションの一部である場合に便利です。
この設定は、使用されるリソースが消失した直後にその削除をリプレイすることによって、使用されるリソースの削除時間を根本的に短縮します。これにより、Kubernetes ガーベジコレクタの長い指数バックオフ期間を待つ代わりに、所有するコンポジットも削除されます。
{{< /hint >}}

## コンポジションでの使用

Usages の典型的なユースケースは、コンポジション内のリソース間で削除の順序を定義することです。Usages は
[コントローラー参照の一致]({{<ref "./compositions#match-a-controller-reference" >}})
をサポートしており、セレクター内で一致するリソースが同じコンポジットリソースにあることを保証します。これは、[クロスリソース参照]({{<ref "./compositions#cross-resource-references" >}})と同様です。

以下の例は、`Cluster` と `Release` リソース間で削除の順序を定義するコンポジションを示しています。`Usage` は、`Release` リソースが正常に削除されるまで `Cluster` リソースの削除をブロックします。

```yaml {label="composition"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  resources:
    - name: cluster
      base:
        apiVersion: container.gcp.upbound.io/v1beta1
        kind: Cluster
        # Removed for brevity
    - name: release
      base:
        apiVersion: helm.crossplane.io/v1beta1
        kind: Release
        # Removed for brevity
    - name: release-uses-cluster
      base:
        apiVersion: apiextensions.crossplane.io/v1alpha1
        kind: Usage
        spec:
          replayDeletion: true
          of:
            apiVersion: container.gcp.upbound.io/v1beta1
            kind: Cluster
            resourceSelector:
              matchControllerRef: true
          by:
            apiVersion: helm.crossplane.io/v1beta1
            kind: Release
            resourceSelector:
              matchControllerRef: true
```

{{<hint "tip" >}}

<!-- vale write-good.Passive = NO -->
コンポジション内に同じタイプのリソースが複数ある場合、{{<hover label="composition" line="18">}}Usage{{</hover>}} リソースは、使用中のリソースまたは使用するリソースを一意に識別する必要があります。これは、追加のラベルを使用し、{{<hover label="composition" line="24">}}matchControllerRef{{</hover>}} を `matchLabels` セレクターと組み合わせることで実現できます。別の選択肢として、`resourceRef.name` を直接パッチし、`ToCompositeFieldPath` および `FromCompositeFieldPath` または `ToEnvironmentFieldPath` および `FromEnvironmentFieldPath` タイプのパッチを使用することもできます。
<!-- vale write-good.Passive = YES -->
{{< /hint >}}
