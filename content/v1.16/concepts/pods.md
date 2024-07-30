---
title: Crossplane Pods
weight: 1
description: Crossplaneにインストールされるコンポーネントの背景とその機能。
---

基本的なCrossplaneのインストールは、`crossplane`ポッドと
`crossplane-rbac-manager`ポッドの2つで構成されています。両方のポッドはデフォルトで`crossplane-system`
名前空間にインストールされます。 


## Crossplaneポッド

### Initコンテナ
コアのCrossplaneコンテナを開始する前に、_init_コンテナが実行されます。init
コンテナはコアのCrossplane 
[カスタムリソース定義](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions)
（`CRDs`）をインストールし、CrossplaneのWebhookを構成し、提供されたプロバイダーや
構成をインストールします。 

{{<hint "tip" >}}
Kubernetesのドキュメントには、[initコンテナ](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)に関する詳細情報が含まれています。
{{< /hint >}}

initコンテナが設定する設定には、Crossplaneでのプロバイダーや構成パッケージのインストール、Crossplaneがインストールされる名前空間のカスタマイズ、およびWebhook構成の定義が含まれます。 

initコンテナによってインストールされるコアのCRDには以下が含まれます: 
* CompositeResourceDefinitions、Compositions、Configurations、Providers
* パッケージ依存関係を管理するためのロック
* インストールされたプロバイダーや関数に設定を適用するためのDeploymentRuntimeConfigs
* [HashiCorp Vault](https://www.vaultproject.io/)のような外部シークレットストアに接続するためのStoreConfigs

{{< hint "note" >}}

[Crossplaneのインストール]({{< ref "../software/install" >}})セクションには、Crossplaneのインストールをカスタマイズするための詳細情報があります。
{{< /hint >}}

Crossplaneポッドのステータス`Init`は、initコンテナが実行中であることを示しています。 

```shell
kubectl get pods -n crossplane-system
NAME                                       READY   STATUS     RESTARTS   AGE
crossplane-9f6d5cd7b-r9j8w                 0/1     Init:0/1   0          6s
```

initコンテナが完了すると、自動的にCrossplaneコアコンテナが開始されます。

```shell
kubectl get pods -n crossplane-system
NAME                                       READY   STATUS    RESTARTS   AGE
crossplane-9f6d5cd7b-r9j8w                 1/1     Running   0          15s
```

### コアコンテナ

メインのCrossplaneコンテナである_コア_コンテナは、Crossplaneリソースの望ましい状態を強制し、リーダー選出を管理し、Webhookを処理します。

{{<hint "note" >}}
Crossplaneポッドは、Claimsや複合リソースを含むコアCrossplaneコンポーネントのみを調整します。プロバイダーは、管理対象リソースの調整を担当します。
{{< /hint >}}

#### 調整ループ

コアコンテナは_調整ループ_で動作し、デプロイされたリソースの状態を常にチェックし、「ドリフト」を修正します。リソースをチェックした後、Crossplaneはしばらく待って再度チェックします。

CrossplaneはKubernetesの[_ウォッチ_](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes)を通じてリソースを監視するか、定期的にポーリングします。一部のリソースはウォッチされ、ポーリングされる場合があります。

CrossplaneはAPIサーバーに対して、オブジェクトの変更をCrossplaneに通知するよう要求します。この通知ツールは_ウォッチ_です。

ウォッチされるオブジェクトには、プロバイダー、管理リソース、およびCompositeResourceDefinitionsが含まれます。

Kubernetesがウォッチを提供できないオブジェクトについては、Crossplaneはリソースの状態を確認するために定期的にポーリングします。デフォルトのポーリングレートは1分です。`--poll-interval`ポッド引数を使用してポーリングレートを変更します。

ポーリング間隔の値を減らすと、Crossplaneはリソースをより頻繁にポーリングします。これにより、Crossplaneポッドの負荷が増加し、プロバイダーAPI呼び出しがより頻繁に行われます。

<!-- vale write-good.TooWordy = NO -->
<!-- allow "maximum" -->
ポーリング間隔を増やすと、Crossplaneはリソースをより少なくポーリングします。これにより、Crossplaneがクラウドプロバイダーの変更を発見するまでの最大時間が増加します。
<!-- vale write-good.TooWordy = YES -->

管理リソースはポーリングを使用します。

{{< hint "note" >}}
管理リソースは、削除や`spec`の変更などのKubernetesイベントを監視します。管理リソースは、外部システムの変更を検出するためにポーリングに依存しています。
{{< /hint >}}

Crossplaneはすべてのリソースを再確認して、望ましい状態にあることを確認します。Crossplaneはデフォルトで1時間ごとにこれを行います。この間隔を変更するには、`--sync-interval` Crossplaneポッド引数を使用します。


`--max-reconcile-rate` は、Crossplane がリソースを調整する回数（秒あたりの回数）を定義します。

`--max-reconcile-rate` を減少させる、または小さくすることで、Crossplane が使用する CPU リソースが減りますが、変更されたリソースが完全に同期されるまでの時間が増加します。

`--max-reconcile-rate` を増加させる、または大きくすることで、Crossplane が使用する CPU リソースが増加しますが、Crossplane がすべてのリソースをより早く調整できるようになります。

{{< hint "important" >}}
ほとんどのプロバイダーは独自の `--max-reconcile-rate` を使用します。これは、プロバイダーとその管理リソースに対して同じ設定を決定します。`--max-reconcile-rate` を Crossplane に適用することは、コア Crossplane リソースのレートのみを制御します。
{{< /hint >}}

##### リアルタイムコンポジションの有効化

リアルタイムコンポジションが有効になっていると、Crossplane はすべての構成リソースを Kubernetes ウォッチで監視します。Crossplane は、構成リソースが変更されたときに Kubernetes API サーバーからイベントを受け取ります。たとえば、プロバイダーが `Ready` 条件を `true` に設定したときです。

{{<hint "important" >}}
リアルタイムコンポジションはアルファ機能です。アルファ機能はデフォルトでは有効になっていません。
{{< /hint >}}

リアルタイムコンポジションが有効になっている場合、Crossplane は `--poll-interval` 設定を使用しません。

リアルタイムコンポジションのサポートを有効にするには、  
[changing the Crossplane pod setting]({{<ref "./pods#change-pod-settings">}})  
を行い、  
{{<hover label="deployment" line="12">}}--enable-realtime-compositions{{</hover>}}  
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
        - --enable-realtime-compositions
```

{{<hint "tip" >}}

[Crossplane install guide]({{<ref "../software/install#feature-flags">}}) では、Helm を使用して  
{{<hover label="deployment" line="12">}}--enable-realtime-compositions{{</hover>}}  
のような機能フラグを有効にする方法が説明されています。
{{< /hint >}}

##### 調整リトライレート

`--max-reconcile-rate` 設定は、Crossplane またはプロバイダーがリソースを修正しようとする回数（秒あたりの回数）を構成します。デフォルト値は 10 回/秒です。

すべてのコア Crossplane コンポーネントは、調整レートを共有します。各プロバイダーは独自の最大調整レート設定を実装しています。

##### リコンシリエーションの数

2つ目の値 `--max-reconcile-rate` が定義するのは、Crossplane が一度にリコンシリエーションできるリソースの数です。設定された `--max-reconcile-rate` よりも多くのリソースがある場合、残りのリソースは Crossplane が既存のリソースをリコンシリエーションするまで待たなければなりません。

これらの設定を適用する手順については、[Change Pod Settings]({{<ref "#change-pod-settings">}}) セクションをお読みください。

<!-- vale Microsoft.HeadingAcronyms = NO -->
<!-- allow 'RBAC' since that's the name -->
## RBAC マネージャーポッド
<!-- vale Microsoft.HeadingAcronyms = YES -->
Crossplane RBAC マネージャーポッドは、Crossplane と Crossplane プロバイダーのために必要な Kubernetes RBAC 権限を自動化します。

{{<hint "note" >}}
Crossplane はデフォルトで RBAC マネージャーをインストールし、有効にします。
RBAC マネージャーを無効にするには、Crossplane の適切な操作のために手動で Kubernetes 権限の定義が必要です。

[RBAC マネージャー設計文書](https://github.com/crossplane/crossplane/blob/master/design/design-doc-rbac-manager.md) 
は、Crossplane の RBAC 要件に関するより包括的な詳細を提供します。
{{< /hint >}}

### RBAC マネージャーの無効化

インストール後に RBAC マネージャーを無効にするには、`crossplane-system` 名前空間から `crossplane-rbac-manager` デプロイメントを削除します。

インストール前に RBAC マネージャーを無効にするには、Helm の `values.yaml` ファイルを編集し、`rbacManager.deploy` を `false` に設定します。

{{< hint "note" >}}

インストール中に Crossplane ポッド設定を変更する手順は、[Crossplane Install]({{<ref "../software/install">}}) セクションにあります。
{{< /hint >}}

<!-- vale Microsoft.HeadingAcronyms = NO -->
<!-- allow 'RBAC' since that's the name -->
### RBAC 初期コンテナ
<!-- vale Microsoft.HeadingAcronyms = YES -->

RBAC マネージャーは、開始する前に `CompositeResourceDefinition` と `ProviderRevision` リソースが利用可能である必要があります。

RBAC マネージャー初期コンテナは、メインの RBAC マネージャーコンテナが開始する前にこれらのリソースを待機します。

### RBAC マネージャーコンテナ

RBAC マネージャーコンテナは以下のタスクを実行します：
* プロバイダー ServiceAccount に RBAC ロールを作成し、バインドして、管理リソースを制御できるようにする
* `crossplane` ServiceAccount に管理リソースを作成する権限を与える
* すべての名前空間で Crossplane リソースにアクセスするための ClusterRoles を作成する


[ClusterRoles]({{<ref "#crossplane-clusterroles">}})を使用して、クラスター内のすべてのCrossplaneリソースへのアクセスを付与します。  

#### Crossplane ClusterRoles

RBACマネージャーは、4つのKubernetes ClusterRolesを作成します。これらのロールは、クラスター全体のCrossplaneリソースに対する権限を付与します。 

<!-- vale Google.Headings = NO -->
<!-- disable heading checking for the role names -->
<!-- vale Google.WordList = NO -->
<!-- allow "admin" -->
##### crossplane-admin
<!-- vale Google.WordList = YES -->
<!-- vale Crossplane.Spelling = NO -->
`crossplane-admin` ClusterRoleには、以下の権限があります：
  * すべてのCrossplaneタイプへの完全なアクセス
  * すべてのシークレットおよび名前空間への完全なアクセス（Crossplaneに関連しないものも含む）
  * すべてのクラスターRBACロール、CustomResourceDefinitions、およびイベントへの読み取り専用アクセス
  * 他のエンティティにRBACロールをバインドする能力。 
<!-- vale Crossplane.Spelling = YES -->
完全なRBACポリシーを表示するには、 

```shell
kubectl describe clusterrole crossplane-admin
```

##### crossplane-edit

`crossplane-edit` ClusterRoleには、以下の権限があります：

  * すべてのCrossplaneタイプへの完全なアクセス
  * すべてのシークレットへの完全なアクセス（Crossplaneに関連しないものも含む）
  * すべての名前空間およびイベントへの読み取り専用アクセス（Crossplaneに関連しないものも含む）。

完全なRBACポリシーを表示するには、 

```shell
kubectl describe clusterrole crossplane-edit
```

##### crossplane-view

`crossplane-view` ClusterRoleには、以下の権限があります：

  * すべてのCrossplaneタイプへの読み取り専用アクセス
  * すべての名前空間およびイベントへの読み取り専用アクセス（Crossplaneに関連しないものも含む）。

完全なRBACポリシーを表示するには、 

```shell
kubectl describe clusterrole crossplane-view
```

##### crossplane-browse

`crossplane-browse` ClusterRoleには、以下の権限があります：

  * Crossplaneの構成およびXRDへの読み取り専用アクセス。これにより、リソースクレーム作成者は適切な構成を発見し、選択することができます。

完全なRBACポリシーを表示するには、 

```shell
kubectl describe clusterrole crossplane-browse
```

## リーダー選挙

デフォルトでは、クラスター内で単一のCrossplaneポッドのみが実行されます。複数のCrossplaneポッドが実行されると、両方のポッドがCrossplaneリソースを管理しようとします。競合を防ぐために、Crossplaneは_リーダー選挙_を使用して、同時に制御するポッドを1つだけにします。他のCrossplaneポッドは、リーダーが失敗するまで待機します。

{{< hint "note" >}}
複数のCrossplaneまたはRBACマネージャーポッドを冗長性のために実行することが可能です。

Kubernetesは、失敗したCrossplaneまたはRBACマネージャーポッドを再起動します。
冗長ポッドはほとんどのデプロイメントでは必要ありません。
{{< /hint >}}

CrossplaneポッドとRBACマネージャーポッドの両方はリーダー選挙をサポートしています。

`--leader-election`ポッド引数を使用してリーダー選挙を有効にします。

{{< hint "warning" >}}
<!-- vale write-good.TooWordy = NO -->
<!-- "multiple" -->
<!-- vale write-good.Passive = NO -->
<!-- allow "is unsupported" --> 
リーダー選挙なしで複数のCrossplaneポッドを実行することはサポートされていません。
<!-- vale write-good.Passive = YES -->
<!-- vale write-good.TooWordy = YES -->
{{< /hint >}}


## ポッド設定の変更

Crossplaneポッドの設定は、Helmの`values.yml`ファイルを編集してCrossplaneをインストールする前に、またはインストール後に`Deployment`を編集して変更できます。

[構成オプション]({{<ref "../software/install#customize-the-crossplane-helm-chart">}}) 
および 
[機能フラグ]({{<ref "../software/install#customize-the-crossplane-helm-chart">}}) 
の完全なリストは、 
[Crossplane Install]({{<ref "../software/install">}}) 
セクションで入手できます。 

{{< hint "note" >}}

インストール中にCrossplaneポッドの設定を変更する手順は、 
[Crossplane Install]({{<ref "../software/install">}}) 
セクションにあります。 
{{< /hint >}}

### デプロイメントの編集
{{< hint "note" >}}
これらの設定は、`crossplane`および`rbac-manager`ポッドと`Deployments`の両方に適用されます。
{{< /hint >}}

インストールされたCrossplaneポッドの設定を変更するには、次のコマンドで`crossplane-system`名前空間内の`crossplane`デプロイメントを編集します。

`kubectl edit deployment crossplane --namespace crossplane-system`

{{< hint "warning" >}}
Crossplaneデプロイメントを更新すると、Crossplaneポッドが再起動します。
{{< /hint >}}

Crossplaneポッド引数を 
{{<hover label="args" line="9" >}}spec.template.spec.containers[].args{{< /hover >}}
デプロイメントのセクションに追加します。

例えば、`sync-interval`を変更するには、 
{{<hover label="args" line="12" >}}--sync-interval=30m{{< /hover >}}を追加します。

```yaml {label="args", copy-lines="1"}
kubectl edit deployment crossplane --namespace crossplane-system
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
        - --sync-interval=30m
```

### 環境変数の使用

コアCrossplaneポッドは、起動時に設定された環境変数をチェックして
デフォルト設定を変更します。

設定可能な環境変数の完全なリストは、 
[Crossplane Install]({{<ref "../software/install">}}) セクションで利用できます。
