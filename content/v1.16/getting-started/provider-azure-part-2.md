---
title: Azure クイックスタート パート 2
weight: 120
tocHidden: true
aliases:
  - /master/getting-started/provider-azure-part-3
---

{{< hint "重要" >}}
このガイドはシリーズのパート2です。  

[**パート 1**]({{<ref "provider-azure" >}}) では
CrossplaneのインストールとKubernetesクラスターをAzureに接続する方法について説明します。

{{< /hint >}}

このガイドでは、Crossplaneを使用してカスタムAPIを構築し、アクセスする方法を説明します。

## 前提条件
* [クイックスタート パート 1]({{<ref "provider-azure" >}})を完了し、Kubernetesを
  Azureに接続します。
* Azure仮想マシン、リソースグループ、仮想ネットワークを作成する権限を持つAzureアカウント。

{{<expand "パート 1をスキップして始める" >}}
1. Crossplane Helmリポジトリを追加し、Crossplaneをインストールします。
```shell
helm repo add \
crossplane-stable https://charts.crossplane.io/stable
helm repo update
&&
helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace
```

2. Crossplaneポッドのインストールが完了し、準備が整ったら、Azure 
   プロバイダーを適用します。
   
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

3. Azure CLIを使用してサービスプリンシパルを作成し、JSON出力を 
   `azure-crednetials.json`として保存します。
{{< editCode >}}
```console
az ad sp create-for-rbac \
--sdk-auth \
--role Owner \
--scopes /subscriptions/$@<subscription_id>$@
```
{{</ editCode >}}

4. Azure JSONファイルからKubernetesシークレットを作成します。
```shell {label="kube-create-secret",copy-lines="all"}
kubectl create secret \
generic azure-secret \
-n crossplane-system \
--from-file=creds=./azure-credentials.json
```

5. _ProviderConfig_を作成します。
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
{{</expand >}}

## カスタムAPIの作成

<!-- vale alex.Condescending = NO -->
Crossplaneを使用すると、ユーザー向けに独自のカスタムAPIを構築でき、クラウドプロバイダーやそのリソースに関する詳細を抽象化できます。APIは、複雑でもシンプルでも自由に設計できます。 
<!-- vale alex.Condescending = YES -->

カスタムAPIはKubernetesオブジェクトです。  
以下はカスタムAPIの例です。

```yaml {label="exAPI"}
apiVersion: compute.example.com/v1alpha1
kind: VirtualMachine
metadata:
  name: my-vm
spec: 
  location: "US"
```

他のKubernetesオブジェクトと同様に、APIには 
{{<hover label="exAPI" line="1">}}version{{</hover>}}, 
{{<hover label="exAPI" line="2">}}kind{{</hover>}} および 
{{<hover label="exAPI" line="5">}}spec{{</hover>}}があります。

### グループとバージョンの定義
独自のAPIを作成するには、まず 
[APIグループ](https://kubernetes.io/docs/reference/using-api/#api-groups) と 
[バージョン](https://kubernetes.io/docs/reference/using-api/#api-versioning)を定義します。  

_group_ は任意の値を指定できますが、一般的な慣習として完全修飾ドメイン名にマッピングされます。

<!-- vale gitlab.SentenceLength = NO -->
バージョンはAPIの成熟度や安定性を示し、API内のフィールドを変更、追加、または削除する際にインクリメントされます。
<!-- vale gitlab.SentenceLength = YES -->

Crossplaneは特定のバージョンや特定のバージョン命名規則を必要としませんが、 
[Kubernetes APIバージョニングガイドライン](https://kubernetes.io/docs/reference/using-api/#api-versioning)に従うことを強く推奨します。

* `v1alpha1` - いつでも変更される可能性のある新しいAPIです。
* `v1beta1` - 安定と見なされる既存のAPIです。破壊的変更は強く推奨されません。
* `v1` - 破壊的変更がない安定したAPIです。

このガイドではグループ 
{{<hover label="version" line="1">}}compute.example.com{{</hover>}}を使用します。

これはAPIの最初のバージョンであるため、このガイドではバージョン
{{<hover label="version" line="1">}}v1alpha1{{</hover>}}を使用します。

```yaml {label="version",copy-lines="none"}
apiVersion: compute.example.com/v1alpha1
```

### 種類を定義する

APIグループは関連するAPIの論理的なコレクションです。グループ内には異なるリソースを表す個々の種類があります。

例えば、`compute`グループには`VirtualMachine`と`BareMetal`の種類があるかもしれません。

`kind`は何でも構いませんが、 
[UpperCamelCased](https://kubernetes.io/docs/contribute/style/style-guide/#use-upper-camel-case-for-api-objects)である必要があります。

このAPIの種類は 
{{<hover label="kind" line="2">}}VirtualMachine{{</hover>}}です。

```yaml {label="kind",copy-lines="none"}
apiVersion: compute.example.com/v1alpha1
kind: VirtualMachine
```

### スペックを定義する

APIの最も重要な部分はスキーマです。スキーマはユーザーから受け入れられる入力を定義します。

このAPIでは、ユーザーがクラウドリソースを実行する場所の 
{{<hover label="spec" line="4">}}location{{</hover>}}を提供することを許可しています。

他のすべてのリソース設定はユーザーによって構成できません。これにより、Crossplaneはユーザーエラーを心配することなく、ポリシーや基準を強制することができます。

```yaml {label="spec",copy-lines="none"}
apiVersion: compute.example.com/v1alpha1
kind: VirtualMachine
spec: 
  location: "US"
```

### APIの適用

Crossplaneは 
{{<hover label="xrd" line="3">}}Composite Resource Definitions{{</hover>}} 
（`XRD`とも呼ばれます）を使用して、KubernetesにカスタムAPIをインストールします。

XRDの{{<hover label="xrd" line="6">}}spec{{</hover>}}には、APIに関するすべての情報が含まれています。これには 
{{<hover label="xrd" line="7">}}group{{</hover>}},
{{<hover label="xrd" line="12">}}version{{</hover>}},
{{<hover label="xrd" line="9">}}kind{{</hover>}}および 
{{<hover label="xrd" line="13">}}schema{{</hover>}}が含まれます。

XRDの{{<hover label="xrd" line="5">}}name{{</hover>}}は、{{<hover label="xrd" line="10">}}plural{{</hover>}}と 
{{<hover label="xrd" line="7">}}group{{</hover>}}の組み合わせでなければなりません。

{{<hover label="xrd" line="13">}}schema{{</hover>}}は、APIの{{<hover label="xrd" line="17">}}spec{{</hover>}}を定義するために
{{<hover label="xrd" line="14">}}OpenAPIv3{{</hover>}}仕様を使用します。  

APIは、{{<hover label="xrd" line="20">}}location{{</hover>}}を定義し、それは{{<hover label="xrd" line="22">}}oneOf{{</hover>}}で
{{<hover label="xrd" line="23">}}EU{{</hover>}}または 
{{<hover label="xrd" line="24">}}US{{</hover>}}のいずれかでなければなりません。

このXRDを適用して、KubernetesクラスターにカスタムAPIを作成します。

```yaml {label="xrd",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: virtualmachines.compute.example.com
spec:
  group: compute.example.com
  names:
    kind: VirtualMachine
    plural: virtualmachines
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              location:
                type: string
                oneOf:
                  - pattern: '^EU$'
                  - pattern: '^US$'
            required:
              - location
    served: true
    referenceable: true
  claimNames:
    kind: VirtualMachineClaim
    plural: virtualmachineclaims
EOF
```

{{<hover label="xrd" line="29">}}claimNames{{</hover>}}を追加することで、ユーザーはこのAPIにアクセスできます。
クラスター全体で{{<hover label="xrd" line="9">}}VirtualMachine{{</hover>}}エンドポイントを使用するか、名前空間で
{{<hover label="xrd" line="30">}}VirtualMachineClaim{{</hover>}}エンドポイントを使用します。

名前空間スコープのAPIは、Crossplaneの_Claim_です。

{{<hint "tip" >}}
Composite Resource Definitionsのフィールドとオプションの詳細については、
[XRDドキュメント]({{<ref "../concepts/composite-resource-definitions">}})をお読みください。 
{{< /hint >}}

インストールされたXRDを表示するには、`kubectl get xrd`を実行します。

```shell {copy-lines="1"}
kubectl get xrd
NAME                                  ESTABLISHED   OFFERED   AGE
virtualmachines.compute.example.com   True          True      43s
```


新しいカスタムAPIエンドポイントを表示するには、次のコマンドを実行します `kubectl api-resources | grep VirtualMachine`

```shell {copy-lines="1",label="apiRes"}
kubectl api-resources | grep VirtualMachine
virtualmachineclaims              compute.example.com/v1alpha1           true         VirtualMachineClaim
virtualmachines                   compute.example.com/v1alpha1           false        VirtualMachine
```

## デプロイメントテンプレートの作成

ユーザーがカスタムAPIにアクセスすると、Crossplaneはその入力を取得し、デプロイするインフラストラクチャを説明するテンプレートと組み合わせます。Crossplaneはこのテンプレートを _Composition_ と呼びます。

{{<hover label="comp" line="3">}}Composition{{</hover>}} は、デプロイするすべてのクラウドリソースを定義します。
テンプレート内の各エントリは、リソース設定やメタデータ（ラベルやアノテーションなど）を定義する完全なリソース定義です。

このテンプレートは、Azureの
{{<hover label="comp" line="11">}}LinuxVirtualMachine{{</hover>}}
{{<hover label="comp" line="46">}}NetworkInterface{{</hover>}}, 
{{<hover label="comp" line="69">}}Subnet{{</hover>}}
{{<hover label="comp" line="90">}}VirtualNetwork{{</hover>}} および
{{<hover label="comp" line="110">}}ResourceGroup{{</hover>}} を作成します。

Crossplaneは、{{<hover label="comp" line="34">}}patches{{</hover>}}を使用して
ユーザーの入力をリソーステンプレートに適用します。  
このCompositionは、ユーザーの
{{<hover label="comp" line="36">}}location{{</hover>}} 入力を取得し、個々の
リソースで使用される{{<hover label="comp" line="37">}}location{{</hover>}}として使用します。

このCompositionをクラスターに適用します。

```yaml {label="comp",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: crossplane-quickstart-vm-with-network
spec:
  resources:
    - name: quickstart-vm
      base:
        apiVersion: compute.azure.upbound.io/v1beta1
        kind: LinuxVirtualMachine
        spec:
          forProvider:
            adminUsername: adminuser
            adminSshKey:
              - publicKey: ssh-rsa
                  AAAAB3NzaC1yc2EAAAADAQABAAABAQC+wWK73dCr+jgQOAxNsHAnNNNMEMWOHYEccp6wJm2gotpr9katuF/ZAdou5AaW1C61slRkHRkpRRX9FA9CYBiitZgvCCz+3nWNN7l/Up54Zps/pHWGZLHNJZRYyAB6j5yVLMVHIHriY49d/GZTZVNB8GoJv9Gakwc/fuEZYYl4YDFiGMBP///TzlI4jhiJzjKnEvqPFki5p2ZRJqcbCiF4pJrxUQR/RXqVFQdbRLZgYfJ8xGB878RENq3yQ39d8dVOkq4edbkzwcUmwwwkYVPIoDGsYLaRHnG+To7FvMeyO7xDVQkMKzopTQV8AuKpyvpqu0a9pWOMaiCyDytO7GGN
                  example@docs.crossplane.io
                username: adminuser
            location: "Central US"
            osDisk:
              - caching: ReadWrite
                storageAccountType: Standard_LRS
            resourceGroupNameSelector:
              matchControllerRef: true
            size: Standard_B1ms
            sourceImageReference:
              - offer: debian-11
                publisher: Debian
                sku: 11-backports-gen2
                version: latest
            networkInterfaceIdsSelector:
              matchControllerRef: true
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.location"
          transforms:
            - type: map
              map: 
                EU: "Sweden Central"
                US: "Central US"
    - name: quickstart-nic
      base:
        apiVersion: network.azure.upbound.io/v1beta1
        kind: NetworkInterface
        spec:
          forProvider:
            ipConfiguration:
              - name: crossplane-quickstart-configuration
                privateIpAddressAllocation: Dynamic
                subnetIdSelector:
                  matchControllerRef: true
            location: "Central US"
            resourceGroupNameSelector:
              matchControllerRef: true
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.location"
          transforms:
            - type: map
              map: 
                EU: "Sweden Central"
                US: "Central US"            
    - name: quickstart-subnet
      base:
        apiVersion: network.azure.upbound.io/v1beta1
        kind: Subnet
        spec:
          forProvider:
            addressPrefixes:
              - 10.0.1.0/24
            virtualNetworkNameSelector:
              matchControllerRef: true
            resourceGroupNameSelector:
              matchControllerRef: true
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.location"
          transforms:
            - type: map
              map: 
                EU: "Sweden Central"
                US: "Central US"
    - name: quickstart-network
      base:
        apiVersion: network.azure.upbound.io/v1beta1
        kind: VirtualNetwork
        spec:
          forProvider:
            addressSpace:
              - 10.0.0.0/16
            location: "Central US"
            resourceGroupNameSelector:
              matchControllerRef: true
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.location"
          transforms:
            - type: map
              map: 
                EU: "Sweden Central"
                US: "Central US"
    - name: crossplane-resourcegroup
      base:
        apiVersion: azure.upbound.io/v1beta1
        kind: ResourceGroup
        spec:
          forProvider:
            location: Central US
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.location"
          toFieldPath: "spec.forProvider.location"
          transforms:
            - type: map
              map: 
                EU: "Sweden Central"
                US: "Central US"
  compositeTypeRef:
    apiVersion: compute.example.com/v1alpha1
    kind: VirtualMachine
EOF
```

{{<hover label="comp" line="52">}}compositeTypeRef{{</hover >}} は、
このテンプレートを使用してリソースを作成できるカスタムAPIを定義します。

{{<hint "tip" >}}
[Compositionのドキュメント]({{<ref "../concepts/compositions">}})を読んで、
Compositionの構成や利用可能なオプションについての詳細を確認してください。

[Patch and Transformのドキュメント]({{<ref "../concepts/patch-and-transform">}})を読んで、
Crossplaneがユーザーの入力をCompositionリソーステンプレートにマッピングするために
patchesをどのように使用するかについての詳細を確認してください。
{{< /hint >}}


`kubectl get composition`を使用してCompositionを表示します。

```shell {copy-lines="1"}
kubectl get composition
NAME                                    XR-KIND           XR-APIVERSION                     AGE
crossplane-quickstart-vm-with-network   XVirtualMachine   custom-api.example.org/v1alpha1   77s
```

## Azure仮想マシンプロバイダーのインストール

パート1ではAzure Virtual Network Providerのみがインストールされました。仮想マシンをデプロイするには、Azure Computeプロバイダーも必要です。

新しいプロバイダーをクラスターに追加します。

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azure-compute
spec:
  package: xpkg.upbound.io/upbound/provider-azure-compute:v0.42.1
EOF
```

`kubectl get providers`を使用して新しいComputeプロバイダーを表示します。

```shell {copy-lines="1"}
kubectl get providers
NAME                            INSTALLED   HEALTHY   PACKAGE                                                  AGE
provider-azure-compute          True        True      xpkg.upbound.io/upbound/provider-azure-compute:v0.42.1   25s
provider-azure-network          True        True      xpkg.upbound.io/upbound/provider-azure-network:v0.42.1   3h
upbound-provider-family-azure   True        True      xpkg.upbound.io/upbound/provider-family-azure:v0.42.1    3h
```

## カスタムAPIにアクセスする

カスタムAPI（XRD）がインストールされ、リソーステンプレート（Composition）に関連付けられると、ユーザーはAPIにアクセスしてリソースを作成できます。

{{<hover label="xr" line="3">}}VirtualMachine{{</hover>}}オブジェクトを作成して、クラウドリソースを作成します。

```yaml {copy-lines="all",label="xr"}
cat <<EOF | kubectl apply -f -
apiVersion: compute.example.com/v1alpha1
kind: VirtualMachine
metadata:
  name: my-vm
spec: 
  location: "EU"
EOF
```

`kubectl get VirtualMachine`を使用してリソースを表示します。

{{< hint "note" >}}
リソースのプロビジョニングには最大5分かかる場合があります。
{{< /hint >}}

```shell {copy-lines="1"}
kubectl get VirtualMachine
NAME    SYNCED   READY   COMPOSITION                             AGE
my-vm   True     True    crossplane-quickstart-vm-with-network   3m3s
```

このオブジェクトはCrossplaneの_複合リソース_（`XR`とも呼ばれます）です。  
これは、Compositionテンプレートから作成されたリソースのコレクションを表す単一のオブジェクトです。

`kubectl get managed`を使用して個々のリソースを表示します。

```shell {copy-lines="1"}
kubectl get managed
NAME                                         READY   SYNCED   EXTERNAL-NAME   AGE
resourcegroup.azure.upbound.io/my-vm-7jb4n   True    True     my-vm-7jb4n     3m43s

NAME                                                       READY   SYNCED   EXTERNAL-NAME   AGE
linuxvirtualmachine.compute.azure.upbound.io/my-vm-5h7p4   True    True     my-vm-5h7p4     3m43s

NAME                                                    READY   SYNCED   EXTERNAL-NAME   AGE
networkinterface.network.azure.upbound.io/my-vm-j7fpx   True    True     my-vm-j7fpx     3m43s

NAME                                          READY   SYNCED   EXTERNAL-NAME   AGE
subnet.network.azure.upbound.io/my-vm-b2dqt   True    True     my-vm-b2dqt     3m43s

NAME                                                  READY   SYNCED   EXTERNAL-NAME   AGE
virtualnetwork.network.azure.upbound.io/my-vm-pd2sw   True    True     my-vm-pd2sw     3m43s
```

APIにアクセスすることで、テンプレートで定義された5つのリソースがすべて作成され、それらがリンクされました。

特定のリソースを見て、APIで使用された場所に作成されていることを確認します。

```yaml {copy-lines="1"}
kubectl describe linuxvirtualmachine | grep Location
    Location:                         Sweden Central
    Location:                         swedencentral
```

`kubectl delete VirtualMachine`を使用してリソースを削除します。

```shell {copy-lines="1"}
kubectl delete VirtualMachine my-vm
virtualmachine.compute.example.com "my-vm" が削除されました
```

Crossplaneがリソースを削除したことを`kubectl get managed`で確認します。

{{<hint "note" >}}
リソースの削除には最大で5分かかる場合があります。
{{< /hint >}}

```shell {copy-lines="1"}
kubectl get managed
リソースは見つかりませんでした
```

## 名前空間を使用したAPIの利用

API `VirtualMachine`へのアクセスはクラスターのスコープで行われます。  
ほとんどの組織は
ユーザーを名前空間に隔離します。  

Crossplaneの_Claim_は、名前空間内のカスタムAPIです。

_Claim_を作成することは、カスタムAPIエンドポイントにアクセスするのと同じですが、
{{<hover label="claim" line="3">}}kind{{</hover>}} 
はカスタムAPIの`claimNames`から取得します。

Claimを作成するために新しい名前空間を作成します。

```shell
kubectl create namespace crossplane-test
```

次に、`crossplane-test`名前空間にClaimを作成します。

```yaml {label="claim",copy-lines="all"}
cat <<EOF | kubectl apply -f -
apiVersion: compute.example.com/v1alpha1
kind: VirtualMachineClaim
metadata:
  name: my-namespaced-vm
  namespace: crossplane-test
spec: 
  location: "EU"
EOF
```
`kubectl get claim -n crossplane-test`でClaimを表示します。

```shell {copy-lines="1"}
kubectl get claim -n crossplane-test
NAME               SYNCED   READY   CONNECTION-SECRET   AGE
my-namespaced-vm   True     True                        5m11s
```

Claimは自動的に複合リソースを作成し、それが管理リソースを作成します。

`kubectl get composite`でCrossplaneが作成した複合リソースを表示します。

```shell {copy-lines="1"}
kubectl get composite
NAME                     SYNCED   READY   COMPOSITION                             AGE
my-namespaced-vm-r7gdr   True     True    crossplane-quickstart-vm-with-network   5m33s
```

再度、`kubectl get managed`で管理リソースを表示します。

```shell {copy-lines="1"}
NAME                                                          READY   SYNCED   EXTERNAL-NAME                  AGE
resourcegroup.azure.upbound.io/my-namespaced-vm-r7gdr-cvzw6   True    True     my-namespaced-vm-r7gdr-cvzw6   5m51s

NAME                                                                        READY   SYNCED   EXTERNAL-NAME                  AGE
linuxvirtualmachine.compute.azure.upbound.io/my-namespaced-vm-r7gdr-vrbgb   True    True     my-namespaced-vm-r7gdr-vrbgb   5m51s

NAME                                                                     READY   SYNCED   EXTERNAL-NAME                  AGE
networkinterface.network.azure.upbound.io/my-namespaced-vm-r7gdr-hwrb8   True    True     my-namespaced-vm-r7gdr-hwrb8   5m51s

NAME                                                           READY   SYNCED   EXTERNAL-NAME                  AGE
subnet.network.azure.upbound.io/my-namespaced-vm-r7gdr-gh468   True    True     my-namespaced-vm-r7gdr-gh468   5m51s

NAME                                                                   READY   SYNCED   EXTERNAL-NAME                  AGE
virtualnetwork.network.azure.upbound.io/my-namespaced-vm-r7gdr-5qhz7   True    True     my-namespaced-vm-r7gdr-5qhz7   5m51s
```

Claimを削除すると、すべてのCrossplane生成リソースが削除されます。

`kubectl delete claim -n crossplane-test my-VirtualMachine-database`

```shell {copy-lines="1"}
kubectl delete claim -n crossplane-test my-namespaced-vm
virtualmachineclaim.compute.example.com "my-namespaced-vm" が削除されました
```

{{<hint "note" >}}
リソースの削除には最大で5分かかる場合があります。
{{< /hint >}}

Crossplaneが複合リソースを削除したことを`kubectl get composite`で確認します。

```shell {copy-lines="1"}
kubectl get composite
No resources found
```

Crossplaneが管理リソースを削除したことを`kubectl get managed`で確認します。

```shell {copy-lines="1"}
kubectl get managed
No resources found
```

## 次のステップ
* Crossplaneが構成できるAzureリソースを[Provider CRDリファレンス](https://marketplace.upbound.io/providers/upbound/provider-family-azure/)で探ります。
* [Crossplane Slack](https://slack.crossplane.io/)に参加して、Crossplaneのユーザーや貢献者とつながります。
* [Crossplaneの概念]({{<ref "../concepts">}})についてもっと読み、Crossplaneでできることを見つけます。
