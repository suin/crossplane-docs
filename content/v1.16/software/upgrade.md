---
title: Crossplaneのアップグレード
weight: 200
---

既存のCrossplaneインストールの推奨アップグレード方法は、[Helm](http://helm.io)を使用することです。

## 前提条件
* [Helm](https://helm.sh/docs/intro/install/) バージョン `v3.2.0` 以上
 

## Crossplane Helmリポジトリの追加
HelmにCrossplaneリポジトリが追加されていることを確認します。

```shell
helm repo add crossplane-stable https://charts.crossplane.io/stable
```

## Helmリポジトリの更新

`helm repo update`を使用して、ローカルのCrossplane Helmチャートを更新します。

```shell
helm repo update
```

{{<hint "重要" >}}
Helmチャートを更新せずにCrossplaneをアップグレードすると、ローカルにキャッシュされたHelmチャートの最新バージョンがインストールされます。
{{< /hint >}}

## Crossplaneのアップグレード

`helm upgrade`を使用してCrossplaneをアップグレードし、Crossplaneの名前空間を指定します。 
デフォルトでは、Crossplaneは`crossplane-system`名前空間にインストールされます。

```shell
helm upgrade crossplane --namespace crossplane-system crossplane-stable/crossplane
```

Helmは、Crossplaneをインストールする際に元々使用された引数やフラグを保持します。

Crossplaneは、`helm upgrade`コマンドで変更されない限り、新しいデフォルトの動作を使用します。

例えば、v1.15.0ではCrossplaneがデフォルトのイメージレジストリを`index.docker.io`から`xpkg.upbound.io`に変更しました。v1.15.0以前のバージョンからCrossplaneをアップグレードすると、デフォルトのパッケージレジストリが更新されます。

新しいデフォルトを上書きするには、アップグレードコマンドで[Helmチャートをカスタマイズ]({{<ref "install#customize-the-crossplane-helm-chart" >}})します。

例えば、元のイメージレジストリを維持するには、次のようにします。
```shell 
helm upgrade crossplane --namespace crossplane-system crossplane-stable/crossplane `--set 'args={"--registry=index.docker.io"}'
```
