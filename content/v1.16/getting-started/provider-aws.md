---
title: AWS クイックスタート
weight: 100
---

Crossplane を AWS に接続して、Kubernetes からクラウドリソースを作成および管理します。
[Upbound AWS Provider](https://marketplace.upbound.io/providers/upbound/provider-family-aws) を使用します。

このガイドは2つの部分に分かれています:
* パート1では、Crossplane のインストール、プロバイダーの設定、AWS への認証、および Kubernetes クラスターから直接 AWS に _Managed Resource_ を作成する手順を説明します。これにより、Crossplane が AWS と通信できることを示します。
* [パート2]({{< ref "provider-aws-part-2" >}}) では、Crossplane を使用してカスタム API を構築およびアクセスする方法を示します。


## 前提条件
このクイックスタートには以下が必要です:
* 少なくとも 2 GB の RAM を持つ Kubernetes クラスター
* Kubernetes クラスターでポッドとシークレットを作成する権限
* [Helm](https://helm.sh/) バージョン v3.2.0 以降
* S3 ストレージバケットを作成する権限を持つ AWS アカウント
* AWS の [アクセスキー](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds)

{{<include file="/master/getting-started/install-crossplane-include.md" type="page" >}}

## AWS プロバイダーのインストール

Kubernetes 構成ファイルを使用して AWS S3 プロバイダーを Kubernetes クラスターにインストールします。

```yaml {label="provider",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1.1.0
EOF
```

Crossplane {{< hover label="provider" line="3" >}}Provider{{</hover>}} は、AWS S3 サービスを表す Kubernetes の _Custom Resource Definitions_ (CRDs) をインストールします。これらの CRD により、Kubernetes 内で直接 AWS リソースを作成できます。

`kubectl get providers` を使用してプロバイダーがインストールされたことを確認します。

```shell {copy-lines="1",label="getProvider"}
kubectl get providers
NAME                          INSTALLED   HEALTHY   PACKAGE                                               AGE
provider-aws-s3               True        True      xpkg.upbound.io/upbound/provider-aws-s3:1.1.0         97s
upbound-provider-family-aws   True        True      xpkg.upbound.io/upbound/provider-family-aws:1.1.0     88s
```

S3 プロバイダーは、2つ目のプロバイダーである
{{<hover label="getProvider" line="4">}}upbound-provider-family-aws{{</hover >}} をインストールします。
ファミリープロバイダーは、すべての AWS ファミリープロバイダーにわたる AWS への認証を管理します。

`kubectl get crds` を使用して新しい CRD を表示できます。
各 CRD は、Crossplane がプロビジョニングおよび管理できる一意の AWS サービスにマップされます。

{{< hint type="tip" >}}
サポートされているすべての CRD の詳細については、
[Upbound Marketplace](https://marketplace.upbound.io/providers/upbound/provider-aws-s3/v1.1.0) を参照してください。
{{< /hint >}}

## AWS用のKubernetesシークレットを作成する
プロバイダーはAWSリソースを作成および管理するために認証情報を必要とします。  
プロバイダーはKubernetesの_Secret_を使用して認証情報をプロバイダーに接続します。

AWSキー・ペアからKubernetesの_Secret_を生成し、それを使用するようにプロバイダーを設定します。

### AWSキー・ペアファイルを生成する
基本的なユーザー認証には、AWSアクセスキーのキー・ペアファイルを使用します。

{{< hint type="tip" >}}
[AWSのドキュメント](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds)には、AWSアクセスキーの生成方法が記載されています。
{{< /hint >}}

AWSアカウントの`aws_access_key_id`と`aws_secret_access_key`を含むテキストファイルを作成します。

{{< editCode >}}
```ini {copy-lines="all"}
[default]
aws_access_key_id = $@<aws_access_key>$@
aws_secret_access_key = $@<aws_secret_key>$@
```
{{< /editCode >}}

このテキストファイルを`aws-credentials.txt`として保存します。

{{< hint type="note" >}}
AWSプロバイダードキュメントの[認証](https://docs.upbound.io/providers/provider-aws/authentication/)セクションには、他の認証方法が記載されています。
{{< /hint >}}

### AWS認証情報を含むKubernetesシークレットを作成する
Kubernetesのジェネリックシークレットには名前と内容があります。  
{{< hover label="kube-create-secret" line="1">}}kubectl create secret{{</hover >}}を使用して、  
{{< hover label="kube-create-secret" line="2">}}aws-secret{{< /hover >}}という名前のシークレットオブジェクトを  
{{< hover label="kube-create-secret" line="3">}}crossplane-system{{</ hover >}}ネームスペースに生成します。

{{< hover label="kube-create-secret" line="4">}}--from-file={{</hover>}}引数を使用して、値を{{< hover label="kube-create-secret" line="4">}}aws-credentials.txt{{< /hover >}}ファイルの内容に設定します。

```shell {label="kube-create-secret",copy-lines="all"}
kubectl create secret \
generic aws-secret \
-n crossplane-system \
--from-file=creds=./aws-credentials.txt
```

`kubectl describe secret`を使用してシークレットを表示します。

{{< hint type="note" >}}
テキストファイルに余分な空白がある場合、サイズが大きくなることがあります。
{{< /hint >}}

```shell {copy-lines="1"}
kubectl describe secret aws-secret -n crossplane-system
Name:         aws-secret
Namespace:    crossplane-system
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
creds:  114 bytes
```

## ProviderConfig を作成する
{{< hover label="providerconfig" line="3">}}ProviderConfig{{</ hover >}} は AWS プロバイダーの設定をカスタマイズします。

この Kubernetes 構成ファイルを使用して {{< hover label="providerconfig" line="3">}}ProviderConfig{{</ hover >}} を適用します:
```yaml {label="providerconfig",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
EOF
```

これは、Kubernetes シークレットとして保存された AWS 資格情報を {{< hover label="providerconfig" line="9">}}secretRef{{</ hover>}} として添付します。

{{< hover label="providerconfig" line="11">}}spec.credentials.secretRef.name{{< /hover >}} の値は、{{< hover label="providerconfig" line="10">}}spec.credentials.secretRef.namespace{{< /hover >}} 内の AWS 資格情報を含む Kubernetes シークレットの名前です。

## 管理リソースを作成する
_管理リソース_ は、Crossplane が Kubernetes クラスターの外部で作成および管理するものです。

このガイドでは、Crossplane を使用して AWS S3 バケットを作成します。

S3 バケットは _管理リソース_ です。

{{< hint type="note" >}}
AWS S3 バケット名はグローバルに一意でなければなりません。一意の名前を生成するために、この例ではランダムハッシュを使用しています。任意の一意の名前が許容されます。
{{< /hint >}}

```yaml {label="xr"}
cat <<EOF | kubectl create -f -
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  generateName: crossplane-bucket-
spec:
  forProvider:
    region: us-east-2
  providerConfigRef:
    name: default
EOF
```

{{< hover label="xr" line="3">}}apiVersion{{< /hover >}} と {{< hover label="xr" line="4">}}kind{{</hover >}} はプロバイダーの CRD から取得されます。

{{< hover label="xr" line="6">}}metadata.name{{< /hover >}} の値は、AWS に作成された S3 バケットの名前です。
この例では、{{< hover label="xr" line="6">}}$bucket{{</hover >}} 変数に生成された名前 `crossplane-bucket-<hash>` を使用しています。

{{< hover label="xr" line="9">}}spec.forProvider.region{{< /hover >}} は、リソースをデプロイする際に使用する AWS リージョンを AWS に指示します。

リージョンは任意の [AWS リージョナルエンドポイント](https://docs.aws.amazon.com/general/latest/gr/rande.html#regional-endpoints) コードを使用できます。

`kubectl get buckets` を使用して、Crossplane がバケットを作成したことを確認します。

{{< hint type="tip" >}}
`READY` と `SYNCED` の値が `True` であるとき、Crossplane はバケットを作成しました。これには最大で 5 分かかる場合があります。
{{< /hint >}}

```shell {copy-lines="1"}
kubectl get buckets
NAME                      READY   SYNCED   EXTERNAL-NAME             AGE
crossplane-bucket-hhdzh   True    True     crossplane-bucket-hhdzh   5s
```

## 管理リソースの削除
Kubernetes クラスターをシャットダウンする前に、作成したばかりの S3 バケットを削除します。

`kubectl delete bucket <bucketname>` を使用してバケットを削除します。

```shell {copy-lines="1"}
kubectl delete bucket crossplane-bucket-hhdzh
bucket.s3.aws.upbound.io "crossplane-bucket-hhdzh" deleted
```

## 次のステップ
* [**パート 2 に進む**]({{< ref "provider-aws-part-2">}}) で、Crossplane を使用してカスタム API を作成および使用します。
* Crossplane が設定できる AWS リソースを [Provider CRD リファレンス](https://marketplace.upbound.io/providers/upbound/provider-family-aws/) で探索します。
* [Crossplane Slack](https://slack.crossplane.io/) に参加して、Crossplane のユーザーやコントリビューターとつながりましょう。
```
