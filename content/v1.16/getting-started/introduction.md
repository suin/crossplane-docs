---
title: Crossplaneの紹介
weight: 2
---

Crossplaneは、Kubernetesクラスターを外部の非Kubernetesリソースに接続し、プラットフォームチームがそれらのリソースを利用するためのカスタムKubernetes APIを構築できるようにします。

<!-- vale gitlab.SentenceLength = NO -->
Crossplaneは、外部リソースをネイティブな
[Kubernetesオブジェクト](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)として表現するために、Kubernetes
[カスタムリソース定義](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
（`CRDs`）を作成します。 
ネイティブなKubernetesオブジェクトとして、`kubectl create`
や`kubectl describe`のような標準コマンドを使用できます。完全な
[Kubernetes API](https://kubernetes.io/docs/reference/using-api/)が
すべてのCrossplaneリソースに対して利用可能です。 
<!-- vale gitlab.SentenceLength = YES -->

Crossplaneはまた、外部リソースの状態を監視し、状態の強制を提供する
[Kubernetesコントローラー](https://kubernetes.io/docs/concepts/architecture/controller/)としても機能します。もし
Kubernetesの外部でリソースが変更または削除されると、Crossplaneは
その変更を元に戻すか、削除されたリソースを再作成します。

{{<img src="/media/crossplane-intro-diagram.png" alt="ユーザーがKubernetesと通信している図。CrossplaneがKubernetesに接続され、CrossplaneがAWS、Azure、GCPと通信している" align="center">}}
KubernetesクラスターにCrossplaneがインストールされると、ユーザーはKubernetesとだけ通信します。Crossplaneは、AWS、
Azure、またはGoogle Cloudのような外部リソースとの通信を管理します。

Crossplaneは、カスタムKubernetes APIの作成も可能にします。プラットフォームチームは
外部リソースを組み合わせ、プラットフォームの消費者に提示されるAPIを簡素化またはカスタマイズできます。

## Crossplaneコンポーネントの概要
この表は、Crossplaneコンポーネントとその役割の概要を提供します。

{{< table "table table-hover table-sm">}}
| コンポーネント | 略称 | スコープ | 概要 |
| --- | --- | --- | ---- | 
| [プロバイダー]({{<ref "#providers">}}) | | クラスター | 外部サービスのための新しいKubernetesカスタムリソース定義を作成します。 |
| [ProviderConfig]({{<ref "#provider-configurations">}}) | `PC` | クラスター | _プロバイダー_の設定を適用します。 |
| [管理リソース]({{<ref "#managed-resources">}}) | `MR` | クラスター | Kubernetesクラスター内でCrossplaneによって作成および管理されるプロバイダーリソース。 | 
| [Composition]({{<ref "#compositions">}}) |  | クラスター | 複数の_管理リソース_を一度に作成するためのテンプレート。 |
| [Composite Resources]({{<ref "#composite-resources" >}}) | `XR` | クラスター | _Composition_テンプレートを使用して、複数の_管理リソース_を単一のKubernetesオブジェクトとして作成します。 |
| [CompositeResourceDefinitions]({{<ref "#composite-resource-definitions" >}}) | `XRD` | クラスター | _Composite Resources_および_Claims_のAPIスキーマを定義します。 |
| [Claims]({{<ref "#claims" >}}) | `XC` | 名前空間 | _Composite Resource_のようですが、名前空間スコープです。 | 
{{< /table >}}

## Crossplane Pod
Kubernetes クラスターにインストールされると、Crossplane はコア Crossplane コンポーネントの初期セットのカスタムリソース定義（`CRDs`）を作成します。

{{< expand "初期 Crossplane CRDs を表示" >}}
Crossplane をインストールした後、`kubectl get crds` を使用してインストールされた Crossplane CRDs を表示します。

```shell
❯ kubectl get crd
NAME                                                    
compositeresourcedefinitions.apiextensions.crossplane.io
compositionrevisions.apiextensions.crossplane.io        
compositions.apiextensions.crossplane.io                
configurationrevisions.pkg.crossplane.io                
configurations.pkg.crossplane.io                        
controllerconfigs.pkg.crossplane.io                     
deploymentruntimeconfigs.pkg.crossplane.io              
environmentconfigs.apiextensions.crossplane.io          
functionrevisions.pkg.crossplane.io                     
functions.pkg.crossplane.io                             
locks.pkg.crossplane.io                                 
providerrevisions.pkg.crossplane.io                     
providers.pkg.crossplane.io                             
storeconfigs.secrets.crossplane.io                      
usages.apiextensions.crossplane.io                                        
```
{{< /expand >}}

以下のセクションでは、これらの CRDs のいくつかの機能について説明します。

<!-- vale Google.Headings = NO -->
<!-- allow "Providers" -->
## プロバイダー
<!-- vale Google.Headings = YES -->
Crossplane _プロバイダー_ は、Crossplane が非 Kubernetes サービスに接続する方法を定義する第二の CRD セットを作成します。各外部サービスは独自のプロバイダーに依存しています。たとえば、 
[AWS](https://marketplace.upbound.io/providers/upbound/provider-aws)、 
[Azure](https://marketplace.upbound.io/providers/upbound/provider-azure) 
および [GCP](https://marketplace.upbound.io/providers/upbound/provider-gcp)
は、それぞれのクラウドサービスの異なるプロバイダーです。

{{< hint "tip" >}}
ほとんどのプロバイダーはクラウドサービス用ですが、Crossplane は API を持つ任意のサービスに接続するためにプロバイダーを使用できます。
{{< /hint >}}

たとえば、AWS プロバイダーは、EC2 コンピュートインスタンスや S3 ストレージバケットなどの AWS リソースのための Kubernetes CRD を定義します。

プロバイダーは外部リソースの Kubernetes API 定義を定義します。たとえば、 
[Upbound Provider AWS](https://marketplace.upbound.io/providers/upbound/provider-aws/)
は、AWS S3 ストレージバケットを作成および管理するための 
[`bucket`](https://marketplace.upbound.io/providers/upbound/provider-aws/v0.25.0/resources/s3.aws.upbound.io/Bucket/v1beta1) 
リソースを定義します。

`bucket` CRD には、バケットをデプロイする AWS リージョンを定義する 
[`spec.forProvider.region`](https://marketplace.upbound.io/providers/upbound/provider-aws/v0.25.0/resources/s3.aws.upbound.io/Bucket/v1beta1#doc:spec-forProvider-region)
値があります。

Upbound Marketplace には、大規模な 
[Crossplane プロバイダーのコレクション](https://marketplace.upbound.io/providers) が含まれています。

さらに多くのプロバイダーは [Crossplane Contrib リポジトリ](https://github.com/crossplane-contrib/) で入手できます。


プロバイダーはクラスター範囲であり、すべてのクラスター名前空間で利用可能です。

インストールされているすべてのプロバイダーを表示するには、コマンド `kubectl get providers` を使用します。

## プロバイダー設定
プロバイダーには _ProviderConfigs_ があります。 _ProviderConfigs_ は、認証やプロバイダーのグローバルデフォルトなど、プロバイダーに関連する設定を構成します。

ProviderConfigs の API エンドポイントは、各プロバイダーに固有です。

_ProviderConfigs_ はクラスター範囲であり、すべてのクラスター名前空間で利用可能です。

インストールされているすべての ProviderConfigs を表示するには、コマンド `kubectl get providerconfig` を使用します。

## 管理リソース
プロバイダーの CRD は、プロバイダー内の個々の _resources_ にマッピングされます。Crossplane がリソースを作成および監視する際、それは _Managed Resource_ です。

プロバイダーの CRD を使用すると、一意の _Managed Resource_ が作成されます。たとえば、プロバイダー AWS の `bucket` CRD を使用すると、Crossplane は AWS S3 ストレージバケットに接続された Kubernetes クラスター内に `bucket` _Managed Resource_ を作成します。

Crossplane コントローラーは _Managed Resources_ の状態を強制します。Crossplane は _Managed Resources_ の設定と存在を強制します。この「コントローラーパターン」は、Kubernetes の 
[kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)
がポッドの状態を強制する方法に似ています。

_Managed Resources_ はクラスター範囲であり、すべてのクラスター名前空間で利用可能です。

`kubectl get managed` を使用して、すべての _managed resources_ を表示します。
{{<hint "warning" >}}
`kubectl get managed` は多くの Kubernetes API クエリを生成します。
`kubectl` クライアントと kube-apiserver の両方が API クエリを制限します。

API サーバーのサイズと管理リソースの数によっては、このコマンドは返答に数分かかる場合やタイムアウトする場合があります。

詳細については、 
[Kubernetes issue #111880](https://github.com/kubernetes/kubernetes/issues/111880)
および 
[Crossplane issue #3459](https://github.com/crossplane/crossplane/issues/3459) をお読みください。
{{< /hint >}}

## コンポジション

_Composition_ は、_managed resource_ のコレクションのテンプレートです。 _Compositions_ は、プラットフォームチームが一連の _managed resources_ を単一のオブジェクトとして定義できるようにします。

例えば、コンピュート _管理リソース_ は、ストレージリソースと仮想ネットワークの作成を必要とする場合があります。単一の _Composition_ は、単一の _Composition_ オブジェクト内でこれら3つのリソースを定義できます。

_コンポジション_ を使用すると、複数の _管理リソース_ で構成されるインフラストラクチャのデプロイが簡素化されます。_コンポジション_ は、デプロイメント全体で標準と設定を強制します。

プラットフォームチームは、_Composition_ 内の各 _管理リソース_ に対して固定またはデフォルトの設定を定義するか、ユーザーが変更できるフィールドと設定を定義できます。

前述の例を使用すると、プラットフォームチームはコンピュートリソースのサイズと仮想ネットワークの設定を設定できます。しかし、プラットフォームチームはユーザーがストレージリソースのサイズを定義することを許可します。

_コンポジション_ を作成すると、Crossplane は管理リソースを作成しません。_コンポジション_ は、_管理リソース_ とその設定のコレクションのテンプレートに過ぎません。_コンポジットリソース_ が特定のリソースを作成します。

{{< hint "note" >}}
[_コンポジットリソース_]({{<ref "#composite-resources">}}) セクションでは
_コンポジットリソース_ について説明します。
{{< /hint >}}

_コンポジション_ はクラスター範囲であり、すべてのクラスター名前空間で利用可能です。

`kubectl get compositions` を使用してすべての _コンポジション_ を表示します。

## コンポジットリソース

_コンポジットリソース_ (`XR`) は、プロビジョニングされた _管理リソース_ のセットです。_コンポジットリソース_ は、_コンポジション_ によって定義されたテンプレートを使用し、ユーザーが定義した設定を適用します。

複数のユニークな _コンポジットリソース_ オブジェクトが同じ _コンポジション_ を使用できます。たとえば、_コンポジション_ テンプレートは、コンピュート、ストレージ、およびネットワーキングのセットの _管理リソース_ を作成できます。Crossplane は、ユーザーがこのリソースセットを要求するたびに同じ _コンポジション_ テンプレートを使用します。

_コンポジション_ がユーザーにリソース設定を定義することを許可する場合、ユーザーはそれらを _コンポジットリソース_ に適用します。

<!-- _コンポジション_ は、どの _コンポジットリソース_ が _コンポジション_ テンプレートを使用できるかを定義します。これは、_コンポジション_ の `spec.compositeTypeRef` 値によって定義されます。これにより、使用できる _コンポジットリソース_ の {{<hover label="comp" line="7">}}apiVersion{{< /hover >}} と {{<hover label="comp" line="8">}}kind{{< /hover >}} が定義されます。 -->

例えば、_Composition_ では：
```yaml {label="comp"}
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: test.example.org
spec:
  compositeTypeRef:
    apiVersion: test.example.org/v1alpha1
    kind: myComputeResource
    # Removed for brevity
```

このテンプレートを使用できる _Composite Resource_ は、次の 
{{<hover label="comp" line="7">}}apiVersion{{< /hover >}} と {{<hover
label="comp" line="8">}}kind{{< /hover >}} に一致する必要があります。

```yaml {label="xr"}
apiVersion: test.example.org/v1alpha1
kind: myComputeResource
metadata:
  name: myResource
spec:
  storage: "large"
```

_Composite Resource_ {{<hover label="xr" line="1">}}apiVersion{{< /hover >}}
は、_Composition_ 
{{<hover label="comp" line="7">}}apiVersion{{</hover >}} に一致し、 
_Composite Resource_  {{<hover label="xr" line="2">}}kind{{< /hover >}}
は _Composition_ {{<hover label="comp" line="8">}}kind{{< /hover >}} に一致します。

この例では、_Composite Resource_ も 
{{<hover label="xr" line="7">}}storage{{< /hover >}} 設定を設定します。 
_Composition_ は、この値を使用して、この _Composite Resource_ によって所有される関連する _managed resources_ を作成します。 -->

{{< hint "tip" >}}
_Compositions_ は一連の _managed resources_ のテンプレートです。  
_Composite Resources_ はテンプレートを埋めて _managed resources_ を作成します。

_Composite Resource_ を削除すると、それが作成したすべての _managed resources_ が削除されます。
{{< /hint >}}

_Composite Resources_ はクラスター範囲であり、すべてのクラスター名前空間で利用可能です。

`kubectl get composite` を使用して、すべての _Composite Resources_ を表示します。

## Composite Resource Definitions
_Composite Resource Definitions_ (`XRDs`) は、_Claims_ と _Composite Resources_ によって使用されるカスタム Kubernetes API を作成します。

{{< hint "note" >}}
[_Claims_]({{<ref "#claims">}}) セクションでは
_Claims_ について説明しています。
{{< /hint >}}

プラットフォームチームがカスタム API を定義します。  
これらの API は、ギガバイト単位のストレージスペースのような特定の値や、`small` や `large` のような一般的な設定、`cloud` や `onprem` のようなデプロイメントオプションを定義できます。Crossplane は API 定義を制限しません。

_Composite Resource Definition_ の `kind` は Crossplane から来ています。
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
```

_Composite Resource Definition_ の `spec` は、_Composite Resource_ の `apiVersion`、`kind` および `spec` を作成します。

{{< hint "tip" >}}
_コンポジットリソース定義_は、_コンポジットリソース_のパラメータを定義します。
{{< /hint >}}

_コンポジットリソース定義_には、4つの主要な`spec`パラメータがあります：
* _コンポジットリソース_の{{<hover label="specGroup" line="3" >}}group{{< /hover >}}を定義するための 
{{< hover label="xr2" line="2" >}}apiVersion{{</hover >}} 
* _コンポジットリソース_で使用されるバージョンを定義する{{< hover label="specGroup" line="7" >}}versions.name{{</hover >}} 
* _コンポジットリソース_の{{< hover label="specGroup" line="5" >}}names.kind{{</hover >}}を定義するための 
{{< hover label="xr2" line="3" >}}kind{{</hover>}} 
* _コンポジットリソース_の{{<hover label="xr2" line="6" >}}spec{{</hover >}}を定義するための{{< hover label="specGroup" line="8" >}}versions.schema{{</hover>}}セクション

```yaml {label="specGroup"}
# Composite Resource Definition (XRD)
spec:
  group: test.example.org
  names:
    kind: myComputeResource
  versions:
  - name: v1alpha1
    schema:
      # Removed for brevity
```

この_コンポジットリソース定義_に基づく_コンポジットリソース_は次のようになります：

```yaml {label="xr2"}
# Composite Resource (XR)
apiVersion: test.example.org/v1alpha1
kind: myComputeResource
metadata:
  name: myResource
spec:
  storage: "large"
```

_コンポジットリソース定義_の{{< hover label="specGroup" line="8" >}}schema{{</hover >}}は、_コンポジットリソース_の
{{<hover label="xr2" line="6" >}}spec{{</hover >}}パラメータを定義します。

これらのパラメータは、開発者が使用できる新しいカスタムAPIです。

たとえば、コンピュート_マネージドリソース_を作成するには、AWSの`m6in.large`やGCPの`e2-standard-2`のようなクラウドプロバイダーのコンピュートクラス名の知識が必要です。

_コンポジットリソース定義_は、選択肢を`small`または`large`に制限できます。
_コンポジットリソース_はそれらのオプションを使用し、_コンポジション_はそれらを特定のクラウドプロバイダー設定にマッピングします。

次の_コンポジットリソース定義_は、{{<hover label="specVersions" line="17" >}}storage{{< /hover >}}パラメータを定義します。ストレージは
{{<hover label="specVersions" line="18">}}string{{< /hover >}}であり、OpenAPIの
{{<hover label="specVersions" line="19" >}}oneOf{{< /hover >}}は、オプションが{{<hover label="specVersions" line="20" >}}small{{< /hover >}}または{{<hover label="specVersions" line="21" >}}large{{< /hover >}}のいずれかであることを要求します。

```yaml {label="specVersions"}
# Composite Resource Definition (XRD)
spec:
  group: test.example.org
  names:
    kind: myComputeResource
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
              storage:
                type: string
                oneOf:
                  - pattern: '^small$'
                  - pattern: '^large$'
            required:
            - storage  
```

_コンポジットリソース定義_は、さまざまな設定やオプションを定義できます。

_コンポジットリソース定義_を作成することで、_コンポジットリソース_の作成が可能になりますが、_クレーム_の作成も可能です。

`spec.claimNames`を持つ_コンポジットリソース定義_は、開発者が_クレーム_を作成することを許可します。

たとえば、 
{{< hover label="xrdClaim" line="6" >}}claimNames.kind{{</hover >}}
は、`kind: computeClaim`の_クレーム_の作成を許可します。
```yaml {label="xrdClaim"}
# Composite Resource Definition (XRD)
spec:
  group: test.example.org
  names:
    kind: myComputeResource
  claimNames:
    kind: computeClaim
  # Removed for brevity 
```

## クレーム
_クレーム_は、開発者がCrossplaneと対話する主な方法です。

_クレーム_は、プラットフォームチームが_コンポジットリソース定義_で定義したカスタムAPIにアクセスします。

_クレーム_は_コンポジットリソース_のように見えますが、名前空間スコープであり、_コンポジットリソース_はクラスター全体のスコープです。

{{< hint "note" >}}
**名前空間スコープが重要な理由は何ですか？**  
名前空間スコープの_クレーム_を持つことで、ユニークな名前空間を使用する複数のチームが、互いに独立して同じ種類のリソースを作成できます。チームAの計算リソースは、チームBの計算リソースとは異なります。

_コンポジットリソース_を直接作成するには、すべてのチームと共有されるクラスター全体の権限が必要です。  
_クレーム_は同じセットのリソースを作成しますが、名前空間レベルで作成されます。
{{< /hint >}}

前の_コンポジットリソース定義_は、  
{{<hover label="xrdClaim2" line="7" >}}computeClaim{{</hover>}}の種類の_クレーム_の作成を許可します。

クレームは、_コンポジットリソース定義_で定義され、_コンポジットリソース_でも使用される同じ 
{{< hover label="xrdClaim2" line="3" >}}apiVersion{{< /hover >}}を使用します。
```yaml {label="xrdClaim2"}
# Composite Resource Definition (XRD)
spec:
  group: test.example.org
  names:
    kind: myComputeResource
  claimNames:
    kind: computeClaim
  # Removed for brevity 
```

例の_クレーム_では、 
{{<hover label="claim" line="2">}}apiVersion{{< /hover >}}
が_コンポジットリソース定義_の{{<hover label="xrdClaim2" line="3">}}group{{< /hover >}}と一致します。

_クレーム_の{{<hover label="claim" line="3">}}kind{{< /hover >}}は、_コンポジットリソース定義_の 
{{<hover label="xrdClaim2" line="7">}}claimNames.kind{{< /hover >}}と一致します。

```yaml {label="claim"}
# Claim
apiVersion: test.example.org/v1alpha1
kind: computeClaim
metadata:
  name: myClaim
  namespace: devGroup
spec:
  size: "large"
```

_Claim_ は {{<hover label="claim" line="6">}}namespace{{</hover >}} にインストールできます。  
_Composite Resource Definition_ は 
{{<hover label="claim" line="7">}}spec{{< /hover >}} オプションを _Composite Resource_ 
{{<hover label="xr-claim" line="6">}}spec{{< /hover >}} と同じように定義します。

{{< hint "tip" >}}
_Composite Resources_ と _Claims_ は似ています。  
_Claims_ のみが 
{{<hover label="claim" line="6">}}namespace{{</hover >}} に存在できます。  
また、_Composite Resource_ の {{<hover label="xr-claim"
line="3">}}kind{{</hover >}} は _Claim_ の 
{{<hover label="claim" line="3">}}kind{{< /hover >}} と異なる場合があります。  
_Composite Resource Definition_ は 
{{<hover label="xrdClaim2" line="7">}}kind{{</hover >}} 値を定義します。
{{< /hint >}}

```yaml {label="xr-claim"}
# Composite Resource (XR)
apiVersion: test.example.org/v1alpha1
kind: myComputeResource
metadata:
  name: myResource
spec:
  storage: "large"
```

_Claims_ は名前空間スコープです。

コマンド `kubectl get claim` を使用して、利用可能なすべての Claims を表示します。

## 次のステップ
クイックスタートガイドのいずれかを使用して、自分自身の Crossplane プラットフォームを構築します。
* [Azure Quickstart]({{<ref "provider-azure" >}})
* [AWS Quickstart]({{<ref "provider-aws" >}})
* [GCP Quickstart]({{<ref "provider-gcp" >}})
