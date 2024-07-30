---
title: 接続の詳細
weight: 110
description: "Crossplane 管理リソース、複合リソース、コンポジション、およびクレーム全体で接続の詳細を作成および管理する方法"
---

Crossplane で接続の詳細を使用するには、以下のコンポーネントが必要です：
* [クレーム]({{<ref "/master/concepts/claims#claim-connection-secrets">}})で `writeConnectionSecretToRef.name` を定義します。
* [コンポジション]({{<ref "/master/concepts/compositions#composite-resource-combined-secret">}})で `writeConnectionSecretsToNamespace` の値を定義します。
* 各リソースの `writeConnectionSecretToRef` 名と名前空間を
  [コンポジション]({{<ref "/master/concepts/compositions#composed-resource-secrets">}})で定義します。
* 各構成リソースによって生成される秘密鍵のリストを `connectionDetails` で
  [コンポジション]({{<ref "/master/concepts/compositions#define-secret-keys">}})で定義します。
* 必要に応じて、[CompositeResourceDefinition]({{<ref "/master/concepts/composite-resource-definitions#manage-connection-secrets">}})で `connectionSecretKeys` を定義します。

{{<hint "note">}}
このガイドでは Kubernetes シークレットの作成について説明します。  
Crossplane は [HashiCorp Vault](https://www.vaultproject.io/) のような外部シークレットストアの使用もサポートしています。

Crossplane を外部シークレットストアと一緒に使用する方法についての詳細は、[外部シークレットストアガイド]({{<ref "../guides/vault-as-secret-store">}})をお読みください。
{{</hint >}}

## 背景
[プロバイダー]({{<ref "/master/concepts/providers">}})が管理リソースを作成すると、そのリソースはリソース固有の詳細を生成する場合があります。これらの詳細には、ユーザー名、パスワード、または IP アドレスのような接続の詳細が含まれることがあります。

Crossplane はこの情報を _接続の詳細_ または _接続シークレット_ と呼びます。

プロバイダーは、管理リソースから _接続の詳細_ として表示する情報を定義します。

<!-- vale gitlab.SentenceLength = NO -->
<!-- wordy because of type names -->
管理リソースが [コンポジション]({{<ref "/master/concepts/compositions">}}) の一部である場合、コンポジション、[複合リソース定義]({{<ref "/master/concepts/composite-resource-definitions">}}) および必要に応じて [クレーム]({{<ref "/master/concepts/claims">}}) が、どの詳細が表示され、どこに保存されるかを定義します。
<!-- vale gitlab.SentenceLength = YES -->

```markdown
{{<hint "note">}}
以下のすべての例は、同じセットのコンポジション、コンポジットリソース定義、およびクレームを使用しています。

すべての例は、リソースを作成するために
[Upbound provider-aws-iam](https://marketplace.upbound.io/providers/upbound/provider-aws-iam/)に依存しています。

{{<expand "Reference Composition" >}}
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xsecrettest.example.org
spec:
  writeConnectionSecretsToNamespace: other-namespace
  compositeTypeRef:
    apiVersion: example.org/v1alpha1
    kind: XSecretTest
  resources:
    - name: key
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
          forProvider:
            userSelector:
              matchControllerRef: true
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
      connectionDetails:
        - fromConnectionSecretKey: username
        - fromConnectionSecretKey: password
        - fromConnectionSecretKey: attribute.secret
        - fromConnectionSecretKey: attribute.ses_smtp_password_v4
      patches:
        - fromFieldPath: "metadata.uid"
          toFieldPath: "spec.writeConnectionSecretToRef.name"
          transforms:
            - type: string
              string:
                fmt: "%s-secret1"
    - name: user
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: User
        spec:
          forProvider: {}
    - name: user2
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: User
        metadata:
          labels:
            docs.crossplane.io: user
        spec:
          forProvider: {}
    - name: key2
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
          forProvider:
            userSelector:
              matchLabels:
                docs.crossplane.io: user
          writeConnectionSecretToRef:
            namespace: docs
            name: key2
      connectionDetails:
        - name: key2-user
          fromConnectionSecretKey: username
        - name: key2-password
          fromConnectionSecretKey: password
        - name: key2-secret
          fromConnectionSecretKey: attribute.secret
        - name: key2-smtp
          fromConnectionSecretKey: attribute.ses_smtp_password_v4
      patches:
        - fromFieldPath: "metadata.uid"
          toFieldPath: "spec.writeConnectionSecretToRef.name"
          transforms:
            - type: string
              string:
                fmt: "%s-secret2"
```
{{</expand >}}

{{<expand "Reference CompositeResourceDefinition" >}}

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xsecrettests.example.org
spec:
  group: example.org
  connectionSecretKeys:
    - username
    - password
    - attribute.secret
    - attribute.ses_smtp_password_v4
    - key2-user
    - key2-pass
    - key2-secret
    - key2-smtp
  names:
    kind: XSecretTest
    plural: xsecrettests
  claimNames:
    kind: SecretTest
    plural: secrettests
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
```
{{</ expand >}}

{{<expand "Reference Claim" >}}
```yaml
apiVersion: example.org/v1alpha1
kind: SecretTest
metadata:
  name: test-secrets
  namespace: default
spec:
  writeConnectionSecretToRef:
    name: my-access-key-secret
```
{{</expand >}}
{{</hint >}}

## 管理されたリソースにおける接続シークレット

<!-- vale gitlab.Substitutions = NO -->
<!-- vale gitlab.SentenceLength = NO -->
<!-- 25語未満 -->
管理されたリソースが接続シークレットを作成すると、Crossplaneはシークレットを
[Kubernetesシークレット]({{<ref "/master/concepts/managed-resources#publish-secrets-to-kubernetes">}})
または
[外部シークレットストア]({{<ref "/master/concepts/managed-resources#publish-secrets-to-an-external-secrets-store">}})に書き込むことができます。
<!-- vale gitlab.SentenceLength = YES -->
<!-- vale gitlab.Substitutions = YES -->

個々の管理リソースを作成すると、そのリソースが作成する接続シークレットが表示されます。

{{<hint "note" >}}
[管理リソース]({{<ref "/master/concepts/managed-resources">}})
のドキュメントを読んで、リソースの構成や個々のリソースの接続シークレットの保存に関する詳細情報を確認してください。
{{< /hint >}}

例えば、{{<hover label="mr" line="2">}}AccessKey{{</hover>}}リソースを作成し、接続シークレットを
{{<hover label="mr" line="12">}}my-accesskey-secret{{</hover>}}という名前のKubernetesシークレットに保存します。
これは、{{<hover label="mr" line="11">}}default{{</hover>}}名前空間にあります。

```yaml {label="mr"}
apiVersion: iam.aws.upbound.io/v1beta1
kind: AccessKey
metadata:
    name: test-accesskey
spec:
    forProvider:
        userSelector:
            matchLabels:
                docs.crossplane.io: user
    writeConnectionSecretToRef:
        namespace: default
        name: my-accesskey-secret
```

Kubernetesシークレットを表示して、管理リソースからの接続詳細を確認します。  
これには、{{<hover label="mrSecret" line="11">}}attribute.secret{{</hover>}}、
{{<hover label="mrSecret" line="12">}}attribute.ses_smtp_password_v4{{</hover>}}、
{{<hover label="mrSecret" line="13">}}password{{</hover>}}および
{{<hover label="mrSecret" line="14">}}username{{</hover>}}が含まれます。
```

```yaml {label="mrSecret",copy-lines="1"}
kubectl describe secret my-accesskey-secret
Name:         my-accesskey-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  connection.crossplane.io/v1alpha1

Data
====
attribute.secret:                40 bytes
attribute.ses_smtp_password_v4:  44 bytes
password:                        40 bytes
username:                        20 bytes
```

Composition と CompositeResourceDefinitions は、リソースによって生成されたシークレットの正確な名前を必要とします。

## Composition における接続シークレット

接続詳細を作成する Composition 内のリソースは、接続詳細を含むシークレットオブジェクトを作成します。  
Crossplane はまた、すべての定義されたリソースからのシークレットを含む、各コンポジットリソースのための別のシークレットオブジェクトを生成します。

例えば、Composition は二つの 
{{<hover label="comp1" line="9">}}AccessKey{{</hover>}}
オブジェクトを定義します。  
各 {{<hover label="comp1" line="9">}}AccessKey{{</hover>}} は、リソースによって定義された 
{{<hover label="comp1" line="14">}}namespace{{</hover>}} 内の 
{{<hover label="comp1" line="15">}}name{{</hover>}} に接続シークレットを書き込みます。 
リソース 
{{<hover label="comp1" line="13">}}writeConnectionSecretToRef{{</hover>}} によって。

Crossplane はまた、 
{{<hover label="comp1" line="4">}}writeConnectionSecretsToNamespace{{</hover>}} によって定義された名前空間に保存された、全体の Composition のためのシークレットオブジェクトを作成します。 

```yaml {label="comp1",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  writeConnectionSecretsToNamespace: other-namespace
  resources:
    - name: key1
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
          forProvider:
            # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1-secret
    - name: key2
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
          forProvider:
            # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key2-secret
    # Removed for brevity
```

Claim を適用した後、Kubernetes シークレットを表示して、作成された三つのシークレットオブジェクトを確認します。

シークレット 
{{<hover label="compGetSec" line="3">}}key1-secret{{</hover>}} はリソース 
{{<hover label="comp1" line="6">}}key1{{</hover>}} から、 
{{<hover label="compGetSec" line="4">}}key2-secret{{</hover>}} はリソース 
{{<hover label="comp1" line="16">}}key2{{</hover>}} からです。

Crossplane は、Composition 内のリソースからのシークレットを持つ 
{{<hover label="compGetSec" line="5">}}other-namespace{{</hover>}} に別のシークレットを作成します。 

```shell {label="compGetSec",copy-lines="1"}
kubectl get secrets -A
NAMESPACE           NAME                                   TYPE                                DATA   AGE
docs                key1-secret                            connection.crossplane.io/v1alpha1   4      4s
docs                key2-secret                            connection.crossplane.io/v1alpha1   4      4s
other-namespace     70975471-c44f-4f6d-bde6-6bbdc9de1eb8   connection.crossplane.io/v1alpha1   0      6s
```

Crossplane はシークレットオブジェクトを作成しますが、デフォルトでは、Crossplane はオブジェクトにデータを追加しません。 

```yaml {copy-lines="none"}
kubectl describe secret 70975471-c44f-4f6d-bde6-6bbdc9de1eb8 -n other-namespace
Name:         70975471-c44f-4f6d-bde6-6bbdc9de1eb8
Namespace:    other-namespace

Type:  connection.crossplane.io/v1alpha1

Data
====
```

Composition は、各リソースのために保存する接続シークレットをリストする必要があります。  
各リソースの下にある 
{{<hover label="comp2" line="16">}}connectionDetails{{</hover>}} オブジェクトを使用して、リソースが生成するシークレットキーを定義します。  

{{<hint "warning">}}
Compositionの
{{<hover label="comp2" line="16">}}connectionDetails{{</hover>}} 
を変更することはできません。  
変更するにはCompositionを削除して
再作成する必要があります。
{{<hover label="comp2" line="16">}}connectionDetails{{</hover>}}を変更するには、  
{{</hint >}}

```yaml {label="comp2",copy-lines="16-20"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  writeConnectionSecretsToNamespace: other-namespace
  resources:
    - name: key
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: AccessKey
        spec:
          forProvider:
            # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
      connectionDetails:
        - fromConnectionSecretKey: username
        - fromConnectionSecretKey: password
        - fromConnectionSecretKey: attribute.secret
        - fromConnectionSecretKey: attribute.ses_smtp_password_v4
    # Removed for brevity
```

Claimを適用した後、合成リソースのシークレットオブジェクトには
{{<hover label="comp2" line="16">}}connectionDetails{{</hover>}}にリストされた
キーのリストが含まれます。

```shell {copy-lines="1"}
kubectl describe secret -n other-namespace
Name:         b0dc71f8-2688-4ebc-818a-bbad6a2c4f9a
Namespace:    other-namespace

Type:  connection.crossplane.io/v1alpha1

Data
====
username:                        20 bytes
attribute.secret:                40 bytes
attribute.ses_smtp_password_v4:  44 bytes
password:                        40 bytes
```

{{<hint "important">}}
キーが
{{<hover label="comp2" line="16">}}connectionDetails{{</hover>}}にリストされていない場合、
シークレットオブジェクトには保存されません。
{{< /hint >}}

### 競合するシークレットキーの管理 
リソースが競合するキーを生成する場合、接続の詳細を持つ
ユニークな名前を作成します
{{<hover label="comp3" line="25">}}name{{</hover>}}。

```yaml {label="comp3",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  writeConnectionSecretsToNamespace: other-namespace
  resources:
    - name: key
      base:
        kind: AccessKey
        spec:
          # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
      connectionDetails:
        - fromConnectionSecretKey: username
    - name: key2
      base:
        kind: AccessKey
        spec:
          # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key2
      connectionDetails:
        - name: key2-user
          fromConnectionSecretKey: username
```

シークレットオブジェクトには両方のキーが含まれます、
{{<hover label="comp3Sec" line="9">}}username{{</hover>}}
と
{{<hover label="comp3Sec" line="10">}}key2-user{{</hover>}}

```shell {label="comp3Sec",copy-lines="1"}
kubectl describe secret -n other-namespace
Name:         b0dc71f8-2688-4ebc-818a-bbad6a2c4f9a
Namespace:    other-namespace

Type:  connection.crossplane.io/v1alpha1

Data
====
username:                        20 bytes
key2-user:                       20 bytes
# Removed for brevity.
```

## 合成リソース定義における接続シークレット

合成リソース定義（`XRD`）は、どのシークレットキーが
結合されたシークレットに入れられ、Claimに提供されるかを制限できます。

デフォルトでは、XRDは合成リソースの
`connectionDetails`にリストされたすべてのシークレットキーを
結合されたシークレットオブジェクトに書き込みます。

{{<hover label="xrd" line="4">}}connectionSecretKeys{{</hover>}}オブジェクトを使用して、
結合されたシークレットオブジェクトとClaimに渡されるキーを制限します。

{{<hover label="xrd" line="4">}}connectionSecretKeys{{</hover>}}リスト内に
作成するシークレットキー名をリストします。Crossplaneは、リストされたキーのみを
結合されたシークレットに追加します。

{{<hint "warning">}}
XRDの
{{<hover label="xrd" line="4">}}connectionSecretKeys{{</hover>}}を変更することはできません。 
変更するにはXRDを削除して
再作成する必要があります。
{{</hint >}}

例えば、XRDは秘密を以下のみに制限することがあります。
{{<hover label="xrd" line="5">}}username{{</hover>}},
{{<hover label="xrd" line="6">}}password{{</hover>}} およびカスタム名の
{{<hover label="xrd" line="7">}}key2-user{{</hover>}} キー。

```yaml {label="xrd",copy-lines="4-12"}
kind: CompositeResourceDefinition
spec:
  # Removed for brevity.
  connectionSecretKeys:
    - username
    - password
    - key2-user
```

個々のリソースからの秘密は、Compositionの `connectionDetails` に詳細が記載されているすべてのリソースを含みます。

```shell {label="xrdSec",copy-lines="1"}
kubectl describe secret key1 -n docs
Name:         key1
Namespace:    docs

Data
====
password:                        40 bytes
username:                        20 bytes
attribute.secret:                40 bytes
attribute.ses_smtp_password_v4:  44 bytes
```

Claimの秘密は、XRDによって許可された
{{<hover label="xrd" line="4">}}connectionSecretKeys{{</hover>}} 
フィールドのキーのみを含みます。

```shell {label="xrdSec2",copy-lines="2"}
kubectl describe secret my-access-key-secret
Name:         my-access-key-secret

Data
====
key2-user:  20 bytes
password:   40 bytes
username:   20 bytes
```

## 秘密オブジェクト
Compositionは各リソースのために秘密オブジェクトを作成し、すべてのリソースからの秘密を含む追加の秘密を作成します。

Crossplaneはリソースの秘密オブジェクトをリソースの
{{<hover label="comp4" line="11">}}writeConnectionSecretToRef{{</hover>}} で定義された場所に保存します。

CrossplaneはCompositionの
{{<hover label="comp4" line="4">}}writeConnectionSecretsToNamespace{{</hover>}} で定義された名前空間に、Crossplane生成の名前で結合された秘密を保存します。

```yaml {label="comp4",copy-lines="none"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
spec:
  writeConnectionSecretsToNamespace: other-namespace
  resources:
    - name: key
      base:
        kind: AccessKey
        spec:
          # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key1
      connectionDetails:
        - fromConnectionSecretKey: username
    - name: key2
      base:
        kind: AccessKey
        spec:
          # Removed for brevity
          writeConnectionSecretToRef:
            namespace: docs
            name: key2
      connectionDetails:
        - name: key2-user
          fromConnectionSecretKey: username
```

Claimが秘密を使用する場合、それはClaimの
{{<hover label="claim3" line="7">}}writeConnectionSecretToRef{{</hover>}} で定義された名前でClaimと同じ名前空間に保存されます。

```yaml {label="claim3",copy-lines="none"}
apiVersion: example.org/v1alpha1
kind: SecretTest
metadata:
  name: test-secrets
  namespace: default
spec:
  writeConnectionSecretToRef:
    name: my-access-key-secret
```

Claimを適用した後、Crossplaneは以下の秘密を作成します：
* Claimの秘密、{{<hover label="allSec" line="3">}}my-access-key-secret{{</hover>}} 
  Claimの {{<hover label="claim3" line="5">}}namespace{{</hover>}} に。
* 最初のリソースの秘密オブジェクト、{{<hover label="allSec" line="4">}}key1{{</hover>}}。
* 2番目のリソースの秘密オブジェクト、{{<hover label="allSec" line="5">}}key2{{</hover>}}。
* Compositionの `writeConnectionSecretsToNamespace` で定義された
  {{<hover label="allSec" line="6">}}other-namespace{{</hover>}} にある複合リソース秘密オブジェクト。

```shell {label="allSec",copy-lines="none"}
 kubectl get secret -A
NAMESPACE           NAME                                   TYPE                                DATA   AGE
default             my-access-key-secret                   connection.crossplane.io/v1alpha1   8      29m
docs                key1                                   connection.crossplane.io/v1alpha1   4      31m
docs                key2                                   connection.crossplane.io/v1alpha1   4      31m
other-namespace     b0dc71f8-2688-4ebc-818a-bbad6a2c4f9a   connection.crossplane.io/v1alpha1   8      31m
```
