---  
title: 自己署名CA証明書  
weight: 270   
---  

>  自己署名証明書を本番環境で使用することは推奨されていません。自己署名証明書はテストのみに使用することをお勧めします。

CrossplaneがプライベートレジストリからConfigurationおよびProviderパッケージをロードする際には、CAおよび中間証明書を信頼するように設定する必要があります。

Crossplaneは、`registryCaBundleConfig.name`および`registryCaBundleConfig.key`パラメータが定義されたHelmチャートを介してインストールする必要があります。詳細は[Crossplaneのインストール]({{<ref "../../master/software/install" >}})を参照してください。

## 設定

1. CAバンドルを作成します（特定の順序でルートおよび中間証明書を含むファイル）。これは、結果のファイルが必要なcrtファイルを正しい順序で含む限り、任意のテキストエディタまたはコマンドラインから行うことができます。多くの場合、これは単一の自己署名ルートCA crtファイル、または中間crtとルートcrtファイルのいずれかになります。crtファイルの順序は、署名順に最も低いものから最も高いものへと並べる必要があります。たとえば、ルート証明書の下に2つの証明書のチェーンがある場合、最下層の中間証明書をファイルの先頭に配置し、その証明書に署名した中間証明書、次にその証明書に署名したルート証明書を配置します。

2. ファイルを`[yourdomain].ca-bundle`として保存します。

3. CrossplaneシステムネームスペースにKubernetes ConfigMapを作成します：

```
kubectl -n [Crossplane system namespace] create cm ca-bundle-config \
--from-file=ca-bundle=./[yourdomain].ca-bundle
```

4. `registryCaBundleConfig.name` Helmチャートパラメータを`ca-bundle-config`に、`registryCaBundleConfig.key`パラメータを`ca-bundle`に設定します。

> Helmにパラメータ値を提供する方法は、Helmのドキュメント[Helm install](https://helm.sh/docs/helm/helm_install/)で説明されています。`override.yaml`ファイルの例のブロックは次のようになります：
```
  registryCaBundleConfig:
    name: ca-bundle-config
    key: ca-bundle
```
