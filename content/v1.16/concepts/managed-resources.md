---
title: 管理されたリソース
weight: 10
description: "管理されたリソースは、外部プロバイダーリソースのCrossplaneによる表現です"
---

_管理されたリソース_（`MR`）は、プロバイダー内の外部サービスを表します。ユーザーが新しい管理されたリソースを作成すると、プロバイダーはプロバイダーの環境内に外部リソースを作成することで反応します。Crossplaneによって管理されるすべての外部サービスは、管理されたリソースにマッピングされます。

{{< hint "note" >}}
CrossplaneはKubernetes内のオブジェクトを_管理されたリソース_と呼び、プロバイダー内の外部オブジェクトを_外部リソース_と呼びます。
{{< /hint >}}

管理されたリソースの例には以下が含まれます：
* Amazon AWS EC2 [`Instance`](https://marketplace.upbound.io/providers/upbound/provider-aws/latest/resources/ec2.aws.upbound.io/Instance/v1beta1)
* Google Cloud GKE [`Cluster`](https://marketplace.upbound.io/providers/upbound/provider-gcp/latest/resources/container.gcp.upbound.io/Cluster/v1beta1)
* Microsoft Azure PostgreSQL [`Database`](https://marketplace.upbound.io/providers/upbound/provider-azure/latest/resources/dbforpostgresql.azure.upbound.io/Database/v1beta1)

{{< hint "tip" >}}

個別の管理されたリソースを作成することもできますが、Crossplaneは
[コンポジション]({{<ref "./compositions" >}})とクレームを使用して
管理されたリソースを作成することを推奨します。
{{< /hint >}}

## 管理されたリソースのフィールド

プロバイダーは、管理されたリソースのグループ、種類、およびバージョンを定義します。
プロバイダーはまた、管理されたリソースの利用可能な設定を定義します。

### グループ、種類、およびバージョン
各管理されたリソースは、独自のグループ、種類、およびバージョンを持つユニークなAPIエンドポイントです。

たとえば、[Upbound AWSプロバイダー](https://marketplace.upbound.io/providers/upbound/provider-aws/latest/)
は、グループ{{<hover label="gkv" line="1">}}ec2.aws.upbound.io{{</hover>}}から
種類{{<hover label="gkv" line="2">}}Instance{{</hover>}}を定義します。

```yaml {label="gkv",copy-lines="none"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Instance
```

<!-- vale off -->
### deletionPolicy
<!-- vale on -->

管理されたリソースの`deletionPolicy`は、管理されたリソースを削除した後にプロバイダーが何をすべきかを示します。`deletionPolicy`が`Delete`の場合、プロバイダーは外部リソースも削除します。`deletionPolicy`が`orphan`の場合、プロバイダーは管理されたリソースを削除しますが、外部リソースは削除しません。

#### オプション
* `deletionPolicy: Delete` - **デフォルト** - 管理リソースを削除する際に外部リソースを削除します。
* `deletionPolicy: Orphan` - 管理リソースを削除する際に外部リソースを残します。

#### 管理ポリシーとの相互作用

[管理ポリシー](#managementpolicies)は以下の場合に
`deletionPolicy`よりも優先されます。
<!-- vale write-good.Passive = NO -->
- 関連する管理ポリシーのアルファ機能が有効になっている場合。
<!-- vale write-good.Passive = YES -->
- リソースがデフォルト値以外の管理ポリシーを設定している場合。

詳細については、以下の表を参照してください。

{{< table "table table-sm table-hover">}}
| managementPolicies          | deletionPolicy   | result  |
|-----------------------------|------------------|---------|
| "*" (デフォルト)               | Delete (デフォルト) | Delete  |
| "*" (デフォルト)               | Orphan           | Orphan  |
| "Delete"を含む           | Delete (デフォルト) | Delete  |
| "Delete"を含む           | Orphan           | Delete  |
| "Delete"を含まない   | Delete (デフォルト) | Orphan  |
| "Delete"を含まない   | Orphan           | Orphan  |
{{< /table >}}

<!-- vale off -->
### forProvider
<!-- vale on -->

管理リソースの {{<hover label="forProvider" line="4">}}spec.forProvider{{</hover>}} は外部リソースのパラメータにマッピングされます。

例えば、AWS EC2インスタンスを作成する際、プロバイダーはAWSの {{<hover label="forProvider" line="5">}}region{{</hover>}} とVMサイズ、すなわち {{<hover label="forProvider" line="6">}}instanceType{{</hover>}} を定義することをサポートします。

{{< hint "note" >}}
プロバイダーは設定とその有効な値を定義します。プロバイダーはまた、`forProvider` 定義における必須およびオプションの値も定義します。

詳細については、特定のプロバイダーのドキュメントを参照してください。
{{< /hint >}}


```yaml {label="forProvider",copy-lines="none"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Instance
# Removed for brevity
spec:
  forProvider:
    region: us-west-1
    instanceType: t2.micro
```

{{< hint "important">}}
Crossplaneは管理リソースの`forProvider`フィールドを外部リソースの「真実の源」と見なします。Crossplaneは、Crossplaneの外部で行われた外部リソースへの変更を上書きします。ユーザーがプロバイダーのウェブコンソール内で変更を行った場合、Crossplaneはその変更を`forProvider`設定で構成された内容に戻します。
{{< /hint >}}

#### 他のリソースの参照

管理リソースのいくつかのフィールドは、他の管理リソースからの値に依存する場合があります。たとえば、VMは使用するための仮想ネットワークの名前が必要です。

管理リソースは、外部名、名前参照、またはセレクタによって他の管理リソースを参照できます。

##### 外部名による一致

リソースを名前で一致させるとき、Crossplaneはプロバイダー内の外部リソースの名前を探します。

たとえば、`my-test-vpc`という名前のAWS VPCオブジェクトは、外部名`vpc-01353cfe93950a8ff`を持っています。

```shell {copy-lines="1"}
kubectl get vpc
NAME            READY   SYNCED   EXTERNAL-NAME           AGE
my-test-vpc     True    True     vpc-01353cfe93950a8ff   49m
```

VPCを名前で一致させるには、外部名を使用します。たとえば、このVPCに接続されたサブネット管理リソースを作成します。

```yaml {copy-lines="none"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Subnet
spec:
  forProvider:
    # Removed for brevity
    vpcId: vpc-01353cfe93950a8ff
```      

##### 名前参照による一致

管理リソースの名前に基づいてリソースを一致させるには、プロバイダー内の外部リソース名ではなく、`nameRef`を使用します。

たとえば、`my-test-vpc`という名前のAWS VPCオブジェクトは、外部名`vpc-01353cfe93950a8ff`を持っています。

```shell {copy-lines="1"}
kubectl get vpc
NAME            READY   SYNCED   EXTERNAL-NAME           AGE
my-test-vpc     True    True     vpc-01353cfe93950a8ff   49m
```

VPCを名前参照で一致させるには、管理リソース名を使用します。たとえば、このVPCに接続されたサブネット管理リソースを作成します。

```yaml {copy-lines="none"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Subnet
spec:
  forProvider:
    # Removed for brevity
    vpcIdRef: 
      name: my-test-vpc
```      


##### セレクタによる一致

セレクタによる一致は、最も柔軟な一致方法です。

{{<hint "note" >}}

[Compositions]({{<ref "./compositions">}})セクションでは、`matchControllerRef`セレクタについて説明しています。
{{</hint >}}

`matchLabels`を使用して、リソースに適用されたラベルを一致させます。たとえば、このサブネットリソースは、ラベル`my-label: label-value`を持つVPCリソースのみと一致します。

```yaml {copy-lines="none"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Subnet
spec:
  forProvider:
    # Removed for brevity
    vpcIdSelector: 
      matchLabels:
        my-label: label-value
```


#### 不変フィールド

一部のプロバイダーは、作成後に一部の管理リソースのフィールドを変更することをサポートしていません。たとえば、Amazon AWSの`RDSInstance`の`region`を変更することはできません。これらのフィールドは_不変フィールド_です。Amazonは、リソースを削除して再作成することを要求します。

Crossplaneは、管理リソースの不変フィールドを編集することを許可しますが、変更は適用されません。Crossplaneは、`forProvider`の変更に基づいてリソースを削除することは決してありません。

{{<hint "note" >}}
<!-- vale write-good.Passive = NO -->
Crossplaneは、Terraformのような他のツールとは異なる動作をします。Terraformは、不変フィールドを変更するためにリソースを削除して再作成します。Crossplaneは、対応する管理リソースオブジェクトがKubernetesから削除され、`deletionPolicy`が`Delete`の場合にのみ、外部リソースを削除します。
<!-- vale write-good.Passive = YES -->
{{< /hint >}}

#### 遅延初期化

Crossplaneは、デフォルトで管理リソースを真実のソースとして扱います。`spec.forProvider`のすべての値（オプションのものを含む）を持つことを期待しています。提供されない場合、Crossplaneは空のフィールドをプロバイダーによって割り当てられた値で埋めます。たとえば、`region`や`availabilityZone`のようなフィールドを考えてみてください。地域のみを指定し、クラウドプロバイダーに可用性ゾーンを選択させることができます。この場合、プロバイダーが可用性ゾーンを割り当てると、Crossplaneはその値を使用して`spec.forProvider.availabilityZone`フィールドを埋めます。

{{<hint "note" >}}
<!-- vale write-good.Passive = NO -->
[managementPolicies]({{<ref "./managed-resources#managementpolicies" >}})を使用すると、`managementPolicies`リストに`LateInitialize`ポリシーを含めないことで、この動作をオフにすることができます。
<!-- vale write-good.Passive = YES -->
{{< /hint >}}

<!-- vale off -->
### initProvider
<!-- vale on -->

{{<hint "important" >}}
管理リソースの`initProvider`オプションは、[managementPolicies]({{<ref "./managed-resources#managementpolicies" >}})に関連するベータ機能です。

{{< /hint >}}

{{<hover label="initProvider" line="7">}}initProvider{{</hover>}}は、新しい管理リソースを作成する際にCrossplaneが適用する設定を定義します。  
Crossplaneは、作成後に変更された{{<hover label="initProvider" line="7">}}initProvider{{</hover>}}フィールドで定義された設定を無視します。

{{<hint "note" >}}
`forProvider`の設定は常にCrossplaneによって強制されます。Crossplaneは、外部リソースの`forProvider`フィールドへの変更を元に戻します。


`initProvider`の設定はCrossplaneによって強制されません。Crossplaneは外部リソースの`initProvider`フィールドへの変更を無視します。
{{</hint >}}

`initProvider`を使用することは、プロバイダーが自動的に変更する可能性のある初期値を設定するのに便利です。たとえば、初期の
{{<hover label="initProvider" line="2">}}NodeGroup{{</hover>}}
を作成し、初期の
{{<hover label="initProvider" line="9">}}desiredSize{{</hover>}}を設定します。  
Crossplaneは、オートスケーラーがNode Group外部リソースをスケールする際に
{{<hover label="initProvider" line="9">}}desiredSize{{</hover>}}
設定を元に戻しません。

{{< hint "tip" >}}
Crossplaneは、`initProvider`設定との競合を避けるために
{{<hover label="initProvider" line="6">}}managementPolicies{{</hover>}}を`LateInitialize`なしで構成することを推奨します。
{{< /hint >}}

```yaml {label="initProvider",copy-lines="none"}
apiVersion: eks.aws.upbound.io/v1beta1
kind: NodeGroup
metadata:
  name: sample-eks-ng
spec:
  managementPolicies: ["Observe", "Create", "Update", "Delete"]
  initProvider:
    scalingConfig:
      - desiredSize: 1
  forProvider:
    region: us-west-1
    scalingConfig:
      - maxSize: 4
        minSize: 1
```

<!-- vale off -->
### managementPolicies
<!-- vale on --> 

{{<hint "note" >}}
管理リソースの`managementPolicies`オプションはベータ機能です。Crossplaneはデフォルトでベータ機能を有効にします。

プロバイダーが管理ポリシーのサポートを決定します。  
プロバイダーが管理ポリシーをサポートしているかどうかは、プロバイダーのドキュメントを参照してください。
{{< /hint >}}

Crossplaneの
{{<hover label="managementPol1" line="4">}}managementPolicies{{</hover>}}
は、Crossplaneが管理リソースおよびその対応する外部リソースに対してどのアクションを実行できるかを決定します。  
1つ以上の
{{<hover label="managementPol1" line="4">}}managementPolicies{{</hover>}}
を管理リソースに適用して、Crossplaneがそのリソースに対してどのような権限を持つかを決定します。

たとえば、Crossplaneに外部リソースを作成および削除する権限を与えますが、変更は行わないようにするには、ポリシーを
{{<hover label="managementPol1" line="4">}}["Create", "Delete", "Observe"]{{</hover>}}に設定します。

```yaml {label="managementPol1"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Subnet
spec:
  managementPolicies: ["Create", "Delete", "Observe"]
  forProvider:
    # Removed for brevity
```

デフォルトのポリシーは、Crossplaneにリソースに対する完全な制御を付与します。  
`managementPolicies`フィールドを空の配列で定義すると、リソースが[一時停止](#paused)します。


{{<hint "重要" >}}
プロバイダーは管理ポリシーのサポートを決定します。  
プロバイダーが管理ポリシーをサポートしているかどうかは、プロバイダーのドキュメントを参照してください。
{{< /hint >}}

Crossplaneは以下のポリシーをサポートしています：
{{<table "table table-sm table-hover">}}
| ポリシー | 説明 |
| --- | --- |
| `*` | _デフォルトポリシー_。Crossplaneはリソースに対して完全な制御を持ちます。 |
| `Create` | 外部リソースが存在しない場合、Crossplaneは管理リソースの設定に基づいてそれを作成します。 |
| `Delete` | Crossplaneは管理リソースを削除する際に外部リソースを削除できます。 |
| `LateInitialize` | Crossplaneは管理リソースの`spec.forProvider`に定義されていない外部リソースの設定を初期化します。詳細については[遅延初期化]({{<ref "./managed-resources#late-initialization" >}})セクションを参照してください。 |
| `Observe` | Crossplaneはリソースを観察するだけで、変更を加えません。[観察専用リソース]({{<ref "../guides/import-existing-resources#import-resources-automatically">}})に使用されます。 |
| `Update` | Crossplaneは管理リソースを変更する際に外部リソースを変更します。 |
{{</table >}}

以下は一般的なポリシーの組み合わせのリストです：
{{<table "table table-sm table-hover table-striped-columns" >}}
| Create | Delete | LateInitialize | Observe | Update | 説明 |
| :---:  | :---:  | :---:          | :---:   | :---:  | ---         |
| {{<check>}}      | {{<check>}}      | {{<check>}}              | {{<check>}}       | {{<check>}}      | _デフォルトポリシー_。Crossplaneはリソースに対して完全な制御を持ちます。                                                                                                     |
| {{<check>}}      | {{<check>}}      | {{<check>}}              | {{<check>}}       |        | 作成後、管理リソースに加えられた変更は外部リソースに渡されません。変更不可能な外部リソースに便利です。 |
| {{<check>}}      | {{<check>}}      |                | {{<check>}}       | {{<check>}}      | 管理リソースに定義されていない設定をCrossplaneが管理するのを防ぎます。外部リソースの不変フィールドに便利です。 |
| {{<check>}}      | {{<check>}}      |                | {{<check>}}       |        | Crossplaneは外部リソースから設定をインポートせず、管理リソースに変更をプッシュしません。外部リソースが削除された場合、Crossplaneはそれを再作成します。 |
| {{<check>}}      |        | {{<check>}}              | {{<check>}}       | {{<check>}}      | 管理リソースを削除する際にCrossplaneは外部リソースを削除しません。 |
| {{<check>}}      |        | {{<check>}}              | {{<check>}}       |        | 管理リソースを削除する際にCrossplaneは外部リソースを削除しません。作成後、Crossplaneは外部リソースに変更を適用しません。 |
| {{<check>}}      |        |                | {{<check>}}       | {{<check>}}      | 管理リソースを削除する際にCrossplaneは外部リソースを削除しません。Crossplaneは外部リソースから設定をインポートしません。 |
| {{<check>}}      |        |                | {{<check>}}       |        | Crossplaneは外部リソースを作成しますが、外部リソースまたは管理リソースに変更を適用しません。Crossplaneはリソースを削除できません。 |
|        |        |                | {{<check>}}       |        | Crossplaneはリソースを観察するだけです。[観察専用リソース]({{<ref "../guides/import-existing-resources#import-resources-automatically">}})に使用されます。 |
|        |        |                |         |        | ポリシーが設定されていません。リソースを[一時停止](#paused)するための代替手段です。                                                                                              |
{{< /table >}}

<!-- vale off -->
### providerConfigRef
<!-- vale on -->

`providerConfigRef`は、管理リソースがどの
[ProviderConfig]({{<ref "./providers#provider-configuration">}})を
使用して管理リソースを作成するかをProviderに伝えます。  

Providerと通信する際に使用する認証方法を定義するために
ProviderConfigを使用します。

{{< hint "important" >}}
`providerConfigRef`が適用されていない場合、Providersは`default`という名前のProviderConfigを使用します。
{{< /hint >}}

例えば、管理リソースは{{<hover label="pcref" line="6">}}user-keys{{</hover>}}という名前のProviderConfigを参照します。

これはProviderConfigの{{<hover label="pc" line="4">}}name{{</hover>}}と一致します。

```yaml {label="pcref",copy-lines="none"}}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Instance
spec:
  forProvider:
    # Removed for brevity
  providerConfigRef: user-keys
```

```yaml {label="pc"}
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: user-keys
# Removed for brevity
```

{{< hint "tip" >}}
各管理リソースは異なるProviderConfigsを参照できます。これにより、
異なる管理リソースが同じProviderに対して異なる資格情報で認証できます。 
{{< /hint >}}

<!-- vale off -->
### providerRef
<!-- vale on --> 

<!-- vale Crossplane.Spelling = NO -->
Crossplaneは`crossplane-runtime`の`providerRef`フィールドを
[v0.10.0](https://github.com/crossplane/crossplane-runtime/releases/tag/v0.10.0)で非推奨にしました。 
`providerRef`を使用している管理リソースは[`providerConfigRef`](#providerconfigref)を使用する必要があります。
<!-- vale Crossplane.Spelling = YES -->

<!-- vale off -->
### writeConnectionSecretToRef
<!-- vale on --> 

Providerが管理リソースを作成する際、ユーザー名、パスワード、IPアドレスのような接続詳細など、リソース固有の
詳細を生成する場合があります。 

Crossplaneは、`writeConnectionSecretToRef`の値で指定されたKubernetes Secretオブジェクトにこれらの詳細を保存します。 

例えば、Crossplaneの
[community AWS provider](https://marketplace.upbound.io/providers/crossplane-contrib/provider-aws/v0.40.0)を使用してAWS RDSデータベースインスタンスを作成する際に、エンドポイント、パスワード、ポート、ユーザー名のデータが生成されます。Providerは
これらの変数をKubernetesシークレット
{{<hover label="secretname" line="9" >}}rds-secret{{</hover>}}に保存し、
{{<hover label="secretname" line="9" >}}writeConnectionSecretToRef{{</hover>}}
フィールドで参照します。

```yaml {label="secretname",copy-lines="none"}
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: my-rds-instance
spec:
  forProvider:
  # Removed for brevity
  writeConnectionSecretToRef:
    name: rds-secret
```

Secretオブジェクトを表示すると、保存されたフィールドが表示されます。

```yaml {copy-lines="1"}
kubectl describe secret rds-secret
Name:         rds-secret
# Removed for brevity
Data
====
port:      4 bytes
username:  10 bytes
endpoint:  54 bytes
password:  27 bytes
```

{{<hint "重要" >}}
プロバイダーはSecretオブジェクトに書き込まれるデータを決定します。生成されたSecretデータについては、特定のプロバイダーのドキュメントを参照してください。
{{< /hint >}}

<!-- vale off -->
### publishConnectionDetailsTo
<!-- vale on --> 

`publishConnectionDetailsTo`フィールドは、[`writeConnectionSecretToRef`](#writeconnectionsecrettoref)を拡張し、管理リソース情報をKubernetes Secretオブジェクトとして、または[HashiCorp Vault](https://www.vaultproject.io/)のような外部シークレットストアに保存することをサポートします。

`publishConnectionDetailsTo`を使用するには、Crossplane External Secrets Stores (ESS)を有効にする必要があります。プロバイダー内で[DeploymentRuntimeConfig]({{<ref "providers#runtime-configuration" >}})を使用してESSを有効にし、Crossplaneでは`--enable-external-secret-stores`引数を使用します。

{{< hint "注意" >}}
すべてのプロバイダーが`publishConnectionDetailsTo`をサポートしているわけではありません。詳細については、プロバイダーのドキュメントを確認してください。
{{< /hint >}}

#### Kubernetesにシークレットを公開する

管理リソースによって生成されたデータをKubernetes Secretオブジェクトとして公開するには、 
{{<hover label="k8secret" line="7">}}publishConnectionDetailsTo.name{{< /hover >}}を提供します。

```yaml {label="k8secret",copy-lines="none"}
apiVersion: rds.aws.upbound.io/v1beta1
kind: Instance
spec:
  forProvider:
  # Removed for brevity
  publishConnectionDetailsTo:
    name: rds-kubernetes-secret
```

Crossplaneは、Kubernetesシークレットに対してラベルやアノテーションを適用することもでき、 
{{<hover label="k8label" line="8">}}publishConnectionDetailsTo.metadata{{</hover>}}を使用します。

```yaml {label="k8label",copy-lines="none"}
apiVersion: rds.aws.upbound.io/v1beta1
kind: Instance
spec:
  forProvider:
  # Removed for brevity
  publishConnectionDetailsTo:
    name: rds-kubernetes-secret
    metadata:
      labels:
        label-tag: label-value
      annotations:
        annotation-tag: annotation-value
```

#### 外部シークレットストアにシークレットを公開する

[HashiCorp Vault](https://www.vaultproject.io/)のような外部シークレットストアにシークレットデータを公開するには、 
{{<hover label="configref" line="8">}}publishConnectionDetailsTo.configRef{{</hover>}}が必要です。

{{<hover label="configref" line="9">}}configRef.name{{</hover>}}は、 
{{<hover label="storeconfig" line="4">}}StoreConfig{{</hover>}}オブジェクトを参照します。

```yaml {label="configref",copy-lines="none"}
apiVersion: rds.aws.upbound.io/v1beta1
kind: Instance
spec:
  forProvider:
  # Removed for brevity
  publishConnectionDetailsTo:
    name: rds-kubernetes-secret
    configRef: 
      name: my-vault-storeconfig
```

```yaml {label="storeconfig",copy-lines="none"}
apiVersion: secrets.crossplane.io/v1alpha1
kind: StoreConfig
metadata:
  name: my-vault-storeconfig
# Removed for brevity
```

{{<hint "tip" >}}
ストアコンフィグオブジェクトの使用に関する詳細は、[Vault as an External Secrets Store]({{<ref "../guides/vault-as-secret-store">}})ガイドをお読みください。
{{< /hint >}}

## アノテーション

Crossplaneは、管理リソースに標準のKubernetes `annotations`セットを適用します。

{{<table "table table-sm">}}
| アノテーション | 定義 | 
| --- | --- | 
| `crossplane.io/external-name` | プロバイダー内の管理リソースの名前。 |
| `crossplane.io/external-create-pending` | Crossplaneが管理リソースの作成を開始した時刻のタイムスタンプ。 | 
| `crossplane.io/external-create-succeeded` | プロバイダーが管理リソースを正常に作成した時刻のタイムスタンプ。 | 
| `crossplane.io/external-create-failed` | プロバイダーが管理リソースの作成に失敗した時刻のタイムスタンプ。 | 
| `crossplane.io/paused` | Crossplaneがこのリソースを調整していないことを示します。詳細については[Pause Annotation](#paused)をお読みください。 |
| `crossplane.io/composition-resource-name` | コンポジションによって作成された管理リソースの場合、これはコンポジションの`resources.name`値です。 | 
{{</table >}}

### 外部リソースの命名
デフォルトでは、プロバイダーは外部リソースにKubernetesオブジェクトと同じ名前を付けます。

たとえば、{{<hover label="external-name" line="4">}}my-rds-instance{{</hover >}}という名前の管理リソースは、プロバイダーの環境内で外部リソースとして`my-rds-instance`という名前を持ちます。 

```yaml {label="external-name",copy-lines="none"}
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: my-rds-instance
```

```shell
kubectl get rdsinstance
NAME                 READY   SYNCED   EXTERNAL-NAME        AGE
my-rds-instance      True    True     my-rds-instance      11m
```

`crossplane.io/external-name`アノテーションがすでに提供されている管理リソースは、アノテーションの値を外部リソース名として使用します。

たとえば、プロバイダーは{{< hover label="custom-name" line="6">}}my-rds-instance{{</hover>}}という名前の管理リソースを作成しますが、AWS内の外部リソースには{{<hover label="custom-name" line="5">}}my-custom-name{{</hover >}}という名前を使用します。

```yaml {label="custom-name",copy-lines="none"}
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: my-rds-instance  
  annotations: 
    crossplane.io/external-name: my-custom-name
```

```shell {copy-lines="1"}
kubectl get rdsinstance
NAME                 READY   SYNCED   EXTERNAL-NAME        AGE
my-rds-instance      True    True     my-custom-name       11m
```

### 作成アノテーション

AWSのような外部システムが非決定的なリソース名を生成する場合、プロバイダーがリソースを作成することは可能ですが、それを記録しないことがあります。このような場合、プロバイダーはリソースを管理できません。

{{<hint "tip">}}
Crossplaneは、プロバイダーが作成したが管理しないリソースを _漏れたリソース_ と呼びます。
{{</hint>}}

プロバイダーは、漏れたリソースを回避し検出するために3つの作成アノテーションを設定します：

* {{<hover label="creation" line="8">}}crossplane.io/external-create-pending{{</hover>}} -
  プロバイダーがリソースを作成しようとした最後の時間。
* {{<hover label="creation" line="9">}}crossplane.io/external-create-succeeded{{</hover>}} -
  プロバイダーがリソースを成功裏に作成した最後の時間。
* `crossplane.io/external-create-failed` - プロバイダーがリソースの作成に失敗した最後の時間。

`kubectl get`を使用して、管理されたリソースのアノテーションを表示します。例えば、AWS VPCリソース：

```yaml {label="creation" copy-lines="2-9"}
$ kubectl get -o yaml vpc my-vpc
apiVersion: ec2.aws.upbound.io/v1beta1
kind: VPC
metadata:
  name: my-vpc
  annotations:
    crossplane.io/external-name: vpc-1234567890abcdef0
    crossplane.io/external-create-pending: "2023-12-18T21:48:06Z"
    crossplane.io/external-create-succeeded: "2023-12-18T21:48:40Z"
```

プロバイダーは
{{<hover label="creation" line="7">}}crossplane.io/external-name{{</hover>}}
アノテーションを使用して、外部システムで管理されたリソースを検索します。

プロバイダーは、外部システムでリソースが存在するかどうか、またそれが管理されたリソースの望ましい状態と一致するかどうかを判断するためにリソースを検索します。プロバイダーがリソースを見つけられない場合、それを作成します。

一部の外部システムでは、プロバイダーがリソースを作成する際にリソースの名前を指定することを許可していません。代わりに、外部システムが非決定的な名前を生成し、それをプロバイダーに返します。

外部システムがリソースの名前を生成すると、プロバイダーはそれを管理されたリソースの `crossplane.io/external-name` アノテーションに保存しようとします。保存できない場合、それはリソースを _漏らします_。

プロバイダーは、アノテーションを保存できることを保証できません。プロバイダーは、リソースを作成してアノテーションを保存する間に再起動したり、ネットワーク接続を失ったりする可能性があります。

プロバイダーは、リソースが漏洩した可能性があることを検出できます。プロバイダーがリソースが漏洩した可能性があると考えると、それを再調整するのを停止し、プロバイダーに進行しても安全であることを伝えるまで待機します。

{{<hint "important">}}
外部システムがリソースの名前を生成するたびに、プロバイダーがリソースを漏洩するリスクがあります。

プロバイダーがリソースが漏洩した可能性があると検出した場合に最も安全なことは、停止して人間の介入を待つことです。

これにより、プロバイダーが漏洩したリソースの重複を作成しないことが保証されます。
重複リソースはコストがかかり、危険です。
{{</hint>}}

プロバイダーがリソースが漏洩した可能性があると考えると、管理リソースに関連付けられた `cannot determine creation result` イベントを作成します。イベントを見るには `kubectl describe` を使用します。

```shell {copy-lines="1"}
kubectl describe queue my-sqs-queue

# Removed for brevity

Events:
  Type     Reason                           Age                 From                                 Message
  ----     ------                           ----                ----                                 -------
  Warning  CannotInitializeManagedResource  29m (x19 over 19h)  managed/queue.sqs.aws.crossplane.io  cannot determine creation result - remove the crossplane.io/external-create-pending annotation if it is safe to proceed
```

プロバイダーは、作成アノテーションを使用してリソースが漏洩した可能性があることを検出します。

プロバイダーが管理リソースを再調整するたびに、リソースの作成アノテーションをチェックします。プロバイダーが最も最近の作成成功または作成失敗の時間よりも新しい作成保留時間を確認すると、リソースが漏洩した可能性があることを認識します。

{{<hint "note">}}
プロバイダーは作成アノテーションを削除しません。彼らはタイムスタンプを使用してどれが最も最近であるかを判断します。管理リソースが複数の作成アノテーションを持つことは正常です。
{{</hint>}}

プロバイダーは、リソースのすべてのアノテーションを同時に更新するため、リソースが漏洩した可能性があることを知っています。プロバイダーがリソースを作成した後に作成アノテーションを更新できなかった場合、`crossplane.io/external-name` アノテーションも更新できなかったことになります。

{{<hint "tip">}}
リソースに `cannot determine creation result` エラーがある場合は、外部システムを調査してください。

`crossplane.io/external-create-pending` アノテーションのタイムスタンプを使用して、プロバイダーがリソースを漏洩した可能性がある時期を特定します。この時間帯に作成されたリソースを探してください。

漏洩したリソースを見つけた場合、安全であれば、外部システムから削除してください。

`crossplane.io/external-create-pending` アノテーションを管理リソースから削除します。漏れたリソースが存在しないことを確認した後に行ってください。これにより、プロバイダーは管理リソースの調整を再開し、再作成することができます。
{{</hint>}}

プロバイダーは、リソースの漏れを避けるために作成アノテーションを使用します。

プロバイダーが `crossplane.io/external-create-pending` アノテーションを書き込むと、管理リソースの最新バージョンを調整していることがわかります。プロバイダーが古いバージョンの管理リソースを調整している場合、書き込みは失敗します。

プロバイダーが古いバージョンを古い `crossplane.io/external-name` アノテーションで調整した場合、リソースが存在しないと誤って判断する可能性があります。プロバイダーは新しいリソースを作成し、既存のリソースを漏らすことになります。

一部の外部システムでは、プロバイダーがリソースを作成してから、そのリソースが存在すると報告されるまでに遅延があります。プロバイダーは、この遅延を考慮するために、最も最近の作成成功時刻を使用します。

プロバイダーが遅延を考慮しなかった場合、リソースが存在しないと誤って判断する可能性があります。プロバイダーは新しいリソースを作成し、既存のリソースを漏らすことになります。

### 一時停止
`crossplane.io/paused` アノテーションを手動で適用すると、プロバイダーは管理リソースの調整を停止します。

リソースを一時停止することは、プロバイダーを変更したり、Kubernetesオブジェクトを編集する際のレースコンディションを防ぐのに役立ちます。

管理リソースに {{<hover label="pause" line="6">}}crossplane.io/paused: "true"{{</hover>}} アノテーションを適用して、調整を一時停止します。

{{< hint "note" >}}
値 `"true"` のみが調整を一時停止します。
{{< /hint >}}

```yaml {label="pause"}
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Instance
metadata:
  name: my-rds-instance
  annotations:
    crossplane.io/paused: "true"
spec:
  forProvider:
    region: us-west-1
    instanceType: t2.micro
```

アノテーションを削除して調整を再開します。

{{<hint "important">}}
Kubernetes と Crossplane は、`paused` アノテーションを持つリソースを削除できません。`kubectl delete` でさえもです。

詳細については、[Crossplane discussion #4839](https://github.com/crossplane/crossplane/issues/4839) をお読みください。
{{< /hint >}}

## ファイナライザー
Crossplane は、管理リソースの削除を制御するために [ファイナライザー](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/) を適用します。


{{< hint "note" >}}
Kubernetes は Finalizers を持つオブジェクトを削除できません。
{{</hint >}}

Crossplane が管理リソースを削除すると、プロバイダーは外部リソースの削除を開始しますが、外部リソースが完全に削除されるまで管理リソースは残ります。

外部リソースが完全に削除されると、Crossplane は Finalizer を削除し、管理リソースオブジェクトを削除します。

## 条件

Crossplane には管理リソースのための標準的な `Conditions` セットがあります。 `kubectl describe <managed_resource>` を使用して管理リソースの `Conditions` を表示します。


{{<hint "note" >}}
プロバイダーは独自のカスタム `Conditions` を定義することがあります。 
{{</hint >}}


### 利用可能
`Reason: Available` は、プロバイダーが管理リソースを作成し、使用可能であることを示します。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Ready
  Status:                True
  Reason:                Available
```
### 作成中

`Reason: Creating` は、プロバイダーが管理リソースを作成しようとしていることを示します。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Ready
  Status:                False
  Reason:                Creating
```

### 削除中
`Reason: Deleting` は、プロバイダーが管理リソースを削除しようとしていることを示します。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Ready
  Status:                False
  Reason:                Deleting
```

<!-- vale off -->
### ReconcilePaused
<!-- vale on -->
`Reason: ReconcilePaused` は、管理リソースに [Pause](#paused) アノテーションがあることを示します。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Synced
  Status:                False
  Reason:                ReconcilePaused
```

<!-- vale off -->
### ReconcileError
<!-- vale on -->
`Reason: ReconcileError` は、Crossplane が管理リソースの調整中にエラーに遭遇したことを示します。 `Condition` の `Message:` 値は、Crossplane エラーを特定するのに役立ちます。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Synced
  Status:                False
  Reason:                ReconcileError
```

<!-- vale off -->
### ReconcileSuccess
<!-- vale on -->
`Reason: ReconcileSuccess` は、プロバイダーが管理リソースを作成し、監視していることを示します。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Synced
  Status:                True
  Reason:                ReconcileSuccess
```

### 利用不可
`Reason: Unavailable` は、Crossplane が管理リソースが利用可能であると期待しているが、プロバイダーがリソースが不健康であると報告していることを示します。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Ready
  Status:                False
  Reason:                Unavailable
```

### 不明
`Reason: Unknown` は、プロバイダーが管理リソースに対して予期しないエラーが発生したことを示します。 `conditions.message` は、何が問題だったのかについての詳細情報を提供します。

```yaml {copy-lines="none"}
Conditions:
  Type:                  Unknown
  Status:                False
  Reason:                Unknown
```


### Upjet プロバイダーの条件
[Upjet](https://github.com/upbound/upjet) は、Crossplane プロバイダーを生成するためのオープンソースツールで、標準の `Conditions` のセットも持っています。


<!-- vale off -->
#### AsyncOperation
<!-- vale on -->

一部のリソースは作成に1分以上かかる場合があります。Upjet ベースのプロバイダーは、非同期操作を使用することで、管理リソースを作成する前に Kubernetes コマンドを完了できます。


##### 完了
`Reason: Finished` は、非同期操作が正常に完了したことを示します。

```yaml {copy-lines="none"}
Conditions:
  Type:                  AsyncOperation
  Status:                True
  Reason:                Finished
```


##### 進行中

`Reason: Ongoing` は、管理リソースの操作がまだ進行中であることを示します。

```yaml {copy-lines="none"}
Conditions:
  Type:                  AsyncOperation
  Status:                True
  Reason:                Ongoing
```

<!-- vale off -->
#### LastAsyncOperation
<!-- vale on -->

Upjet の `Type: LastAsyncOperation` は、前回の非同期操作の状態を `Success` または失敗の `Reason` としてキャプチャします。

<!-- vale off -->
##### ApplyFailure
<!-- vale on -->

`Reason: ApplyFailure` は、プロバイダーが管理リソースに設定を適用できなかったことを示します。 `conditions.message` は、何が問題だったのかについての詳細情報を提供します。

```yaml {copy-lines="none"}
Conditions:
  Type:                  LastAsyncOperation
  Status:                False
  Reason:                ApplyFailure
```

<!-- vale off -->
##### DestroyFailure
<!-- vale on -->

`Reason: DestroyFailure` は、プロバイダーが管理リソースを削除できなかったことを示します。 `conditions.message` は、何が問題だったのかについての詳細情報を提供します。

```yaml {copy-lines="none"}
Conditions:
  Type:                  LastAsyncOperation
  Status:                False
  Reason:                DestroyFailure
```

##### 成功
`Reason: Success` は、プロバイダーが管理リソースを非同期に正常に作成したことを示します。

```yaml {copy-lines="none"}
Conditions:
  Type:                  LastAsyncOperation
  Status:                True
  Reason:                Success
```
