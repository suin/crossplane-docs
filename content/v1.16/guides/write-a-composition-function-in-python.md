---
title: Pythonでコンポジション関数を書く
state: beta
alphaVersion: "1.11"
betaVersion: "1.14"
weight: 81
description: "コンポジション関数を使用してPythonでリソースをテンプレート化できます"
---

コンポジション関数（略して関数）は、Crossplaneリソースをテンプレート化するカスタムプログラムです。Crossplaneは、合成リソース（XR）を作成する際に、どのリソースを作成すべきかを判断するためにコンポジション関数を呼び出します。コンポジション関数について詳しくは、[concepts]({{<ref "../concepts/composition-functions" >}})ページをお読みください。

一般的なプログラミング言語を使用してリソースをテンプレート化する関数を書くことができます。一般的なプログラミング言語を使用することで、ループや条件文などの高度なロジックを使用してリソースをテンプレート化することが可能になります。このガイドでは、[Python](https://python.org)でコンポジション関数を書く方法を説明します。

{{< hint "important" >}}
このガイドに従う前に、[コンポジション関数の動作]({{<ref "../concepts/composition-functions#how-composition-functions-work" >}})に慣れておくと良いでしょう。
{{< /hint >}}

## ステップを理解する

このガイドでは、{{<hover label="xr" line="2">}}XBuckets{{</hover>}}合成リソース（XR）のためのコンポジション関数を書くことを扱います。

```yaml {label="xr"}
apiVersion: example.crossplane.io/v1
kind: XBuckets
metadata:
  name: example-buckets
spec:
  region: us-east-2
  names:
  - crossplane-functions-example-a
  - crossplane-functions-example-b
  - crossplane-functions-example-c
```

<!-- vale gitlab.FutureTense = NO -->
<!--
このセクションは今後のセクションのための準備をしています。関数がまだ存在しないため、現在形で関数を参照するのは意味がありません。
-->
`XBuckets` XRには、リージョンとバケット名の配列があります。この関数は、名前の配列の各エントリに対してAmazon Web Services（AWS）S3バケットを作成します。
<!-- vale gitlab.FutureTense = YES -->

Pythonで関数を書くには：

1. [関数を書くために必要なツールをインストールする](#install-the-tools-you-need-to-write-the-function)
1. [テンプレートから関数を初期化する](#initialize-the-function-from-a-template)
1. [テンプレートを編集して関数のロジックを追加する](#edit-the-template-to-add-the-functions-logic)
1. [関数をエンドツーエンドでテストする](#test-the-function-end-to-end)
1. [関数をパッケージリポジトリにビルドしてプッシュする](#build-and-push-the-function-to-a-package-registry)

このガイドでは、これらのステップを詳細に説明します。

## 関数を書くために必要なツールをインストールする

Pythonで関数を書くために必要なものは次のとおりです：

* [Python](https://www.python.org/downloads/) v3.11。
* [Hatch](https://hatch.pypa.io/)、Pythonビルドツール。このガイドではv1.7を使用します。
* [Docker Engine](https://docs.docker.com/engine/)。このガイドではEngine v24を使用します。
* [Crossplane CLI]({{<ref "../cli" >}}) v1.14以上。このガイドではCrossplane CLI v1.14を使用します。

{{<hint "note">}}
KubernetesクラスターやCrossplaneコントロールプレーンへのアクセスは、コンポジション関数を構築またはテストするために必要ありません。
{{</hint>}}

## テンプレートから関数を初期化する

`crossplane beta xpkg init`コマンドを使用して新しい関数を初期化します。このコマンドを実行すると、[GitHubリポジトリ](https://github.com/crossplane/function-template-python)をテンプレートとして使用して関数が初期化されます。

```shell {copy-lines=1}
crossplane beta xpkg init function-xbuckets https://github.com/crossplane/function-template-python -d function-xbuckets
Initialized package "function-xbuckets" in directory "/home/negz/control/negz/function-xbuckets" from https://github.com/crossplane/function-template-python/tree/bfed6923ab4c8e7adeed70f41138645fc7d38111 (main)
```

`crossplane beta init xpkg`コマンドは、`function-xbuckets`という名前のディレクトリを作成します。コマンドを実行すると、新しいディレクトリは次のようになります：

```shell {copy-lines=1}
ls function-xbuckets
Dockerfile  example/  function/  LICENSE  package/  pyproject.toml  README.md  renovate.json  tests/
```

関数のコードは`function`ディレクトリにあります：

```shell {copy-lines=1}
ls function/
__version__.py  fn.py  main.py
```

`function/fn.py`ファイルは、関数のコードを追加する場所です。テンプレート内の他のファイルについて知っておくと便利です：

* `function/main.py`は関数を実行します。`main.py`を編集する必要はありません。
* `Dockerfile`は関数のランタイムをビルドします。`Dockerfile`を編集する必要はありません。
* `package`ディレクトリには、関数パッケージをビルドするために使用されるメタデータが含まれています。

{{<hint "tip">}}
<!-- vale gitlab.FutureTense = NO -->
<!--
このヒントはCrossplaneの将来の計画について説明しています。
-->
Crossplane CLIのv1.14では、`crossplane beta xpkg init`はテンプレートGitHubリポジトリをクローンするだけです。将来のCLIリリースでは、テンプレート名を新しい関数の名前に置き換えるなどのタスクが自動化される予定です。詳細については、Crossplaneの問題[#4941](https://github.com/crossplane/crossplane/issues/4941)を参照してください。
<!-- vale gitlab.FutureTense = YES -->
{{</hint>}}

`package/crossplane.yaml`を編集して、コードを追加する前にパッケージの名前を変更します。パッケージの名前を`function-xbuckets`にします。

`package/input`ディレクトリは、関数の入力のOpenAPIスキーマを定義します。このガイドの関数は入力を受け付けません。`package/input`ディレクトリを削除します。

[composition functions]({{<ref "../concepts/composition-functions" >}})のドキュメントでは、composition functionの入力について説明しています。

{{<hint "tip">}}
入力を使用する関数を書いている場合は、入力YAMLファイルを編集して関数の要件を満たしてください。

入力のkindとAPIグループを変更します。`Input`や`template.fn.crossplane.io`は使用しないでください。代わりに、関数にとって意味のあるものを使用してください。
{{</hint>}}

## テンプレートを編集して関数のロジックを追加する

関数のロジックは、`function/fn.py`の{{<hover label="hello-world" line="1">}}RunFunction{{</hover>}}メソッドに追加します。ファイルを最初に開くと、「hello world」関数が含まれています。

```python {label="hello-world"}
async def RunFunction(self, req: fnv1beta1.RunFunctionRequest, _: grpc.aio.ServicerContext) -> fnv1beta1.RunFunctionResponse:
    log = self.log.bind(tag=req.meta.tag)
    log.info("Running function")

    rsp = response.to(req)

    example = ""
    if "example" in req.input:
        example = req.input["example"]

    # TODO: Add your function logic here!
    response.normal(rsp, f"I was run with input {example}!")
    log.info("I was run!", input=example)

    return rsp
```

すべてのPython composition functionsには`RunFunction`メソッドがあります。Crossplaneは、関数が実行するために必要なすべての情報を{{<hover label="hello-world" line="1">}}RunFunctionRequest{{</hover>}}オブジェクトに渡します。

関数は、{{<hover label="hello-world" line="15">}}RunFunctionResponse{{</hover>}}オブジェクトを返すことで、Crossplaneにどのリソースを構成すべきかを指示します。

`RunFunction`メソッドを編集して、以下のコードに置き換えます。

```python {hl_lines="7-28"}
async def RunFunction(self, req: fnv1beta1.RunFunctionRequest, _: grpc.aio.ServicerContext) -> fnv1beta1.RunFunctionResponse:
    log = self.log.bind(tag=req.meta.tag)
    log.info("Running function")

    rsp = response.to(req)

    region = req.observed.composite.resource["spec"]["region"]
    names = req.observed.composite.resource["spec"]["names"]

    for name in names:
        rsp.desired.resources[f"xbuckets-{name}"].resource.update(
            {
                "apiVersion": "s3.aws.upbound.io/v1beta1",
                "kind": "Bucket",
                "metadata": {
                    "annotations": {
                        "crossplane.io/external-name": name,
                    },
                },
                "spec": {
                    "forProvider": {
                        "region": region,
                    },
                },
            }
        )

    log.info("Added desired buckets", region=region, count=len(names))

    return rsp
```

以下のブロックを展開して、インポートや関数のロジックを説明するコメントを含む完全な`fn.py`を表示します。

{{<expand "The full fn.py file" >}}
```python
"""A Crossplane composition function."""

import grpc
from crossplane.function import logging, response
from crossplane.function.proto.v1beta1 import run_function_pb2 as fnv1beta1
from crossplane.function.proto.v1beta1 import run_function_pb2_grpc as grpcv1beta1


class FunctionRunner(grpcv1beta1.FunctionRunnerService):
    """A FunctionRunner handles gRPC RunFunctionRequests."""

    def __init__(self):
        """Create a new FunctionRunner."""
        self.log = logging.get_logger()

    async def RunFunction(
        self, req: fnv1beta1.RunFunctionRequest, _: grpc.aio.ServicerContext
    ) -> fnv1beta1.RunFunctionResponse:
        """Run the function."""
        # Create a logger for this request.
        log = self.log.bind(tag=req.meta.tag)
        log.info("Running function")

        # Create a response to the request. This copies the desired state and
        # pipeline context from the request to the response.
        rsp = response.to(req)

        # Get the region and a list of bucket names from the observed composite
        # resource (XR). Crossplane represents resources using the Struct
        # well-known protobuf type. The Struct Python object can be accessed
        # like a dictionary.
        region = req.observed.composite.resource["spec"]["region"]
        names = req.observed.composite.resource["spec"]["names"]

        # Add a desired S3 bucket for each name.
        for name in names:
            # Crossplane represents desired composed resources using a protobuf
            # map of messages. This works a little like a Python defaultdict.
            # Instead of assigning to a new key in the dict-like map, you access
            # the key and mutate its value as if it did exist.
            #
            # The below code works because accessing the xbuckets-{name} key
            # automatically creates a new, empty fnv1beta1.Resource message. The
            # Resource message has a resource field containing an empty Struct
            # object that can be populated from a dictionary by calling update.
            #
            # https://protobuf.dev/reference/python/python-generated/#map-fields
            rsp.desired.resources[f"xbuckets-{name}"].resource.update(
                {
                    "apiVersion": "s3.aws.upbound.io/v1beta1",
                    "kind": "Bucket",
                    "metadata": {
                        "annotations": {
                            "crossplane.io/external-name": name,
                        },
                    },
                    "spec": {
                        "forProvider": {
                            "region": region,
                        },
                    },
                }
            )

        # Log what the function did. This will only appear in the function's pod
        # logs. A function can use response.normal() and response.warning() to
        # emit Kubernetes events associated with the XR it's operating on.
        log.info("Added desired buckets", region=region, count=len(names))

        return rsp
```
{{</expand>}}

このコードは次のことを行います：

1. `RunFunctionRequest`から観測された複合リソースを取得します。
1. 観測された複合リソースからリージョンとバケット名を取得します。
1. 各バケット名に対して1つの希望するS3バケットを追加します。
1. 希望するS3バケットを`RunFunctionResponse`で返します。

Crossplaneは、Pythonでcomposition functionsを書くための[ソフトウェア開発キット](https://github.com/crossplane/function-sdk-python)（SDK）を提供しています。この関数はSDKのユーティリティを使用しています。


{{<hint "tip">}}
[Python Function SDKのドキュメント](https://crossplane.github.io/function-sdk-python)をお読みください。
{{</hint>}}

{{<hint "important">}}
Python SDKは、[Protocol Buffers](https://protobuf.dev)スキーマから`RunFunctionRequest`および`RunFunctionResponse`のPythonオブジェクトを自動的に生成します。スキーマは、[Buf Schema Registry](https://buf.build/crossplane/crossplane/docs/main:apiextensions.fn.proto.v1beta1)で確認できます。

生成されたPythonオブジェクトのフィールドは、辞書やリストのような組み込みPython型と似た動作をします。ただし、いくつかの違いがあることに注意してください。

特に、観測されたリソースと望ましいリソースのマップには辞書のようにアクセスしますが、マップキーに割り当てることで新しい望ましいリソースを追加することはできません。代わりに、マップキーが既に存在するかのようにアクセスして変更してください。

このように新しいリソースを追加するのではなく：

```python
resource = {"apiVersion": "example.org/v1", "kind": "Composed", ...}
rsp.desired.resources["new-resource"] = fnv1beta1.Resource(resource=resource)
```

既に存在するかのように振る舞い、変更します：

```python
resource = {"apiVersion": "example.org/v1", "kind": "Composed", ...}
rsp.desired.resources["new-resource"].resource.update(resource)
```

詳細については、Protocol Buffersの[Python Generated Code Guide](https://protobuf.dev/reference/python/python-generated/#fields)を参照してください。
{{</hint>}}

## 関数のエンドツーエンドテスト

ユニットテストを追加し、`crossplane beta render`コマンドを使用して関数をテストします。

テンプレートから関数を初期化すると、`tests/test_fn.py`にいくつかのユニットテストが追加されます。これらのテストは、Python標準ライブラリの[`unittest`](https://docs.python.org/3/library/unittest.html)モジュールを使用しています。

テストケースを追加するには、`test_run_function`の`cases`リストを更新します。関数の完全な`tests/test_fn.py`ファイルを表示するには、以下のブロックを展開してください。

{{<expand "The full test_fn.py file" >}}
```python
import dataclasses
import unittest

from crossplane.function import logging, resource
from crossplane.function.proto.v1beta1 import run_function_pb2 as fnv1beta1
from google.protobuf import duration_pb2 as durationpb
from google.protobuf import json_format
from google.protobuf import struct_pb2 as structpb

from function import fn


class TestFunctionRunner(unittest.IsolatedAsyncioTestCase):
    def setUp(self) -> None:
        logging.configure(level=logging.Level.DISABLED)
        self.maxDiff = 2000

    async def test_run_function(self) -> None:
        @dataclasses.dataclass
        class TestCase:
            reason: str
            req: fnv1beta1.RunFunctionRequest
            want: fnv1beta1.RunFunctionResponse

        cases = [
            TestCase(
                reason="The function should compose two S3 buckets.",
                req=fnv1beta1.RunFunctionRequest(
                    observed=fnv1beta1.State(
                        composite=fnv1beta1.Resource(
                            resource=resource.dict_to_struct(
                                {
                                    "apiVersion": "example.crossplane.io/v1alpha1",
                                    "kind": "XBuckets",
                                    "metadata": {"name": "test"},
                                    "spec": {
                                        "region": "us-east-2",
                                        "names": ["test-bucket-a", "test-bucket-b"],
                                    },
                                }
                            )
                        )
                    )
                ),
                want=fnv1beta1.RunFunctionResponse(
                    meta=fnv1beta1.ResponseMeta(ttl=durationpb.Duration(seconds=60)),
                    desired=fnv1beta1.State(
                        resources={
                            "xbuckets-test-bucket-a": fnv1beta1.Resource(
                                resource=resource.dict_to_struct(
                                    {
                                        "apiVersion": "s3.aws.upbound.io/v1beta1",
                                        "kind": "Bucket",
                                        "metadata": {
                                            "annotations": {
                                                "crossplane.io/external-name": "test-bucket-a"
                                            },
                                        },
                                        "spec": {
                                            "forProvider": {"region": "us-east-2"}
                                        },
                                    }
                                )
                            ),
                            "xbuckets-test-bucket-b": fnv1beta1.Resource(
                                resource=resource.dict_to_struct(
                                    {
                                        "apiVersion": "s3.aws.upbound.io/v1beta1",
                                        "kind": "Bucket",
                                        "metadata": {
                                            "annotations": {
                                                "crossplane.io/external-name": "test-bucket-b"
                                            },
                                        },
                                        "spec": {
                                            "forProvider": {"region": "us-east-2"}
                                        },
                                    }
                                )
                            ),
                        },
                    ),
                    context=structpb.Struct(),
                ),
            ),
        ]

        runner = fn.FunctionRunner()

        for case in cases:
            got = await runner.RunFunction(case.req, None)
            self.assertEqual(
                json_format.MessageToDict(got),
                json_format.MessageToDict(case.want),
                "-want, +got",
            )


if __name__ == "__main__":
    unittest.main()
```
{{</expand>}}

`hatch run`を使用してユニットテストを実行します：

```shell {copy-lines="1"}
hatch run test:unit
.
----------------------------------------------------------------------
Ran 1 test in 0.003s

OK
```

```markdown
{{<hint "tip">}}
[Hatch](https://hatch.pypa.io/) は Python のビルドツールです。Python
アーティファクト（ホイールなど）をビルドします。また、`virtualenv` や `venv` に似た仮想環境を管理します。`hatch run` コマンドは仮想環境を作成し、その環境でコマンドを実行します。
{{</hint>}}

この関数を使用する Composition の出力を Crossplane CLI を使用してプレビューできます。これを行うために Crossplane コントロールプレーンは必要ありません。

`function-xbuckets` の下に `example` という名前のディレクトリを作成し、Composite Resource、Composition、および Function の YAML ファイルを作成します。

以下のブロックを展開して、例のファイルを確認してください。

{{<expand "The xr.yaml, composition.yaml and function.yaml files">}}

これらのファイルを使用して `crossplane beta render` を実行することで、以下の出力を再現できます。

`xr.yaml` ファイルには、レンダリングするためのコンポジットリソースが含まれています：

```yaml
apiVersion: example.crossplane.io/v1
kind: XBuckets
metadata:
  name: example-buckets
spec:
  region: us-east-2
  names:
  - crossplane-functions-example-a
  - crossplane-functions-example-b
  - crossplane-functions-example-c
```

<br />

`composition.yaml` ファイルには、コンポジットリソースをレンダリングするために使用する Composition が含まれています：

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: create-buckets
spec:
  compositeTypeRef:
    apiVersion: example.crossplane.io/v1
    kind: XBuckets
  mode: Pipeline
  pipeline:
  - step: create-buckets
    functionRef:
      name: function-xbuckets
```

<br />

`functions.yaml` ファイルには、Composition がパイプラインステップで参照する Functions が含まれています：

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-xbuckets
  annotations:
    render.crossplane.io/runtime: Development
spec:
  # The CLI ignores this package when using the Development runtime.
  # You can set it to any value.
  package: xpkg.upbound.io/negz/function-xbuckets:v0.1.0
```
{{</expand>}}

`functions.yaml` の Function は
{{<hover label="development" line="6">}}Development{{</hover>}}
ランタイムを使用しています。これは `crossplane beta render` に対して、あなたの関数がローカルで実行されていることを示します。Docker を使用して関数をプルして実行するのではなく、ローカルで実行されている関数に接続します。

```yaml {label="development"}
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-xbuckets
  annotations:
    render.crossplane.io/runtime: Development
```

`hatch run development` を使用して、ローカルで関数を実行します。

```shell {label="run"}
hatch run development
```

{{<hint "warning">}}
`hatch run development` は、暗号化や認証なしで関数を実行します。テストおよび開発中のみ使用してください。
{{</hint>}}

別のターミナルで `crossplane beta render` を実行します。

```shell
crossplane beta render xr.yaml composition.yaml functions.yaml
```

このコマンドはあなたの関数を呼び出します。関数が実行されているターミナルでは、今後ログ出力が表示されるはずです：
```

```shell
hatch run development
2024-01-11T22:12:58.153572Z [info     ] Running function               filename=fn.py lineno=22 tag=
2024-01-11T22:12:58.153792Z [info     ] Added desired buckets          count=3 filename=fn.py lineno=68 region=us-east-2 tag=
```

`crossplane beta render` コマンドは、関数が返す望ましいリソースを出力します。

```yaml
---
apiVersion: example.crossplane.io/v1
kind: XBuckets
metadata:
  name: example-buckets
---
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  annotations:
    crossplane.io/composition-resource-name: xbuckets-crossplane-functions-example-b
    crossplane.io/external-name: crossplane-functions-example-b
  generateName: example-buckets-
  labels:
    crossplane.io/composite: example-buckets
  ownerReferences:
    # Omitted for brevity
spec:
  forProvider:
    region: us-east-2
---
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  annotations:
    crossplane.io/composition-resource-name: xbuckets-crossplane-functions-example-c
    crossplane.io/external-name: crossplane-functions-example-c
  generateName: example-buckets-
  labels:
    crossplane.io/composite: example-buckets
  ownerReferences:
    # Omitted for brevity
spec:
  forProvider:
    region: us-east-2
---
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  annotations:
    crossplane.io/composition-resource-name: xbuckets-crossplane-functions-example-a
    crossplane.io/external-name: crossplane-functions-example-a
  generateName: example-buckets-
  labels:
    crossplane.io/composite: example-buckets
  ownerReferences:
    # Omitted for brevity
spec:
  forProvider:
    region: us-east-2
```

{{<hint "tip">}}
構成関数のドキュメントを読んで、[構成関数のテスト]({{< ref "../concepts/composition-functions#test-a-composition-that-uses-functions" >}})について学びましょう。
{{</hint>}}

## 関数をパッケージレジストリにビルドしてプッシュする

関数は2つの段階でビルドします。最初に関数のランタイムをビルドします。これは、Crossplaneが関数を実行するために使用するOpen Container Initiative (OCI) イメージです。その後、そのランタイムをパッケージに埋め込み、パッケージレジストリにプッシュします。Crossplane CLIは、デフォルトのパッケージレジストリとして `xpkg.upbound.io` を使用します。

関数はデフォルトで `linux/amd64` のような単一のプラットフォームをサポートします。各プラットフォームのためにランタイムとパッケージをビルドし、すべてのパッケージをレジストリの単一のタグにプッシュすることで、複数のプラットフォームをサポートできます。

関数をレジストリにプッシュすることで、Crossplaneコントロールプレーンで関数を使用できるようになります。関数をコントロールプレーンで使用する方法については、[構成関数のドキュメント]({{<ref "../concepts/composition-functions" >}})を参照してください。

Dockerを使用して、各プラットフォームのランタイムをビルドします。

```shell {copy-lines="1"}
docker build . --quiet --platform=linux/amd64 --tag runtime-amd64
sha256:fdf40374cc6f0b46191499fbc1dbbb05ddb76aca854f69f2912e580cfe624b4b
```

```shell {copy-lines="1"}
docker build . --quiet --platform=linux/arm64 --tag runtime-arm64
sha256:cb015ceabf46d2a55ccaeebb11db5659a2fb5e93de36713364efcf6d699069af
```

{{<hint "tip">}}
任意のタグを使用できます。ランタイムイメージをレジストリにプッシュする必要はありません。タグは、`crossplane xpkg build` にどのランタイムを埋め込むかを伝えるためだけに使用されます。
{{</hint>}}

{{<hint "important">}}
Dockerは、異なるプラットフォームのイメージを作成するためにエミュレーションを使用します。異なるプラットフォームのイメージのビルドが失敗した場合は、`binfmt` がインストールされていることを確認してください。手順については、[Dockerのドキュメント](https://docs.docker.com/build/building/multi-platform/#qemu)を参照してください。
{{</hint>}}


Crossplane CLIを使用して、各プラットフォーム用のパッケージをビルドします。各パッケージには
ランタイムイメージが埋め込まれています。

{{<hover label="build" line="2">}}--package-root{{</hover>}} フラグは
`package` ディレクトリを指定します。このディレクトリには `crossplane.yaml` が含まれています。
これにはパッケージに関するメタデータが含まれています。

{{<hover label="build" line="3">}}--embed-runtime-image{{</hover>}} フラグは
Dockerを使用してビルドされたランタイムイメージタグを指定します。

{{<hover label="build" line="4">}}--package-file{{</hover>}} フラグは
パッケージファイルをディスクに書き込む場所を指定します。Crossplaneパッケージファイルは
拡張子 `.xpkg` を使用します。

```shell {label="build"}
crossplane xpkg build \
    --package-root=package \
    --embed-runtime-image=runtime-amd64 \
    --package-file=function-amd64.xpkg
```

```shell
crossplane xpkg build \
    --package-root=package \
    --embed-runtime-image=runtime-arm64 \
    --package-file=function-arm64.xpkg
```

{{<hint "tip">}}
Crossplaneパッケージは特別なOCIイメージです。パッケージについての詳細は
[パッケージのドキュメント]({{< ref "../concepts/packages" >}})をお読みください。
{{</hint>}}

両方のパッケージファイルをレジストリにプッシュします。両方のファイルをレジストリの
1つのタグにプッシュすると、`linux/arm64` と `linux/amd64` ホストの両方で動作する
[multi-platform](https://docs.docker.com/build/building/multi-platform/)
パッケージが作成されます。

```shell
crossplane xpkg push \
  --package-files=function-amd64.xpkg,function-arm64.xpkg \
  negz/function-xbuckets:v0.1.0
```

{{<hint "tip">}}
関数をGitHubリポジトリにプッシュすると、テンプレートが自動的に
[GitHub Actions](https://github.com/features/actions)を使用して継続的インテグレーション（CI）を設定します。CIワークフローは
関数をリント、テスト、ビルドします。テンプレートがCIをどのように設定しているかは
`.github/workflows/ci.yaml`を読むことで確認できます。

CIワークフローは自動的にパッケージを `xpkg.upbound.io` にプッシュできます。これを機能させるには、
https://marketplace.upbound.io にリポジトリを作成する必要があります。APIトークンを作成し、
[リポジトリに追加する](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository)ことで、
CIワークフローにマーケットプレイスへのプッシュアクセスを与えます。
APIトークンアクセスIDを `XPKG_ACCESS_ID` という名前のシークレットとして保存し、
APIトークンを `XPKG_TOKEN` という名前のシークレットとして保存します。
{{</hint>}}

It seems that there is no content provided for translation. Please paste the Markdown content you'd like me to translate into Japanese.
