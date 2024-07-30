---
weight: 50
title: コマンドリファレンス
description: "Crossplane CLIのコマンドリファレンス"
---


<!-- vale Google.Headings = NO -->
`crossplane` CLIは、Crossplaneの使用を容易にするためのユーティリティを提供します。

`crossplane`のインストールに関する情報は、[Crossplane CLIの概要]({{<ref "../cli">}})ページを参照してください。

## グローバルフラグ
以下のフラグはすべてのコマンドで使用できます。

{{< table "table table-sm table-striped">}}
| 短いフラグ | 長いフラグ   | 説明                          |
|------------|-------------|------------------------------|
| `-h`       | `--help`    | コンテキストに応じたヘルプを表示します。 |
|            | `--verbose` | 詳細な出力を表示します。        |
{{< /table >}}

## version

`crossplane version`コマンドは、Crossplane CLIとコントロールプレーンのバージョンを返します。

```shell
crossplane version
Client Version: v1.16.0
Server Version: v1.16.0
```

## xpkg

`crossplane xpkg`コマンドは、Crossplaneの[パッケージ]({{<ref "../concepts/packages">}})を作成、インストール、更新するほか、CrossplaneパッケージをCrossplaneパッケージレジストリに認証および公開する機能を提供します。

### xpkg build

`crossplane xpkg build`を使用すると、Crossplaneパッケージのビルドを自動化し、簡素化できます。

Crossplane CLIは、YAMLファイルのディレクトリを組み合わせて、[OCIコンテナイメージ](https://opencontainers.org/)としてパッケージ化します。

CLIは、[Crossplane XPKG仕様](https://github.com/crossplane/crossplane/blob/master/contributing/specifications/xpkg.md)を満たすために必要なアノテーションと値を適用します。

`crossplane` CLIは、[構成]({{< ref "../concepts/packages" >}})、[関数]({{<ref "../concepts/composition-functions">}})、および[プロバイダー]({{<ref "../concepts/providers" >}})パッケージタイプのビルドをサポートしています。

#### フラグ
{{< table "table table-sm table-striped">}}
| 短いフラグ   | 長いフラグ                            | 説明                          |
| ------------ | -------------                        | ------------------------------ |
|              | `--embed-runtime-image-name=NAME`    | パッケージに含めるイメージの名前とタグ。プロバイダーおよび関数パッケージ専用。 |
|              | `--embed-runtime-image-tarball=PATH` | パッケージに含めるイメージのファイル名。プロバイダーおよび関数パッケージ専用。                              |
| `-e`         | `--examples-root="./examples"`       | パッケージに関連する例のディレクトリへのパス。                               |
|              | `--ignore=PATH,...`                  | 無視するファイルおよびディレクトリのリスト。                              |
| `-o`         | `--package-file=PATH`                | 作成されたパッケージのディレクトリとファイル名。                             |
| `-f`         | `--package-root="."`                 | YAMLファイルを検索するディレクトリ。                              |
{{< /table >}}

`crossplane xpkg build` コマンドは、`--package-root` で設定されたディレクトリを再帰的に検索し、`.yml` または `.yaml` で終わるファイルをパッケージにまとめようとします。

すべての YAML ファイルは、`apiVersion`、`kind`、`metadata`、および `spec` フィールドを持つ有効な Kubernetes マニフェストである必要があります。

#### 無視するファイル

`--ignore` を使用して、無視するファイルとディレクトリのリストを提供します。

例えば、  
`crossplane xpkg build --ignore="./test/*,kind-config.yaml"`

#### パッケージ名の設定

`crossplane` は、新しいパッケージに `metadata.name` とパッケージ内容のハッシュの組み合わせを自動的に名前付けし、内容を `--package-root` と同じ場所に保存します。特定の場所とファイル名を `--package-file` または `-o` で定義します。

例えば、  
`crossplane xpkg build -o /home/crossplane/example.xpkg`。

#### 例を含める

`--examples-root` を使用して、パッケージの使用方法を示す YAML ファイルを含めます。

[Upbound Marketplace](https://marketplace.upbound.io/) は、公開されたパッケージのドキュメントとして `--examples-root` で含まれるファイルを使用します。

#### ランタイムイメージを含める

Functions と Providers は、依存関係と設定を説明する YAML ファイルと、ランタイム用のコンテナイメージを必要とします。

`--embed-runtime-image-name` を使用すると、指定されたイメージが実行され、関数またはプロバイダーのパッケージ内にイメージが含まれます。

{{<hint "note" >}}
`--embed-runtime-image-name` で参照されるイメージは、ローカルの Docker キャッシュに存在する必要があります。

不足しているイメージをダウンロードするには、`docker pull` を使用します。
{{< /hint >}}

`--embed-runtime-image-tarball` フラグは、ローカルの OCI イメージタールボールを関数またはプロバイダーのパッケージ内に含めます。

### xpkg install

`crossplane xpkg install` を使用して、パッケージを Crossplane にダウンロードしてインストールします。

デフォルトでは、`crossplane xpkg install` コマンドは `~/.kube/config` で定義された Kubernetes 構成を使用します。

環境変数 `KUBECONFIG` でカスタム Kubernetes 構成ファイルの場所を定義します。

パッケージの種類、パッケージファイル、およびオプションで Crossplane 内のパッケージに付ける名前を指定します。

```
`crossplane xpkg install <package-kind> <registry URL package name and tag> [<optional-name>]`

`<package-kind>` は `configuration`、`function` または `provider` のいずれかです。

例えば、バージョン 0.42.0 の 
[AWS S3 provider](https://marketplace.upbound.io/providers/upbound/provider-aws-s3/v0.42.0) をインストールするには:

`crossplane xpkg install provider xpkg.upbound.io/upbound/provider-aws-s3:v0.42.0`

#### フラグ
{{< table "table table-sm table-striped">}}
| 短いフラグ   | 長いフラグ                                        | 説明                                                                                     |
| ------------ | -------------                                    | ------------------------------                                                                  |
|              | `--runtime-config=<runtime config name>`         | ランタイム構成でパッケージをインストールします。                                               |
| `-m`         | `--manual-activation`                            | `revisionActiviationPolicy` を `Manual` に設定します。                                                |
|              | `--package-pull-secrets=<list of secrets>`       | パッケージレジストリへの認証に使用するKubernetesシークレットのカンマ区切りリストです。 |
| `-r`         | `--revision-history-limit=<number of revisions>` | `revisionHistoryLimit` を設定します。デフォルトは `1` です。                                                |
| `-w`         | `--wait=<number of seconds>`                     | パッケージがインストールされるまで待機する秒数です。                                             |

{{< /table >}}

#### パッケージインストールの待機

パッケージをインストールする際、`crossplane xpkg install` コマンドは
パッケージのダウンロードとインストールを待機しません。ダウンロードやインストールの問題を確認するには、`kubectl describe configuration` で `configuration` を調査してください。

`--wait` を使用すると、`crossplane xpkg install` コマンドがパッケージが `HEALTHY` の状態になるまで待機します。コマンドは、`wait` 時間が経過する前にパッケージが `HEALTHY` でない場合、エラーを返します。
```

#### 手動パッケージアクティベーションが必要

パッケージを手動アクティベーションを必要とするように設定し、 
`--manual-activation` でパッケージの自動アップグレードを防ぎます。

#### プライベートレジストリへの認証

プライベートパッケージレジストリに認証するには、`--package-pull-secrets` を使用し、 
Kubernetes Secret オブジェクトのリストを提供します。

{{<hint "重要" >}}
シークレットは Crossplane ポッドと同じネームスペースに存在する必要があります。 
{{< /hint >}}

#### 保存するパッケージバージョンの数をカスタマイズ

デフォルトでは、Crossplane はローカルパッケージキャッシュに 
単一の非アクティブパッケージのみを保存します。

`--revision-history-limit` を使用して、パッケージの非アクティブコピーを 
さらに保存します。

パッケージドキュメントで 
[パッケージリビジョン]({{< ref "../concepts/packages#configuration-revisions" >}}) 
について詳しく読むことができます。

### xpkg login

`xpkg login` を使用して `xpkg.upbound.io` に認証します。 
[Upbound Marketplace](https://marketplace.upbound.io/) コンテナレジストリです。

[Upbound Marketplace に登録](https://accounts.upbound.io/register) 
してパッケージをプッシュし、プライベートリポジトリを作成します。

#### フラグ

{{< table "table table-sm table-striped">}}
| 短いフラグ   | 長いフラグ                            | 説明                    |
| ------------ | -------------                        | ------------------------------ |
| `-u` | `--username=<username>`    | 認証に使用するユーザー名。 | 
| `-p` | `--password=<password>`    | 認証に使用するパスワード。 | 
| `-t` | `--token=<token string>`   | 認証に使用するユーザートークン文字列。 | 
| `-a` | `--account=<organization>` | 認証中に Upbound 組織を指定します。 |
{{< /table >}}

#### 認証オプション

`crossplane xpkg login` コマンドは、ユーザー名とパスワードまたは Upbound API トークンを使用できます。

デフォルトでは、引数なしの `crossplane xpkg login` は、ユーザー名とパスワードを 
入力するように促します。

`--username` および `--password` フラグを使用してユーザー名とパスワードを提供するか、 
環境変数 `UP_USER` にユーザー名を、`UP_PASSWORD` にパスワードを設定します。


ユーザーネームとパスワードの代わりに、`--token` または `UP_TOKEN` 環境変数を使用して、Upbound ユーザートークンを使用します。 

{{< hint "important" >}}
`--token` または `UP_TOKEN` 環境変数は、ユーザーネームとパスワードよりも優先されます。
{{< /hint >}}

`--password` または `--token` の入力として `-` を使用すると、stdin から入力を読み取ります。  
例えば、`crossplane xpkg login --password -`。

Crossplane CLI にログインすると、`.crossplane/config.json` に `profile` が作成され、特権のないアカウント情報がキャッシュされます。 

{{<hint "note" >}}
`config.json` ファイルの `session` フィールドは、セッションクッキー識別子です。 

`session` 値は認証には使用されません。これは `token` ではありません。
{{< /hint >}}

#### 登録済みの Upbound 組織での認証

ユーザーネームとパスワードまたはトークンとともに、`--account` オプションを使用して、Upbound Marketplace の登録済み組織に認証します。 

例えば、 
`crossplane xpkg login --account=Upbound --username=my-user --password -`。

### xpkg logout

`crossplane xpkg logout` を使用して、現在の `crossplane xpkg login` 
セッションを無効にします。

{{< hint "note" >}}
`crossplane xpkg logout` を使用すると、`~/.crossplane/config.json` ファイルから `session` が削除されますが、設定ファイルは削除されません。
{{< /hint >}}

### xpkg push

Crossplane パッケージファイルをパッケージレジストリにプッシュします。 

Crossplane CLI は、デフォルトで `xpkg.upbound.io` の 
[Upbound Marketplace](https://marketplace.upbound.io/) にイメージをプッシュします。

{{< hint "note" >}}
パッケージをプッシュするには、[`crossplane xpkg login`](#xpkg-login) での認証が必要な場合があります。
{{< /hint >}}

組織、パッケージ名、タグを指定して、  
`crossplane xpkg push <package>` を実行します。

デフォルトでは、コマンドはプッシュするための単一の `.xpkg` ファイルを現在のディレクトリで探します。 

複数のファイルをプッシュするか、特定の `.xpkg` ファイルを指定するには、`-f` フラグを使用します。

例えば、`my-package` というローカルパッケージを 
`crossplane-docs/my-package:v0.14.0` にプッシュするには、次のようにします：

`crossplane xpkg push -f my-package.xpkg crossplane-docs/my-package:v0.14.0`

他のパッケージレジストリ、例えば [DockerHub](https://hub.docker.com/) にプッシュするには、パッケージ名とともに完全なURLを指定します。

例えば、ローカルパッケージ `my-package` を DockerHub の組織 `crossplane-docs/my-package:v0.14.0` にプッシュするには、次のコマンドを使用します：
`crossplane xpkg push -f my-package.xpkg index.docker.io/crossplane-docs/my-package:v0.14.0`。


#### フラグ

{{< table "table table-sm table-striped">}}
| 短いフラグ   | 長いフラグ              | 説明                                   |
| ------------ | -------------          | ------------------------------                |
| `-f`         | `--package-files=PATH` | プッシュするxpkgファイルのカンマ区切りリスト。 |
{{< /table >}}

### xpkg update

`crossplane xpkg update` コマンドは、既存のパッケージをダウンロードして更新します。

デフォルトでは、`crossplane xpkg update` コマンドは `~/.kube/config` に定義されたKubernetes設定を使用します。

環境変数 `KUBECONFIG` を使用してカスタムKubernetes設定ファイルの場所を定義します。

パッケージの種類、パッケージファイル、およびオプションでCrossplaneに既にインストールされているパッケージの名前を指定します。

`crossplane xpkg update <package-kind> <registry package name and tag> [<optional-name>]`

パッケージファイルは、[Upbound Marketplace](https://marketplace.upbound.io/) の `xpkg.upbound.io` レジストリ上の組織、イメージ、およびタグである必要があります。

例えば、[AWS S3プロバイダー](https://marketplace.upbound.io/providers/upbound/provider-aws-s3/v0.42.0) のバージョン0.42.0に更新するには：

`crossplane xpkg update provider xpkg.upbound.io/upbound/provider-aws-s3:v0.42.0`


## ベータ

Crossplaneの `beta` コマンドは実験的です。これらのコマンドは、将来のリリースでフラグ、オプション、または出力が変更される可能性があります。

Crossplaneのメンテナーは、将来のリリースで `beta` の下にあるコマンドを昇格または削除することがあります。


### beta convert

Crossplaneが進化するにつれて、そのAPIやリソースが変更される可能性があります。新しいAPIやリソースへの移行を支援するために、`crossplane beta convert` コマンドはCrossplaneリソースを新しいバージョンまたは種類に変換します。

`crossplane beta convert` コマンドを使用して、既存の
[ControllerConfig]({{<ref "../concepts/providers#controller-configuration">}})
を [DeploymentRuntimeConfig]({{<ref "../concepts/providers#runtime-configuration">}}) 
または [patch and transforms]({{<ref "../concepts/patch-and-transform">}}) を使用して 
[Composition pipeline function]({{< ref "../concepts/compositions#use-composition-functions" >}}) に変換します。


`crossplane beta convert` コマンドに変換タイプ、入力ファイル、およびオプションで出力ファイルを指定します。デフォルトでは、コマンドは出力を標準出力に書き込みます。

例えば、ControllerConfigをDeploymentRuntimeConfigに変換するには、`crossplane beta convert deployment-runtime`を使用します。例えば、

`crossplane beta convert deployment-runtime controllerConfig.yaml -o deploymentConfig.yaml`

パッチと変換を使用してCompositionをパイプライン関数に変換するには、`crossplane beta convert pipeline-composition`を使用します。

オプションで、`-f`フラグを使用して関数の名前を指定します。デフォルトでは、関数名は「function-patch-and-transform」です。

`crossplane beta convert pipeline-composition oldComposition.yaml -o newComposition.yaml -f patchFunctionName`


#### フラグ
{{< table "table table-sm table-striped">}}
| 短いフラグ   | 長いフラグ       | 説明                                                                                     |
| ------------ | --------------- | ------------------------------                                                             |
| `-o`         | `--output-file` | 書き込む出力YAMLファイル。デフォルトではstdoutに出力されます。  |
| `-f`         | `--function-name` | 新しい関数の名前。デフォルトは`function-patch-and-transform`です。 |
<!-- vale Crossplane.Spelling = YES -->
{{< /table >}}


### beta render 

`crossplane beta render` コマンドは、[合成リソース]({{<ref "../concepts/composite-resources">}})の出力をプレビューします。これは、任意の[合成関数]({{<ref "../concepts/composition-functions">}})を適用した後のものです。

{{< hint "important" >}}
`crossplane beta render` コマンドは、[パッチと変換の合成パッチ]({{<ref "../concepts/patch-and-transform">}})を適用しません。

このコマンドは「パッチと変換」関数のみをサポートしています。
{{< /hint >}}

`crossplane beta render` コマンドは、ローカルで実行されているDockerエンジンに接続して、合成関数をプルして実行します。

{{<hint "important">}} 
`crossplane beta render`を実行するには、[Docker](https://www.docker.com/)が必要です。
{{< /hint >}}


コンポジットリソース、コンポジション、およびコンポジション関数のYAML定義を提供し、出力をローカルでレンダリングするためのコマンドを示します。

例えば、  
`crossplane beta render xr.yaml composition.yaml function.yaml`

出力には、元のコンポジットリソースと生成された管理リソースが含まれます。

{{<expand "An example render output" >}}
```yaml
---
apiVersion: nopexample.org/v1
kind: XBucket
metadata:
  name: test-xrender
status:
  bucketRegion: us-east-2
---
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  annotations:
    crossplane.io/composition-resource-name: my-bucket
  generateName: test-xrender-
  labels:
    crossplane.io/composite: test-xrender
  ownerReferences:
  - apiVersion: nopexample.org/v1
    blockOwnerDeletion: true
    controller: true
    kind: XBucket
    name: test-xrender
    uid: ""
spec:
  forProvider:
    region: us-east-2
```
{{< /expand >}}

#### フラグ

{{< table "table table-sm table-striped">}}
| 短いフラグ   | 長いフラグ                             | 説明                                           |
| ------------ | -------------                         | ------------------------------                        |
|              | `--context-files=<key>=<file>,<key>=<file>`    | 関数の「コンテキスト」に読み込むファイルのカンマ区切りリスト。 |
|              | `--context-values=<key>=<value>,<key>=<value>` | 関数の「コンテキスト」に読み込むキーと値のペアのカンマ区切りリスト。                                                    |
| `-r`         | `--include-function-results`          | 関数からの「結果」またはイベントを含める。   |
| `-o`         | `--observed-resources=<directory or file>`               |
関数に人工的な管理リソースデータを提供します。|
| `-x`         | `--include-full-xr`          | レンダリングされた出力に入力コンポジットリソースの仕様とメタデータフィールドのコピーを含める。   |
|              | `--timeout=`                          | 関数が終了するまでの待機時間。                    |
{{< /table >}}

`crossplane beta render`コマンドは、標準の  
[Docker環境変数](https://docs.docker.com/engine/reference/commandline/cli/#environment-variables)  
を利用してローカルDockerエンジンに接続し、コンポジション関数を実行します。

#### 関数コンテキストを提供する

`--context-files`および`--context-values`フラグは、関数の`context`にデータを提供できます。  
コンテキストはJSON形式のデータです。

#### 関数の結果を含める

関数がステータスを持つKubernetesイベントを生成する場合は、  
`--include-function-results`を使用して、管理リソースの出力とともにそれらを印刷します。

#### 合成リソースを含める

Composition関数は合成リソースの`status`フィールドのみを変更できます。デフォルトでは、`crossplane beta render`コマンドは`metadata.name`とともに`status`フィールドのみを出力します。

`--include-full-xr`を使用して、`spec`および`metadata`フィールドを含む完全な合成リソースを出力します。

#### モック管理リソース

`--observed-resources`を使用して、管理リソースを表すモックまたは人工データを提供します。`crossplane beta render`コマンドは、提供された入力をCrossplaneクラスター内のリソースのように扱います。

関数は、関数を実行する一部として含まれたリソースを参照および操作できます。

`observed-resources`は、複数のリソースを含む単一のYAMLファイルまたは複数のリソースを表すYAMLファイルのディレクトリである可能性があります。

YAMLファイル内には、  
{{<hover label="apiVersion" line="1">}}apiVersion{{</hover>}}、  
{{<hover label="apiVersion" line="2">}}kind{{</hover>}}、  
{{<hover label="apiVersion" line="3">}}metadata{{</hover>}}、および  
{{<hover label="apiVersion" line="7">}}spec{{</hover>}}を含めてください。

```yaml {label="apiVersion"}
apiVersion: example.org/v1alpha1
kind: ComposedResource
metadata:
  name: test-render-b
  annotations:
    crossplane.io/composition-resource-name: resource-b
spec:
  coolerField: "I'm cooler!"
```

リソースのスキーマは検証されず、任意のデータを含むことができます。

### beta top

コマンド`crossplane beta top`は、Crossplane関連のポッドのCPUおよびメモリ使用量を表示します。

```shell
crossplane beta top 
TYPE         NAMESPACE   NAME                                                       CPU(cores)   MEMORY
crossplane   default     crossplane-f98f9ddfd-tnm46                                 4m           32Mi
crossplane   default     crossplane-rbac-manager-74ff459b88-94p8p                   4m           14Mi
provider     default     provider-aws-s3-1f1a3fb08cbc-5c49d84447-sggrq              3m           108Mi
provider     default     upbound-provider-family-aws-48b3b5ccf964-76c9686b6-bgg65   2m           89Mi
```

{{<hint "important" >}}
`crossplane beta top`を使用するには、Kubernetes 
[metrics server](https://github.com/kubernetes-sigs/metrics-server)がCrossplaneを実行しているクラスターで有効になっている必要があります。

[metrics-server GitHubページ](https://github.com/kubernetes-sigs/metrics-server#installation)のインストール手順に従ってください。
{{< /hint >}}

#### フラグ
{{< table "table table-sm table-striped">}}
<!-- vale Crossplane.Spelling = NO -->
<!-- vale flags `dot` as an error but only the trailing tick. -->
| 短いフラグ   | 長いフラグ                   | 説明                                                                        |
| ------------ | -------------               | ------------------------------                                                     |
| `-n`         | `--namespace`               | Crossplaneポッドが実行される名前空間。デフォルトは`crossplane-system`です。                                                    |
| `-s`         | `--summary`                 | 出力とともにすべてのCrossplaneポッドの概要を印刷します。                |
|              | `--verbose`                 | 出力とともに詳細なログ情報を印刷します。                                                     |
<!-- vale Crossplane.Spelling = YES -->
{{< /table >}}


Kubernetes メトリクスサーバーは、`crossplane beta top` コマンドのデータを収集するのに時間がかかる場合があります。メトリクスサーバーが準備できるまで、`top` コマンドを実行するとエラーが発生することがあります。例えば、

`crossplane: error: error adding metrics to pod, check if metrics-server is running or wait until metrics are available for the pod: the server is currently unable to handle the request (get pods.metrics.k8s.io crossplane-contrib-provider-helm-b4cc4c2c8db3-6d787f9686-qzmz2)`


### beta trace

`crossplane beta trace` コマンドを使用して、Crossplane オブジェクトの視覚的関係を表示します。`trace` コマンドは、クレーム、構成、関数、管理リソース、またはパッケージをサポートしています。

このコマンドは、リソースタイプとリソース名を必要とします。

`crossplane beta trace <resource kind> <resource name>`

例えば、`example.crossplane.io` タイプの `my-claim` というリソースを表示するには：  
`crossplane beta trace example.crossplane.io my-claim`

このコマンドは、Kubernetes CLI スタイルの `<kind>/<name>` 入力も受け付けます。  
例えば、  
`crossplane beta trace example.crossplane.io/my-claim`

デフォルトでは、`crossplane beta trace` コマンドは `~/.kube/config` に定義された Kubernetes 設定を使用します。

環境変数 `KUBECONFIG` を使用して、カスタム Kubernetes 設定ファイルの場所を定義します。

#### フラグ
{{< table "table table-sm table-striped">}}
<!-- vale Crossplane.Spelling = NO -->
<!-- vale flags `dot` as an error but only the trailing tick. -->
| 短いフラグ   | 長いフラグ                   | 説明                                                                        |
| ------------ | -------------               | ------------------------------                                                     |
| `-n`         | `--namespace`               | リソースの名前空間。                                                     |
| `-o`         | `--output=`                 | グラフ出力を `wide`、`json`、または [Graphviz dot](https://graphviz.org/docs/layouts/dot/) 出力のために `dot` に変更します。 |
|              | `--show-connection-secrets` | 接続シークレット名を印刷します。シークレット値は印刷しません。                |
|              | `--show-package-dependencies <filter>` | パッケージの依存関係を表示します。オプションは、すべての依存関係を表示する `all`、パッケージを一度だけ印刷する `unique`、または依存関係を印刷しない `none` です。デフォルトでは、`trace` コマンドは `--show-package-dependencies unique` を使用します。                |
|              | `--show-package-revisions <output>`    | パッケージのリビジョンバージョンを印刷します。オプションは、アクティブなリビジョンのみを表示する `active`、すべてのリビジョンを表示する `all`、またはリビジョンを印刷しない `none` です。                 |
|              | `--show-package-runtime-configs` | DeploymentRuntimeConfig の依存関係を印刷します。                |
<!-- vale Crossplane.Spelling = YES -->
{{< /table >}}

#### 出力オプション

デフォルトでは `crossplane beta trace` は端末に直接出力し、"Ready" 条件と "Status" メッセージを64文字に制限します。

以下は、複数のコンポジションと構成リソースを含むAWSリファレンスプラットフォームからの "cluster" クレームの出力例です：

```shell {copy-lines="1"}
crossplane beta trace cluster.aws.platformref.upbound.io platform-ref-aws
NAME                                                                               VERSION   INSTALLED   HEALTHY   STATE    STATUS
Configuration/platform-ref-aws                                                     v0.9.0    True        True      -        HealthyPackageRevision
├─ ConfigurationRevision/platform-ref-aws-9ad7b5db2899                             v0.9.0    -           True      Active   HealthyPackageRevision
├─ Configuration/upbound-configuration-aws-network                                 v0.7.0    True        True      -        HealthyPackageRevision
│  ├─ ConfigurationRevision/upbound-configuration-aws-network-97be9100cfe1         v0.7.0    -           True      Active   HealthyPackageRevision
│  ├─ Provider/upbound-provider-aws-ec2                                            v0.47.0   True        True      -        HealthyPackageRevision
│  │  ├─ ProviderRevision/upbound-provider-aws-ec2-cfeb0cd0f1d2                    v0.47.0   -           True      Active   HealthyPackageRevision
│  │  └─ Provider/upbound-provider-family-aws                                      v1.0.0    True        True      -        HealthyPackageRevision
│  │     └─ ProviderRevision/upbound-provider-family-aws-48b3b5ccf964              v1.0.0    -           True      Active   HealthyPackageRevision
│  └─ Function/upbound-function-patch-and-transform                                v0.2.1    True        True      -        HealthyPackageRevision
│     └─ FunctionRevision/upbound-function-patch-and-transform-a2f88f8d8715        v0.2.1    -           True      Active   HealthyPackageRevision
├─ Configuration/upbound-configuration-aws-database                                v0.5.0    True        True      -        HealthyPackageRevision
│  ├─ ConfigurationRevision/upbound-configuration-aws-database-3112f0a765c5        v0.5.0    -           True      Active   HealthyPackageRevision
│  └─ Provider/upbound-provider-aws-rds                                            v0.47.0   True        True      -        HealthyPackageRevision
│     └─ ProviderRevision/upbound-provider-aws-rds-58f96aa9fc4b                    v0.47.0   -           True      Active   HealthyPackageRevision
├─ Configuration/upbound-configuration-aws-eks                                     v0.5.0    True        True      -        HealthyPackageRevision
│  ├─ ConfigurationRevision/upbound-configuration-aws-eks-83c9d65f4a47             v0.5.0    -           True      Active   HealthyPackageRevision
│  ├─ Provider/crossplane-contrib-provider-helm                                    v0.16.0   True        True      -        HealthyPackageRevision
│  │  └─ ProviderRevision/crossplane-contrib-provider-helm-b4cc4c2c8db3            v0.16.0   -           True      Active   HealthyPackageRevision
│  ├─ Provider/crossplane-contrib-provider-kubernetes                              v0.10.0   True        True      -        HealthyPackageRevision
│  │  └─ ProviderRevision/crossplane-contrib-provider-kubernetes-63506a3443e0      v0.10.0   -           True      Active   HealthyPackageRevision
│  ├─ Provider/upbound-provider-aws-eks                                            v0.47.0   True        True      -        HealthyPackageRevision
│  │  └─ ProviderRevision/upbound-provider-aws-eks-641a096d79d8                    v0.47.0   -           True      Active   HealthyPackageRevision
│  └─ Provider/upbound-provider-aws-iam                                            v0.47.0   True        True      -        HealthyPackageRevision
│     └─ ProviderRevision/upbound-provider-aws-iam-438eac423037                    v0.47.0   -           True      Active   HealthyPackageRevision
├─ Configuration/upbound-configuration-app                                         v0.2.0    True        True      -        HealthyPackageRevision
│  └─ ConfigurationRevision/upbound-configuration-app-5d95726dba8c                 v0.2.0    -           True      Active   HealthyPackageRevision
├─ Configuration/upbound-configuration-observability-oss                           v0.2.0    True        True      -        HealthyPackageRevision
│  ├─ ConfigurationRevision/upbound-configuration-observability-oss-a51529457ad7   v0.2.0    -           True      Active   HealthyPackageRevision
│  └─ Provider/grafana-provider-grafana                                            v0.8.0    True        True      -        HealthyPackageRevision
│     └─ ProviderRevision/grafana-provider-grafana-ac529c8ce1c6                    v0.8.0    -           True      Active   HealthyPackageRevision
└─ Configuration/upbound-configuration-gitops-flux                                 v0.2.0    True        True      -        HealthyPackageRevision
   └─ ConfigurationRevision/upbound-configuration-gitops-flux-2e80ec62738d         v0.2.0    -           True      Active   HealthyPackageRevision
```

#### ワイド出力
`--output=wide` を使用して、"Ready" または "Status" メッセージが64文字を超える場合は、全体を印刷します。

例えば、出力は長すぎる "Status" メッセージを切り捨てます。

```shell {copy-lines="1"
crossplane trace cluster.aws.platformref.upbound.io platform-ref-aws
NAME                                                              SYNCED   READY   STATUS
Cluster/platform-ref-aws (default)                                True     False   Waiting: ...resource claim is waiting for composite resource to become Ready
```

完全なメッセージを見るには `--output=wide` を使用します。

```shell {copy-lines="1"
crossplane trace cluster.aws.platformref.upbound.io platform-ref-aws --output=wide
NAME                                                              SYNCED   READY   STATUS
Cluster/platform-ref-aws (default)                                True     False   Waiting: Composite resource claim is waiting for composite resource to become Ready
```

#### Graphviz dotファイル出力

`--output=dot` を使用して、テキスト形式の 
[Graphviz dot](https://graphviz.org/docs/layouts/dot/) 出力を印刷します。

出力を保存してエクスポートするか、出力を直接Graphviz `dot` に渡して画像をレンダリングします。

例えば、出力を `graph.png` ファイルとして保存するには 
`dot -Tpng -o graph.png` を使用します。

`crossplane beta trace cluster.aws.platformref.upbound.io platform-ref-aws -o dot | dot -Tpng -o graph.png`

#### 接続シークレットの印刷

`-s` を使用して、他のリソースとともに接続シークレット名を印刷します。

{{<hint "important">}}
`crossplane beta trace` コマンドはシークレット値を印刷しません。
{{< /hint >}}

出力には、シークレット名とシークレットのネームスペースの両方が含まれます。

```shell
crossplane beta trace configuration platform-ref-aws -s
NAME                                                                        SYNCED   READY   STATUS
Cluster/platform-ref-aws (default)                                          True     True    Available
└─ XCluster/platform-ref-aws-mlnwb                                          True     True    Available
   ├─ XNetwork/platform-ref-aws-mlnwb-6nvkx                                 True     True    Available
   │  ├─ SecurityGroupRule/platform-ref-aws-mlnwb-szgxp                     True     True    Available
   │  └─ Secret/3f11c30b-dd94-4f5b-aff7-10fe4318ab1f (upbound-system)       -        -
   ├─ XEKS/platform-ref-aws-mlnwb-fqjzz                                     True     True    Available
   │  ├─ OpenIDConnectProvider/platform-ref-aws-mlnwb-h26xx                 True     True    Available
   │  └─ Secret/9666eccd-929c-4452-8658-c8c881aee137-eks (upbound-system)   -        -
   ├─ XServices/platform-ref-aws-mlnwb-bgndx                                True     True    Available
   │  ├─ Release/platform-ref-aws-mlnwb-7hfkv                               True     True    Available
   │  └─ Secret/d0955929-892d-40c3-b0e0-a8cabda55895 (upbound-system)       -        -
   └─ Secret/9666eccd-929c-4452-8658-c8c881aee137 (upbound-system)          -        -
```

#### パッケージ依存関係の表示

`--show-package-dependencies` フラグを使用して、パッケージ依存関係に関する詳細情報を含めます。

デフォルトでは `crossplane beta trace` は `--show-package-dependencies unique` を使用して、出力に必要なパッケージを一度だけ含めます。

`--show-package-dependencies all` を使用して、同じ依存関係を必要とするすべてのパッケージを表示します。

```shell
crossplane beta trace configuration platform-ref-aws --show-package-dependencies all
NAME                                                                               VERSION   INSTALLED   HEALTHY   STATE    STATUS
Configuration/platform-ref-aws                                                     v0.9.0    True        True      -        HealthyPackageRevision
├─ ConfigurationRevision/platform-ref-aws-9ad7b5db2899                             v0.9.0    -           True      Active   HealthyPackageRevision
├─ Configuration/upbound-configuration-aws-network                                 v0.7.0    True        True      -        HealthyPackageRevision
│  ├─ ConfigurationRevision/upbound-configuration-aws-network-97be9100cfe1         v0.7.0    -           True      Active   HealthyPackageRevision
│  ├─ Provider/upbound-provider-aws-ec2                                            v0.47.0   True        True      -        HealthyPackageRevision
│  │  ├─ ProviderRevision/upbound-provider-aws-ec2-cfeb0cd0f1d2                    v0.47.0   -           True      Active   HealthyPackageRevision
│  │  └─ Provider/upbound-provider-family-aws                                      v1.0.0    True        True      -        HealthyPackageRevision
│  │     └─ ProviderRevision/upbound-provider-family-aws-48b3b5ccf964              v1.0.0    -           True      Active   HealthyPackageRevision
│  └─ Function/upbound-function-patch-and-transform                                v0.2.1    True        True      -        HealthyPackageRevision
│     └─ FunctionRevision/upbound-function-patch-and-transform-a2f88f8d8715        v0.2.1    -           True      Active   HealthyPackageRevision
├─ Configuration/upbound-configuration-aws-database                                v0.5.0    True        True      -        HealthyPackageRevision
│  ├─ ConfigurationRevision/upbound-configuration-aws-database-3112f0a765c5        v0.5.0    -           True      Active   HealthyPackageRevision
│  ├─ Provider/upbound-provider-aws-rds                                            v0.47.0   True        True      -        HealthyPackageRevision
│  │  ├─ ProviderRevision/upbound-provider-aws-rds-58f96aa9fc4b                    v0.47.0   -           True      Active   HealthyPackageRevision
│  │  └─ Provider/upbound-provider-family-aws                                      v1.0.0    True        True      -        HealthyPackageRevision
│  │     └─ ProviderRevision/upbound-provider-family-aws-48b3b5ccf964              v1.0.0    -           True      Active   HealthyPackageRevision
│  └─ Configuration/upbound-configuration-aws-network                              v0.7.0    True        True      -        HealthyPackageRevision
│     ├─ ConfigurationRevision/upbound-configuration-aws-network-97be9100cfe1      v0.7.0    -           True      Active   HealthyPackageRevision
│     ├─ Provider/upbound-provider-aws-ec2                                         v0.47.0   True        True      -        HealthyPackageRevision
│     │  ├─ ProviderRevision/upbound-provider-aws-ec2-cfeb0cd0f1d2                 v0.47.0   -           True      Active   HealthyPackageRevision
│     │  └─ Provider/upbound-provider-family-aws                                   v1.0.0    True        True      -        HealthyPackageRevision
│     │     └─ ProviderRevision/upbound-provider-family-aws-48b3b5ccf964           v1.0.0    -           True      Active   HealthyPackageRevision
│     └─ Function/upbound-function-patch-and-transform                             v0.2.1    True        True      -        HealthyPackageRevision
│        └─ FunctionRevision/upbound-function-patch-and-transform-a2f88f8d8715     v0.2.1    -           True      Active   HealthyPackageRevision
├─ Configuration/upbound-configuration-aws-eks                                     v0.5.0    True        True      -        HealthyPackageRevision
│  ├─ ConfigurationRevision/upbound-configuration-aws-eks-83c9d65f4a47             v0.5.0    -           True      Active   HealthyPackageRevision
│  ├─ Configuration/upbound-configuration-aws-network                              v0.7.0    True        True      -        HealthyPackageRevision
│  │  ├─ ConfigurationRevision/upbound-configuration-aws-network-97be9100cfe1      v0.7.0    -           True      Active   HealthyPackageRevision
│  │  ├─ Provider/upbound-provider-aws-ec2                                         v0.47.0   True        True      -        HealthyPackageRevision
│  │  │  ├─ ProviderRevision/upbound-provider-aws-ec2-cfeb0cd0f1d2                 v0.47.0   -           True      Active   HealthyPackageRevision
│  │  │  └─ Provider/upbound-provider-family-aws                                   v1.0.0    True        True      -        HealthyPackageRevision
│  │  │     └─ ProviderRevision/upbound-provider-family-aws-48b3b5ccf964           v1.0.0    -           True      Active   HealthyPackageRevision
│  │  └─ Function/upbound-function-patch-and-transform                             v0.2.1    True        True      -        HealthyPackageRevision
│  │     └─ FunctionRevision/upbound-function-patch-and-transform-a2f88f8d8715     v0.2.1    -           True      Active   HealthyPackageRevision
│  ├─ Provider/crossplane-contrib-provider-helm                                    v0.16.0   True        True      -        HealthyPackageRevision
│  │  └─ ProviderRevision/crossplane-contrib-provider-helm-b4cc4c2c8db3            v0.16.0   -           True      Active   HealthyPackageRevision
│  ├─ Provider/crossplane-contrib-provider-kubernetes                              v0.10.0   True        True      -        HealthyPackageRevision
│  │  └─ ProviderRevision/crossplane-contrib-provider-kubernetes-63506a3443e0      v0.10.0   -           True      Active   HealthyPackageRevision
│  ├─ Provider/upbound-provider-aws-ec2                                            v0.47.0   True        True      -        HealthyPackageRevision
│  │  ├─ ProviderRevision/upbound-provider-aws-ec2-cfeb0cd0f1d2                    v0.47.0   -           True      Active   HealthyPackageRevision
│  │  └─ Provider/upbound-provider-family-aws                                      v1.0.0    True        True      -        HealthyPackageRevision
│  │     └─ ProviderRevision/upbound-provider-family-aws-48b3b5ccf964              v1.0.0    -           True      Active   HealthyPackageRevision
│  ├─ Provider/upbound-provider-aws-eks                                            v0.47.0   True        True      -        HealthyPackageRevision
│  │  ├─ ProviderRevision/upbound-provider-aws-eks-641a096d79d8                    v0.47.0   -           True      Active   HealthyPackageRevision
│  │  └─ Provider/upbound-provider-family-aws                                      v1.0.0    True        True      -        HealthyPackageRevision
│  │     └─ ProviderRevision/upbound-provider-family-aws-48b3b5ccf964              v1.0.0    -           True      Active   HealthyPackageRevision
│  ├─ Provider/upbound-provider-aws-iam                                            v0.47.0   True        True      -        HealthyPackageRevision
│  │  ├─ ProviderRevision/upbound-provider-aws-iam-438eac423037                    v0.47.0   -           True      Active   HealthyPackageRevision
│  │  └─ Provider/upbound-provider-family-aws                                      v1.0.0    True        True      -        HealthyPackageRevision
│  │     └─ ProviderRevision/upbound-provider-family-aws-48b3b5ccf964              v1.0.0    -           True      Active   HealthyPackageRevision
│  └─ Function/upbound-function-patch-and-transform                                v0.2.1    True        True      -        HealthyPackageRevision
│     └─ FunctionRevision/upbound-function-patch-and-transform-a2f88f8d8715        v0.2.1    -           True      Active   HealthyPackageRevision
├─ Configuration/upbound-configuration-app                                         v0.2.0    True        True      -        HealthyPackageRevision
│  ├─ ConfigurationRevision/upbound-configuration-app-5d95726dba8c                 v0.2.0    -           True      Active   HealthyPackageRevision
│  ├─ Provider/crossplane-contrib-provider-helm                                    v0.16.0   True        True      -        HealthyPackageRevision
│  │  └─ ProviderRevision/crossplane-contrib-provider-helm-b4cc4c2c8db3            v0.16.0   -           True      Active   HealthyPackageRevision
│  └─ Function/upbound-function-patch-and-transform                                v0.2.1    True        True      -        HealthyPackageRevision
│     └─ FunctionRevision/upbound-function-patch-and-transform-a2f88f8d8715        v0.2.1    -           True      Active   HealthyPackageRevision
├─ Configuration/upbound-configuration-observability-oss                           v0.2.0    True        True      -        HealthyPackageRevision
│  ├─ ConfigurationRevision/upbound-configuration-observability-oss-a51529457ad7   v0.2.0    -           True      Active   HealthyPackageRevision
│  ├─ Provider/crossplane-contrib-provider-helm                                    v0.16.0   True        True      -        HealthyPackageRevision
│  │  └─ ProviderRevision/crossplane-contrib-provider-helm-b4cc4c2c8db3            v0.16.0   -           True      Active   HealthyPackageRevision
│  ├─ Provider/crossplane-contrib-provider-kubernetes                              v0.10.0   True        True      -        HealthyPackageRevision
│  │  └─ ProviderRevision/crossplane-contrib-provider-kubernetes-63506a3443e0      v0.10.0   -           True      Active   HealthyPackageRevision
│  ├─ Provider/grafana-provider-grafana                                            v0.8.0    True        True      -        HealthyPackageRevision
│  │  └─ ProviderRevision/grafana-provider-grafana-ac529c8ce1c6                    v0.8.0    -           True      Active   HealthyPackageRevision
│  └─ Function/upbound-function-patch-and-transform                                v0.2.1    True        True      -        HealthyPackageRevision
│     └─ FunctionRevision/upbound-function-patch-and-transform-a2f88f8d8715        v0.2.1    -           True      Active   HealthyPackageRevision
└─ Configuration/upbound-configuration-gitops-flux                                 v0.2.0    True        True      -        HealthyPackageRevision
   ├─ ConfigurationRevision/upbound-configuration-gitops-flux-2e80ec62738d         v0.2.0    -           True      Active   HealthyPackageRevision
   ├─ Provider/crossplane-contrib-provider-helm                                    v0.16.0   True        True      -        HealthyPackageRevision
   │  └─ ProviderRevision/crossplane-contrib-provider-helm-b4cc4c2c8db3            v0.16.0   -           True      Active   HealthyPackageRevision
   └─ Function/upbound-function-patch-and-transform                                v0.2.1    True        True      -        HealthyPackageRevision
      └─ FunctionRevision/upbound-function-patch-and-transform-a2f88f8d8715        v0.2.1    -           True      Active   HealthyPackageRevision
```

`--show-package-dependencies none` を使用して、すべての依存関係を非表示にします。

```shell
crossplane beta trace configuration platform-ref-aws --show-package-dependencies none
NAME                                                     VERSION   INSTALLED   HEALTHY   STATE    STATUS
Configuration/platform-ref-aws                           v0.9.0    True        True      -        HealthyPackageRevision
└─ ConfigurationRevision/platform-ref-aws-9ad7b5db2899   v0.9.0    -           True      Active   HealthyPackageRevision
```

#### パッケージリビジョンの表示

デフォルトでは `crossplane beta trace` コマンドは、現在使用中のパッケージリビジョンのみを表示します。アクティブおよび非アクティブのリビジョンの両方を表示するには、`--show-package-revisions all` を使用します。

```shell
crossplane beta trace configuration platform-ref-aws --show-package-revisions all
NAME                                                                               VERSION   INSTALLED   HEALTHY   STATE      STATUS
Configuration/platform-ref-aws                                                     v0.9.0    True        True      -          HealthyPackageRevision
├─ ConfigurationRevision/platform-ref-aws-ad01153c1179                             v0.8.0    -           True      Inactive   HealthyPackageRevision
├─ ConfigurationRevision/platform-ref-aws-9ad7b5db2899                             v0.9.0    -           True      Active     HealthyPackageRevision
├─ Configuration/upbound-configuration-aws-network                                 v0.2.0    True        True      -          HealthyPackageRevision
│  ├─ ConfigurationRevision/upbound-configuration-aws-network-288fcd1b88dd         v0.2.0    -           True      Active     HealthyPackageRevision
│  └─ Provider/upbound-provider-aws-ec2                                            v1.0.0    True        True      -          HealthyPackageRevision
│     ├─ ProviderRevision/upbound-provider-aws-ec2-5cfd948d082f                    v1.0.0    -           True      Active     HealthyPackageRevision
│     └─ Provider/upbound-provider-family-aws                                      v1.0.0    True        True      -          HealthyPackageRevision
│        └─ ProviderRevision/upbound-provider-family-aws-48b3b5ccf964              v1.0.0    -           True      Active     HealthyPackageRevision
# Removed for brevity
```

すべてのリビジョンを非表示にするには、`--show-package-revision none` を使用します。

```shell
crossplane beta trace configuration platform-ref-aws --show-package-revisions none
NAME                                                       VERSION   INSTALLED   HEALTHY   STATE   STATUS
Configuration/platform-ref-aws                             v0.9.0    True        True      -       HealthyPackageRevision
├─ Configuration/upbound-configuration-aws-network         v0.2.0    True        True      -       HealthyPackageRevision
│  └─ Provider/upbound-provider-aws-ec2                    v1.0.0    True        True      -       HealthyPackageRevision
│     └─ Provider/upbound-provider-family-aws              v1.0.0    True        True      -       HealthyPackageRevision
# Removed for brevity
```

### beta validate

`crossplane beta validate` コマンドは、Kubernetes API サーバーの検証ライブラリを使用して、プロバイダーまたは XRD スキーマに対して [コンポジション]({{<ref "../concepts/compositions">}}) を検証します。

`crossplane beta validate` コマンドは、以下のシナリオの検証をサポートしています：

- 管理リソースまたは複合リソースを 
  [プロバイダーまたは XRD スキーマに対して検証](#validate-resources-against-a-schema)します。 
- `crossplane beta render` の出力を [検証入力](#validate-render-command-output)として使用します。 
- [XRD を Kubernetes 共通表現言語](#validate-common-expression-language-rules) 
  (CEL) ルールに対して検証します。
- リソースを [スキーマのディレクトリに対して検証](#validate-against-a-directory-of-schemas)します。


{{< hint "note" >}}
`crossplane beta validate` コマンドは、すべての検証をオフラインで実行します。 

Crossplane を実行している Kubernetes クラスターは必要ありません。 
{{< /hint >}}

#### フラグ

{{< table "table table-sm table-striped" >}}
| 短いフラグ   | 長いフラグ                | 説明                                           |
| ------------ | ------------------------ | ----------------------------------------------------- |
| `-h`         | `--help`                 | コンテキストに応じたヘルプを表示します。                          |
| `-v`         | `--version`              | バージョンを表示して終了します。                               |
|              | `--cache-dir=".crossplane/cache"` | ダウンロードしたスキーマを保存するキャッシュディレクトリの絶対パスを指定します。 |
|              | `--clean-cache`          | パッケージスキーマをダウンロードする前にキャッシュディレクトリをクリーンします。 |
|              | `--skip-success-results` | 成功結果の印刷をスキップします。                        |
|              | `--verbose`              | 詳細なログメッセージを表示します。                     |
{{< /table >}}

#### スキーマに対するリソースの検証

`crossplane beta validate` コマンドは、XR および 1 つ以上の管理リソースをプロバイダーのスキーマに対して検証できます。

{{<hint "important" >}}
プロバイダーに対して検証する際、`crossplane beta validate` コマンドはプロバイダー パッケージを `--cache-dir` ディレクトリにダウンロードします。デフォルトでは、Crossplane は `.crossplane` を `--cache-dir` の場所として使用します。

Kubernetes クラスターや Crossplane ポッドへのアクセスは必要ありません。  
検証にはプロバイダー パッケージをダウンロードする能力が必要です。
{{< /hint >}}

`crossplane beta validate` コマンドは、スキーマ CRD ファイルを `--cache-dir` ディレクトリにダウンロードしてキャッシュします。デフォルトで、Crossplane CLI は `.crossplane/cache` をキャッシュの場所として使用します。

キャッシュをクリアして CRD ファイルを再度ダウンロードするには、`--clean-cache` フラグを使用します。

プロバイダーに対して管理リソースを検証するには、まずプロバイダー マニフェストファイルを作成します。たとえば、プロバイダー AWS から IAM ロールを検証するには、 
[Provider AWS IAM](https://marketplace.upbound.io/providers/upbound/provider-aws-iam/v1.0.0) 
マニフェストを使用します。

{{<hint "tip" >}}
"[ファミリープロバイダー](https://blog.upbound.io/new-provider-families)" を検証するには、検証するリソースのプロバイダー マニフェストを使用してください。
{{< /hint >}}

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-iam
spec:
  package: xpkg.upbound.io/upbound/provider-aws-iam:v1.0.0
```

検証する XR または管理リソースを含めます。

たとえば、 
{{<hover label="iamAK" line="2">}}AccessKey{{</hover>}} 管理リソースを検証するには、管理リソース YAML ファイルを提供します。

```yaml {label="iamAK"}
apiVersion: iam.aws.upbound.io/v1beta1
kind: AccessKey
metadata:
  name: sample-access-key-0
spec:
  forProvider:
    userSelector:
      matchLabels:
        example-name: test-user-0
```

プロバイダーおよび管理リソース YAML ファイルを入力として提供して、`crossplane beta validate` コマンドを実行します。

```shell
crossplane beta validate provider.yaml managedResource.yaml
[✓] iam.aws.upbound.io/v1beta1, Kind=AccessKey, sample-access-key-0 validated successfully
Total 1 resources: 0 missing schemas, 1 success case, 0 failure cases
```


#### レンダーコマンド出力の検証

`crossplane beta render` の出力を `crossplane beta validate` にパイプすることで、XR、コンポジション、コンポジション関数を含む完全な Crossplane リソースパイプラインを検証できます。

`crossplane beta render` コマンドに `--include-full-xr` オプションを使用し、`crossplane beta validate` コマンドに `-` オプションを使用して、`crossplane beta render` の出力を `crossplane beta validate` の入力にパイプします。

```shell {copy-lines="1"}
crossplane beta render xr.yaml composition.yaml function.yaml --include-full-xr | crossplane beta validate schemas.yaml -
[x] schema validation error example.crossplane.io/v1beta1, Kind=XR, example : status.conditions[0].lastTransitionTime: Invalid value: "null": status.conditions[0].lastTransitionTime in body must be of type string: "null"
[x] schema validation error example.crossplane.io/v1beta1, Kind=XR, example : spec: Required value
[✓] iam.aws.upbound.io/v1beta1, Kind=AccessKey, sample-access-key-0 validated successfully
[✓] iam.aws.upbound.io/v1beta1, Kind=AccessKey, sample-access-key-1 validated successfully
[✓] iam.aws.upbound.io/v1beta1, Kind=User, test-user-0 validated successfully
[✓] iam.aws.upbound.io/v1beta1, Kind=User, test-user-1 validated successfully
Total 5 resources: 0 missing schemas, 4 success cases, 1 failure cases
```

#### 共通式言語ルールの検証
XRD は、共通式言語 ([CEL](https://kubernetes.io/docs/reference/using-api/cel/)) で表現された [検証ルール](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#validation-rules) を定義できます。

XRD のスキーマ {{<hover label="celXRD" line="10" >}}spec{{</hover>}} オブジェクト内に、{{<hover label="celXRD" line="12" >}}x-kubernetes-validations{{</hover>}} キーを使用して CEL ルールを適用します。

```yaml {label="celXRD"}
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: myXR.crossplane.io
spec:
# Removed for brevity
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              x-kubernetes-validations:
              - rule: "self.minReplicas <= self.replicas && self.replicas <= self.maxReplicas"
                message: "replicas should be in between minReplicas and maxReplicas."
              properties:
                minReplicas:
                  type: integer
                maxReplicas:
                  type: integer
                replicas: 
                  type: integer
# Removed for brevity
```

この例のルールは、XR の {{<hover label="celXR" line="6">}}replicas{{</hover>}} フィールドの値が {{<hover label="celXR" line="7">}}minReplicas{{</hover>}} と {{<hover label="celXR" line="8">}}maxReplicas{{</hover>}} の間にあることを確認します。

```yaml {label="celXR"}
apiVersion: example.crossplane.io/v1beta1
kind: XR
metadata:
  name: example
spec:
  replicas: 49
  minReplicas: 1
  maxReplicas: 30
```

例の XRD と XR で `crossplane beta validate` を実行すると、エラーが発生します。

```shell
`crossplane beta validate xrd.yaml xr.yaml
[x] CEL validation error example.crossplane.io/v1beta1, Kind=XR, example : spec: Invalid value: "object": replicas should be in between minReplicas and maxReplicas.
Total 1 resources: 0 missing schemas, 0 success cases, 1 failure cases
```

#### スキーマのディレクトリに対して検証

`crossplane beta render` コマンドは、YAML ファイルのディレクトリを検証できます。

このコマンドは `.yaml` および `.yml` ファイルのみを処理し、他のすべてのファイルタイプは無視します。

ファイルのディレクトリを使用して、検証するディレクトリとリソースを指定します。

例えば、XRD と Provider スキーマを含む {{<hover label="validateDir" line="2">}}schemas{{</hover>}} という名前のディレクトリを使用します。

```shell {label="validateDir"}
tree
schemas
|-- platform-ref-aws.yaml
|-- providers
|   |-- a.txt
|   `-- provider-aws-iam.yaml
`-- xrds
    `-- xrd.yaml
```

ディレクトリ名とリソース YAML ファイルを `crossplane beta validate` コマンドに提供します。

```shell
crossplane beta validate schema resources.yaml
[x] schema validation error example.crossplane.io/v1beta1, Kind=XR, example : status.conditions[0].lastTransitionTime: Invalid value: "null": status.conditions[0].lastTransitionTime in body must be of type string: "null"
[x] CEL validation error example.crossplane.io/v1beta1, Kind=XR, example : spec: Invalid value: "object": no such key: minReplicas evaluating rule: replicas should be greater than or equal to minReplicas.
[✓] iam.aws.upbound.io/v1beta1, Kind=AccessKey, sample-access-key-0 validated successfully
[✓] iam.aws.upbound.io/v1beta1, Kind=AccessKey, sample-access-key-1 validated successfully
[✓] iam.aws.upbound.io/v1beta1, Kind=User, test-user-0 validated successfully
[✓] iam.aws.upbound.io/v1beta1, Kind=User, test-user-1 validated successfully
Total 5 resources: 0 missing schemas, 4 success cases, 1 failure cases
```

### beta xpkg init

`crossplane beta xpkg init` コマンドは、パッケージを構築するためのファイルで現在のディレクトリを埋めます。

パッケージに使用する名前と、開始するパッケージテンプレートをコマンドで指定します  
`crossplane beta xpkg init <name> <template>`

`<name>` の入力は使用されません。Crossplane は将来のリリースのために `<name>` を予約しています。

`<template>` の値は、次の4つのよく知られたテンプレートのいずれかである必要があります：
* `configuration-template` - [crossplane/configuration-template](https://github.com/crossplane/configuration-template) リポジトリから Crossplane [Configuration]({{<ref "../concepts/packages">}}) を構築するためのテンプレート。
* `function-template-go` - [crossplane/function-template-go](https://github.com/crossplane/function-template-go) リポジトリから Crossplane Go [composition functions]({{<ref "../concepts/composition-functions">}}) を構築するためのテンプレート。
* `function-template-python` - [crossplane/function-template-python](https://github.com/crossplane/function-template-go) リポジトリから Crossplane Python [composition functions]({{<ref "../concepts/composition-functions">}}) を構築するためのテンプレート。
* `provider-template` - [Crossplane/provider-template](https://github.com/crossplane/provider-template) リポジトリから基本的な Crossplane プロバイダーを構築するためのテンプレート。
* `provider-template-upjet` - 既存の Terraform プロバイダーから [Upjet](https://github.com/crossplane/upjet) ベースの Crossplane プロバイダーを構築するためのテンプレート。 [upbound/upjet-provider-template](https://github.com/upbound/upjet-provider-template) リポジトリからコピーします。

よく知られたテンプレートの代わりに、`<template>` の値は git リポジトリの URL であることもできます。

#### NOTES.txt

テンプレートリポジトリのルートディレクトリに `NOTES.txt` ファイルが含まれている場合、`crossplane beta xpkg init` コマンドは、テンプレートファイルでディレクトリを埋めた後にファイルの内容をターミナルに出力します。これは、テンプレートに関する情報を提供するのに役立ちます。

#### init.sh

テンプレートリポジトリのルートディレクトリに `init.sh` ファイルが含まれている場合、`crossplane beta xpkg init` コマンドは、テンプレートファイルでディレクトリを埋めた後にダイアログを開始します。ダイアログは、ユーザーにスクリプトを表示または実行するかどうかを促します。初期化スクリプトを使用して、テンプレートを自動的にパーソナライズします。

#### フラグ
{{< table "table table-sm table-striped">}}
| 短いフラグ   | 長いフラグ               | 説明                                                                                      |
| ------------ | ----------------------- | ------------------------------                                                                   |
| `-b`         | `--ref-name`            | テンプレートリポジトリからクローンするブランチまたはタグ。                                         |
| `-d`         | `--directory`           | テンプレートファイルを作成して読み込むディレクトリ。デフォルトでは現在のディレクトリを使用します。 |
| `-r`         | `--run-init-script`     | 存在する場合、プロンプトなしでinit.shスクリプトを実行します。                                                        |
<!-- vale Crossplane.Spelling = YES -->
{{< /table >}}
