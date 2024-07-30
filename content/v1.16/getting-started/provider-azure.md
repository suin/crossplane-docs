---
title: Azure クイックスタート 
weight: 110
---

Crossplane を Azure に接続して、Kubernetes からクラウドリソースを作成および管理します 
[Upbound Azure Provider](https://marketplace.upbound.io/providers/upbound/provider-family-azure/) を使用します。

このガイドは2つの部分で構成されています：
* パート1では、Crossplane のインストール、プロバイダーの設定を通じて Azure への認証、および Kubernetes クラスターから直接 Azure に _Managed Resource_ を作成する手順を説明します。これにより、Crossplane が Azure と通信できることが示されます。
* [パート2]({{< ref "provider-azure-part-2" >}}) では、Crossplane を使用してカスタム API を構築し、アクセスする方法を示します。

## 前提条件
このクイックスタートには以下が必要です：
* 最低 2 GB の RAM を持つ Kubernetes クラスター
* Kubernetes クラスター内でポッドとシークレットを作成する権限
* [Helm](https://helm.sh/) バージョン v3.2.0 以上
* [Azure Virtual Machine](https://learn.microsoft.com/en-us/azure/virtual-machines/) を作成する権限を持つ Azure アカウント
  および
  [Virtual Network](https://learn.microsoft.com/en-us/azure/virtual-network/)
* Azure [service principal](https://learn.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals#service-principal-object) および [Azure resource group](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal) を作成する権限を持つ Azure アカウント

{{<include file="/master/getting-started/install-crossplane-include.md" type="page" >}}

## Azure プロバイダーのインストール

Kubernetes 構成ファイルを使用して、Kubernetes クラスターに Azure Network リソースプロバイダーをインストールします。

```yaml {label="provider",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-network
spec:
  package: xpkg.upbound.io/upbound/provider-azure-network:v0.42.1
EOF
```

Crossplane {{< hover label="provider" line="3" >}}Provider{{</hover>}}
は、Azure Networking サービスを表す Kubernetes _Custom Resource Definitions_ (CRDs) をインストールします。これらの CRD により、Kubernetes 内で直接 Azure リソースを作成できます。

`kubectl get providers` を使用してプロバイダーがインストールされたことを確認します。

```shell {copy-lines="1",label="getProvider"}
kubectl get providers
NAME                            INSTALLED   HEALTHY   PACKAGE                                                  AGE
provider-azure-network          True        True      xpkg.upbound.io/upbound/provider-azure-network:v0.42.1   38s
upbound-provider-family-azure   True        True      xpkg.upbound.io/upbound/provider-family-azure:v0.42.1    26s
```


ネットワークプロバイダーは、2番目のプロバイダーである
{{<hover label="getProvider" line="4">}}upbound-provider-family-azure{{</hover>}} 
プロバイダーをインストールします。  
ファミリープロバイダーは、すべてのAzureファミリープロバイダーにわたってAzureへの認証を管理します。 

新しいCRDは`kubectl get crds`で表示できます。  
すべてのCRDは、Crossplaneがプロビジョニングおよび管理できるユニークなAzureサービスにマッピングされています。

{{< hint type="tip" >}}
サポートされているすべてのCRDの詳細は、 
[Upbound Marketplace](https://marketplace.upbound.io/providers/upbound/provider-family-azure/v0.42.1)を参照してください。
{{< /hint >}}


## Azure用のKubernetesシークレットを作成する
プロバイダーは、Azureリソースを作成および管理するための資格情報を必要とします。 
プロバイダーは、資格情報をプロバイダーに接続するためにKubernetes _Secret_ を使用します。

このガイドでは、AzureサービスプリンシパルのJSONファイルを生成し、Kubernetes _Secret_ として保存します。

### Azureコマンドラインをインストールする
[認証ファイル](https://docs.microsoft.com/en-us/azure/developer/go/azure-sdk-authorization#use-file-based-authentication)を生成するには、Azureコマンドラインが必要です。  
Microsoftのドキュメントに従って、[Azureコマンドラインをダウンロードしてインストール](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)してください。

Azureコマンドラインにログインします。

```command
az login
```
### Azureサービスプリンシパルを作成する
Azureポータルから[サブスクリプションIDを見つける](https://docs.microsoft.com/en-us/azure/azure-portal/get-subscription-tenant-id)ためにAzureのドキュメントに従ってください。

Azureコマンドラインを使用し、サブスクリプションIDを提供してサービスプリンシパルと認証ファイルを作成します。

{{< editCode >}}
```console {copy-lines="all"}
az ad sp create-for-rbac \
--sdk-auth \
--role Owner \
--scopes /subscriptions/$@<subscription_id>$@
```
{{< /editCode >}}

AzureのJSON出力を`azure-credentials.json`として保存します。

{{< hint type="note" >}}
Azureプロバイダーのドキュメントの
[認証](https://docs.upbound.io/providers/provider-azure/authentication/) 
セクションでは、他の認証方法について説明しています。
{{< /hint >}}

### Azure資格情報を使用してKubernetesシークレットを作成する
Kubernetesの一般的なシークレットには名前と内容があります。 {{< hover label="kube-create-secret" line="1">}}kubectl create secret{{< /hover >}}を使用して、{{< hover label="kube-create-secret" line="2">}}azure-secret{{< /hover >}}という名前のシークレットオブジェクトを{{< hover label="kube-create-secret" line="3">}}crossplane-system{{</ hover >}}名前空間に生成します。  

<!-- vale gitlab.Substitutions = NO -->
<!-- ignore .json file name -->
`{{< hover label="kube-create-secret" line="4">}}--from-file={{</hover>}}` 引数を使用して、`{{< hover label="kube-create-secret" line="4">}}azure-credentials.json{{< /hover >}}` ファイルの内容を値として設定します。
<!-- vale gitlab.Substitutions = YES -->
```shell {label="kube-create-secret",copy-lines="all"}
kubectl create secret \
generic azure-secret \
-n crossplane-system \
--from-file=creds=./azure-credentials.json
```

`kubectl describe secret` でシークレットを表示します。

{{< hint type="note" >}}
テキストファイルに余分な空白がある場合、サイズが大きくなることがあります。
{{< /hint >}}

```shell {copy-lines="1"}
kubectl describe secret azure-secret -n crossplane-system
Name:         azure-secret
Namespace:    crossplane-system
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
creds:  629 bytes
```

## ProviderConfigの作成
`ProviderConfig` はAzureプロバイダーの設定をカスタマイズします。  

次のコマンドで `{{< hover label="providerconfig" line="5">}}ProviderConfig{{</ hover >}}` を適用します：
```yaml {label="providerconfig",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: azure.upbound.io/v1beta1
metadata:
  name: default
kind: ProviderConfig
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azure-secret
      key: creds
EOF
```

これにより、Kubernetesシークレットとして保存されたAzureの資格情報が `{{< hover label="providerconfig" line="9">}}secretRef{{</ hover>}}` として添付されます。

`{{< hover label="providerconfig" line="11">}}spec.credentials.secretRef.name{{< /hover >}}` の値は、`{{< hover label="providerconfig" line="10">}}spec.credentials.secretRef.namespace{{< /hover >}}` にあるAzureの資格情報を含むKubernetesシークレットの名前です。


## 管理リソースの作成
_管理リソース_ は、CrossplaneがKubernetesクラスターの外部で作成および管理するすべてのものです。この例では、Crossplaneを使用してAzure仮想ネットワークを作成します。仮想ネットワークは_管理リソース_です。

{{< hint type="note" >}}
Azureリソースグループ名を追加してください。リソースグループを作成するには、Azureのドキュメントに従って 
[リソースグループを作成](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal)してください。
{{< /hint >}}

{{< editCode >}}
```yaml {label="xr"}
cat <<EOF | kubectl create -f -
apiVersion: network.azure.upbound.io/v1beta1
kind: VirtualNetwork
metadata:
  name: crossplane-quickstart-network
spec:
  forProvider:
    addressSpace:
      - 10.0.0.0/16
    location: "Sweden Central"
    resourceGroupName: docs
EOF
```
{{< /editCode >}}

`{{< hover label="xr" line="2">}}apiVersion{{< /hover >}}` と 
`{{< hover label="xr" line="3">}}kind{{</hover >}}` はプロバイダーのCRDからのものです。


{{< hover label="xr" line="10">}}spec.forProvider.location{{< /hover >}} 
は、Azureにリソースをデプロイする際に使用する場所を指示します。 

`kubectl get virtualnetwork.network`を使用して、CrossplaneがAzure Virtual Networkを作成したことを確認します。

{{< hint type="tip" >}}
Crossplaneは、値が`READY`および`SYNCED`が`True`のときに仮想ネットワークを作成しました。  
これには最大5分かかる場合があります。  
{{< /hint >}}

```shell {copy-lines="1"}
kubectl get virtualnetwork.network
NAME                            READY   SYNCED   EXTERNAL-NAME                   AGE
crossplane-quickstart-network   True    True     crossplane-quickstart-network   10m
```

## 管理リソースの削除
Kubernetesクラスターをシャットダウンする前に、作成したばかりの仮想ネットワークを削除します。

`kubectl delete virtualnetwork.network`を使用して、仮想ネットワークを削除します。 


```shell {copy-lines="1"}
kubectl delete virtualnetwork.network crossplane-quickstart-network
virtualnetwork.network.azure.upbound.io "crossplane-quickstart-network" deleted
```

## 次のステップ
* [**パート2に進む**]({{< ref "provider-azure-part-2">}}) で、Crossplaneを使用してカスタムAPIを作成および使用します。
* Crossplaneが構成できるAzureリソースを[Provider CRDリファレンス](https://marketplace.upbound.io/providers/upbound/provider-family-azure/)で探索します。
* [Crossplane Slack](https://slack.crossplane.io/)に参加し、Crossplaneのユーザーや貢献者とつながります。
