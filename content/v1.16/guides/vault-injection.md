---
title: Vault 認証情報の注入
weight: 230
---


> このガイドは、[Minikube 上の Vault] および [Vault Kubernetes サイドカー] ガイドから適応されています。

ほとんどの Crossplane プロバイダーは、少なくとも以下のソースから認証情報を提供することをサポートしています：
- Kubernetes Secret
- 環境変数
- ファイルシステム

プロバイダーは追加の認証情報ソースをオプションでサポートすることがありますが、一般的なソースはさまざまなユースケースをカバーしています。[Vault] を秘密管理に使用する組織の間で人気のある特定のユースケースは、サイドカーを使用してファイルシステムに認証情報を注入することです。このガイドでは、[Vault Kubernetes サイドカー] を使用して [provider-gcp] および [provider-aws] のために認証情報を提供する方法を示します。

> 注：このガイドでは、GCP 認証情報と AWS アクセスキーを Vault の KV シークレットエンジンにコピーします。これは Vault を使用したシークレット管理のシンプルで一般的なアプローチですが、[AWS]、[Azure]、および [GCP] のための Vault の専用クラウドプロバイダーシークレットエンジンを使用するほど堅牢ではありません。

## セットアップ

> 注：このガイドでは、Crossplane と同じクラスターで実行されている Vault のセットアップ手順を説明します。クラスター外で実行されている既存の Vault インスタンスを使用することもできますが、その場合は Kubernetes 認証が有効になっている必要があります。

始める前に、Crossplane と Vault がインストールされており、クラスター内で実行されていることを確認する必要があります。

1. Crossplane をインストール

```console
kubectl create namespace crossplane-system

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

2. Vault Helm チャートをインストール

```console
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault
```

3. Vault インスタンスのアンシール

Vault が物理ストレージから暗号化データにアクセスするためには、[アンシール] されている必要があります。

```console
kubectl exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json
VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
```

4. Kubernetes 認証メソッドを有効にする

Vault が Kubernetes サービスアカウントに基づいてリクエストを認証できるようにするためには、[Kubernetes 認証バックエンド] を有効にする必要があります。これには、Vault にログインし、サービスアカウントトークン、API サーバーアドレス、および証明書で構成する必要があります。Vault を Kubernetes で実行しているため、これらの値はすでにコンテナのファイルシステムと環境変数を介して利用可能です。

```console
cat cluster-keys.json | jq -r ".root_token" # get root token

kubectl exec -it vault-0 -- /bin/sh
vault login # use root token from above
vault auth enable kubernetes

vault write auth/kubernetes/config \
        token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
        kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
        kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

5. Vaultコンテナから退出

次のステップは、あなたのローカル環境で実行されます。

```console
exit
```

{{< tabs >}}
{{< tab "GCP" >}}

## GCPサービスアカウントの作成

GCP上でインフラをプロビジョニングするためには、適切な権限を持つサービスアカウントを作成する必要があります。このガイドでは、CloudSQLインスタンスのみをプロビジョニングするため、サービスアカウントは`cloudsql.admin`ロールにバインドされます。以下のステップでは、GCPサービスアカウントを設定し、CrossplaneがCloudSQLインスタンスを管理できるように必要な権限を付与し、サービスアカウントの資格情報をJSONファイルに出力します。

```console
# replace this with your own gcp project id and the name of the service account
# that will be created.
PROJECT_ID=my-project
NEW_SA_NAME=test-service-account-name

# create service account
SA="${NEW_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
gcloud iam service-accounts create $NEW_SA_NAME --project $PROJECT_ID

# enable cloud API
SERVICE="sqladmin.googleapis.com"
gcloud services enable $SERVICE --project $PROJECT_ID

# grant access to cloud API
ROLE="roles/cloudsql.admin"
gcloud projects add-iam-policy-binding --role="$ROLE" $PROJECT_ID --member "serviceAccount:$SA"

# create service account keyfile
gcloud iam service-accounts keys create creds.json --project $PROJECT_ID --iam-account $SA
```

現在、`creds.json`に有効なサービスアカウントの資格情報があるはずです。

## Vaultに資格情報を保存

Vaultを設定した後、[kvシークレットエンジン]に資格情報を保存する必要があります。

> 注: 以下のステップでは、Vaultに保存する前に資格情報をコンテナのファイルシステムにコピーすることが含まれています。また、コンテナをローカル環境にポートフォワードして、VaultのHTTP APIまたはUIを使用することもできます
> （`kubectl port-forward vault-0 8200:8200`）。

1. 資格情報ファイルをVaultコンテナにコピー

資格情報をコンテナのファイルシステムにコピーして、Vaultに保存できるようにします。

```console
kubectl cp creds.json vault-0:/tmp/creds.json
```

2. KVシークレットエンジンを有効にする

シークレットエンジンは使用する前に有効にする必要があります。`secret`パスで`kv-v2`シークレットエンジンを有効にします。

```console
kubectl exec -it vault-0 -- /bin/sh

vault secrets enable -path=secret kv-v2
```

3. KVエンジンにGCP資格情報を保存

GCP資格情報のパスは、`provider-gcp`コントローラーの`Pod`に注入する際にシークレットが参照される方法です。

```console
vault kv put secret/provider-creds/gcp-default @tmp/creds.json
```

4. 資格情報ファイルをクリーンアップ

コンテナのファイルシステムにGCP資格情報ファイルはもう必要ないので、クリーンアップしてください。

```console
rm tmp/creds.json
```

{{< /tab >}}
{{< tab "AWS" >}}

## AWS IAMユーザーの作成

AWS上でインフラをプロビジョニングするためには、既存のIAMユーザーを使用するか、新しいIAMユーザーを適切な権限で作成する必要があります。以下の手順でAWS IAMユーザーを作成し、必要な権限を付与します。

> 注: 既存の適切な権限を持つIAMユーザーがいる場合は、このステップをスキップできますが、`ACCESS_KEY_ID`および`AWS_SECRET_ACCESS_KEY`環境変数の値を提供する必要があります。

```console
# create a new IAM user
IAM_USER=test-user
aws iam create-user --user-name $IAM_USER

# grant the IAM user the necessary permissions
aws iam attach-user-policy --user-name $IAM_USER --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# create a new IAM access key for the user
aws iam create-access-key --user-name $IAM_USER > creds.json
# assign the access key values to environment variables
ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId creds.json)
AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey creds.json)
```

## Vaultに資格情報を保存

Vaultを設定した後、[kvシークレットエンジン]に資格情報を保存する必要があります。

1. KVシークレットエンジンを有効にする

シークレットエンジンは使用する前に有効にする必要があります。`secret`パスで`kv-v2`シークレットエンジンを有効にします。

```console
kubectl exec -it vault-0 -- env \
  ACCESS_KEY_ID=${ACCESS_KEY_ID} \
  AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY} \
  /bin/sh

vault secrets enable -path=secret kv-v2
```

2. KVエンジンにAWS資格情報を保存

AWS資格情報のパスは、`provider-aws`コントローラーの`Pod`に注入する際にシークレットが参照される方法です。

```
vault kv put secret/provider-creds/aws-default access_key="$ACCESS_KEY_ID" secret_key="$AWS_SECRET_ACCESS_KEY"
```

{{< /tab >}}
{{< /tabs >}}

## プロバイダー資格情報を読み取るためのVaultポリシーを作成

コントローラーがVaultサイドカーに資格情報をファイルシステムに注入させるためには、`Pod`を[ポリシー]に関連付ける必要があります。このポリシーは、`kv-v2`シークレットエンジンの`provider-creds`パスにあるすべてのシークレットを読み取り、リストすることを許可します。

```console
vault policy write provider-creds - <<EOF
path "secret/data/provider-creds/*" {
    capabilities = ["read", "list"]
}
EOF
```

## CrossplaneプロバイダーPodのためのロールを作成

1. ロールを作成

最後のステップは、作成したポリシーにバインドされたロールを作成し、Kubernetesサービスアカウントのグループに関連付けることです。このロールは、`crossplane-system`ネームスペース内の任意の（`*`）サービスアカウントによって引き受けることができます。

```console
vault write auth/kubernetes/role/crossplane-providers \
        bound_service_account_names="*" \
        bound_service_account_namespaces=crossplane-system \
        policies=provider-creds \
        ttl=24h
```

2. Vaultコンテナから退出

次のステップは、ローカル環境で実行されます。

```console
exit
```

{{< tabs >}}
{{< tab "GCP" >}}

## provider-gcpのインストール

これで`provider-gcp`をインストールする準備が整いました。Crossplaneは、プロバイダーのコントローラー`Pod`のデプロイをカスタマイズするための`ControllerConfig`タイプを提供します。`ControllerConfig`は、その設定を使用したい任意の数の`Provider`オブジェクトによって作成され、参照されることができます。以下の例では、`Pod`のアノテーションは、`secret/provider-creds/gcp-default`に保存されたシークレットを`crossplane-providers`ロールを引き受けることによってコンテナのファイルシステムに注入することをVaultのミューテイティングWebhookに示しています。また、シークレットデータが`provider-gcp`が期待する形式で提示されるように、テンプレートフォーマットも追加されています。

```console
echo "apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: vault-config
spec:
  metadata:
    annotations:
      vault.hashicorp.com/agent-inject: \"true\"
      vault.hashicorp.com/role: "crossplane-providers"
      vault.hashicorp.com/agent-inject-secret-creds.txt: "secret/provider-creds/gcp-default"
      vault.hashicorp.com/agent-inject-template-creds.txt: |
        {{- with secret \"secret/provider-creds/gcp-default\" -}}
         {{ .Data.data | toJSON }}
        {{- end -}}
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-gcp:v0.22.0
  controllerConfigRef:
    name: vault-config" | kubectl apply -f -
```

## プロバイダー-gcpの設定

`provider-gcp`がインストールされて実行中の場合、ファイルシステム内の資格情報を指定する`ProviderConfig`を作成する必要があります。この`ProviderConfig`を参照する管理リソースをプロビジョニングするために使用されます。この`ProviderConfig`の名前は`default`であるため、明示的に`ProviderConfig`を参照しない管理リソースによって使用されます。

> 注: 以前に定義された`PROJECT_ID`環境変数が正しく設定されていることを確認してください。

```console
echo "apiVersion: gcp.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: ${PROJECT_ID}
  credentials:
    source: Filesystem
    fs:
      path: /vault/secrets/creds.txt" | kubectl apply -f -
```

GCPの資格情報がコンテナに注入されていることを確認するには、次のコマンドを実行します。

```console
PROVIDER_CONTROLLER_POD=$(kubectl -n crossplane-system get pod -l pkg.crossplane.io/provider=provider-gcp -o name --no-headers=true)
kubectl -n crossplane-system exec -it $PROVIDER_CONTROLLER_POD -c provider-gcp -- cat /vault/secrets/creds.txt
```

## インフラストラクチャのプロビジョニング

最終ステップは、実際に`CloudSQLInstance`をプロビジョニングすることです。以下のオブジェクトを作成すると、GCP上にCloud SQL Postgresデータベースが作成されます。

```console
echo "apiVersion: database.gcp.crossplane.io/v1beta1
kind: CloudSQLInstance
metadata:
  name: postgres-vault-demo
spec:
  forProvider:
    databaseVersion: POSTGRES_12
    region: us-central1
    settings:
      tier: db-custom-1-3840
      dataDiskType: PD_SSD
      dataDiskSizeGb: 10
  writeConnectionSecretToRef:
    namespace: crossplane-system
    name: cloudsqlpostgresql-conn" | kubectl apply -f -
```

データベースのプロビジョニングの進行状況を監視するには、次のコマンドを使用します。

```console
kubectl get cloudsqlinstance -w
```

{{< /tab >}}
{{< tab "AWS" >}}

## プロバイダー-awsのインストール

`provider-aws`をインストールする準備が整いました。Crossplaneは、プロバイダーのコントローラー`Pod`のデプロイをカスタマイズするための`ControllerConfig`タイプを提供します。`ControllerConfig`は、その設定を使用したい任意の数の`Provider`オブジェクトによって作成および参照できます。以下の例では、`Pod`のアノテーションは、`secret/provider-creds/aws-default`に保存されたシークレットを`crossplane-providers`ロールを引き受けることによってコンテナのファイルシステムに注入することをVaultのミューテイティングWebhookに示しています。また、シークレットデータが`provider-aws`が期待する形式で表示されるように、いくつかのテンプレートフォーマットが追加されています。

{% raw  %}
```console
echo "apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: aws-vault-config
spec:
  args:
    - --debug
  metadata:
    annotations:
      vault.hashicorp.com/agent-inject: \"true\"
      vault.hashicorp.com/role: \"crossplane-providers\"
      vault.hashicorp.com/agent-inject-secret-creds.txt: \"secret/provider-creds/aws-default\"
      vault.hashicorp.com/agent-inject-template-creds.txt: |
        {{- with secret \"secret/provider-creds/aws-default\" -}}
          [default]
          aws_access_key_id="{{ .Data.data.access_key }}"
          aws_secret_access_key="{{ .Data.data.secret_key }}"
        {{- end -}}
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v0.33.0
  controllerConfigRef:
    name: aws-vault-config" | kubectl apply -f -
```
{% endraw %}

## provider-awsの設定

`provider-aws`がインストールされ、実行されている状態になったら、ファイルシステム内の資格情報を指定する`ProviderConfig`を作成する必要があります。この`ProviderConfig`を参照する管理リソースをプロビジョニングするために使用されます。この`ProviderConfig`の名前は`default`であるため、明示的に`ProviderConfig`を参照しない管理リソースによって使用されます。

```console
echo "apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Filesystem
    fs:
      path: /vault/secrets/creds.txt" | kubectl apply -f -
```

AWSの資格情報がコンテナに注入されていることを確認するには、次のコマンドを実行します。

```console
PROVIDER_CONTROLLER_POD=$(kubectl -n crossplane-system get pod -l pkg.crossplane.io/provider=provider-aws -o name --no-headers=true)
kubectl -n crossplane-system exec -it $PROVIDER_CONTROLLER_POD -c provider-aws -- cat /vault/secrets/creds.txt
```

## インフラストラクチャのプロビジョニング

最後のステップは、実際に`Bucket`をプロビジョニングすることです。以下のオブジェクトを作成すると、AWS上にS3バケットが作成されます。

```console
echo "apiVersion: s3.aws.crossplane.io/v1beta1
kind: Bucket
metadata:
  name: s3-vault-demo
spec:
  forProvider:
    acl: private
    locationConstraint: us-east-1
    publicAccessBlockConfiguration:
      blockPublicPolicy: true
    tagging:
      tagSet:
        - key: Name
          value: s3-vault-demo
  providerConfigRef:
    name: default" | kubectl apply -f -
```

バケットのプロビジョニングの進行状況を監視するには、次のコマンドを使用します。

```console
kubectl get bucket -w
```

{{< /tab >}}
{{< /tabs >}}

<!-- named links -->

[Vault on Minikube]: https://learn.hashicorp.com/tutorials/vault/kubernetes-minikube
[Vault Kubernetes Sidecar]: https://learn.hashicorp.com/tutorials/vault/kubernetes-sidecar
[Vault]: https://www.vaultproject.io/
[Vault Kubernetes Sidecar]: https://www.vaultproject.io/docs/platform/k8s/injector
[provider-gcp]: https://marketplace.upbound.io/providers/crossplane-contrib/provider-gcp
[provider-aws]: https://marketplace.upbound.io/providers/crossplane-contrib/provider-aws
[AWS]: https://www.vaultproject.io/docs/secrets/aws
[Azure]: https://www.vaultproject.io/docs/secrets/azure
[GCP]: https://www.vaultproject.io/docs/secrets/gcp 
[unsealed]: https://www.vaultproject.io/docs/concepts/seal
[Kubernetes authentication backend]: https://www.vaultproject.io/docs/auth/kubernetes
[kv secrets engine]: https://www.vaultproject.io/docs/secrets/kv/kv-v2
[policy]: https://www.vaultproject.io/docs/concepts/policies

It seems that there is no content provided for translation. Please paste the Markdown content you'd like me to translate into Japanese.
