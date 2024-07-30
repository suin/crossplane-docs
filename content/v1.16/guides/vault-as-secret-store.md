---
title: Vaultを外部シークレットストアとして使用する
weight: 230
---

このガイドでは、Crossplaneとそのプロバイダーを[Vault]を[外部シークレットストア]（`ESS`）として使用するために必要な手順を説明します。[ESSプラグインVault]を使用します。

{{<hint "warning" >}}
外部シークレットストアはアルファ機能です。

本番環境での使用は推奨されません。Crossplaneはデフォルトで外部シークレットストアを無効にしています。
{{< /hint >}}

Crossplaneは、プロバイダーの資格情報、管理リソースへの入力、接続詳細などの機密情報を使用します。

[Vault資格情報注入ガイド]({{<ref "vault-injection" >}})では、プロバイダーの資格情報にVaultとCrossplaneを使用する方法について詳しく説明しています。

Crossplaneは、管理リソースの入力にVaultを使用することをサポートしていません。
[Crossplane issue #2985](https://github.com/crossplane/crossplane/issues/2985)では、この機能のサポートを追跡しています。

Vaultを使用した接続詳細のサポートには、Crossplane外部シークレットストアが必要です。

## 前提条件
このガイドでは、[Helm](https://helm.sh)のバージョン3.11以降が必要です。

## Vaultのインストール

{{<hint "note" >}}
[Vaultのインストール](https://developer.hashicorp.com/vault/docs/platform/k8s/helm)に関する詳細な手順は、Vaultのドキュメントにあります。
{{< /hint >}}

### Vault Helmチャートの追加

`hashicorp`のHelmリポジトリを追加します。
```shell
helm repo add hashicorp https://helm.releases.hashicorp.com --force-update 
```

Helmを使用してVaultをインストールします。
```shell
helm -n vault-system upgrade --install vault hashicorp/vault --create-namespace
```

### Vaultのアンシール

Vaultが[シールされている](https://developer.hashicorp.com/vault/docs/concepts/seal)場合は、アンシールキーを使用してVaultをアンシールします。

Vaultキーを取得します。
```shell
kubectl -n vault-system exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json
VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
```

キーを使用してVaultをアンシールします。
```shell {copy-lines="1"}
kubectl -n vault-system exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.13.1
Build Date      2023-03-23T12:51:35Z
Storage Type    file
Cluster Name    vault-cluster-df884357
Cluster ID      b3145d26-2c1a-a7f2-a364-81753033c0d9
HA Enabled      false
```

## Vault Kubernetes認証の構成

VaultがKubernetesサービスアカウントに基づいてリクエストを認証できるように、[Kubernetes認証メソッド]を有効にします。

### Vaultのルートトークンを取得する

Vaultのルートトークンは、[Vaultのアンシール](#unseal-vault)時に作成されるJSONファイルの中にあります。

```shell
cat cluster-keys.json | jq -r ".root_token"
```

### Kubernetes認証を有効にする

Vaultポッドのシェルに接続します。

```shell {copy-lines="1"}
kubectl -n vault-system exec -it vault-0 -- /bin/sh
/ $
```

Vaultシェルから、_ルートトークン_を使用してVaultにログインします。
```shell {copy-lines="1"}
vault login # use the root token from above
Token (will be hidden):
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                hvs.TSN4SssfMBM0HAtwGrxgARgn
token_accessor       qodxHrINVlRXKyrGeeDkxnih
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

VaultでKubernetes認証メソッドを有効にします。
```shell {copy-lines="1"}
vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
```

Kubernetesと通信するようにVaultを設定し、Vaultシェルを終了します。

```shell {copy-lines="1-4"}
vault write auth/kubernetes/config \
        token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
        kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
        kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
Success! Data written to: auth/kubernetes/config
/ $ exit
```

## Crossplane統合のためのVaultの設定

Crossplaneは、情報を保存するためにVaultのキー・バリューシークレットエンジンに依存しており、VaultはCrossplaneサービスアカウントのための権限ポリシーを必要とします。

<!-- vale Crossplane.Spelling = NO -->
<!-- allow "kv" -->
### Vault kvシークレットエンジンを有効にする
<!-- vale Crossplane.Spelling = YES -->

[Vault KVシークレットエンジン]を有効にします。

{{< hint "important" >}}
Vaultには2つのバージョンの
[KVシークレットエンジン](https://developer.hashicorp.com/vault/docs/secrets/kv)があります。
この例ではバージョン2を使用します。
{{</hint >}}

```shell {copy-lines="1"}
kubectl -n vault-system exec -it vault-0 -- vault secrets enable -path=secret kv-v2
Success! Enabled the kv-v2 secrets engine at: secret/
```

### CrossplaneのためのVaultポリシーを作成する

CrossplaneがVaultからデータを読み書きできるようにするためのVaultポリシーを作成します。
```shell {copy-lines="1-8"}
kubectl -n vault-system exec -i vault-0 -- vault policy write crossplane - <<EOF
path "secret/data/*" {
    capabilities = ["create", "read", "update", "delete"]
}
path "secret/metadata/*" {
    capabilities = ["create", "read", "update", "delete"]
}
EOF
Success! Uploaded policy: crossplane
```

ポリシーをVaultに適用します。
```shell {copy-lines="1-5"}
kubectl -n vault-system exec -it vault-0 -- vault write auth/kubernetes/role/crossplane \
    bound_service_account_names="*" \
    bound_service_account_namespaces=crossplane-system \
    policies=crossplane \
    ttl=24h
Success! Data written to: auth/kubernetes/role/crossplane
```

## Crossplaneをインストールする

{{<hint "important" >}}
Crossplane v1.12ではプラグインサポートが導入されました。お使いのCrossplaneのバージョンがプラグインをサポートしていることを確認してください。
{{< /hint >}}

External Secrets Stores機能を有効にしてCrossplaneをインストールします。

```shell 
helm upgrade --install crossplane crossplane-stable/crossplane --namespace crossplane-system --create-namespace --set args='{--enable-external-secret-stores}'
```

## Crossplane Vaultプラグインのインストール

Crossplane Vaultプラグインは、デフォルトのCrossplaneインストールの一部ではありません。
このプラグインは、[Vault Agent Sidecar Injection]を使用してVaultシークレットストアをCrossplaneに接続するユニークなPodとしてインストールされます。

まず、VaultプラグインPodのアノテーションを設定します。

```yaml
cat > values.yaml <<EOF
podAnnotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/agent-inject-token: "true"
  vault.hashicorp.com/role: crossplane
  vault.hashicorp.com/agent-run-as-user: "65532"
EOF
```
次に、Crossplane ESSプラグインPodを`crossplane-system`ネームスペースにインストールし、Vaultアノテーションを適用します。

```shell
helm upgrade --install ess-plugin-vault oci://xpkg.upbound.io/crossplane-contrib/ess-plugin-vault --namespace crossplane-system -f values.yaml
```

## Crossplaneの設定

Vaultプラグインを使用するには、Vaultサービスに接続するための設定が必要です。
プラグインは、外部シークレットストアを有効にするためにプロバイダーも必要とします。

プラグインとプロバイダーが設定されたら、CrossplaneはVaultとの通信方法を説明するために2つの`StoreConfig`オブジェクトを必要とします。

### プロバイダーで外部シークレットストアを有効にする

{{<hint "note">}}
この例ではプロバイダーGCPを使用していますが、
{{<hover label="ControllerConfig" line="2">}}ControllerConfig{{</hover>}}はすべてのプロバイダーで同じです。
{{</hint >}}

外部シークレットストアを有効にするための`ControllerConfig`オブジェクトを作成します。

```yaml {label="ControllerConfig"}
echo "apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: vault-config
spec:
  args:
    - --enable-external-secret-stores" | kubectl apply -f -
```

プロバイダーをインストールし、ControllerConfigを適用します。
```yaml
echo "apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-gcp:v0.23.0-rc.0.19.ge9b75ee5
  controllerConfigRef:
    name: vault-config" | kubectl apply -f -
```

### CrossplaneプラグインをVaultに接続する
プラグインがVaultサービスに接続するための{{<hover label="VaultConfig" line="2">}}VaultConfig{{</hover>}}リソースを作成します：

```yaml {label="VaultConfig"}
echo "apiVersion: secrets.crossplane.io/v1alpha1
kind: VaultConfig
metadata:
  name: vault-internal
spec:
  server: http://vault.vault-system:8200
  mountPath: secret/
  version: v2
  auth:
    method: Token
    token:
      source: Filesystem
      fs:
        path: /vault/secrets/token" | kubectl apply -f -
```

### Crossplane StoreConfigの作成

{{<hover label="xp-storeconfig" line="2">}}StoreConfig{{</hover >}}オブジェクトを
{{<hover label="xp-storeconfig" line="1">}}secrets.crossplane.io{{</hover >}}グループから作成します。
CrossplaneはStoreConfigを使用してVaultプラグインサービスに接続します。

{{<hover label="xp-storeconfig" line="10">}}configRef{{</hover >}}は
StoreConfigを特定のVaultプラグイン設定に接続します。

```yaml {label="xp-storeconfig"}
echo "apiVersion: secrets.crossplane.io/v1alpha1
kind: StoreConfig
metadata:
  name: vault
spec:
  type: Plugin
  defaultScope: crossplane-system
  plugin:
    endpoint: ess-plugin-vault.crossplane-system:4040
    configRef:
      apiVersion: secrets.crossplane.io/v1alpha1
      kind: VaultConfig
      name: vault-internal" | kubectl apply -f -
```


### プロバイダー StoreConfig の作成
プロバイダーの API グループから {{<hover label="gcp-storeconfig" line="2">}}StoreConfig{{</hover >}} オブジェクトを作成します。
{{<hover label="gcp-storeconfig" line="1">}}gcp.crossplane.io{{</hover >}}。
プロバイダーは、この StoreConfig を使用して、Managed Resources のために Vault と通信します。

{{<hover label="gcp-storeconfig" line="10">}}configRef{{</hover >}} は
StoreConfig を特定の Vault プラグイン構成に接続します。

```yaml {label="gcp-storeconfig"}
echo "apiVersion: gcp.crossplane.io/v1alpha1
kind: StoreConfig
metadata:
  name: vault
spec:
  type: Plugin
  defaultScope: crossplane-system
  plugin:
    endpoint: ess-plugin-vault.crossplane-system:4040
    configRef:
      apiVersion: secrets.crossplane.io/v1alpha1
      kind: VaultConfig
      name: vault-internal" | kubectl apply -f -
```

## プロバイダーリソースの作成

Crossplane がプロバイダーをインストールし、プロバイダーが正常であることを確認します。

```shell {copy-lines="1"}
kubectl get providers
NAME           INSTALLED   HEALTHY   PACKAGE                                                                     AGE
provider-gcp   True        True      xpkg.upbound.io/crossplane-contrib/provider-gcp:v0.23.0-rc.0.19.ge9b75ee5   10m
```

### CompositeResourceDefinition の作成

カスタム API エンドポイントを定義するために `CompositeResourceDefinition` を作成します。

```yaml
echo "apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: compositeessinstances.ess.example.org
  annotations:
    feature: ess
spec:
  group: ess.example.org
  names:
    kind: CompositeESSInstance
    plural: compositeessinstances
  claimNames:
    kind: ESSInstance
    plural: essinstances
  connectionSecretKeys:
    - publicKey
    - publicKeyType
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              parameters:
                type: object
                properties:
                  serviceAccount:
                    type: string
                required:
                  - serviceAccount
            required:
              - parameters" | kubectl apply -f -
```

### Composition の作成
GCP 内にサービスアカウントとサービスアカウントキーを作成するために `Composition` を作成します。

サービスアカウントキーを作成すると、
{{<hover label="comp" line="39" >}}connectionDetails{{</hover>}} が生成され、
プロバイダーはそれを Vault に保存します。
{{<hover label="comp" line="31">}}publishConnectionDetailsTo{{</hover>}} の詳細を使用します。

```yaml {label="comp"}
echo "apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: essinstances.ess.example.org
  labels:
    feature: ess
spec:
  publishConnectionDetailsWithStoreConfigRef: 
    name: vault
  compositeTypeRef:
    apiVersion: ess.example.org/v1alpha1
    kind: CompositeESSInstance
  resources:
    - name: serviceaccount
      base:
        apiVersion: iam.gcp.crossplane.io/v1alpha1
        kind: ServiceAccount
        metadata:
          name: ess-test-sa
        spec:
          forProvider:
            displayName: a service account to test ess
    - name: serviceaccountkey
      base:
        apiVersion: iam.gcp.crossplane.io/v1alpha1
        kind: ServiceAccountKey
        spec:
          forProvider:
            serviceAccountSelector:
              matchControllerRef: true
          publishConnectionDetailsTo:
            name: ess-mr-conn
            metadata:
              labels:
                environment: development
                team: backend
            configRef:
              name: vault
      connectionDetails:
        - fromConnectionSecretKey: publicKey
        - fromConnectionSecretKey: publicKeyType" | kubectl apply -f -
```

### クレームの作成
次に、Crossplane に GCP リソースと関連するシークレットを作成させるために `Claim` を作成します。

Composition と同様に、Claim は
{{<hover label="claim" line="12">}}publishConnectionDetailsTo{{</hover>}} を使用して
Vault に接続し、シークレットを保存します。

```yaml {label="claim"}
echo "apiVersion: ess.example.org/v1alpha1
kind: ESSInstance
metadata:
  name: my-ess
  namespace: default
spec:
  parameters:
    serviceAccount: ess-test-sa
  compositionSelector:
    matchLabels:
      feature: ess
  publishConnectionDetailsTo:
    name: ess-claim-conn
    metadata:
      labels:
        environment: development
        team: backend
    configRef:
      name: vault" | kubectl apply -f -
```

## リソースの確認

すべてのリソースが `READY` で `SYNCED` であることを確認します：

```shell {copy-lines="1"}
kubectl get managed
NAME                                                      READY   SYNCED   DISPLAYNAME                     EMAIL                                                            DISABLED
serviceaccount.iam.gcp.crossplane.io/my-ess-zvmkz-vhklg   True    True     a service account to test ess   my-ess-zvmkz-vhklg@testingforbugbounty.iam.gserviceaccount.com

NAME                                                         READY   SYNCED   KEY_ID                                     CREATED_AT             EXPIRES_AT
serviceaccountkey.iam.gcp.crossplane.io/my-ess-zvmkz-bq8pz   True    True     5cda49b7c32393254b5abb121b4adc07e140502c   2022-03-23T10:54:50Z
```

クレームを表示
```shell {copy-lines="1"}
kubectl -n default get claim
NAME     READY   CONNECTION-SECRET   AGE
my-ess   True                        19s
```


複合リソースを表示します。
```shell {copy-lines="1"}
kubectl get composite
NAME           READY   COMPOSITION                    AGE
my-ess-zvmkz   True    essinstances.ess.example.org   32s
```

## Vault シークレットの確認

Vault 内を見て、管理リソースからのシークレットを表示します。

```shell {copy-lines="1",label="vault-key"}
kubectl -n vault-system exec -i vault-0 -- vault kv list /secret/default
Keys
----
ess-claim-conn
```

キー {{<hover label="vault-key" line="4">}}ess-claim-conn{{</hover>}}
は、Claim の
{{<hover label="claim" line="12">}}publishConnectionDetailsTo{{</hover>}}
設定の名前です。

"crossplane-system" Vault スコープ内の接続シークレットを確認します。
```shell {copy-lines="1",label="scope-key"}
kubectl -n vault-system exec -i vault-0 -- vault kv list /secret/crossplane-system
Keys
----
d2408335-eb88-4146-927b-8025f405da86
ess-mr-conn
```

キー
{{<hover label="scope-key"line="4">}}d2408335-eb88-4146-927b-8025f405da86{{</hover>}}
は以下から来ています。

<!-- ## どこから来るのか？ -->

そしてキー
{{<hover label="scope-key"line="5">}}ess-mr-conn{{</hover>}}
は Composition の
{{<hover label="comp" line="31">}}publishConnectionDetailsTo{{</hover>}}
設定から来ています。


Claim の接続シークレット `ess-claim-conn` の内容を確認して、管理リソースによって作成されたキーを見ます。
```shell {copy-lines="1"}
kubectl -n vault-system exec -i vault-0 -- vault kv get /secret/default/ess-claim-conn
======= Metadata =======
Key                Value
---                -----
created_time       2022-03-18T21:24:07.2085726Z
custom_metadata    map[environment:development secret.crossplane.io/ner-uid:881cd9a0-6cc6-418f-8e1d-b36062c1e108 team:backend]
deletion_time      n/a
destroyed          false
version            1

======== Data ========
Key              Value
---              -----
publicKey        -----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzsEYCokmYEsZJCc9QN/8
Fm1M/kTPp7Gat/MXLTP3zFyCTBFVNLN79MbAKdinWi6ePXEb75vzB79IdZcWj8lo
8trnS64QjNB9Vs4Xk5UvDALwleFN/bZeperxivDPwVPvT9Aqy/U9kohoS/LHyE8w
uWQb5AuMeVQ1gtCTnCqQZ4d2MSVhQXYVvAWax1spJ9LT7mHub5j95xDdYIcOV3VJ
l9CIo4VrWIT8THFN2NnjTrGq9+0TzXY0bV674bjJkfBC6v6yXs5HTetG+Uekq/xf
FCjrrDi1+2UR9Mu2WTuvl8qn50be+mbwdJO5wE32jewxdYrVVmj19+PkaEeAwGTc
vwIDAQAB
-----END PUBLIC KEY-----
publicKeyType    TYPE_RAW_PUBLIC_KEY
```

管理リソース接続シークレット `ess-mr-conn` の内容を確認します。公開キーは Claim で使用されているこの管理リソースの公開キーと同一です。
```shell {copy-lines="1"}
kubectl -n vault-system exec -i vault-0 -- vault kv get /secret/crossplane-system/ess-mr-conn
======= Metadata =======
Key                Value
---                -----
created_time       2022-03-18T21:21:07.9298076Z
custom_metadata    map[environment:development secret.crossplane.io/ner-uid:4cd973f8-76fc-45d6-ad45-0b27b5e9252a team:backend]
deletion_time      n/a
destroyed          false
version            2

========= Data =========
Key               Value
---               -----
privateKey        {
  "type": "service_account",
  "project_id": "REDACTED",
  "private_key_id": "REDACTED",
  "private_key": "-----BEGIN PRIVATE KEY-----\nREDACTED\n-----END PRIVATE KEY-----\n",
  "client_email": "ess-test-sa@REDACTED.iam.gserviceaccount.com",
  "client_id": "REDACTED",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/ess-test-sa%40REDACTED.iam.gserviceaccount.com"
}
privateKeyType    TYPE_GOOGLE_CREDENTIALS_FILE
publicKey         -----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAzsEYCokmYEsZJCc9QN/8
Fm1M/kTPp7Gat/MXLTP3zFyCTBFVNLN79MbAKdinWi6ePXEb75vzB79IdZcWj8lo
8trnS64QjNB9Vs4Xk5UvDALwleFN/bZeperxivDPwVPvT9Aqy/U9kohoS/LHyE8w
uWQb5AuMeVQ1gtCTnCqQZ4d2MSVhQXYVvAWax1spJ9LT7mHub5j95xDdYIcOV3VJ
l9CIo4VrWIT8THFN2NnjTrGq9+0TzXY0bV674bjJkfBC6v6yXs5HTetG+Uekq/xf
FCjrrDi1+2UR9Mu2WTuvl8qn50be+mbwdJO5wE32jewxdYrVVmj19+PkaEeAwGTc
vwIDAQAB
-----END PUBLIC KEY-----
publicKeyType     TYPE_RAW_PUBLIC_KEY
```

### リソースの削除

Claim を削除すると、管理リソースと Vault からの関連キーが削除されます。

```shell
kubectl delete claim my-ess
```

<!-- 名前付きリンク -->

[Vault]: https://www.vaultproject.io/
[External Secret Store]: https://github.com/crossplane/crossplane/blob/master/design/design-doc-external-secret-stores.md
[this issue]: https://github.com/crossplane/crossplane/issues/2985
[Kubernetes Auth Method]: https://www.vaultproject.io/docs/auth/kubernetes
[Unseal]: https://www.vaultproject.io/docs/concepts/seal
[Vault KV Secrets Engine]: https://developer.hashicorp.com/vault/docs/secrets/kv
[Vault Agent Sidecar Injection]: https://www.vaultproject.io/docs/platform/k8s/injector
[ESS Plugin Vault]: https://github.com/crossplane-contrib/ess-plugin-vault
