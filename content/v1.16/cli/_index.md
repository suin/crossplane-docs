---
weight: 200
title: CLIリファレンス
description: "Crossplaneコマンドラインインターフェースのドキュメント"
---

Crossplane CLIは、Crossplaneの開発および管理のいくつかの側面を簡素化するのに役立ちます。

Crossplane CLIには以下が含まれています：
* Crossplaneパッケージのビルド、インストール、更新、プッシュのためのツール
* Crossplaneを実行しているKubernetesクラスターにアクセスすることなく、スタンドアロンのComposition Functionのテストとレンダリング
* CrossplaneのComposition、Composite Resources、およびManaged Resourcesのトラブルシューティング

## CLIのインストール

Crossplane CLIは、外部依存関係のない単一のスタンドアロンバイナリです。

{{<hint "note" >}}
ユーザーのコンピュータにCrossplane CLIをインストールします。

ほとんどのCrossplane CLIコマンドはKubernetesに依存せず、
Crossplaneポッドへのアクセスを必要としません。
{{< /hint >}} 

Crossplaneインストールスクリプトを使用して、CPUアーキテクチャに合わせた最新バージョンをダウンロードします。

```shell
curl -sL "https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh" | sh
```

[スクリプト](https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh)
は、CPUアーキテクチャを検出し、最新の安定リリースをダウンロードします。

{{<expand "Crossplane CLIを手動でインストール" >}}

シェルスクリプトを実行したくない場合は、Crossplaneリリースリポジトリから
バイナリを手動でダウンロードできます。
https://releases.crossplane.io/stable/current/bin

{{<hint "important" >}}
<!-- vale write-good.Passive = NO -->
CLIはリリースリポジトリで`crank`という名前です。このファイルをダウンロードしてください。
<!-- vale write-good.Passive = YES -->

`crossplane`バイナリはKubernetesのCrossplaneポッドイメージです。
{{< /hint >}}

バイナリを`$PATH`内の場所に移動します。例えば`/usr/local/bin`です。
{{< /expand >}}

### 他のCLIバージョンのダウンロード

`XP_CHANNEL`および`XP_VERSION`環境変数を使用して、異なるCrossplane CLIバージョンや異なるリリースブランチをダウンロードします。

デフォルトでは、CLIは`XP_CHANNEL`が`stable`、`XP_VERSION`が`current`の状態でインストールされ、最新の安定リリースに一致します。

例えば、CLIバージョン`v1.14.0`をインストールするには、ダウンロードスクリプトのcurlコマンドに`XP_VERSION=v1.14.0`を追加します：  

```
`curl -sL "https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh" | XP_VERSION=v1.14.0 sh`
```
