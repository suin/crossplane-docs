---
title: "概要"
weight: -1
cascade:
    version: "1.16"
---

{{< img src="/media/banner.png" alt="Crossplane ポプシクルトラック" size="large" >}}

<br />

Crossplaneは、Kubernetesクラスターを**ユニバーサルコントロールプレーン**に変えるオープンソースのKubernetes拡張機能です。

Crossplaneを使用すると、標準のKubernetes APIを通じて、どこでも何でも管理できます。Crossplaneは、Kubernetesから直接
[ピザを注文する](https://blog.crossplane.io/providers-101-ordering-pizza-with-kubernetes-and-crossplane/)
ことさえ可能です。APIがあれば、Crossplaneはそれに接続できます。

Crossplaneを使用することで、プラットフォームチームはKubernetesポリシー、ネームスペース、ロールベースのアクセス制御などの完全な機能を活用して、新しい抽象化とカスタムAPIを作成できます。Crossplaneは、すべての非Kubernetesリソースを一つの屋根の下に集約します。

プラットフォームチームによって作成されたカスタムAPIは、リソースやクラウド全体でのセキュリティとコンプライアンスの強制を可能にし、開発者に複雑さを露呈することなく実現します。単一のAPI呼び出しで、複数のリソースを複数のクラウドで作成し、すべての制御プレーンとしてKubernetesを使用できます。

{{< hint "tip" >}}
**コントロールプレーンとは何ですか？**
<!-- vale Google.WordList = NO -->
コントロールプレーンは、リソースのライフサイクルを作成および管理します。コントロールプレーンは、意図したリソースが存在することを常に_確認_し、意図した状態が現実と一致しない場合に_報告_し、物事を正すために_行動_します。

Crossplaneは、Kubernetesコントロールプレーンを拡張して、どこでも任意のリソースを確認、報告、行動する**ユニバーサルコントロールプレーン**にします。
<!-- vale Google.WordList = YES -->
{{< /hint >}}


# 始める
* [KubernetesクラスターにCrossplaneをインストール]({{<ref "software/install">}})する
* [Crossplaneの紹介]({{<ref "getting-started/introduction" >}})でCrossplaneの動作について詳しく学ぶ
* [Crossplane Slack](https://slack.crossplane.io/)に参加し、7,000人以上のオペレーターのコミュニティと会話を始めましょう。


Crossplaneは、[Cloud Native Compute Foundation](https://www.cncf.io/)プロジェクトです。
