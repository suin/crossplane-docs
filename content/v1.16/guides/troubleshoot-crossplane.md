---
title: トラブルシュート Crossplane
weight: 306
---
## リクエストされたリソースが見つかりません

Crossplane CLIを使用して`Provider`または
`Configuration`（例：`crossplane install provider
xpkg.upbound.io/crossplane-contrib/provider-aws:v0.33.0`）をインストールし、`the server
could not find the requested resource`エラーが発生した場合、ほとんどの場合、それは使用しているCrossplane CLIが古いことを示しています。言い換えれば、いくつかのCrossplane APIがアルファからベータまたは安定版に移行しており、古いプラグインはこの変更を認識していません。

## リソースの状態と条件

ほとんどのCrossplaneリソースには、その特定のリソースの現在の状態を表す`status`セクションがあります。Crossplaneリソースに対して`kubectl describe`を実行すると、その条件に関する洞察に満ちた情報が得られることがよくあります。たとえば、GCPの`CloudSQLInstance`管理リソースの状態を確認するには、そのリソースに対して`kubectl describe`を使用します。

```shell {copy-lines="1"}
kubectl describe cloudsqlinstance my-db
Status:
  Conditions:
    Last Transition Time:  2019-09-16T13:46:42Z
    Reason:                Creating
    Status:                False
    Type:                  Ready
```

ほとんどのCrossplaneリソースは`Ready`条件を設定します。`Ready`はリソースの可用性を表します - それが作成中、削除中、利用可能、利用不可、バインディング中などであるかどうかです。

## リソースイベント

ほとんどのCrossplaneリソースは、何か興味深いことが起こると_イベント_を発生させます。リソースに関連するイベントは、`kubectl describe`を実行することで確認できます - 例：`kubectl describe cloudsqlinstance my-db`。特定の名前空間内のすべてのイベントを確認するには、`kubectl get events`を実行します。

```console
Events:
  Type     Reason                   Age                From                                                   Message
  ----     ------                   ----               ----                                                   -------
  Warning  CannotConnectToProvider  16s (x4 over 46s)  managed/postgresqlserver.database.azure.crossplane.io  cannot get referenced ProviderConfig: ProviderConfig.azure.crossplane.io "default" not found
```

> イベントは名前空間に依存しますが、多くのCrossplaneリソース（XRなど）はクラスター範囲です。Crossplaneはクラスター範囲のリソースに対してイベントを`default`名前空間に発生させます。

## Crossplaneログ

さらなる情報を得たり、障害を調査したりするために次に見るべき場所は、`crossplane-system`名前空間で実行されているCrossplaneポッドのログです。現在のCrossplaneログを取得するには、次のコマンドを実行します。

```shell
kubectl -n crossplane-system logs -lapp=crossplane
```

> Crossplaneはデフォルトで少ないログを発生させます - イベントは通常、Crossplaneが何をしているかに関する情報を探すのに最適な場所です。探している情報が見つからない場合は、`--debug`フラグを使用してCrossplaneを再起動する必要があるかもしれません。

## プロバイダーログ

Crossplaneの機能の多くはプロバイダーによって提供されることを忘れないでください。`kubectl logs`を使用してプロバイダーのログも表示できます。慣例として、デフォルトでいくつかのログも出力されます。

```shell
kubectl -n crossplane-system logs <name-of-provider-pod>
```

Crossplaneコミュニティによって維持されているすべてのプロバイダーは、Crossplaneの`--debug`フラグのサポートを反映しています。プロバイダーにフラグを設定する最も簡単な方法は、`ControllerConfig`を作成し、それを`Provider`から参照することです。

```yaml
apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: debug-config
spec:
  args:
    - --debug
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v0.33.0
  controllerConfigRef:
    name: debug-config
```

> `ControllerConfig`への参照は、すでにインストールされている`Provider`に追加でき、その`Deployment`が適切に更新されます。

## コンポジションと複合リソース定義

### 一般的なトラブルシューティング手順

Crossplaneとそのプロバイダーは、ほとんどのエラーメッセージをリソースのイベントフィールドにログします。Composite Resourcesがプロビジョニングされない場合は、以下の手順に従ってください。

1. `kubectl describe`または`kubectl get event`を使用して、ルートリソースのイベントを取得します。
2. イベントにエラーがある場合は、それに対処します。
3. エラーがない場合は、そのサブリソースを確認します。

    `kubectl get <KIND> <NAME> -o=jsonpath='{.spec.resourceRef}{" "}{.spec.resourceRefs}' | jq`
4. 返された各リソースについてこのプロセスを繰り返します。

{{< hint "note" >}}
このセクションの残りでは、外部ツールを使用せずにコンポジションに関連する問題をデバッグする方法を示します。
ArgoCDやFluxCDをUIで使用している場合、UIでオブジェクトの関係を視覚化できます。
また、kube-lineageプラグインを使用して、ターミナルでオブジェクトの関係を視覚化することもできます。
{{< /hint >}}

### 例

#### コンポジション
<!-- vale Google.WordList = NO --> 
クレームを使用してサンプルアプリケーションをデプロイしました。種類 = `ExampleApp`。名前 = `example-application`。

サンプルアプリケーションは、以下のように利用可能な状態に到達しません。

1. クレームを表示します。

    ```bash
    kubectl describe exampleapp example-application

    Status:
    Conditions:
        Last Transition Time:  2022-03-01T22:57:38Z
        Reason:                Composite resource claim is waiting for composite resource to become Ready
        Status:                False
        Type:                  Ready
    Events:                    <none>
    ```

2. クレームにエラーがない場合は、クレームの`.spec.resourceRef`フィールドを確認します。

    ```bash
    kubectl get exampleapp example-application -o=jsonpath='{.spec.resourceRef}{" "}{.spec.resourceRefs}' | jq

    {
      "apiVersion": "awsblueprints.io/v1alpha1",
      "kind": "XExampleApp",
      "name": "example-application-xqlsz"
    }
    ```
3. 前の出力では、このクレームのクラスター範囲のリソースが表示されます。種類 = `XExampleApp` 名前 = `example-application-xqlsz`
4. クラスター範囲のリソースのイベントを表示します。

```bash
    kubectl describe xexampleapp example-application-xqlsz

    Events:
    Type     Reason                   Age               From                                                             Message
    ----     ------                   ----              ----                                                             -------
    Normal   PublishConnectionSecret  9s (x2 over 10s)  defined/compositeresourcedefinition.apiextensions.crossplane.io  Successfully published connection details
    Normal   SelectComposition        6s (x6 over 11s)  defined/compositeresourcedefinition.apiextensions.crossplane.io  Successfully selected composition
    Warning  ComposeResources         6s (x6 over 10s)  defined/compositeresourcedefinition.apiextensions.crossplane.io  can't render composed resource from resource template at index 3: can't use dry-run create to name composed resource: an empty namespace may not be set during creation
    Normal   ComposeResources         6s (x6 over 10s)  defined/compositeresourcedefinition.apiextensions.crossplane.io  Successfully composed resources
    ```
5. イベントにエラーが表示されます。これは、構成で名前空間を指定していないことを不満に思っています。この特定の種類のエラーについては、そのサブリソースを取得し、どれが作成されていないかを確認できます。

    ```bash
    kubectl get xexampleapp example-application-xqlsz -o=jsonpath='{.spec.resourceRef}{" "}{.spec.resourceRefs}' | jq
    
    [
        {
            "apiVersion": "awsblueprints.io/v1alpha1",
            "kind": "XDynamoDBTable",
            "name": "example-application-xqlsz-6j9nm"
        },
        {
            "apiVersion": "awsblueprints.io/v1alpha1",
            "kind": "XIAMPolicy",
            "name": "example-application-xqlsz-lp9wt"
        },
        {
            "apiVersion": "awsblueprints.io/v1alpha1",
            "kind": "XIAMPolicy",
            "name": "example-application-xqlsz-btwkn"
        },
        {
            "apiVersion": "awsblueprints.io/v1alpha1",
            "kind": "IRSA"
        }
    ]
    ```
6. 配列の最後の要素に名前がないことに注意してください。構成内のリソースが検証に失敗すると、リソースオブジェクトは作成されず、名前を持ちません。この特定の問題については、IRSAリソースの名前空間を指定する必要があります。

#### 複合リソース定義

複合リソース定義（XRD）のデバッグは、構成のデバッグと似ています。

1. XRDを取得します。

    ```bash
    kubectl get xrd testing.awsblueprints.io

    NAME                       ESTABLISHED   OFFERED   AGE
    testing.awsblueprints.io                           66s
    ```
2. そのステータスが確立されていないことに注意してください。このXRDを記述して、そのイベントを取得します。

    ```bash
    kubectl describe xrd testing.awsblueprints.io

    Events:
    Type     Reason              Age                    From                                                             Message
    ----     ------              ----                   ----                                                             -------
    Normal   ApplyClusterRoles   3m19s (x3 over 3m19s)  rbac/compositeresourcedefinition.apiextensions.crossplane.io     Applied RBAC ClusterRoles
    Normal   RenderCRD           18s (x9 over 3m19s)    defined/compositeresourcedefinition.apiextensions.crossplane.io  Rendered composite resource CustomResourceDefinition
    Warning  EstablishComposite  18s (x9 over 3m19s)    defined/compositeresourcedefinition.apiextensions.crossplane.io  can't apply rendered composite resource CustomResourceDefinition: can't create object: CustomResourceDefinition.apiextensions.k8s.io "testing.awsblueprints.io" is invalid: metadata.name: Invalid value: "testing.awsblueprints.io": must be spec.names.plural+"."+spec.group
    ```
3. イベントで、CrossplaneがこのXRDに対応するCRDを生成できないことがわかります。この場合、名前が `spec.names.plural+"."+spec.group` であることを確認してください。

#### プロバイダー

プロバイダーをインストールするには、`configuration.pkg.crossplane.io` と `provider.pkg.crossplane.io` の2つの方法があります。どちらを使用しても、プロバイダー自体に機能的な違いはありません。
`configuration.pkg.crossplane.io` オブジェクトを定義すると、Crossplaneは `provider.pkg.crossplane.io` オブジェクトを作成し、それを管理します。Crossplaneパッケージに関する詳細は、[パッケージのドキュメント]({{<ref "/master/concepts/packages">}})を参照してください。

プロバイダーの問題が発生している場合、以下の手順は良い出発点です。

1. プロバイダーオブジェクトのステータスを確認します。
    ```bash
    kubectl describe provider.pkg.crossplane.io provider-aws

    Status:
        Conditions:
            Last Transition Time:  2022-08-04T16:19:44Z
            Reason:                HealthyPackageRevision
            Status:                True
            Type:                  Healthy
            Last Transition Time:  2022-08-04T16:14:29Z
            Reason:                ActivePackageRevision
            Status:                True
            Type:                  Installed
        Current Identifier:      crossplane/provider-aws:v0.29.0
        Current Revision:        provider-aws-a2e16ca2fc1a
    Events:
        Type    Reason                  Age                      From                                 Message
        ----    ------                  ----                     ----                                 -------
        Normal  InstallPackageRevision  9m49s (x237 over 4d17h)  packages/provider.pkg.crossplane.io  Successfully installed package revision
    ```
    上記の出力では、このプロバイダーが正常であることがわかります。このプロバイダーに関する詳細情報を取得するには、さらに掘り下げることができます。`Current Revision` フィールドは、次に見るべきオブジェクトを知らせてくれます。

2. プロバイダーオブジェクトを作成すると、CrossplaneはOCIイメージの内容に基づいて `ProviderRevision` オブジェクトを作成します。この例では、OCIイメージを `crossplane/provider-aws:v0.29.0` に指定しています。このイメージには、Deployment、ServiceAccount、CRDなどのKubernetesオブジェクトを定義するYAMLファイルが含まれています。
`ProviderRevision` オブジェクトは、YAMLファイルの内容に基づいてプロバイダーが機能するために必要なリソースを作成します。プロバイダーパッケージの一部としてデプロイされているものを調査するには、ProviderRevisionオブジェクトを調査します。上記の `Current Revision` フィールドは、このプロバイダーが使用しているProviderRevisionオブジェクトを示しています。

```bash
(kubectl get providerrevision provider-aws-a2e16ca2fc1a

    NAME                        HEALTHY   REVISION   IMAGE                             STATE    DEP-FOUND   DEP-INSTALLED   AGE
    provider-aws-a2e16ca2fc1a   True      1          crossplane/provider-aws:v0.29.0   Active                               19d
```

オブジェクトを説明すると、このオブジェクトによって管理されているすべてのCRDが見つかります。

```bash
(kubectl describe providerrevision provider-aws-a2e16ca2fc1a

    Status:
        Controller Ref:
            Name:  provider-aws-a2e16ca2fc1a
        Object Refs:
            API Version:  apiextensions.k8s.io/v1
            Kind:         CustomResourceDefinition
            Name:         natgateways.ec2.aws.crossplane.io
            UID:          5c36d1bc-61b8-44f8-bca0-47e368af87a9
            ....
    Events:
        Type    Reason             Age                    From                                         Message
        ----    ------             ----                   ----                                         -------
        Normal  SyncPackage        22m (x369 over 4d18h)  packages/providerrevision.pkg.crossplane.io  Successfully configured package revision
        Normal  BindClusterRole    15m (x348 over 4d18h)  rbac/providerrevision.pkg.crossplane.io      Bound system ClusterRole to provider ServiceAccount
        Normal  ApplyClusterRoles  15m (x364 over 4d18h)  rbac/providerrevision.pkg.crossplane.io      Applied RBAC ClusterRoles
```

イベントフィールドは、このプロセス中に発生した可能性のある問題も示します。
<!-- vale  Google.WordList = YES -->
3. 上記のイベントフィールドにエラーが表示されない場合は、Crossplaneがデプロイメントをプロビジョニングし、そのステータスを確認しているかどうかを確認してください。

```bash
(kubectl get deployment -n crossplane-system

    NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
    crossplane                  1/1     1            1           105d
    crossplane-rbac-manager     1/1     1            1           105d
    provider-aws-a2e16ca2fc1a   1/1     1            1           19d

    kubectl get pods -n crossplane-system

    NAME                                         READY   STATUS    RESTARTS   AGE
    crossplane-54db688c8d-qng6b                  2/2     Running   0          4d19h
    crossplane-rbac-manager-5776c9fbf4-wn5rj     1/1     Running   0          4d19h
    provider-aws-a2e16ca2fc1a-776769ccbd-4dqml   1/1     Running   0          4d23h
```
失敗しているポッドがある場合は、そのログを確認し、問題を解決してください。


## Crossplaneの一時停止

時々、バグに遭遇したときなど、リソースの管理を積極的に停止したい場合は、Crossplaneを一時停止することが有用です。すべてのリソースを削除せずにCrossplaneを一時停止するには、次のコマンドを実行してデプロイメントを単にスケールダウンします：

```bash
kubectl -n crossplane-system scale --replicas=0 deployment/crossplane
```

問題を修正したり、状況を整えたりできたら、デプロイメントを再度スケールアップすることでCrossplaneを再開できます：

```bash
kubectl -n crossplane-system scale --replicas=1 deployment/crossplane
```

## プロバイダーの一時停止

プロバイダーも、問題をトラブルシューティングしたり、リソースの複雑な移行を調整したりする際に一時停止できます。`ControllerConfig`を作成して参照することが、プロバイダーをスケールダウンする最も簡単な方法であり、`ControllerConfig`を変更するか、参照を削除することで再度スケールアップできます：

```yaml
apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: scale-config
spec:
  replicas: 0
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v0.33.0
  controllerConfigRef:
    name: scale-config
```

> すでにインストールされている`Provider`に`ControllerConfig`への参照を追加すると、それに応じて`Deployment`が更新されることに注意してください。

## リソースがハングしたときの削除

Crossplaneが管理するリソースは、自動的にクリーンアップされ、何も残らないようにします。これはファイナライザーを使用して実現されますが、特定のシナリオではファイナライザーがKubernetesオブジェクトの削除を妨げることがあります。

これに対処するために、基本的にはオブジェクトのファイナライザーを削除するためにパッチを適用したいと考えています。これにより、オブジェクトは完全に削除されることができます。ただし、これによりCrossplaneが管理していた外部リソースが必ずしも削除されるわけではないため、クラウドプロバイダーのコンソールに移動して、残っているリソースをクリーンアップする必要があります。

一般的に、ファイナライザーは次のコマンドでオブジェクトから削除できます：

```shell
kubectl patch <resource-type> <resource-name> -p '{"metadata":{"finalizers": []}}' --type=merge
```

たとえば、`my-db`という名前の`CloudSQLInstance`管理リソース（`database.gcp.crossplane.io`）のファイナライザーを削除するには、次のようにします：

```shell
kubectl patch cloudsqlinstance my-db -p '{"metadata":{"finalizers": []}}' --type=merge
```

## ヒント、コツ、およびトラブルシューティング

このセクションでは、Composite Resourcesを操作する際の一般的なヒント、コツ、およびトラブルシューティング手順について説明します。Composite Resourcesが機能しない理由を追跡しようとしている場合は、[トラブルシューティング][trouble-ref]ページにも役立つ情報があります。

### クレームとXRのトラブルシューティング

Crossplaneはトラブルシューティングのためにステータス条件とイベントに大きく依存しています。
これらは`kubectl describe`を使用して確認できます。たとえば：

```console
# my-dbという名前のPostgreSQLInstanceクレームを説明する
kubectl describe postgresqlinstance.database.example.org my-db
```

Kubernetesの慣例に従い、Crossplaneはエラーを発生した場所に近くに保持します。これは、`Composition`や構成リソースに問題があるためにクレームが準備完了にならない場合、なぜそうなっているのかを知るために「参照をたどる」必要があることを意味します。クレームはXRがまだ準備完了でないことだけを教えてくれます。

参照をたどるには：

1. クレームで`kubectl describe`を実行し、その「Resource Ref」（別名`spec.resourceRef`）を探してXRを見つけます。
1. XRで`kubectl describe`を実行します。ここで、使用している`Composition`に関する問題があるかどうかがわかります。
1. 問題がない場合でもXRが準備完了にならないようであれば、「Resource Refs」（または`spec.resourceRefs`）を探して構成リソースを見つけます。
1. 各参照された構成リソースで`kubectl describe`を実行して、それが準備完了かどうか、または問題があるかどうかを確認します。




<!-- Named Links -->
[Requested Resource Not Found]: #requested-resource-not-found
[install Crossplane CLI]: "../getting-started/install-configure"
[Resource Status and Conditions]: #resource-status-and-conditions
[Resource Events]: #resource-events
[Crossplane Logs]: #crossplane-logs
[Provider Logs]: #provider-logs
[Pausing Crossplane]: #pausing-crossplane
[Pausing Providers]: #pausing-providers
[Deleting When a Resource Hangs]: #deleting-when-a-resource-hangs
[Installing Crossplane Package]: #installing-crossplane-package
[Crossplane package]: /master/concepts/packages/
[Handling Crossplane Package Dependency]: #handling-crossplane-package-dependency
[semver spec]: https://github.com/Masterminds/semver#basic-comparisons

It seems that there is no content provided for translation. Please paste the Markdown content you would like me to translate into Japanese.
