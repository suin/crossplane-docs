---
title: 設定パッケージ
description: "パッケージは複数のCrossplaneリソースを単一のポータブルOCIイメージにまとめます。"
altTitle: "Crossplaneパッケージ"
weight: 200
---

_設定_ パッケージは、 
[OCIコンテナイメージ](https://opencontainers.org/)であり、 
[コンポジション]({{<ref "./compositions" >}})、 
[複合リソース定義]({{<ref "./composite-resource-definitions" >}}) 
および必要な[プロバイダー]({{<ref "./providers">}})または 
[関数]({{<ref "./composition-functions" >}})のコレクションを含んでいます。

設定パッケージは、あなたのCrossplane設定を完全にポータブルにします。

{{<hint "important" >}}
Crossplane [プロバイダー]({{<ref "./providers">}})および 
[関数]({{<ref "./composition-functions">}})もCrossplaneパッケージです。  

この文書では、設定パッケージのインストールと管理方法について説明します。  

パッケージの使用に関する詳細は、 
[プロバイダー]({{<ref "./providers">}})および 
[コンポジション関数]({{<ref "./composition-functions">}})の章を参照してください。 
{{< /hint >}}

## 設定のインストール

Crossplane 
{{<hover line="2" label="install">}}設定{{</hover>}}オブジェクトを使用して設定をインストールするには、 
{{<hover line="6" label="install">}}spec.package{{</hover>}}の値を
設定パッケージの場所に設定します。

{{< hint "important" >}}
Crossplaneバージョン1.15.0以降、Crossplaneはデフォルトで`xpkg.upbound.io`のUpbound Marketplace
Crossplaneパッケージレジストリを使用してパッケージをダウンロードおよびインストールします。 

`package`で完全なドメイン名を指定するか、[Crossplaneポッド]({{<ref "./pods">}})
で`--registry`フラグを使用してデフォルトのCrossplaneレジストリを変更します。 
{{< /hint >}}

例えば、 
[Upbound AWSリファレンスプラットフォーム](https://marketplace.upbound.io/configurations/upbound/platform-ref-aws/v0.6.0)をインストールするには、 

```yaml {label="install"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  package: xpkg.upbound.io/upbound/platform-ref-aws:v0.6.0
```

Crossplaneは、設定にリストされたコンポジション、複合リソース定義、および
プロバイダーをインストールします。

### Helmを使用したインストール

Crossplaneは、Crossplane Helmチャートを使用して初期のCrossplane
インストール中に設定をインストールすることをサポートしています。


`helm install` に対して 
{{<hover label="helm" line="5" >}}--set configuration.packages{{</hover >}} 
引数を使用します。

たとえば、Upbound AWS リファレンスプラットフォームをインストールするには、

```shell {label="helm"}
helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace \
--set configuration.packages='{xpkg.upbound.io/upbound/platform-ref-aws:v0.6.0}'
```

### オフラインインストール

Crossplane パッケージをオフラインでインストールするには、パッケージをホストするための 
[Harbor](https://goharbor.io/) のようなローカルコンテナレジストリが必要です。Crossplane はコンテナレジストリからのパッケージのインストールのみをサポートしています。

Crossplane は Kubernetes ボリュームから直接パッケージをインストールすることをサポートしていません。

### インストールオプション

構成は、構成パッケージに関連する設定を変更するための複数のオプションをサポートしています。

#### 構成のリビジョン

既存の構成の新しいバージョンをインストールする際、Crossplane は新しい構成リビジョンを作成します。

構成リビジョンを表示するには 
{{<hover label="rev" line="1">}}kubectl get configurationrevisions{{</hover>}} を使用します。

```shell {label="rev",copy-lines="1"}
kubectl get configurationrevisions
NAME                            HEALTHY   REVISION   IMAGE                                             STATE      DEP-FOUND   DEP-INSTALLED   AGE
platform-ref-aws-1735d56cd88d   True      2          xpkg.upbound.io/upbound/platform-ref-aws:v0.5.0   Active     2           2               46s
platform-ref-aws-3ac761211893   True      1          xpkg.upbound.io/upbound/platform-ref-aws:v0.4.1   Inactive                               5m13s
```

同時にアクティブなリビジョンは1つだけです。アクティブなリビジョンは、Composition や Composite Resource Definition を含む利用可能なリソースを決定します。

デフォルトでは、Crossplane は単一の _Inactive_ リビジョンのみを保持します。

Crossplane が保持するリビジョンの数を構成パッケージ 
{{<hover label="revHistory" line="6">}}revisionHistoryLimit{{</hover>}} で変更します。

{{<hover label="revHistory" line="6">}}revisionHistoryLimit{{</hover>}} 
フィールドは整数です。  
デフォルト値は `1` です。  
{{<hover label="revHistory" line="6">}}revisionHistoryLimit{{</hover>}} を `0` に設定することで、リビジョンの保存を無効にします。

たとえば、デフォルト設定を変更して10のリビジョンを保存するには 
{{<hover label="revHistory" line="6">}}revisionHistoryLimit: 10{{</hover>}} を使用します。

```yaml {label="revHistory"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  revisionHistoryLimit: 10
# Removed for brevity
```

#### 構成パッケージのプルポリシー

{{<hover label="pullpolicy" line="6">}}packagePullPolicy{{</hover>}} を使用して、Crossplane が構成パッケージをローカルの Crossplane パッケージキャッシュにダウンロードするタイミングを定義します。

`packagePullPolicy` オプションは次のとおりです：
* `IfNotPresent` - (**デフォルト**) キャッシュにパッケージがない場合のみ、パッケージをダウンロードします。
* `Always` - 毎分新しいパッケージをチェックし、キャッシュにない一致するパッケージをダウンロードします。
* `Never` - パッケージを決してダウンロードしません。パッケージはローカルのパッケージキャッシュからのみインストールされます。

{{<hint "tip" >}}
Crossplane 
{{<hover label="pullpolicy" line="6">}}packagePullPolicy{{</hover>}} は Kubernetes コンテナイメージの 
[image pull policy](https://kubernetes.io/docs/concepts/containers/images/#image-pull-policy) のように機能します。

Crossplane は Kubernetes イメージのようにタグとパッケージダイジェストハッシュの使用をサポートしています。
{{< /hint >}}

たとえば、特定の構成パッケージを `Always` ダウンロードするには、 
{{<hover label="pullpolicy" line="6">}}packagePullPolicy: Always{{</hover>}} 
構成を使用します。

```yaml {label="pullpolicy",copy-lines="6"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  packagePullPolicy: Always
# Removed for brevity
```

#### リビジョンアクティベーションポリシー

`Active` パッケージリビジョンは、パッケージコントローラーがリソースを積極的に調整している状態です。

デフォルトでは、Crossplane は最も最近インストールされたパッケージリビジョンを `Active` として設定します。

構成のアップグレード動作を制御するには、 
{{<hover label="revision" line="6">}}revisionActivationPolicy{{</hover>}} を使用します。

{{<hover label="revision" line="6">}}revisionActivationPolicy{{</hover>}} 
オプションは次のとおりです：
* `Automatic` - (**デフォルト**) 最後にインストールされた構成を自動的にアクティブ化します。
* `Manual` - 構成を自動的にアクティブ化しません。

たとえば、アップグレード動作を手動アップグレードを必要とするように変更するには、 
{{<hover label="revision" line="6">}}revisionActivationPolicy: Manual{{</hover>}} を設定します。

```yaml {label="revision"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  revisionActivationPolicy: Manual
# Removed for brevity
```

#### プライベートレジストリからの構成のインストール

Kubernetes が `imagePullSecrets` を使用して 
[プライベートレジストリからイメージをインストールする](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/) のとおり、 
Crossplane は `packagePullSecrets` を使用してプライベートレジストリから構成パッケージをインストールします。

{{<hover label="pps" line="6">}}packagePullSecrets{{</hover>}} を使用して、 
構成パッケージをダウンロードする際の認証に使用する Kubernetes シークレットを提供します。


{{<hint "important" >}}
KubernetesのシークレットはCrossplaneと同じ名前空間に存在する必要があります。
{{</hint >}}

{{<hover label="pps" line="6">}}packagePullSecrets{{</hover>}}はシークレットのリストです。

例えば、{{<hover label="pps" line="6">}}example-secret{{</hover>}}という名前のシークレットを使用するには、 
{{<hover label="pps" line="6">}}packagePullSecrets{{</hover>}}を設定します。

```yaml {label="pps"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  packagePullSecrets: 
    - name: example-secret
# Removed for brevity
```

#### 依存関係を無視する

デフォルトでは、CrossplaneはConfigurationパッケージにリストされているすべての[依存関係](#manage-dependencies)をインストールします。

Crossplaneは、{{<hover label="pkgDep" line="6" >}}skipDependencyResolution{{</hover>}}を使用してConfigurationパッケージの依存関係を無視できます。

{{< hint "warning" >}}
ほとんどのConfigurationには、必要なプロバイダーの依存関係が含まれています。

Configurationが依存関係を無視する場合、必要なプロバイダーは手動でインストールする必要があります。
{{< /hint >}}

例えば、依存関係の解決を無効にするには、 
{{<hover label="pkgDep" line="6" >}}skipDependencyResolution: true{{</hover>}}を設定します。

```yaml {label="pkgDep"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  skipDependencyResolution: true
# Removed for brevity
```

#### Crossplaneのバージョン要件を無視する

Configurationパッケージは、インストール前に特定のCrossplaneバージョンまたは最小バージョンを要求する場合があります。デフォルトでは、Crossplaneは要求されたバージョンを満たさない場合、Configurationをインストールしません。

Crossplaneは、{{<hover label="xpVer" line="6">}}ignoreCrossplaneConstraints{{</hover>}}を使用して要求されたバージョンを無視できます。

例えば、サポートされていないCrossplaneバージョンにConfigurationパッケージをインストールするには、 
{{<hover label="xpVer" line="6">}}ignoreCrossplaneConstraints: true{{</hover>}}を設定します。

```yaml {label="xpVer"}
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: platform-ref-aws
spec:
  ignoreCrossplaneConstraints: true
# Removed for brevity
```


### Configurationの検証

{{<hover label="verify" line="1">}}kubectl get configuration{{</hover >}}を使用してConfigurationを検証します。

動作しているConfigurationは、`Installed`と`Healthy`が`True`として報告されます。

```shell {label="verify",copy-lines="1"}
kubectl get configuration
NAME               INSTALLED   HEALTHY   PACKAGE                                           AGE
platform-ref-aws   True        True      xpkg.upbound.io/upbound/platform-ref-aws:v0.6.0   54s
```

### 依存関係の管理

構成パッケージには、Functions、Providers、または他の構成を含む他のパッケージへの依存関係が含まれる場合があります。

Crossplaneが構成の依存関係を満たせない場合、構成は `HEALTHY` を `False` として報告します。

例えば、この Upbound AWS リファレンスプラットフォームのインストールは `HEALTHY: False` です。

```shell {copy-lines="1"}
kubectl get configuration
NAME               INSTALLED   HEALTHY   PACKAGE                                           AGE
platform-ref-aws   True        False     xpkg.upbound.io/upbound/platform-ref-aws:v0.6.0   71s
```

構成が `HEALTHY` でない理由の詳細を確認するには、 
{{<hover label="depend" line="1">}}kubectl describe configurationrevisions{{</hover>}} を使用します。

```yaml {copy-lines="1",label="depend"}
kubectl describe configurationrevision
Name:         platform-ref-aws-a30ad655c769
API Version:  pkg.crossplane.io/v1
Kind:         ConfigurationRevision
# Removed for brevity
Spec:
  Desired State:                  Active
  Image:                          xpkg.upbound.io/upbound/platform-ref-aws:v0.6.0
  Revision:                       1
Status:
  Conditions:
    Last Transition Time:  2023-10-06T20:08:14Z
    Reason:                UnhealthyPackageRevision
    Status:                False
    Type:                  Healthy
  Controller Ref:
    Name:
Events:
  Type     Reason       Age                From                                              Message
  ----     ------       ----               ----                                              -------
  Warning  LintPackage  29s (x2 over 29s)  packages/configurationrevision.pkg.crossplane.io  incompatible Crossplane version: package is not compatible with Crossplane version (v1.12.0)
```

{{<hover label="depend" line="18">}}イベント{{</hover>}}は、 
{{<hover label="depend" line="21">}}警告{{</hover>}}を示し、現在の Crossplane のバージョンが構成パッケージの要件を満たしていないというメッセージを表示します。

## 構成の作成

Crossplane 構成パッケージは、1つ以上の YAML ファイルを含む 
[OCI コンテナイメージ](https://opencontainers.org/) です。

{{<hint "important" >}}
構成パッケージは完全に OCI 準拠です。OCI イメージを構築するツールは、構成パッケージを構築できます。

Crossplane コマンドラインツールを使用して、Crossplane パッケージビルドにエラーチェックとフォーマットを提供することを強く推奨します。

サードパーティツールを使用してパッケージを構築する際のパッケージ要件については、 
[Crossplane パッケージ仕様](https://github.com/crossplane/crossplane/blob/master/contributing/specifications/xpkg.md) をお読みください。
{{</hint >}}

構成パッケージには `crossplane.yaml` ファイルが必要で、Composition および CompositeResourceDefinition ファイルを含むことができます。

<!-- vale Google.Headings = NO -->
### crossplane.yaml ファイル
<!-- vale Google.Headings = YES -->

Crossplane CLI を使用して構成パッケージを構築するには、 
{{<hover label="cfgMeta" line="1">}}crossplane.yaml{{</hover>}} という名前のファイルを作成します。  
{{<hover label="cfgMeta" line="1">}}crossplane.yaml{{</hover>}} 
ファイルは、構成の要件と名前を定義します。


{{<hint "important" >}}
Crossplane CLIは`crossplane.yaml`という名前のファイルのみをサポートしています。
{{< /hint >}}

構成パッケージは
{{<hover label="cfgMeta" line="2">}}meta.pkg.crossplane.io{{</hover>}}
Crossplane APIグループを使用します。

他の構成、関数、またはプロバイダーを
{{<hover label="cfgMeta" line="7">}}dependsOn{{</hover>}}リストに指定します。  
オプションとして、 
{{<hover label="cfgMeta" line="9">}}version{{</hover>}}オプションで特定のパッケージバージョンまたは最小バージョンを要求できます。

この構成に対して特定のCrossplaneのバージョンまたは最小バージョンを
{{<hover label="cfgMeta" line="11">}}crossplane.version{{</hover>}}オプションで定義することもできます。

{{<hint "note" >}}
{{<hover label="cfgMeta" line="10">}}crossplane{{</hover>}}オブジェクトや必要なバージョンの定義はオプションです。 
{{< /hint >}}

```yaml {label="cfgMeta",copy-lines="all"}
$ cat crossplane.yaml
apiVersion: meta.pkg.crossplane.io/v1alpha1
kind: Configuration
metadata:
  name: test-configuration
spec:
  dependsOn:
    - provider: xpkg.upbound.io/crossplane-contrib/provider-aws
      version: ">=v0.36.0"
  crossplane:
    version: ">=v1.12.1-0"
```

### パッケージをビルドする

[Crossplane CLI]({{<ref "../cli">}})コマンドを使用して、 
`crossplane xpkg build --package-root=<directory>`でパッケージを作成します。

ここで、`<directory>`は`crossplane.yaml`ファイルと
任意のCompositionまたはCompositeResourceDefinition YAMLファイルを含むディレクトリです。

CLIはディレクトリ内の`.yml`または`.yaml`ファイルを再帰的に検索して
パッケージに含めます。

{{<hint "important" >}}
`--ignore=<file_list>`で他のYAMLファイルを無視する必要があります。  
例えば、`crossplane xpkg build --package-root=test-directory --ignore=".tmp/*"`のように。

CompositionやCompositeResourceDefinitionsでないYAMLファイルを含めることは、
Claimsを含めてサポートされていません。
{{</hint >}}

デフォルトでは、Crossplaneは構成名と
パッケージ内容のSHA-256ハッシュの`.xpkg`ファイルを作成します。

例えば、{{<hover label="xpkgName" line="2">}}Configuration{{</hover>}}という名前の
{{<hover label="xpkgName" line="4">}}test-configuration{{</hover>}}。  
Crossplane CLIは`test-configuration-e8c244f6bf21.xpkg`という名前のパッケージをビルドします。

```yaml {label="xpkgName"}
apiVersion: meta.pkg.crossplane.io/v1alpha1
kind: Configuration
metadata:
  name: test-configuration
# Removed for brevity
```

出力ファイルは`--package-file=<filename>.xpkg`オプションで指定します。

例えば、`test-directory`という名前のディレクトリからパッケージをビルドし、現在の作業ディレクトリに`test-package.xpkg`という名前のパッケージを生成するには、次のコマンドを使用します：

```shell
crossplane xpkg build --package-root=test-directory --package-file=test-package.xpkg
```

```shell
ls -1 ./
test-directory
test-package.xpkg
```
