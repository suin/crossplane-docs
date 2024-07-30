---
title: GCP クイックスタート
weight: 140
---

CrossplaneをGCPに接続して、Kubernetesからクラウドリソースを作成および管理します 
[Upbound GCP Provider](https://marketplace.upbound.io/providers/upbound/provider-family-gcp/)を使用します。

このガイドは2つの部分で構成されています：
* パート1では、Crossplaneのインストール、GCPへの認証を行うためのプロバイダーの設定、およびKubernetesクラスターから直接GCPに_Managed Resource_を作成する手順を説明します。これにより、CrossplaneがGCPと通信できることが示されます。
* [パート2]({{< ref "provider-gcp-part-2" >}})では、Crossplaneを使用してカスタムAPIを構築し、アクセスする方法を示します。
## 前提条件
このクイックスタートには以下が必要です：
* 最低2GBのRAMを持つKubernetesクラスター
* Kubernetesクラスター内でポッドとシークレットを作成する権限
* [Helm](https://helm.sh/) バージョンv3.2.0以上
* ストレージバケットを作成する権限を持つGCPアカウント
* GCP [アカウントキー](https://cloud.google.com/iam/docs/creating-managing-service-account-keys)
* GCP [プロジェクトID](https://support.google.com/googleapi/answer/7014113?hl=en)

{{<include file="/master/getting-started/install-crossplane-include.md" type="page" >}}

## GCPプロバイダーのインストール

Kubernetes構成ファイルを使用して、プロバイダーをKubernetesクラスターにインストールします。

```shell {label="provider",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp-storage
spec:
  package: xpkg.upbound.io/upbound/provider-gcp-storage:v0.41.0
EOF
```

Crossplane {{< hover label="provider" line="3" >}}プロバイダー{{</hover>}}
は、GCPストレージサービスを表すKubernetes _カスタムリソース定義_ (CRD)をインストールします。これらのCRDを使用すると、Kubernetes内で直接GCPリソースを作成できます。

`kubectl get providers`を使用して、プロバイダーがインストールされたことを確認します。

```shell {copy-lines="1",label="getProvider"}
kubectl get providers
NAME                          INSTALLED   HEALTHY   PACKAGE                                                AGE
provider-gcp-storage          True        True      xpkg.upbound.io/upbound/provider-gcp-storage:v0.41.0   36s
upbound-provider-family-gcp   True        True      xpkg.upbound.io/upbound/provider-family-gcp:v0.41.0    29s
```

ストレージプロバイダーは、2番目のプロバイダーである
{{<hover label="getProvider" line="4">}}upbound-provider-family-gcp{{</hover>}} 
プロバイダーをインストールします。   
ファミリープロバイダーは、すべてのGCPファミリープロバイダーに対するGCPへの認証を管理します。

`kubectl get crds`を使用して、新しいCRDを表示できます。  
すべてのCRDは、Crossplaneがプロビジョニングおよび管理できるユニークなGCPサービスにマッピングされています。


{{< hint "tip" >}}
すべてのサポートされているCRDの詳細については、 
[Upbound Marketplace](https://marketplace.upbound.io/providers/upbound/provider-family-gcp/)を参照してください。
{{< /hint >}}


## GCP用のKubernetesシークレットを作成する
プロバイダーは、GCPリソースを作成および管理するための資格情報を必要とします。プロバイダーは、Kubernetesの_Secret_を使用して資格情報をプロバイダーに接続します。

まず、Google CloudサービスアカウントのJSONファイルからKubernetesの_Secret_を生成し、それを使用するようにプロバイダーを設定します。

### GCPサービスアカウントJSONファイルを生成する
基本的なユーザー認証には、Google CloudサービスアカウントのJSONファイルを使用します。

{{< hint "tip" >}}
[GCPドキュメント](https://cloud.google.com/iam/docs/creating-managing-service-account-keys) 
には、サービスアカウントのJSONファイルを生成する方法が記載されています。
{{< /hint >}}

このJSONファイルを`gcp-credentials.json`として保存します。


### GCP資格情報を使用してKubernetesシークレットを作成する
Kubernetesの一般的なシークレットには名前と内容があります。 
{{< hover label="kube-create-secret" line="1">}}kubectl create secret{{< /hover >}} 
を使用して、 
{{< hover label="kube-create-secret" line="2">}}gcp-secret{{< /hover >}}という名前のシークレットオブジェクトを 
{{< hover label="kube-create-secret" line="3">}}crossplane-system{{</ hover >}} 
名前空間に生成します。  
{{< hover label="kube-create-secret" line="4">}}--from-file={{</hover>}}
引数を使用して、 
{{< hover label="kube-create-secret" line="4">}}gcp-credentials.json{{< /hover >}} 
ファイルの内容を値として設定します。


```shell {label="kube-create-secret",copy-lines="all"}
kubectl create secret \
generic gcp-secret \
-n crossplane-system \
--from-file=creds=./gcp-credentials.json
```

`kubectl describe secret`を使用してシークレットを表示します。

{{< hint "note" >}}
ファイルサイズは内容によって異なる場合があります。
{{< /hint >}}

```shell {copy-lines="1"}
kubectl describe secret gcp-secret -n crossplane-system
Name:         gcp-secret
Namespace:    crossplane-system
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
creds:  2330 bytes
```

{{< hint type="note" >}}
[GCPプロバイダーのドキュメント](https://docs.upbound.io/providers/provider-gcp/authentication/)の
[認証](https://docs.upbound.io/providers/provider-gcp/authentication/)セクションでは、他の認証方法について説明しています。
{{< /hint >}}

## ProviderConfigを作成する
`ProviderConfig`は、GCPプロバイダーの設定をカスタマイズします。

あなたの 
{{< hover label="providerconfig" line="7" >}}GCPプロジェクトID{{< /hover >}}を
_ProviderConfig_ 設定に含めてください。

{{< hint "tip" >}}
`gcp-credentials.json` ファイルの `project_id` フィールドから GCP プロジェクト ID を見つけてください。
{{< /hint >}}

次のコマンドで 
{{< hover label="providerconfig" line="2">}}ProviderConfig{{</ hover >}} を適用します: 

{{< editCode >}}
```yaml {label="providerconfig",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: $@<PROJECT_ID>$@
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-secret
      key: creds
EOF
```
{{< /editCode >}}

これにより、Kubernetes シークレットとして保存された GCP 認証情報が 
{{< hover label="providerconfig" line="10">}}secretRef{{</ hover>}} として添付されます。

{{< hover label="providerconfig" line="12">}}spec.credentials.secretRef.name{{< /hover >}} の値は、GCP 認証情報を含む Kubernetes シークレットの名前であり、 
{{< hover label="providerconfig" line="11">}}spec.credentials.secretRef.namespace{{< /hover >}} にあります。

## 管理リソースの作成
_管理リソース_ は、Crossplane が Kubernetes クラスターの外部で作成および管理するすべてのものです。この例では、Crossplane を使用して GCP ストレージバケットを作成します。  
ストレージバケットは _管理リソース_ です。

{{< hint "note" >}}
一意の名前を生成するには、`name` の代わりに 
{{<hover label="xr" line="5">}}generateName{{</hover >}} を使用してください。
{{< /hint >}}

次のコマンドでバケットを作成します:

```yaml {label="xr",copy-lines="all"}
cat <<EOF | kubectl create -f -
apiVersion: storage.gcp.upbound.io/v1beta1
kind: Bucket
metadata:
  generateName: crossplane-bucket-
  labels:
    docs.crossplane.io/example: provider-gcp
spec:
  forProvider:
    location: US
  providerConfigRef:
    name: default
EOF
```

{{< hover label="xr" line="2">}}apiVersion{{< /hover >}} と 
{{< hover label="xr" line="3">}}kind{{</hover >}} はプロバイダーの CRD から取得されます。

{{< hover label="xr" line="10">}}spec.forProvider.location{{< /hover >}} 
は、リソースをデプロイする際に使用する GCP リージョンを GCP に指示します。  
{{<hover label="xr" line="3">}}bucket{{</hover >}} の場合、 
リージョンは任意の 
[GCP マルチリージョンロケーション](https://cloud.google.com/storage/docs/locations#location-mr) であることができます。

`kubectl get bucket` を使用して、Crossplane がバケットを作成したことを確認します。

{{< hint type="tip" >}}
Crossplane は、値 `READY` と `SYNCED` が `True` のときにバケットを作成しました。  
これには最大で 5 分かかる場合があります。  
{{< /hint >}}

```shell {copy-lines="1"}
kubectl get bucket
NAME                      READY   SYNCED   EXTERNAL-NAME             AGE
crossplane-bucket-8b7gw   True    True     crossplane-bucket-8b7gw   2m2s
```

## 管理リソースの削除
Kubernetes クラスターをシャットダウンする前に、作成した GCP バケットを削除してください。

`kubectl delete bucket` を使用してバケットを削除します。

{{<hint "tip" >}}
名前ではなくラベルで削除するには、`--selector` フラグを使用してください。
{{</hint>}}

```shell {copy-lines="1"}
kubectl delete bucket --selector docs.crossplane.io/example=provider-gcp
bucket.storage.gcp.upbound.io "crossplane-bucket-8b7gw" deleted
```

## 次のステップ
* [**パート 2 に進む**]({{< ref "provider-gcp-part-2">}}) で Crossplane _コンポジットリソース_ と _クレーム_ を作成します。
* [Provider CRD リファレンス](https://marketplace.upbound.io/providers/upbound/provider-family-gcp/) で Crossplane が構成できる GCP リソースを探索します。
* [Crossplane Slack](https://slack.crossplane.io/) に参加して、Crossplane のユーザーや貢献者とつながりましょう。
