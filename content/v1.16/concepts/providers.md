---
title: プロバイダー
weight: 5
description: "プロバイダーはCrossplaneを外部APIに接続します"
---

プロバイダーはCrossplaneが外部サービス上にインフラストラクチャをプロビジョニングできるようにします。プロバイダーは新しいKubernetes APIを作成し、それを外部APIにマッピングします。

プロバイダーは非Kubernetesリソースへの接続に関するすべての側面を担当します。これには、認証、外部API呼び出しの実行、および外部リソースに対する
[Kubernetesコントローラー](https://kubernetes.io/docs/concepts/architecture/controller/)
ロジックの提供が含まれます。

プロバイダーの例には以下が含まれます：

* [Provider AWS](https://github.com/upbound/provider-aws)
* [Provider Azure](https://github.com/upbound/provider-azure)
* [Provider GCP](https://github.com/upbound/provider-gcp)
* [Provider Kubernetes](https://github.com/crossplane-contrib/provider-kubernetes)

{{< hint "tip" >}}
[Upbound Marketplace](https://marketplace.upbound.io)で他のプロバイダーを見つけてください。
{{< /hint >}}

<!-- vale write-good.Passive = NO -->
<!-- "are Managed" isn't passive in this context -->
プロバイダーは、Kubernetesで作成できるすべての外部リソースをKubernetes APIエンドポイントとして定義します。  
これらのエンドポイントは
[_Managed Resources_]({{<ref "managed-resources" >}})です。
<!-- vale write-good.Passive = YES -->


## プロバイダーのインストール

プロバイダーをインストールすると、プロバイダーのAPIを表す新しいKubernetesリソースが作成されます。プロバイダーをインストールすると、プロバイダーのAPIをKubernetesクラスターに調整する責任を持つプロバイダーポッドも作成されます。プロバイダーは常に希望する管理リソースの状態を監視し、欠落している外部リソースを作成します。

プロバイダーをインストールするには、Crossplane
{{<hover label="install" line="2">}}Provider{{</hover >}}オブジェクトを使用し、
{{<hover label="install" line="6">}}spec.package{{</hover >}}の値をプロバイダー パッケージの場所に設定します。

{{< hint "important" >}}
Crossplaneバージョン1.15.0以降、Crossplaneはデフォルトで`xpkg.upbound.io`のUpbound Marketplace
Crossplaneパッケージレジストリを使用してパッケージをダウンロードおよびインストールします。

`package`で完全なドメイン名を指定するか、[Crossplaneポッド]({{<ref "./pods">}})で`--registry`フラグを使用してデフォルトのCrossplaneレジストリを変更します。
{{< /hint >}}

例えば、[AWS Community Provider](https://github.com/crossplane-contrib/provider-aws)をインストールするには、

```yaml {label="install"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v0.39.0
```

デフォルトでは、ProviderポッドはCrossplaneと同じ名前空間（`crossplane-system`）にインストールされます。

{{<hint "note" >}}
Providersは{{<hover label="install" line="1">}}pkg.crossplane.io{{</hover>}}グループの一部です。  

{{<hover label="meta-pkg" line="1">}}meta.pkg.crossplane.io{{</hover>}}グループはProviderパッケージを作成するためのものです。 

Providerのビルドに関する指示はこの文書の範囲外です。  
詳細については、Crossplaneの貢献ガイド[Provider Development Guide](https://github.com/crossplane/crossplane/blob/master/contributing/guide-provider-development.md)をお読みください。

Providerパッケージの仕様については、[Crossplane Provider Package specification](https://github.com/crossplane/crossplane/blob/master/contributing/specifications/xpkg.md#provider-package-requirements)をお読みください。

```yaml {label="meta-pkg"}
apiVersion: meta.pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
# Removed for brevity
```
{{</hint >}}

### Helmでインストール

Crossplaneは、Crossplane Helmチャートを使用して初期のCrossplaneインストール中にProvidersをインストールすることをサポートしています。

`helm install`で{{<hover label="helm" line="5" >}}--set provider.packages{{</hover >}}引数を使用します。

例えば、AWS Community Providerをインストールするには、

```shell {label="helm"}
helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace \
--set provider.packages='{xpkg.upbound.io/crossplane-contrib/provider-aws:v0.39.0}'
```

### オフラインでインストール

Crossplane Providersをオフラインでインストールするには、Providerパッケージをホストするための[Harbor](https://goharbor.io/)のようなローカルコンテナレジストリが必要です。CrossplaneはコンテナレジストリからのProviderパッケージのインストールのみをサポートしています。

CrossplaneはKubernetesボリュームから直接Providerパッケージをインストールすることをサポートしていません。

### インストールオプション

Providersは、インストール関連の設定を変更するための複数の構成オプションをサポートしています。

#### Providerプルポリシー

{{<hover label="pullpolicy" line="6">}}packagePullPolicy{{</hover>}}を使用して、CrossplaneがProviderパッケージをローカルのCrossplaneパッケージキャッシュにダウンロードするタイミングを定義します。

`packagePullPolicy` オプションは次のとおりです：
* `IfNotPresent` - (**デフォルト**) キャッシュにパッケージがない場合のみ、パッケージをダウンロードします。
* `Always` - 毎分新しいパッケージをチェックし、キャッシュにない一致するパッケージをダウンロードします。
* `Never` - パッケージを決してダウンロードしません。パッケージはローカルパッケージキャッシュからのみインストールされます。

{{<hint "tip" >}}
Crossplane 
{{<hover label="pullpolicy" line="6">}}packagePullPolicy{{</hover>}} は Kubernetes コンテナイメージの 
[image pull policy](https://kubernetes.io/docs/concepts/containers/images/#image-pull-policy) のように機能します。  

Crossplane は Kubernetes イメージのようにタグとパッケージダイジェストハッシュの使用をサポートしています。
{{< /hint >}}

たとえば、特定のプロバイダー パッケージを `Always` ダウンロードするには、 
{{<hover label="pullpolicy" line="6">}}packagePullPolicy: Always{{</hover>}} 
設定を使用します。

```yaml {label="pullpolicy",copy-lines="6"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  packagePullPolicy: Always
# Removed for brevity
```

#### リビジョンアクティベーションポリシー

`Active` パッケージリビジョンは、パッケージコントローラーがリソースを積極的に調整している状態です。

デフォルトでは、Crossplane は最も最近インストールされたパッケージリビジョンを `Active` として設定します。

プロバイダーのアップグレード動作を制御するには、 
{{<hover label="revision" line="6">}}revisionActivationPolicy{{</hover>}} を使用します。

{{<hover label="revision" line="6">}}revisionActivationPolicy{{</hover>}} 
オプションは次のとおりです：
* `Automatic` - (**デフォルト**) 最後にインストールされたプロバイダーを自動的にアクティブ化します。
* `Manual` - プロバイダーを自動的にアクティブ化しません。

たとえば、アップグレード動作を手動アップグレードを必要とするように変更するには、 
{{<hover label="revision" line="6">}}revisionActivationPolicy: Manual{{</hover>}} を設定します。

```yaml {label="revision"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  revisionActivationPolicy: Manual
# Removed for brevity
```

#### パッケージリビジョン履歴制限

Crossplane が同じプロバイダー パッケージの異なるバージョンをインストールすると、Crossplane は新しい _revision_ を作成します。

デフォルトでは、Crossplane は 1 つの _Inactive_ リビジョンを維持します。

{{<hint "note" >}}
パッケージリビジョンの使用に関する詳細情報は、[プロバイダーのアップグレード](#upgrade-a-provider) セクションをお読みください。
{{< /hint >}}


プロバイダーパッケージでCrossplaneが保持するリビジョンの数を変更します  
{{<hover label="revHistoryLimit" line="6">}}revisionHistoryLimit{{</hover>}}。  

{{<hover label="revHistoryLimit" line="6">}}revisionHistoryLimit{{</hover>}}  
フィールドは整数です。  
デフォルト値は `1` です。  
{{<hover label="revHistoryLimit" line="6">}}revisionHistoryLimit{{</hover>}} を `0` に設定することで  
リビジョンの保存を無効にします。  

例えば、デフォルト設定を変更して10のリビジョンを保存するには  
{{<hover label="revHistoryLimit" line="6">}}revisionHistoryLimit: 10{{</hover>}} を使用します。  

```yaml {label="revHistoryLimit"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  revisionHistoryLimit: 10
# Removed for brevity
```

#### プライベートレジストリからプロバイダーをインストールする

Kubernetesが `imagePullSecrets` を使用して  
[プライベートレジストリからイメージをインストールする](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)のと同様に、  
Crossplaneは `packagePullSecrets` を使用してプライベートレジストリからプロバイダーパッケージをインストールします。  

{{<hover label="pps" line="6">}}packagePullSecrets{{</hover>}} を使用して、  
プロバイダーパッケージをダウンロードする際の認証に使用するKubernetesシークレットを提供します。  

{{<hint "important" >}}  
KubernetesシークレットはCrossplaneと同じ名前空間に存在する必要があります。  
{{</hint >}}  

{{<hover label="pps" line="6">}}packagePullSecrets{{</hover>}} はシークレットのリストです。  

例えば、  
{{<hover label="pps" line="6">}}example-secret{{</hover>}} という名前のシークレットを使用するには、  
{{<hover label="pps" line="6">}}packagePullSecrets{{</hover>}} を設定します。  

```yaml {label="pps"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  packagePullSecrets: 
    - name: example-secret
# Removed for brevity
```

{{<hint "note" >}}  
設定された `packagePullSecrets` は、  
いかなるプロバイダーパッケージの依存関係にも渡されません。  
{{< /hint >}}  

#### 依存関係を無視する

デフォルトでは、Crossplaneはプロバイダーパッケージにリストされた  
[依存関係](#manage-dependencies)をインストールします。  

Crossplaneは、  
{{<hover label="pkgDep" line="6" >}}skipDependencyResolution{{</hover>}} を使用して  
プロバイダーパッケージの依存関係を無視できます。  

例えば、依存関係の解決を無効にするには、  
{{<hover label="pkgDep" line="6" >}}skipDependencyResolution: true{{</hover>}} を設定します。

```yaml {label="pkgDep"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  skipDependencyResolution: true
# Removed for brevity
```

#### Crossplaneのバージョン要件を無視する

プロバイダーパッケージは、インストール前に特定のまたは最小のCrossplaneバージョンを必要とする場合があります。デフォルトでは、CrossplaneはCrossplaneバージョンが必要なバージョンを満たさない場合、プロバイダーをインストールしません。

Crossplaneは、{{<hover label="xpVer" line="6">}}ignoreCrossplaneConstraints{{</hover>}}を使用して、必要なバージョンを無視できます。

たとえば、サポートされていないCrossplaneバージョンにプロバイダーパッケージをインストールするには、{{<hover label="xpVer" line="6">}}ignoreCrossplaneConstraints: true{{</hover>}}を設定します。

```yaml {label="xpVer"}
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  ignoreCrossplaneConstraints: true
# Removed for brevity
```

### 依存関係の管理

プロバイダーパッケージには、他のパッケージや構成、または他のプロバイダーへの依存関係が含まれる場合があります。

Crossplaneがプロバイダーパッケージの依存関係を満たせない場合、プロバイダーは`HEALTHY`を`False`として報告します。

たとえば、Upbound AWSリファレンスプラットフォームのこのインストールは`HEALTHY: False`です。

```shell {copy-lines="1"}
kubectl get providers
NAME              INSTALLED   HEALTHY   PACKAGE                                           AGE
provider-aws-s3   True        False     xpkg.upbound.io/upbound/provider-aws-s3:v0.41.0   12s
```

プロバイダーが`HEALTHY`でない理由に関する詳細情報を表示するには、{{<hover label="depend" line="1">}}kubectl describe providerrevisions{{</hover>}}を使用します。

```yaml {copy-lines="1",label="depend"}
kubectl describe providerrevisions
Name:         provider-aws-s3-92206523fff4
API Version:  pkg.crossplane.io/v1
Kind:         ProviderRevision
Spec:
  Desired State:                  Active
  Image:                          xpkg.upbound.io/upbound/provider-aws-s3:v0.41.0
  Revision:                       1
Status:
  Conditions:
    Last Transition Time:  2023-10-10T21:06:39Z
    Reason:                UnhealthyPackageRevision
    Status:                False
    Type:                  Healthy
  Controller Ref:
    Name:
Events:
  Type     Reason             Age                From                                         Message
  ----     ------             ----               ----                                         -------
  Warning  LintPackage        41s (x3 over 47s)  packages/providerrevision.pkg.crossplane.io  incompatible Crossplane version: package is not compatible with Crossplane version (v1.10.0)
```

{{<hover label="depend" line="17">}}イベント{{</hover>}}は、現在のCrossplaneバージョンが構成パッケージの要件を満たしていないというメッセージを伴う{{<hover label="depend" line="20">}}警告{{</hover>}}を示します。

## プロバイダーのアップグレード

既存のプロバイダーをアップグレードするには、新しいプロバイダーマニフェストを適用するか、`kubectl edit providers`を使用してインストールされたプロバイダーパッケージを編集します。

プロバイダーの`spec.package`内のバージョン番号を更新し、変更を適用します。Crossplaneは新しいイメージをインストールし、新しい`ProviderRevision`を作成します。

`ProviderRevision`により、Crossplaneは非推奨のプロバイダーCRDを削除することなく保存できるようになります。


`ProviderRevisions` を表示するには 
{{<hover label="getPR" line="1">}}kubectl get providerrevisions{{</hover>}}

```shell {label="getPR",copy-lines="1"}
kubectl get providerrevisions
NAME                                       HEALTHY   REVISION   IMAGE                                                    STATE      DEP-FOUND   DEP-INSTALLED   AGE
provider-aws-s3-dbc7f981d81f               True      1          xpkg.upbound.io/upbound/provider-aws-s3:v0.37.0          Active     1           1               10d
provider-nop-552a394a8acc                  True      2          xpkg.upbound.io/crossplane-contrib/provider-nop:v0.3.0   Active                                 11d
provider-nop-7e62d2a1a709                  True      1          xpkg.upbound.io/crossplane-contrib/provider-nop:v0.2.0   Inactive                               13d
upbound-provider-family-aws-710d8cfe9f53   True      1          xpkg.upbound.io/upbound/provider-family-aws:v0.40.0      Active                                 10d
```

デフォルトでは、Crossplane は単一の 
{{<hover label="getPR" line="5">}}Inactive{{</hover>}} プロバイダーを保持します。

デフォルト値を変更するには、[revision history limit](#package-revision-history-limit) セクションを参照してください。

プロバイダーの単一のリビジョンは 
{{<hover label="getPR" line="4">}}Active{{</hover>}} である必要があります。

## プロバイダーの削除

`kubectl delete provider` を使用してプロバイダーオブジェクトを削除することで、プロバイダーを削除します。

{{< hint "warning" >}}
プロバイダーの管理リソースを最初に削除せずにプロバイダーを削除すると、リソースが放棄される可能性があります。外部リソースは削除されません。

最初にプロバイダーを削除した場合は、クラウドプロバイダーを通じて外部リソースを手動で削除する必要があります。管理リソースは、ファイナライザーを削除することで手動で削除する必要があります。

放棄されたリソースの削除に関する詳細は、[Crossplane troubleshooting guide]({{<ref "../guides/troubleshoot-crossplane#deleting-when-a-resource-hangs" >}})を参照してください。
{{< /hint >}}

## プロバイダーの確認

プロバイダーは、サポートする管理リソースを表す独自のAPIをインストールします。
プロバイダーは、デプロイメント、サービスアカウント、またはRBAC構成を作成することもあります。

プロバイダーのステータスを表示するには

`kubectl get providers`

インストール中、プロバイダーは `INSTALLED` を `True` として、`HEALTHY` を `Unknown` として報告します。

```shell {copy-lines="1"}
kubectl get providers
NAME                              INSTALLED   HEALTHY   PACKAGE                                                   AGE
crossplane-contrib-provider-aws   True        Unknown   xpkg.upbound.io/crossplane-contrib/provider-aws:v0.39.0   63s
```

プロバイダーのインストールが完了し、使用可能になると、`HEALTHY` ステータスは `True` を報告します。

```shell {copy-lines="1"}
kubectl get providers
NAME                              INSTALLED   HEALTHY   PACKAGE                                                   AGE
crossplane-contrib-provider-aws   True        True      xpkg.upbound.io/crossplane-contrib/provider-aws:v0.39.0   88s
```


{{<hint "important" >}}
いくつかのプロバイダーは、数百のKubernetesカスタムリソース定義（`CRDs`）をインストールします。  
これは、サイズが不十分なAPIサーバーに大きな負担をかけ、プロバイダーのインストール時間に影響を与える可能性があります。

Crossplaneコミュニティには、  
[CRDsのスケーリングに関する詳細](https://github.com/crossplane/crossplane/blob/master/design/one-pager-crd-scaling.md)があります。
{{< /hint >}}

### プロバイダーの状態

Crossplaneは、プロバイダー用の標準的な`Conditions`セットを使用します。  
`kubectl describe provider`を使用して、プロバイダーの`Status`の下にある状態を表示します。

```yaml
kubectl describe provider
Name:         my-provider
API Version:  pkg.crossplane.io/v1
Kind:         Provider
# Removed for brevity
Status:
  Conditions:
    Reason:      HealthyPackageRevision
    Status:      True
    Type:        Healthy
    Reason:      ActivePackageRevision
    Status:      True
    Type:        Installed
# Removed for brevity
```

#### タイプ

プロバイダー`Conditions`は、2つの`Types`をサポートしています：

* `Type: Installed` - プロバイダーのパッケージはインストールされていますが、使用する準備ができていません。
* `Type: Healthy` - プロバイダーのパッケージは使用する準備ができています。

#### 理由

各`Reason`は特定の`Type`および`Status`に関連しています。Crossplaneは、プロバイダー`Conditions`に対して以下の`Reasons`を使用します。

<!-- vale Google.Headings = NO -->
##### InactivePackageRevision

`Reason: InactivePackageRevision`は、プロバイダーのパッケージが非アクティブなプロバイダーのパッケージリビジョンを使用していることを示します。

<!-- vale Google.Headings = YES -->
```yaml
Type: Installed
Status: False
Reason: InactivePackageRevision
```

<!-- vale Google.Headings = NO -->
##### ActivePackageRevision
<!-- vale Google.Headings = YES -->
プロバイダーのパッケージは現在のパッケージリビジョンですが、Crossplaneはまだパッケージリビジョンのインストールを完了していません。

{{< hint "tip" >}}
この状態にあるプロバイダーは、パッケージリビジョンに問題があるためです。

詳細については、`kubectl describe providerrevisions`を使用してください。
{{< /hint >}}

```yaml
Type: Installed
Status: True
Reason: ActivePackageRevision
```

<!-- vale Google.Headings = NO -->
##### HealthyPackageRevision

プロバイダーは完全にインストールされ、使用する準備ができています。

{{<hint "tip" >}}
`Reason: HealthyPackageRevision`は、正常に動作しているプロバイダーの通常の状態です。
{{< /hint >}}

<!-- vale Google.Headings = YES -->
```yaml
Type: Healthy
Status: True
Reason: HealthyPackageRevision
```

<!-- vale Google.Headings = NO -->
##### UnhealthyPackageRevision
<!-- vale Google.Headings = YES -->


プロバイダーパッケージリビジョンのインストール中にエラーが発生し、
Crossplaneがプロバイダーパッケージをインストールできませんでした。

{{<hint "tip" >}}
`kubectl describe providerrevisions`を使用して、パッケージリビジョンが失敗した理由の詳細を確認してください。
{{< /hint >}}

```yaml
Type: Healthy
Status: False
Reason: UnhealthyPackageRevision
```
<!-- vale Google.Headings = NO -->
##### UnknownPackageRevisionHealth
<!-- vale Google.Headings = YES -->

プロバイダーパッケージリビジョンのステータスは`Unknown`です。プロバイダーパッケージリビジョンはインストール中であるか、問題があります。

{{<hint "tip" >}}
`kubectl describe providerrevisions`を使用して、パッケージリビジョンが失敗した理由の詳細を確認してください。
{{< /hint >}}

```yaml
Type: Healthy
Status: Unknown
Reason: UnknownPackageRevisionHealth
```

## プロバイダーの設定

プロバイダーには2種類の設定があります：

* _コントローラー設定_は、Kubernetesクラスター内で実行されているプロバイダーポッドの設定を変更します。たとえば、プロバイダーポッドに`toleration`を設定することです。

* _プロバイダー設定_は、外部プロバイダーとの通信に使用される設定を変更します。たとえば、クラウドプロバイダーの認証です。

{{<hint "important" >}}
`ControllerConfig`オブジェクトをプロバイダーに適用します。  

`ProviderConfig`オブジェクトを管理リソースに適用します。
{{< /hint >}}

### コントローラー設定

{{< hint "important" >}}
<!-- vale write-good.Passive = NO -->
<!-- vale gitlab.FutureTense = NO -->
`ControllerConfig`タイプはv1.11で非推奨となり、将来のリリースで削除されます。
<!-- vale write-good.Passive = YES -->
<!-- vale gitlab.FutureTense = YES -->

[`DeploymentRuntimeConfig`]({{<ref "#runtime-configuration" >}})は
コントローラー設定の代替であり、v1.14+で利用可能です。
{{< /hint >}}

Crossplaneの`ControllerConfig`をプロバイダーに適用すると、
プロバイダーのポッドの設定が変更されます。
[Crossplane ControllerConfigスキーマ]({{< ref "../api#ControllerConfig-spec" >}})
は、サポートされているControllerConfig設定のセットを定義しています。

ControllerConfigsの最も一般的な使用例は、プロバイダーのポッドに`args`を提供してオプションサービスを有効にすることです。たとえば、
[外部シークレットストア]({{< ref "../guides/vault-as-secret-store#enable-external-secret-stores-in-the-provider" >}})
をプロバイダーに対して有効にすることです。

各プロバイダーは、サポートされている `args` のセットを決定します。

### ランタイム構成

{{<hint "重要" >}}
`DeploymentRuntimeConfigs` はベータ機能です。

デフォルトでオンになっており、Crossplane デプロイメントに `--enable-deployment-runtime-configs=false` を渡すことで無効にできます。
{{< /hint >}}

ランタイム構成は、ランタイムを持つ Crossplane パッケージのための一般化された構成メカニズムであり、具体的には `Providers` と `Functions` です。これは、非推奨の `ControllerConfig` タイプに代わるもので、v1.14+ で利用可能です。

デフォルトの構成では、Crossplane は Kubernetes Deployment を使用してパッケージのランタイムをデプロイします。具体的には、`Provider` のためのコントローラーまたは `Function` のための gRPC サーバーです。`DeploymentRuntimeConfig` を適用し、それを `Provider` または `Function` オブジェクトで参照することで、ランタイムマニフェストを構成することが可能です。

{{<hint "注意" >}}
`ControllerConfig` とは異なり、`DeploymentRuntimeConfig` は Kubernetes Deployment スペック全体を埋め込んでおり、ランタイムの構成においてより柔軟性を提供します。詳細については、[設計文書](https://github.com/crossplane/crossplane/blob/2c5e7f07ba9e3d83d1c85169bbde685de8514ab8/design/one-pager-package-runtime-config.md)を参照してください。
{{< /hint >}}

例として、`Provider` に対して外部シークレットストアのアルファ機能を有効にするために、コントローラーに `--enable-external-secret-stores` 引数を追加するには、次のように適用できます。

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp-iam
spec:
  package: xpkg.upbound.io/upbound/provider-gcp-iam:v0.37.0
  runtimeConfigRef:
    name: enable-ess
---
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: enable-ess
spec:
  deploymentTemplate:
    spec:
      selector: {}
      template:
        spec:
          containers:
            - name: package-runtime
              args:
                - --enable-external-secret-stores
```

パッケージマネージャーは、ランタイムコンテナの名前として `package-runtime` を使用することに注意してください。異なるコンテナ名を使用する場合、パッケージマネージャーはそれをサイドカーコンテナとして導入し、パッケージランタイムコンテナを変更することはありません。

<!-- vale write-good.Passive = NO -->
パッケージマネージャーは、ランタイムが正常に動作することを保証するためにいくつかのフィールドに対して意見を持っており、
<!-- vale write-good.Passive = YES -->
ランタイム構成の値の上にそれらをオーバーレイします。たとえば、設定されていない場合、レプリカ数をデフォルトで 1 にし、Deployment と Service が一致するようにラベルセレクターを上書きします。また、必要な環境変数、ポート、ボリュームおよびボリュームマウントを注入します。

`Provider` または `Functions` の `spec.runtimeConfigRef.name` フィールドはデフォルトで値 `default` に設定されており、指定されていない場合は Crossplane がデフォルトのランタイム構成を使用します。Crossplane は常にクラスター内にデフォルトのランタイム
<!-- vale gitlab.FutureTense = NO -->
構成が存在することを保証しますが、既に存在する場合は変更しません。これにより
<!-- vale gitlab.FutureTense = YES -->
ユーザーはデフォルトのランタイム構成を自分のニーズに合わせてカスタマイズできます。

{{<hint "tip" >}}
<!-- vale gitlab.SubstitutionWarning = NO -->
`DeploymentRuntimeConfig` は Kubernetes の `Deployment`
<!-- vale gitlab.SubstitutionWarning = YES -->
spec と同じスキーマを使用しているため、スキーマ検証を回避するために空の値を渡す必要があるかもしれません。
たとえば、`replicas` フィールドだけを変更したい場合は、次のように渡す必要があります：

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: multi-replicas
spec:
  deploymentTemplate:
    spec:
      replicas: 2
      selector: {}
      template: {}
```

{{< /hint >}}

#### ランタイムデプロイメント spec の構成

`DeploymentRuntimeConfig` で提供されるデプロイメント spec をベースにして、パッケージマネージャーは次のルールに従ってパッケージランタイムのデプロイメント spec を構築します：
- パッケージランタイムコンテナを `containers` 配列の最初のコンテナとして注入し、名前を `package-runtime` とします。
- 提供されていない場合は、次のようにデフォルト設定します：
  - `spec.replicas` を 1 に設定。
  - イメージプルポリシーを `IfNotPresent` に設定。
  - Pod セキュリティコンテキストを次のように設定：
    ```yaml
    runAsNonRoot: true
    runAsUser: 2000
    runAsGroup: 2000
    ```
  - ランタイムコンテナのセキュリティコンテキストを次のように設定：
    ```yaml
    allowPrivilegeEscalation: false
    privileged: false
    runAsGroup: 2000
    runAsNonRoot: true
    runAsUser: 2000
    ```
- 次のことを適用します：
  - **metadata.namespace** を Crossplane 名前空間として設定します。
  - **metadata.ownerReferences** を設定し、デプロイメントがパッケージリビジョンに所有されるようにします。
  - **spec.selectors** を生成されたラベルを使用して設定します。
  - **spec.serviceAccount** を作成された **Service Account** で設定します。
  - パッケージ spec で提供されたプルシークレットをイメージプルシークレット `spec.packagePullSecrets` として追加します。
  - パッケージ spec で提供された値 `spec.packagePullPolicy` で **Image Pull Policy** を設定します。
  - ランタイムコンテナに必要な **Ports** を追加します。
  - ランタイムコンテナに必要な **Environments** を追加します。
  - TLS シークレットをマウントするために、ランタイムコンテナに必要な **Volumes**、**Volume Mounts** および **Environments** を追加します。

#### ランタイムリソースのメタデータの設定

`DeploymentRuntimeConfig` は、次のランタイムリソースのメタデータを設定することも可能です。すなわち、`Deployment`、`ServiceAccount`、および `Service`：
- 名前
- ラベル
- アノテーション

以下の例は、ServiceAccountの名前とDeploymentのラベルを設定する方法を示しています：

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: my-runtime-config
spec:
  deploymentTemplate:
    metadata:
      labels:
        my-label: my-value
  serviceAccountTemplate:
    metadata:
      name: my-service-account
```

### プロバイダーの設定

`ProviderConfig` は、プロバイダーが外部プロバイダーと通信する際に使用する設定を決定します。各プロバイダーは、その `ProviderConfig` の利用可能な設定を決定します。

<!-- vale write-good.Weasel = NO -->
<!-- allow "usually" -->
プロバイダーの認証は通常 `ProviderConfig` で設定されます。たとえば、AWSプロバイダーで基本的なキー・ペア認証を使用するには、{{<hover label="providerconfig" line="2" >}}ProviderConfig{{</hover >}} 
{{<hover label="providerconfig" line="5" >}}spec{{</hover >}} 
が
{{<hover label="providerconfig" line="6" >}}credentials{{</hover >}} 
を定義し、プロバイダーのポッドがKubernetesの
{{<hover label="providerconfig" line="7" >}}Secrets{{</hover >}} 
オブジェクトを探し、次の名前のキーを使用する必要があります：
{{<hover label="providerconfig" line="10" >}}aws-creds{{</hover >}}。
<!-- vale write-good.Weasel = YES -->
```yaml {label="providerconfig"}
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: aws-provider
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
```

{{< hint "important" >}}
認証設定はプロバイダーによって異なる場合があります。

特定のプロバイダーの認証設定に関する指示については、そのプロバイダーのドキュメントを参照してください。
{{< /hint >}}

<!-- vale write-good.TooWordy = NO -->
<!-- allow multiple -->
ProviderConfigオブジェクトは、個々のマネージドリソースに適用されます。単一のプロバイダーは、複数のユーザーまたはアカウントを通じてProviderConfigsで認証できます。
<!-- vale write-good.TooWordy = YES -->

各アカウントの認証情報は、ユニークなProviderConfigに結びついています。マネージドリソースを作成する際には、希望するProviderConfigを添付してください。

たとえば、2つのAWS ProviderConfigs、名前が
{{<hover label="user" line="4">}}user-keys{{</hover >}} 
と
{{<hover label="admin" line="4">}}admin-keys{{</hover >}} 
は異なるKubernetesシークレットを使用します。

```yaml {label="user"}
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: user-keys
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: my-key
      key: secret-key
```

```yaml {label="admin"}
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: admin-keys
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: admin-key
      key: admin-secret-key
```

管理リソースを作成する際に ProviderConfig を適用します。

これにより、AWS {{<hover label="user-bucket" line="2" >}}Bucket{{< /hover >}} リソースが
{{<hover label="user-bucket" line="9" >}}user-keys{{< /hover >}} ProviderConfig を使用して作成されます。

```yaml {label="user-bucket"}
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: user-bucket
spec:
  forProvider:
    region: us-east-2
  providerConfigRef:
    name: user-keys
```

これにより、2 番目の {{<hover label="admin-bucket" line="2" >}}Bucket{{< /hover >}} リソースが
{{<hover label="admin-bucket" line="9" >}}admin-keys{{< /hover >}} ProviderConfig を使用して作成されます。

```yaml {label="admin-bucket"}
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: user-bucket
spec:
  forProvider:
    region: us-east-2
  providerConfigRef:
    name: admin-keys
```
