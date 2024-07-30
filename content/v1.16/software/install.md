---
title: Crossplaneのインストール
weight: 100
---

Crossplaneは既存のKubernetesクラスターにインストールされ、`Crossplane`ポッドを作成し、Crossplane _Provider_ リソースのインストールを有効にします。

{{< hint type="tip" >}}
Kubernetesクラスターを持っていない場合は、[Kind](https://kind.sigs.k8s.io/)を使用してローカルに作成してください。
{{< /hint >}}

## 前提条件
* アクティブな[サポートされているKubernetesバージョン](https://kubernetes.io/releases/patch-releases/#support-period)
* [Helm](https://helm.sh/docs/intro/install/) バージョン `v3.2.0` 以上

## Crossplaneのインストール

Crossplaneが公開した_Helmチャート_を使用してCrossplaneをインストールします。


### Crossplane Helmリポジトリの追加

`helm repo add`コマンドを使用してCrossplaneリポジトリを追加します。

```shell
helm repo add crossplane-stable https://charts.crossplane.io/stable
```

`helm repo update`を使用してローカルのHelmチャートキャッシュを更新します。
```shell
helm repo update
```

### Crossplane Helmチャートのインストール

`helm install`を使用してCrossplane Helmチャートをインストールします。

{{< hint "tip" >}}
`helm install --dry-run --debug`オプションを使用して、Crossplaneがクラスターに加える変更を確認します。HelmはKubernetesクラスターに変更を加えずに、適用される設定を表示します。
{{< /hint >}}

Crossplaneは`crossplane-system`ネームスペースに作成され、インストールされます。

```shell
helm install crossplane \
--namespace crossplane-system \
--create-namespace crossplane-stable/crossplane 
```

`kubectl get pods -n crossplane-system`を使用してインストールされたCrossplaneポッドを表示します。

```shell {copy-lines="1"}
kubectl get pods -n crossplane-system
NAME                                       READY   STATUS    RESTARTS   AGE
crossplane-6d67f8cd9d-g2gjw                1/1     Running   0          26m
crossplane-rbac-manager-86d9b5cf9f-2vc4s   1/1     Running   0          26m
```

{{< hint "tip" >}}
`--version <version>`オプションを使用して特定のバージョンのCrossplaneをインストールします。たとえば、バージョン`1.10.0`をインストールするには：

```shell
helm install crossplane \
--namespace crossplane-system \
--create-namespace crossplane-stable/crossplane \
--version 1.10.0
```
{{< /hint >}}



## インストールされたデプロイメント
Crossplaneは`crossplane-system`ネームスペースに2つのKubernetes _デプロイメント_ を作成し、Crossplaneポッドをデプロイします。

```shell {copy-lines="1"}
kubectl get deployments -n crossplane-system
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
crossplane                1/1     1            1           8m13s
crossplane-rbac-manager   1/1     1            1           8m13s
```

### Crossplaneデプロイメント
Crossplaneデプロイメントは`crossplane-init container`から始まります。`init`コンテナはKubernetesクラスターにCrossplane _カスタムリソース定義_ をインストールします。

```
`init` コンテナが終了すると、`crossplane` ポッドは 2 つの Kubernetes コントローラーを管理します。
* _パッケージ マネージャー コントローラー_ は、プロバイダー、関数、および構成パッケージをインストールします。
* _コンポジション コントローラー_ は、Crossplane _コンポジット リソース定義_、_コンポジション_、および _クレーム_ をインストールおよび管理します。

### Crossplane RBAC マネージャーのデプロイメント
`crossplane-rbac-manager` は、インストールされた Crossplane _プロバイダー_ およびその _カスタム リソース定義_ のための Kubernetes _ClusterRoles_ を作成および管理します。

[Crossplane RBAC マネージャー設計文書](https://github.com/crossplane/crossplane/blob/master/design/design-doc-rbac-manager.md) には、インストールされた _ClusterRoles_ に関する詳細情報があります。

## インストールオプション

### Crossplane Helm チャートのカスタマイズ
Crossplane は、Helm チャートを構成することによって、インストール時のカスタマイズをサポートします。

コマンドラインまたは Helm _values_ ファイルを使用してカスタマイズを適用します。

<!-- Generated from Helm README at https://github.com/crossplane/crossplane/blob/master/cluster/charts/crossplane/README.md -->
<!-- vale gitlab.Substitutions = NO -->
<!-- allow lowercase yaml -->
{{<expand "All Crossplane customization options" >}}
{{< table "table table-hover table-striped table-sm">}}
| パラメーター | 説明 | デフォルト |
| --- | --- | --- |
| `affinity` | Crossplane ポッドデプロイメントに `affinities` を追加します。 | `{}` |
| `args` | Crossplane ポッドにカスタム引数を追加します。 | `[]` |
| `configuration.packages` | インストールする構成パッケージのリスト。 | `[]` |
| `customAnnotations` | Crossplane ポッドデプロイメントにカスタム `annotations` を追加します。 | `{}` |
| `customLabels` | Crossplane ポッドデプロイメントにカスタム `labels` を追加します。 | `{}` |
| `deploymentStrategy` | Crossplane および RBAC マネージャーポッドのデプロイメント戦略。 | `"RollingUpdate"` |
| `extraEnvVarsCrossplane` | Crossplane ポッドデプロイメントにカスタム環境変数を追加します。変数名の `.` を `_` に置き換えます。たとえば、`SAMPLE.KEY=value1` は `SAMPLE_KEY=value1` になります。 | `{}` |
| `extraEnvVarsRBACManager` | RBAC マネージャーポッドデプロイメントにカスタム環境変数を追加します。変数名の `.` を `_` に置き換えます。たとえば、`SAMPLE.KEY=value1` は `SAMPLE_KEY=value1` になります。 | `{}` |
| `extraObjects` | Helm インストール中に任意の Kubernetes オブジェクトを追加します | `[]` |
| `extraVolumeMountsCrossplane` | Crossplane ポッドにカスタム `volumeMounts` を追加します。 | `{}` |
| `extraVolumesCrossplane` | Crossplane ポッドにカスタム `volumes` を追加します。 | `{}` |
| `function.packages` | インストールする関数パッケージのリスト。 | `[]` |
| `hostNetwork` | Crossplane デプロイメントのために `hostNetwork` を有効にします。注意: `hostNetwork` を有効にすると、Crossplane ポッドがホストネットワーク名前空間にアクセスできるようになります。 | `false` |
| `image.pullPolicy` | Crossplane および RBAC マネージャーポッドに使用されるイメージプルポリシー。 | `"IfNotPresent"` |
| `image.repository` | Crossplane ポッドイメージのリポジトリ。 | `"xpkg.upbound.io/crossplane/crossplane"` |
| `image.tag` | Crossplane イメージタグ。デフォルトは `Chart.yaml` の `appVersion` の値です。 | `""` |
| `imagePullSecrets` | Crossplane ServiceAccount に追加する imagePullSecret 名。 | `{}` |
| `leaderElection` | Crossplane ポッドのために [リーダー選挙](https://docs.crossplane.io/latest/concepts/pods/#leader-election) を有効にします。 | `true` |
| `metrics.enabled` | Prometheus パス、ポート、およびスクレイプアノテーションを有効にし、Crossplane および RBAC マネージャーポッドの両方にポート 8080 を公開します。 | `false` |
| `nodeSelector` | Crossplane ポッドデプロイメントに `nodeSelectors` を追加します。 | `{}` |
| `packageCache.configMap` | パッケージキャッシュとして使用する ConfigMap の名前。デフォルトのパッケージキャッシュ `emptyDir` ボリュームを無効にします。 | `""` |
| `packageCache.medium` | パッケージキャッシュを RAM バックのファイルシステムに保持するために `Memory` に設定します。Crossplane 開発に便利です。 | `""` |
| `packageCache.pvc` | パッケージキャッシュとして使用する PersistentVolumeClaim の名前。デフォルトのパッケージキャッシュ `emptyDir` ボリュームを無効にします。 | `""` |
| `packageCache.sizeLimit` | パッケージキャッシュのサイズ制限。medium が `Memory` の場合、`sizeLimit` はノードメモリを超えることはできません。 | `"20Mi"` |
| `podSecurityContextCrossplane` | Crossplane ポッドにカスタム `securityContext` を追加します。 | `{}` |
| `podSecurityContextRBACManager` | RBAC マネージャーポッドにカスタム `securityContext` を追加します。 | `{}` |
| `priorityClassName` | Crossplane および RBAC マネージャーポッドに適用する PriorityClass 名。 | `""` |
| `provider.packages` | インストールするプロバイダーパッケージのリスト。 | `[]` |
| `rbacManager.affinity` | RBAC マネージャーポッドデプロイメントに `affinities` を追加します。 | `{}` |
| `rbacManager.args` | RBAC マネージャーポッドにカスタム引数を追加します。 | `[]` |
| `rbacManager.deploy` | RBAC マネージャーポッドとその必要なロールをデプロイします。 | `true` |
| `rbacManager.leaderElection` | RBAC マネージャーポッドのために [リーダー選挙](https://docs.crossplane.io/latest/concepts/pods/#leader-election) を有効にします。 | `true` |
| `rbacManager.nodeSelector` | RBAC マネージャーポッドデプロイメントに `nodeSelectors` を追加します。 | `{}` |
| `rbacManager.replicas` | デプロイする RBAC マネージャーポッドの `replicas` の数。 | `1` |
| `rbacManager.skipAggregatedClusterRoles` | 集約された Crossplane ClusterRoles をインストールしません。 | `false` |
| `rbacManager.tolerations` | RBAC マネージャーポッドデプロイメントに `tolerations` を追加します。 | `[]` |
| `registryCaBundleConfig.key` | 不明または信頼できない証明書を持つレジストリからパッケージを取得できるようにするカスタム CA バンドルを含む ConfigMap キー。 | `""` |
| `registryCaBundleConfig.name` | 不明または信頼できない証明書を持つレジストリからパッケージを取得できるようにするカスタム CA バンドルを含む ConfigMap 名。 | `""` |
| `replicas` | デプロイする Crossplane ポッドの `replicas` の数。 | `1` |
| `resourcesCrossplane.limits.cpu` | Crossplane ポッドの CPU リソース制限。 | `"100m"` |
| `resourcesCrossplane.limits.memory` | Crossplane ポッドのメモリリソース制限。 | `"512Mi"` |
| `resourcesCrossplane.requests.cpu` | Crossplane ポッドの CPU リソース要求。 | `"100m"` |
| `resourcesCrossplane.requests.memory` | Crossplane ポッドのメモリリソース要求。 | `"256Mi"` |
| `resourcesRBACManager.limits.cpu` | RBAC マネージャーポッドの CPU リソース制限。 | `"100m"` |
| `resourcesRBACManager.limits.memory` | RBAC マネージャーポッドのメモリリソース制限。 | `"512Mi"` |
| `resourcesRBACManager.requests.cpu` | RBAC マネージャーポッドの CPU リソース要求。 | `"100m"` |
| `resourcesRBACManager.requests.memory` | RBAC マネージャーポッドのメモリリソース要求。 | `"256Mi"` |
| `securityContextCrossplane.allowPrivilegeEscalation` | Crossplane ポッドのために `allowPrivilegeEscalation` を有効にします。 | `false` |
| `securityContextCrossplane.readOnlyRootFilesystem` | Crossplane ポッドのルートファイルシステムを読み取り専用として設定します。 | `true` |
| `securityContextCrossplane.runAsGroup` | Crossplane ポッドで使用されるグループ ID。 | `65532` |
| `securityContextCrossplane.runAsUser` | Crossplane ポッドで使用されるユーザー ID。 | `65532` |
| `securityContextRBACManager.allowPrivilegeEscalation` | RBAC マネージャーポッドのために `allowPrivilegeEscalation` を有効にします。 | `false` |
| `securityContextRBACManager.readOnlyRootFilesystem` | RBAC マネージャーポッドのルートファイルシステムを読み取り専用として設定します。 | `true` |
| `securityContextRBACManager.runAsGroup` | RBAC マネージャーポッドで使用されるグループ ID。 | `65532` |
| `securityContextRBACManager.runAsUser` | RBAC マネージャーポッドで使用されるユーザー ID。 | `65532` |
| `serviceAccount.customAnnotations` | Crossplane ServiceAccount にカスタム `annotations` を追加します。 | `{}` |
| `tolerations` | Crossplane ポッドデプロイメントに `tolerations` を追加します。 | `[]` |
| `webhooks.enabled` | Crossplane およびインストールされたプロバイダーパッケージのためにウェブフックを有効にします。 | `true` |
{{< /table >}}
{{< /expand >}}
<!-- vale gitlab.Substitutions = YES -->
```

#### コマンドラインのカスタマイズ

コマンドラインでカスタム設定を適用するには 
`helm install crossplane --set <setting>=<value>` を使用します。

例えば、イメージプルポリシーを変更するには：

```shell
helm install crossplane \
--namespace crossplane-system \
--create-namespace \
crossplane-stable/crossplane \
--set image.pullPolicy=Always
```

Helmはカンマ区切りの引数をサポートしています。

例えば、イメージプルポリシーとレプリカの数を変更するには：

```shell
helm install crossplane \
--namespace crossplane-system \
--create-namespace \
crossplane-stable/crossplane \
--set image.pullPolicy=Always,replicas=2
```

#### Helm値ファイル

Helm _values_ ファイルでカスタム設定を適用するには 
`helm install crossplane -f <filename>` を使用します。

YAMLファイルがカスタマイズされた設定を定義します。

例えば、イメージプルポリシーとレプリカの数を変更するには：

カスタマイズされた設定を持つYAMLを作成します。

```yaml
replicas: 2

image:
  pullPolicy: Always
```

`helm install` でファイルを適用します：

```shell
helm install crossplane \
--namespace crossplane-system \
--create-namespace \
crossplane-stable/crossplane \
-f settings.yaml
```

#### フィーチャーフラグ

Crossplaneはフィーチャーフラグの背後に新しい機能を導入します。デフォルトでは
アルファ機能はオフになっています。Crossplaneはデフォルトでベータ機能を有効にします。 
フィーチャーフラグを有効にするには、Helmチャートの `args` 値を設定します。利用可能なフィーチャーフラグは
`crossplane core start --help` を実行することで直接見つけることができるか、以下の表を参照してください。

{{< expand "フィーチャーフラグ" >}}
{{< table caption="フィーチャーフラグ" >}}
| ステータス | フラグ | 説明 |
| --- | --- | --- |
| ベータ | `--enable-composition-functions` | コンポジション関数のサポートを有効にします。 |
| ベータ | `--enable-composition-functions-extra-resources` | コンポジション関数の追加リソースのサポートを有効にします。 `--enable-composition-functions` が有効な場合のみ尊重されます。 |
| ベータ | `--enable-composition-webhook-schema-validation` | スキーマを使用したコンポジションの検証を有効にします。 |
| ベータ | `--enable-deployment-runtime-configs` | DeploymentRuntimeConfigsのサポートを有効にします。 |
| アルファ | `--enable-environment-configs` | EnvironmentConfigsのサポートを有効にします。 |
| アルファ | `--enable-external-secret-stores` | 外部シークレットストアのサポートを有効にします。 |
| アルファ | `--enable-realtime-compositions` | リアルタイムコンポジションのサポートを有効にします。 |
| アルファ | `--enable-ssa-claims` | サーバーサイドアプライを使用してXRsとクレームを同期するサポートを有効にします。 |
| アルファ | `--enable-usages` | 使用のサポートを有効にします。 |
{{< /table >}}
{{< /expand >}}

```
これらのフラグを `values.yaml` ファイル内またはインストール時に `--set` フラグを使用して設定します。例えば: `--set
args='{"--enable-composition-functions","--enable-composition-webhook-schema-validation"}'`。

#### デフォルトのパッケージレジストリを変更する

Crossplane バージョン 1.15.0 以降、Crossplane は DockerHub の代わりに `xpkg.upbound.io` の [Upbound Marketplace](https://marketplace.upbound.io) からパッケージをダウンロードします。

Crossplane のインストール中にデフォルトのレジストリの場所を変更するには、`--set args='{"--registry=index.docker.io"}'` を使用します。

### プレリリースの Crossplane バージョンをインストールする
`master` Crossplane Helm チャンネルからプレリリースバージョンの Crossplane をインストールします。

`master` チャンネルのバージョンはアクティブに開発中であり、不安定な場合があります。

{{< hint "warning" >}}
本番環境で Crossplane `master` リリースを使用しないでください。`stable` チャンネルのみを使用してください。  
テストと開発には `master` のみを使用してください。
{{< /hint >}}

#### Crossplane master Helm リポジトリを追加する

`helm repo add` コマンドを使用して Crossplane リポジトリを追加します。

```shell
helm repo add crossplane-master https://charts.crossplane.io/master/
```

ローカル Helm チャートキャッシュを `helm repo update` で更新します。
```shell
helm repo update
```

#### Crossplane master Helm チャートをインストールする

`helm install` を使用して Crossplane `master` Helm チャートをインストールします。

{{< hint "tip" >}}
`helm install --dry-run --debug` オプションを使用して、Crossplane がクラスターに加える変更を確認します。Helm は Kubernetes クラスターに変更を加えずに、適用される設定を表示します。
{{< /hint >}}

Crossplane は `crossplane-system` 名前空間に作成してインストールします。

```shell
helm install crossplane \
--namespace crossplane-system \
--create-namespace crossplane-master/crossplane \
--devel 
```

## Crossplane ディストリビューション
サードパーティのベンダーは独自の Crossplane ディストリビューションを維持する場合があります。ベンダーがサポートするディストリビューションには、コミュニティの Crossplane ディストリビューションにはない機能やツールが含まれている場合があります。

CNCF によって認定されたサードパーティのディストリビューションは、コミュニティの Crossplane ディストリビューションと "[準拠](https://github.com/cncf/crossplane-conformance)" しています。

### ベンダー
以下は、準拠した Crossplane ディストリビューションを提供しているベンダーです。
```

#### Upbound
Upboundは、Crossplaneの創設者であり、Crossplaneの無料でオープンソースの配布版である 
[Universal Crossplane](https://www.upbound.io/product/universal-crossplane)
（`UXP`）を維持しています。 

UXPに関する情報は、 
[Upbound UXPドキュメント](https://docs.upbound.io/uxp/install/)で見つけることができます。
