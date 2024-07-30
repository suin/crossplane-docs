---
title: 既存リソースのインポート
weight: 200
---

プロバイダーにすでにプロビジョニングされたリソースがある場合、
それらを管理リソースとしてインポートし、Crossplaneに管理させることができます。
管理リソースの [`managementPolicies`]({{<ref "/v1.16/concepts/managed-resources#managementpolicies">}})
フィールドは、外部リソースをCrossplaneにインポートすることを可能にします。

Crossplaneはリソースを [手動で]({{<ref "#import-resources-manually">}})
または [自動で]({{<ref "#import-resources-automatically">}}) インポートできます。

## リソースを手動でインポートする

Crossplaneは、管理リソース内の `crossplane.io/external-name` アノテーションを
一致させることで、既存のプロバイダーリソースを発見し、インポートできます。

プロバイダー内の既存の外部リソースをインポートするには、`crossplane.io/external-name` アノテーションを持つ新しい管理リソースを作成します。
アノテーションの値をプロバイダー内のリソースの名前に設定します。

たとえば、{{<hover label="annotation" line="5">}}my-existing-network{{</hover>}}という名前の
既存のGCPネットワークをインポートするには、新しい管理リソースを作成し、
アノテーションに{{<hover label="annotation" line="5">}}my-existing-network{{</hover>}}を使用します。

```yaml {label="annotation",copy-lines="none"}
apiVersion: compute.gcp.crossplane.io/v1beta1
kind: Network
metadata:
  annotations:
    crossplane.io/external-name: my-existing-network
```

{{<hover label="name" line="5">}}metadata.name{{</hover>}} フィールドは
任意の名前にすることができます。たとえば、{{<hover label="name" line="5">}}imported-network{{</hover>}}です。

{{< hint "note" >}}
この名前はKubernetesオブジェクトの名前です。
プロバイダー内のリソース名とは関係ありません。
{{< /hint >}}

```yaml {label="name",copy-lines="none"}
apiVersion: compute.gcp.crossplane.io/v1beta1
kind: Network
metadata:
  name: imported-network
  annotations:
    crossplane.io/external-name: my-existing-network
```

{{<hover label="fp" line="8">}}spec.forProvider{{</hover>}} フィールドは空のままにします。
Crossplaneは設定をインポートし、それを管理リソースに自動的に適用します。

{{< hint "important" >}}
管理リソースに{{<hover label="fp" line="8">}}spec.forProvider{{</hover>}}内の
_必須_ フィールドがある場合、それを `forProvider` フィールドに追加する必要があります。

それらのフィールドの値は、プロバイダー内の値と一致する必要があります。
さもなければ、Crossplaneは既存の値を上書きします。
{{< /hint >}}

```yaml {label="fp",copy-lines="all"}
apiVersion: compute.gcp.crossplane.io/v1beta1
kind: Network
metadata:
  name: imported-network
  annotations:
    crossplane.io/external-name: my-existing-network
spec:
  forProvider: {}
```


Crossplaneは、インポートされたリソースを制御および管理します。管理されたリソースの`spec`に対する変更は、外部リソースに影響を与えます。

## リソースを自動的にインポートする

`Observe` [管理ポリシー]({{<ref "/v1.16/concepts/managed-resources#managementpolicies">}})を使用して、外部リソースを自動的にインポートします。

Crossplaneは、リソースを観察するだけで、リソースを変更または削除することはありません。

{{<hint "重要" >}}
管理されたリソースの`managementPolicies`オプションはベータ機能です。

プロバイダーが管理ポリシーのサポートを決定します。
プロバイダーが管理ポリシーをサポートしているかどうかは、プロバイダーのドキュメントを参照してください。
{{< /hint >}}

<!-- vale off -->
### Observe管理ポリシーを適用する
<!-- vale on -->

インポートするリソースの
{{<hover label="oo-policy" line="1">}}apiVersion{{</hover>}}と
{{<hover label="oo-policy" line="2">}}kind{{</hover>}}に一致する新しい管理リソースを作成し、
{{<hover label="oo-policy" line="4">}}managementPolicies: ["Observe"]{{</hover>}}を
{{<hover label="oo-policy" line="3">}}spec{{</hover>}}に追加します。

例えば、GCP SQL DatabaseInstanceをインポートするには、
{{<hover label="oo-policy" line="4">}}managementPolicies: ["Observe"]{{</hover>}}
を設定した新しいリソースを作成します。
```yaml {label="oo-policy",copy-lines="none"}
apiVersion: sql.gcp.upbound.io/v1beta1
kind: DatabaseInstance
spec:
  managementPolicies: ["Observe"]
```

### 外部名アノテーションを追加する
リソースのために{{<hover label="oo-ex-name" line="5">}}crossplane.io/external-name{{</hover>}}
アノテーションを追加します。この名前は、プロバイダー内の名前と一致する必要があります。

例えば、GCPデータベースの名前が
{{<hover label="oo-ex-name" line="5">}}my-external-database{{</hover>}}の場合、
値が
{{<hover label="oo-ex-name" line="5">}}my-external-database{{</hover>}}である
{{<hover label="oo-ex-name" line="5">}}crossplane.io/external-name{{</hover>}}アノテーションを適用します。

```yaml {label="oo-ex-name",copy-lines="none"}
apiVersion: sql.gcp.upbound.io/v1beta1
kind: DatabaseInstance
metadata:
  annotations:
    crossplane.io/external-name: my-external-database
spec:
  managementPolicies: ["Observe"]
```

### Kubernetesオブジェクト名を作成する
Kubernetesオブジェクトに使用する{{<hover label="oo-name" line="4">}}name{{</hover>}}を作成します。

例えば、Kubernetesオブジェクトに名前を付けます
{{<hover label="oo-name" line="4">}}my-imported-database{{</hover>}}。

```yaml {label="oo-name",copy-lines="none"}
apiVersion: sql.gcp.upbound.io/v1beta1
kind: DatabaseInstance
metadata:
  name: my-imported-database
  annotations:
    crossplane.io/external-name: my-external-database
spec:
  managementPolicies: ["Observe"]
```

### 特定の外部リソースを特定する
プロバイダー内に同じ名前のリソースが複数ある場合は、ユニークな
{{<hover line="9" label="oo-region">}}spec.forProvider{{</hover>}} フィールドで特定のリソースを識別します。

例えば、{{<hover line="10" label="oo-region">}}us-central1{{</hover>}} リージョンのGCP SQLデータベースのみをインポートします。

```yaml {label="oo-region"}
apiVersion: sql.gcp.upbound.io/v1beta1
kind: DatabaseInstance
metadata:
  name: my-imported-database
  annotations:
    crossplane.io/external-name: my-external-database
spec:
  managementPolicies: ["Observe"]
  forProvider:
    region: "us-central1"
```

### 管理リソースを適用する

新しい管理リソースを適用します。Crossplaneは、クラウド内の外部リソースのステータスを新しく作成された管理リソースと同期します。

### 発見されたリソースを表示する
Crossplaneは管理リソースを発見し、外部リソースからの値で
{{<hover label="ooPopulated" line="12">}}status.atProvider{{</hover>}}
フィールドを埋めます。

```yaml {label="ooPopulated",copy-lines="none"}
apiVersion: sql.gcp.upbound.io/v1beta1
kind: DatabaseInstance
metadata:
  name: my-imported-database
  annotations:
    crossplane.io/external-name: my-external-database
spec:
  managementPolicies: ["Observe"]
  forProvider:
    region: us-central1
status:
  atProvider:
    connectionName: crossplane-playground:us-central1:my-external-database
    databaseVersion: POSTGRES_14
    deletionProtection: true
    firstIpAddress: 35.184.74.79
    id: my-external-database
    publicIpAddress: 35.184.74.79
    region: us-central1
    # Removed for brevity
    settings:
    - activationPolicy: ALWAYS
      availabilityType: REGIONAL
      diskSize: 100
      # Removed for brevity
      pricingPlan: PER_USE
      tier: db-custom-4-26624
      version: 4
  conditions:
  - lastTransitionTime: "2023-02-22T07:16:51Z"
    reason: Available
    status: "True"
    type: Ready
  - lastTransitionTime: "2023-02-22T07:16:51Z"
    reason: ReconcileSuccess
    status: "True"
    type: Synced
```
<!-- vale off -->
## インポートされたObserveOnlyリソースを制御する
<!-- vale on -->

Crossplaneは、インポート後に`managementPolicies`を変更することで、observe onlyインポートリソースのアクティブな制御を行うことができます。

管理リソースの{{<hover label="fc" line="8">}}managementPolicies{{</hover>}}フィールドを
{{<hover label="fc" line="8">}}["*"]{{</hover>}}に変更します。

{{<hover label="fc" line="16">}}status.atProvider{{</hover>}}から必要なパラメータ値をコピーし、
{{<hover label="fc" line="9">}}spec.forProvider{{</hover>}}に提供します。

{{< hint "tip" >}}
重要な`spec.atProvider`の値を手動で`spec.forProvider`にコピーします。
{{< /hint >}}

```yaml {label="fc"}
apiVersion: sql.gcp.upbound.io/v1beta1
kind: DatabaseInstance
metadata:
  name: my-imported-database
  annotations:
    crossplane.io/external-name: my-external-database
spec:
  managementPolicies: ["*"]
  forProvider:
    databaseVersion: POSTGRES_14
    region: us-central1
    settings:
    - diskSize: 100
      tier: db-custom-4-26624
status:
  atProvider:
    databaseVersion: POSTGRES_14
    region: us-central1
    # Removed for brevity
    settings:
    - diskSize: 100
      tier: db-custom-4-26624
      # Removed for brevity
  conditions:
    - lastTransitionTime: "2023-02-22T07:16:51Z"
      reason: Available
      status: "True"
      type: Ready
    - lastTransitionTime: "2023-02-22T11:16:45Z"
      reason: ReconcileSuccess
      status: "True"
      type: Synced
```

Crossplaneは現在、インポートされたリソースを完全に管理しています。Crossplaneは、プロバイダーの外部リソースに対して管理リソースへの変更を適用します。
