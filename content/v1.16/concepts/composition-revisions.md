---
title: コンポジションのリビジョン
weight: 35
---

このガイドでは、「コンポジションのリビジョン」を使用して、Crossplaneの[`Composition`][composition-type]に安全に変更を加えたり、変更を元に戻したりする方法について説明します。Crossplaneに関する基本的な知識、特に[Compositions]に精通していることを前提としています。

`Composition`は、CrossplaneがComposite Resource (XR)をどのように調整するかを設定します。言い換えれば、XRを作成するときに選択した`Composition`が、Crossplaneが応じて作成する管理リソースを決定します。例えば、Azure MySQL Serverといくつかのファイアウォールルールの組織共通のデータベース構成を表す`PlatformDB` XRを定義したとしましょう。この`Composition`には、MySQLサーバーとファイアウォールルールの「基本」構成が含まれており、`PlatformDB`の構成によって拡張されます。

`Composition`とそれを使用するXRとの間には一対多の関係があります。例えば、10個の異なる`PlatformDB` XRで使用される`big-platform-db`という名前の`Composition`を定義することができます。通常、セルフサービスの観点から、`Composition`は実際の`PlatformDB` XRとは異なるチームによって管理されます。例えば、`Composition`はプラットフォームチームのメンバーによって作成および維持され、個々のアプリケーションチームがその`Composition`を使用する`PlatformDB` XRを作成します。

各`Composition`は可変であり、組織のニーズの変化に応じて更新できます。しかし、コンポジションのリビジョンがない場合、`Composition`を更新することはリスクのあるプロセスになる可能性があります。Crossplaneは常に`Composition`を使用して、実際のインフラストラクチャ（MySQLサーバーやファイアウォールルール）が望ましい状態と一致するようにします。もし10個の`PlatformDB` XRがすべて`big-platform-db` `Composition`を使用している場合、`big-platform-db` `Composition`に対する更新に応じて、これら10個のXRはすぐに更新されます。

コンポジションのリビジョンを使用すると、XRは自動更新をオプトアウトできます。代わりに、最新の`Composition`設定を自分のペースで利用するためにXRを更新できます。これにより、インフラストラクチャに対する変更を[カナリア]として行ったり、一部のXRを以前の`Composition`設定に戻したりすることができ、すべてのXRを元に戻す必要がなくなります。

## コンポジションリビジョンの使用

コンポジションリビジョンを有効にすると、3つのことが発生します：

1. Crossplaneは各`Composition`の更新に対して`CompositionRevision`を作成します。
1. コンポジットリソースは、どの`CompositionRevision`を使用するかを指定する`spec.compositionRevisionRef`フィールドを持ちます。
1. コンポジットリソースは、新しいコンポジションリビジョンにどのように更新されるべきかを指定する`spec.compositionUpdatePolicy`フィールドを持ちます。

`Composition`を編集するたびに、Crossplaneはその`Composition`の「リビジョン」を表す`CompositionRevision`を自動的に作成します - それはユニークな状態です。各リビジョンには増加するリビジョン番号が割り当てられます。これにより、`CompositionRevision`の消費者はどのリビジョンが「最新」であるかを把握できます。

Crossplaneは`Composition`の「最新」と「現在」のリビジョンを区別します。つまり、既存の`CompositionRevision`に対応する以前の状態に`Composition`を戻すと、そのリビジョンは「現在」となりますが、「最新」のリビジョンではない場合があります（つまり、最も最近の_ユニーク_な`Composition`構成）。

`kubectl`を使用して、どのリビジョンが存在するかを確認できます：

```console
# 'example'という名前のコンポジションのすべてのリビジョンを見つける
kubectl get compositionrevision -l crossplane.io/composition-name=example
```

これにより、次のような出力が得られます：

```console
NAME            REVISION   CURRENT   AGE
example-18pdg   1          False     4m36s
example-2bgdr   2          True      73s
example-xjrdm   3          False     61s
```

> `Composition`は、ニーズが変化するにつれて更新できる可変リソースです。各`CompositionRevision`は、特定の時点でのそれらのニーズの不変のスナップショットです。

デフォルトでは、コンポジションリビジョンが有効かどうかにかかわらず、Crossplaneは同じように動作します。これは、コンポジションリビジョンを有効にすると、すべてのXRが`Automatic`な`compositionUpdatePolicy`にデフォルトで設定されるためです。XRは2つの更新ポリシーをサポートしています：

* `Automatic`：現在の`CompositionRevision`を自動的に使用します。（デフォルト）
* `Manual`：`CompositionRevision`を変更するために手動の介入が必要です。

以下のXRは`Manual`ポリシーを使用しています。このポリシーが使用されると、XRは最初に作成されたときに現在の`CompositionRevision`を選択しますが、別の`CompositionRevision`を使用する場合は手動で更新する必要があります。

```yaml
apiVersion: example.org/v1alpha1
kind: PlatformDB
metadata:
  name: example
spec:
  parameters:
    storageGB: 20
  # The Manual policy specifies that you do not want this XR to update to the
  # current CompositionRevision automatically.
  compositionUpdatePolicy: Manual
  compositionRef:
    name: example
  writeConnectionSecretToRef:
    name: db-conn
```

Crossplaneは、選択した`compositionUpdatePolicy`に関係なく、XRの`compositionRevisionRef`を作成時に自動的に設定します。`Manual`ポリシーを選択した場合、XRが異なる`CompositionRevision`を使用するようにしたいときは、`compositionRevisionRef`フィールドを編集する必要があります。

```yaml
apiVersion: example.org/v1alpha1
kind: PlatformDB
metadata:
  name: example
spec:
  parameters:
    storageGB: 20
  compositionUpdatePolicy: Manual
  compositionRef:
    name: example
  # Update the referenced CompositionRevision if and when you are ready.
  compositionRevisionRef:
    name: example-18pdg
  writeConnectionSecretToRef:
    name: db-conn
```

## 完全な例

このチュートリアルでは、CompositionRevisionsがどのように機能し、Composite Resource (XR) の更新をどのように管理するかについて説明します。これは、`MyVPC`リソースを定義する`Composition`と`CompositeResourceDefinition` (XRD) から始まり、異なるアップグレードパスを観察するために複数のXRを作成することに続きます。Crossplaneは、構成が更新されるたびに作成された複合リソースに異なるCompositionRevisionsを割り当てます。

### 準備 
##### Crossplaneのインストール
Crossplane v1.11.0以降をインストールし、Crossplaneポッドが実行されるまで待ちます。
```shell
kubectl create namespace crossplane-system
helm repo add crossplane-master https://charts.crossplane.io/master/
helm repo update
helm install crossplane --namespace crossplane-system crossplane-master/crossplane --devel --version 1.11.0-rc.0.108.g0521c32e
kubectl get pods -n crossplane-system
```
期待される出力:
```shell
NAME                                       READY   STATUS    RESTARTS   AGE
crossplane-7f75ddcc46-f4d2z                1/1     Running   0          9s
crossplane-rbac-manager-78bd597746-sdv6w   1/1     Running   0          9s
```

#### CompositionとXRDの例をデプロイ
例のCompositionを適用します。

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  labels:
    channel: dev
  name: myvpcs.aws.example.upbound.io
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  compositeTypeRef:
    apiVersion: aws.example.upbound.io/v1alpha1
    kind: MyVPC
  resources:
  - base:
      apiVersion: ec2.aws.upbound.io/v1beta1
      kind: VPC
      spec:
        forProvider:
          region: us-west-1
          cidrBlock: 192.168.0.0/16
          enableDnsSupport: true
          enableDnsHostnames: true
    name: my-vcp
```

例のXRDを適用します。
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: myvpcs.aws.example.upbound.io
spec:
  group: aws.example.upbound.io
  names:
    kind: MyVPC
    plural: myvpcs
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
              id:
                type: string 
                description: ID of this VPC that other objects will use to refer to it. 
            required:
            - id
```

CrossplaneがCompositionリビジョンを作成したことを確認します。
```shell
kubectl get compositionrevisions -o="custom-columns=NAME:.metadata.name,REVISION:.spec.revision,CHANNEL:.metadata.labels.channel"
```
期待される出力:
```shell
NAME                                    REVISION   CHANNEL
myvpcs.aws.example.upbound.io-ad265bc   1          dev
```

{{< hint "note" >}}
ラベル`dev`はCompositionから自動的に作成されます。
{{< /hint >}}


### 複合リソースの作成
このチュートリアルでは、異なる更新ポリシーと構成選択オプションをカバーするために4つの複合リソースがあります。デフォルトの動作は、XRをCompositionの最新リビジョンに更新することです。ただし、XRで`compositionUpdatePolicy: Manual`を設定することでこれを変更できます。また、`compositionUpdatePolicy: Automatic`とともに`compositionRevisionSelector.matchLabels`を使用して特定のラベルを持つ最新のリビジョンを選択することも可能です。

#### デフォルトの更新ポリシー
`compositionUpdatePolicy` が定義されていない XR を作成します。更新ポリシーはデフォルトで `Automatic` です：
```yaml
apiVersion: aws.example.upbound.io/v1alpha1
kind: MyVPC
metadata:
  name: vpc-auto
spec:
  id: vpc-auto
```
期待される出力：
```shell
myvpc.aws.example.upbound.io/vpc-auto created
``` 

#### 手動更新ポリシー
`compositionUpdatePolicy: Manual` と `compositionRevisionRef` を持つ Composite Resource を作成します。
```yaml
apiVersion: aws.example.upbound.io/v1alpha1
kind: MyVPC
metadata:
  name: vpc-man
spec:
  id: vpc-man
  compositionUpdatePolicy: Manual
  compositionRevisionRef:
    name: myvpcs.aws.example.upbound.io-ad265bc
```

期待される出力：
```shell
myvpc.aws.example.upbound.io/vpc-man created
``` 

#### セレクターを使用する
`channel: dev` の `compositionRevisionSelector` を持つ XR を作成します：
```yaml
apiVersion: aws.example.upbound.io/v1alpha1
kind:  MyVPC
metadata:
  name: vpc-dev
spec:
  id: vpc-dev
  compositionRevisionSelector:
    matchLabels:
      channel: dev
```
期待される出力：
```shell
myvpc.aws.example.upbound.io/vpc-dev created
``` 

`channel: staging` の `compositionRevisionSelector` を持つ XR を作成します：
```yaml
apiVersion: aws.example.upbound.io/v1alpha1
kind: MyVPC
metadata:
  name: vpc-staging
spec:
  id: vpc-staging
  compositionRevisionSelector:
    matchLabels:
      channel: staging
```

期待される出力：
```shell
myvpc.aws.example.upbound.io/vpc-staging created
``` 

ラベル `channel: staging` を持つ Composite Resource に `REVISION` がないことを確認します。  
他のすべての XR には、作成された Composition Revision に一致する `REVISION` があります。
```shell
kubectl get composite -o="custom-columns=NAME:.metadata.name,SYNCED:.status.conditions[0].status,REVISION:.spec.compositionRevisionRef.name,POLICY:.spec.compositionUpdatePolicy,MATCHLABEL:.spec.compositionRevisionSelector.matchLabels"
```
期待される出力：
```shell
NAME          SYNCED   REVISION                                POLICY      MATCHLABEL
vpc-auto      True     myvpcs.aws.example.upbound.io-ad265bc   Automatic   <none>
vpc-dev       True     myvpcs.aws.example.upbound.io-ad265bc   Automatic   map[channel:dev]
vpc-man       True     myvpcs.aws.example.upbound.io-ad265bc   Manual      <none>
vpc-staging   False    <none>                                  Automatic   map[channel:staging]
``` 

{{< hint "note" >}}
`vpc-staging` XR のラベルは、既存の Composition Revisions と一致しません。
{{< /hint >}}

### 新しい Composition revisions を作成する
Crossplane は、Composition が作成または更新されると新しい CompositionRevision を作成します。ラベルやアノテーションの変更も新しい CompositionRevision をトリガーします。

#### Composition ラベルを更新する
`Composition` ラベルを `channel: staging` に更新します：
```shell
kubectl label composition myvpcs.aws.example.upbound.io channel=staging --overwrite
```
期待される出力：
```shell
composition.apiextensions.crossplane.io/myvpcs.aws.example.upbound.io labeled
``` 

Crossplane が新しい Composition revision を作成することを確認します：
```shell
kubectl get compositionrevisions -o="custom-columns=NAME:.metadata.name,REVISION:.spec.revision,CHANNEL:.metadata.labels.channel"
```
期待される出力：
```shell
NAME                                    REVISION   CHANNEL
myvpcs.aws.example.upbound.io-727b3c8   2          staging
myvpcs.aws.example.upbound.io-ad265bc   1          dev
``` 

CrossplaneがCompositeリソース`vpc-auto`と`vpc-staging`をCompositeリビジョン:2に割り当てていることを確認します。  
XRs `vpc-man`と`vpc-dev`はまだ元のリビジョン:1に割り当てられています:

```shell
kubectl get composite -o="custom-columns=NAME:.metadata.name,SYNCED:.status.conditions[0].status,REVISION:.spec.compositionRevisionRef.name,POLICY:.spec.compositionUpdatePolicy,MATCHLABEL:.spec.compositionRevisionSelector.matchLabels"
```
期待される出力:
```shell
NAME          SYNCED   REVISION                                POLICY      MATCHLABEL
vpc-auto      True     myvpcs.aws.example.upbound.io-727b3c8   Automatic   <none>
vpc-dev       True     myvpcs.aws.example.upbound.io-ad265bc   Automatic   map[channel:dev]
vpc-man       True     myvpcs.aws.example.upbound.io-ad265bc   Manual      <none>
vpc-staging   True     myvpcs.aws.example.upbound.io-727b3c8   Automatic   map[channel:staging]
``` 

{{< hint "note" >}}
`vpc-auto`は常に最新のリビジョンを使用します。  
`vpc-staging`は現在、リビジョン:2に適用されたラベルと一致しています。
{{< /hint >}}

#### コンポジション仕様とラベルの更新
VPCでDNSサポートを無効にし、ラベルを`staging`から`dev`に戻すためにコンポジションを更新します。

以下の変更を適用して`Composition`仕様とラベルを更新します:
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  labels:
    channel: dev
  name: myvpcs.aws.example.upbound.io
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  compositeTypeRef:
    apiVersion: aws.example.upbound.io/v1alpha1
    kind: MyVPC
  resources:
  - base:
      apiVersion: ec2.aws.upbound.io/v1beta1
      kind: VPC
      spec:
        forProvider:
          region: us-west-1
          cidrBlock: 192.168.0.0/16
          enableDnsSupport: false
          enableDnsHostnames: true
    name: my-vcp
```

期待される出力:
```shell
composition.apiextensions.crossplane.io/myvpcs.aws.example.upbound.io configured
``` 

Crossplaneが新しいコンポジションリビジョンを作成することを確認します:

```shell
kubectl get compositionrevisions -o="custom-columns=NAME:.metadata.name,REVISION:.spec.revision,CHANNEL:.metadata.labels.channel"
```
期待される出力:
```shell
NAME                                    REVISION   CHANNEL
myvpcs.aws.example.upbound.io-727b3c8   2          staging
myvpcs.aws.example.upbound.io-ad265bc   1          dev
myvpcs.aws.example.upbound.io-f81c553   3          dev
``` 

{{< hint "note" >}}
ラベルと仕様の値を同時に変更することは、`dev`チャネルに新しい変更をデプロイするために重要です。
{{< /hint >}}

CrossplaneがCompositeリソース`vpc-auto`と`vpc-dev`をCompositeリビジョン:3に割り当てていることを確認します。  
`vpc-staging`はリビジョン:2に割り当てられ、`vpc-man`はまだ元のリビジョン:1に割り当てられています:

```shell
kubectl get composite -o="custom-columns=NAME:.metadata.name,SYNCED:.status.conditions[0].status,REVISION:.spec.compositionRevisionRef.name,POLICY:.spec.compositionUpdatePolicy,MATCHLABEL:.spec.compositionRevisionSelector.matchLabels"
```
期待される出力:
```shell
NAME          SYNCED   REVISION                                POLICY      MATCHLABEL
vpc-auto      True     myvpcs.aws.example.upbound.io-f81c553   Automatic   <none>
vpc-dev       True     myvpcs.aws.example.upbound.io-f81c553   Automatic   map[channel:dev]
vpc-man       True     myvpcs.aws.example.upbound.io-ad265bc   Manual      <none>
vpc-staging   True     myvpcs.aws.example.upbound.io-727b3c8   Automatic   map[channel:staging]
``` 


{{< hint "note" >}}
`vpc-dev`はリビジョン:3に適用された更新されたラベルと一致します。  
`vpc-staging`はリビジョン:2に適用されたラベルと一致します。
{{< /hint >}}

```
[composition-type]: {{<ref "../../master/concepts/compositions" >}}
[Compositions]: {{<ref "../../master/concepts/compositions" >}}
[canary]: https://martinfowler.com/bliki/CanaryRelease.html
[install-guide]: {{<ref "../../master/software/install" >}}
```
