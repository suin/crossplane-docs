---
title: Goでコンポジション関数を書く
state: beta
alphaVersion: "1.11"
betaVersion: "1.14"
weight: 80
description: "コンポジション関数を使用してGoでリソースをテンプレート化できます"
---

コンポジション関数（略して関数）は、Crossplaneリソースをテンプレート化するカスタムプログラムです。Crossplaneは、合成リソース（XR）を作成するときに、どのリソースを作成すべきかを判断するためにコンポジション関数を呼び出します。コンポジション関数について詳しくは、[concepts]({{<ref "../concepts/composition-functions" >}})ページをお読みください。

一般的なプログラミング言語を使用してリソースをテンプレート化する関数を書くことができます。一般的なプログラミング言語を使用することで、ループや条件文などの高度なロジックを使用してリソースをテンプレート化することが可能になります。このガイドでは、[Go](https://go.dev)でコンポジション関数を書く方法を説明します。

{{< hint "important" >}}
このガイドに従う前に、[コンポジション関数の動作]({{<ref "../concepts/composition-functions#how-composition-functions-work" >}})に慣れておくと良いでしょう。
{{< /hint >}}

## ステップを理解する

このガイドでは、{{<hover label="xr" line="2">}}XBuckets{{</hover>}}合成リソース（XR）のためのコンポジション関数の作成について説明します。

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
このセクションは今後のセクションの準備をしています。関数はまだ存在しないため、現在形で関数に言及することは意味がありません。
-->
`XBuckets` XRには、リージョンとバケット名の配列があります。この関数は、名前の配列の各エントリに対してAmazon Web Services（AWS）S3バケットを作成します。
<!-- vale gitlab.FutureTense = YES -->

Goで関数を書くには：

1. [関数を書くために必要なツールをインストールする](#install-the-tools-you-need-to-write-the-function)
1. [テンプレートから関数を初期化する](#initialize-the-function-from-a-template)
1. [テンプレートを編集して関数のロジックを追加する](#edit-the-template-to-add-the-functions-logic)
1. [関数をエンドツーエンドでテストする](#test-the-function-end-to-end)
1. [関数をパッケージリポジトリにビルドしてプッシュする](#build-and-push-the-function-to-a-package-registry)

このガイドでは、これらのステップを詳細に説明します。

## 関数を書くために必要なツールをインストールする

Goで関数を書くには、次のものが必要です：

* [Go](https://go.dev/dl/) v1.21以上。このガイドではGo v1.21を使用します。
* [Docker Engine](https://docs.docker.com/engine/)。このガイドではEngine v24を使用します。
* [Crossplane CLI]({{<ref "../cli" >}}) v1.14以上。このガイドではCrossplane CLI v1.14を使用します。

{{<hint "note">}}
KubernetesクラスターやCrossplaneコントロールプレーンへのアクセスは、構成関数をビルドまたはテストするために必要ありません。
{{</hint>}}

## テンプレートから関数を初期化する

`crossplane beta xpkg init`コマンドを使用して新しい関数を初期化します。このコマンドを実行すると、[GitHubリポジトリ](https://github.com/crossplane/function-template-go)をテンプレートとして使用して関数が初期化されます。

```shell {copy-lines=1}
crossplane beta xpkg init function-xbuckets function-template-go -d function-xbuckets 
Initialized package "function-xbuckets" in directory "/home/negz/control/negz/function-xbuckets" from https://github.com/crossplane/function-template-go/tree/91a1a5eed21964ff98966d72cc6db6f089ad63f4 (main)
```

`crossplane beta init xpkg`コマンドは、`function-xbuckets`という名前のディレクトリを作成します。コマンドを実行すると、新しいディレクトリは次のようになります：

```shell {copy-lines=1}
ls function-xbuckets
Dockerfile  fn.go  fn_test.go  go.mod  go.sum  input/  LICENSE  main.go  package/  README.md  renovate.json
```

`fn.go`ファイルは、関数のコードを追加する場所です。テンプレート内の他のファイルについて知っておくと便利です：

* `main.go`は関数を実行します。`main.go`を編集する必要はありません。
* `Dockerfile`は関数のランタイムをビルドします。`Dockerfile`を編集する必要はありません。
* `input`ディレクトリは関数の入力タイプを定義します。
* `package`ディレクトリには、関数パッケージをビルドするために使用されるメタデータが含まれています。

{{<hint "tip">}}
<!-- vale gitlab.FutureTense = NO -->
<!--
このヒントはCrossplaneの将来の計画について説明しています。
-->
Crossplane CLIのv1.14では、`crossplane beta xpkg init`はテンプレートのGitHubリポジトリをクローンするだけです。将来のCLIリリースでは、テンプレート名を新しい関数の名前に置き換えるなどのタスクが自動化される予定です。詳細については、Crossplaneの問題[#4941](https://github.com/crossplane/crossplane/issues/4941)を参照してください。
<!-- vale gitlab.FutureTense = YES -->
{{</hint>}}


コードを追加する前にいくつかの変更を行う必要があります：

* `package/crossplane.yaml`を編集して、パッケージの名前を変更します。
* `go.mod`を編集して、Goモジュールの名前を変更します。

パッケージの名前を`function-xbuckets`にします。

モジュールの名前は、関数コードをどこに保管したいかによって異なります。GoコードをGitHubにプッシュする場合は、GitHubのユーザー名を使用できます。例えば、`module github.com/negz/function-xbuckets`のようになります。

このガイドの関数は入力タイプを使用しません。この関数では、`input`および`package/input`ディレクトリを削除する必要があります。

`input`ディレクトリは、Compositionからの`input`フィールドを使用して、関数が入力を受け取るために使用できるGo構造体を定義します。  
[composition functions]({{<ref "../concepts/composition-functions" >}})のドキュメントでは、入力をコンポジション関数に渡す方法について説明しています。

`package/input`ディレクトリには、`input`ディレクトリ内の構造体から生成されたOpenAPIスキーマが含まれています。

{{<hint "tip">}}  
入力を使用する関数を書いている場合は、関数の要件を満たすように入力を編集してください。

入力の種類とAPIグループを変更します。`Input`や`template.fn.crossplane.io`は使用しないでください。代わりに、関数にとって意味のあるものを使用してください。

`input`ディレクトリ内のファイルを編集した場合は、`go generate`を実行して生成されたファイルを更新する必要があります。詳細は`input/generate.go`を参照してください。

```shell
go generate ./...
```
{{</hint>}}

## テンプレートを編集して関数のロジックを追加する

関数のロジックは、`fn.go`の{{<hover label="hello-world" line="1">}}RunFunction{{</hover>}}メソッドに追加します。ファイルを最初に開くと、「hello world」関数が含まれています。

```go {label="hello-world"}
func (f *Function) RunFunction(_ context.Context, req *fnv1beta1.RunFunctionRequest) (*fnv1beta1.RunFunctionResponse, error) {
	f.log.Info("Running Function", "tag", req.GetMeta().GetTag())

	rsp := response.To(req, response.DefaultTTL)

	in := &v1beta1.Input{}
	if err := request.GetInput(req, in); err != nil {
		response.Fatal(rsp, errors.Wrapf(err, "cannot get Function input from %T", req))
		return rsp, nil
	}

	response.Normalf(rsp, "I was run with input %q", in.Example)
	return rsp, nil
}
```

すべてのGoコンポジション関数には`RunFunction`メソッドがあります。Crossplaneは、関数が実行するために必要なすべての情報を{{<hover label="hello-world" line="1">}}RunFunctionRequest{{</hover>}}構造体で渡します。

関数は、{{<hover label="hello-world" line="13">}}RunFunctionResponse{{</hover>}}構造体を返すことで、Crossplaneにどのリソースを構成すべきかを伝えます。

{{<hint "tip">}}  
Crossplaneは、[Protocol Buffers](http://protobuf.dev)を使用して`RunFunctionRequest`および`RunFunctionResponse`構造体を生成します。`RunFunctionRequest`および`RunFunctionResponse`の詳細なスキーマは、[Buf Schema Registry](https://buf.build/crossplane/crossplane/docs/main:apiextensions.fn.proto.v1beta1)で見つけることができます。  
{{</hint>}}

`RunFunction` メソッドを編集して、次のコードに置き換えます。

```go {hl_lines="4-56"}
func (f *Function) RunFunction(_ context.Context, req *fnv1beta1.RunFunctionRequest) (*fnv1beta1.RunFunctionResponse, error) {
	rsp := response.To(req, response.DefaultTTL)

	xr, err := request.GetObservedCompositeResource(req)
	if err != nil {
		response.Fatal(rsp, errors.Wrapf(err, "cannot get observed composite resource from %T", req))
		return rsp, nil
	}

	region, err := xr.Resource.GetString("spec.region")
	if err != nil {
		response.Fatal(rsp, errors.Wrapf(err, "cannot read spec.region field of %s", xr.Resource.GetKind()))
		return rsp, nil
	}

	names, err := xr.Resource.GetStringArray("spec.names")
	if err != nil {
		response.Fatal(rsp, errors.Wrapf(err, "cannot read spec.names field of %s", xr.Resource.GetKind()))
		return rsp, nil
	}

	desired, err := request.GetDesiredComposedResources(req)
	if err != nil {
		response.Fatal(rsp, errors.Wrapf(err, "cannot get desired resources from %T", req))
		return rsp, nil
	}

	_ = v1beta1.AddToScheme(composed.Scheme)

	for _, name := range names {
		b := &v1beta1.Bucket{
			ObjectMeta: metav1.ObjectMeta{
				Annotations: map[string]string{
					"crossplane.io/external-name": name,
				},
			},
			Spec: v1beta1.BucketSpec{
				ForProvider: v1beta1.BucketParameters{
					Region: ptr.To[string](region),
				},
			},
		}

		cd, err := composed.From(b)
		if err != nil {
			response.Fatal(rsp, errors.Wrapf(err, "cannot convert %T to %T", b, &composed.Unstructured{}))
			return rsp, nil
		}

		desired[resource.Name("xbuckets-"+name)] = &resource.DesiredComposed{Resource: cd}
	}

	if err := response.SetDesiredComposedResources(rsp, desired); err != nil {
		response.Fatal(rsp, errors.Wrapf(err, "cannot set desired composed resources in %T", rsp))
		return rsp, nil
	}

	return rsp, nil
}
```

以下のブロックを展開して、インポートや関数のロジックを説明するコメントを含む完全な `fn.go` を表示します。

{{<expand "The full fn.go file" >}}
```go
package main

import (
	"context"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/utils/ptr"

	"github.com/upbound/provider-aws/apis/s3/v1beta1"

	"github.com/crossplane/function-sdk-go/errors"
	"github.com/crossplane/function-sdk-go/logging"
	fnv1beta1 "github.com/crossplane/function-sdk-go/proto/v1beta1"
	"github.com/crossplane/function-sdk-go/request"
	"github.com/crossplane/function-sdk-go/resource"
	"github.com/crossplane/function-sdk-go/resource/composed"
	"github.com/crossplane/function-sdk-go/response"
)

// Function returns whatever response you ask it to.
type Function struct {
	fnv1beta1.UnimplementedFunctionRunnerServiceServer

	log logging.Logger
}

// RunFunction observes an XBuckets composite resource (XR). It adds an S3
// bucket to the desired state for every entry in the XR's spec.names array.
func (f *Function) RunFunction(_ context.Context, req *fnv1beta1.RunFunctionRequest) (*fnv1beta1.RunFunctionResponse, error) {
	f.log.Info("Running Function", "tag", req.GetMeta().GetTag())

	// Create a response to the request. This copies the desired state and
	// pipeline context from the request to the response.
	rsp := response.To(req, response.DefaultTTL)

	// Read the observed XR from the request. Most functions use the observed XR
	// to add desired managed resources.
	xr, err := request.GetObservedCompositeResource(req)
	if err != nil {
		// If the function can't read the XR, the request is malformed. This
		// should never happen. The function returns a fatal result. This tells
		// Crossplane to stop running functions and return an error.
		response.Fatal(rsp, errors.Wrapf(err, "cannot get observed composite resource from %T", req))
		return rsp, nil
	}

	// Create an updated logger with useful information about the XR.
	log := f.log.WithValues(
		"xr-version", xr.Resource.GetAPIVersion(),
		"xr-kind", xr.Resource.GetKind(),
		"xr-name", xr.Resource.GetName(),
	)

	// Get the region from the XR. The XR has getter methods like GetString,
	// GetBool, etc. You can use them to get values by their field path.
	region, err := xr.Resource.GetString("spec.region")
	if err != nil {
		response.Fatal(rsp, errors.Wrapf(err, "cannot read spec.region field of %s", xr.Resource.GetKind()))
		return rsp, nil
	}

	// Get the array of bucket names from the XR.
	names, err := xr.Resource.GetStringArray("spec.names")
	if err != nil {
		response.Fatal(rsp, errors.Wrapf(err, "cannot read spec.names field of %s", xr.Resource.GetKind()))
		return rsp, nil
	}

	// Get all desired composed resources from the request. The function will
	// update this map of resources, then save it. This get, update, set pattern
	// ensures the function keeps any resources added by other functions.
	desired, err := request.GetDesiredComposedResources(req)
	if err != nil {
		response.Fatal(rsp, errors.Wrapf(err, "cannot get desired resources from %T", req))
		return rsp, nil
	}

	// Add v1beta1 types (including Bucket) to the composed resource scheme.
	// composed.From uses this to automatically set apiVersion and kind.
	_ = v1beta1.AddToScheme(composed.Scheme)

	// Add a desired S3 bucket for each name.
	for _, name := range names {
		// One advantage of writing a function in Go is strong typing. The
		// function can import and use managed resource types from the provider.
		b := &v1beta1.Bucket{
			ObjectMeta: metav1.ObjectMeta{
				// Set the external name annotation to the desired bucket name.
				// This controls what the bucket will be named in AWS.
				Annotations: map[string]string{
					"crossplane.io/external-name": name,
				},
			},
			Spec: v1beta1.BucketSpec{
				ForProvider: v1beta1.BucketParameters{
					// Set the bucket's region to the value read from the XR.
					Region: ptr.To[string](region),
				},
			},
		}

		// Convert the bucket to the unstructured resource data format the SDK
		// uses to store desired composed resources.
		cd, err := composed.From(b)
		if err != nil {
			response.Fatal(rsp, errors.Wrapf(err, "cannot convert %T to %T", b, &composed.Unstructured{}))
			return rsp, nil
		}

		// Add the bucket to the map of desired composed resources. It's
		// important that the function adds the same bucket every time it's
		// called. It's also important that the bucket is added with the same
		// resource.Name every time it's called. The function prefixes the name
		// with "xbuckets-" to avoid collisions with any other composed
		// resources that might be in the desired resources map.
		desired[resource.Name("xbuckets-"+name)] = &resource.DesiredComposed{Resource: cd}
	}

	// Finally, save the updated desired composed resources to the response.
	if err := response.SetDesiredComposedResources(rsp, desired); err != nil {
		response.Fatal(rsp, errors.Wrapf(err, "cannot set desired composed resources in %T", rsp))
		return rsp, nil
	}

	// Log what the function did. This will only appear in the function's pod
	// logs. A function can use response.Normal and response.Warning to emit
	// Kubernetes events associated with the XR it's operating on.
	log.Info("Added desired buckets", "region", region, "count", len(names))

	return rsp, nil
}
```
{{</expand>}}

このコードは：

1. `RunFunctionRequest` から観測された複合リソースを取得します。
1. 観測された複合リソースからリージョンとバケット名を取得します。
1. 各バケット名に対して1つの希望する S3 バケットを追加します。
1. 希望する S3 バケットを `RunFunctionResponse` で返します。

このコードは、[UpboundのAWS S3プロバイダー](https://github.com/upbound/provider-aws) の `v1beta1.Bucket` 型を使用しています。Goで関数を書く利点の1つは、Crossplaneがプロバイダーで使用するのと同じ強く型付けされた構造体を使用してリソースを構成できることです。

この型を使用するには、AWSプロバイダーGoモジュールを取得する必要があります：

```shell
go get github.com/upbound/provider-aws@v0.43.0
```

Crossplaneは、[Go](https://go.dev)で構成関数を書くための[ソフトウェア開発キット](https://github.com/crossplane/function-sdk-go)（SDK）を提供しています。この関数はSDKのユーティリティを使用しています。特に、`request` と `response` パッケージは、`RunFunctionRequest` と `RunFunctionResponse` 型を扱いやすくします。

{{<hint "tip">}}
SDKの[Goパッケージドキュメント](https://pkg.go.dev/github.com/crossplane/function-sdk-go)を読んでください。
{{</hint>}}

## 関数をエンドツーエンドでテストする

ユニットテストを追加し、`crossplane beta render` コマンドを使用して関数をテストします。

Goはユニットテストに対して豊富なサポートを提供しています。テンプレートから関数を初期化すると、いくつかのユニットテストが `fn_test.go` に追加されます。これらのテストはGoの[推奨事項](https://github.com/golang/go/wiki/TestComments)に従っています。標準ライブラリの[`pkg/testing`](https://pkg.go.dev/testing)と[`google/go-cmp`](https://pkg.go.dev/github.com/google/go-cmp/cmp)のみを使用しています。

テストケースを追加するには、`TestRunFunction` の `cases` マップを更新します。以下のブロックを展開して、関数の完全な `fn_test.go` ファイルを表示します。


{{<expand "The full fn_test.go file" >}}
```go
package main

import (
	"context"
	"testing"
	"time"

	"github.com/google/go-cmp/cmp"
	"github.com/google/go-cmp/cmp/cmpopts"
	"google.golang.org/protobuf/testing/protocmp"
	"google.golang.org/protobuf/types/known/durationpb"

	"github.com/crossplane/crossplane-runtime/pkg/logging"

	fnv1beta1 "github.com/crossplane/function-sdk-go/proto/v1beta1"
	"github.com/crossplane/function-sdk-go/resource"
)

func TestRunFunction(t *testing.T) {
	type args struct {
		ctx context.Context
		req *fnv1beta1.RunFunctionRequest
	}
	type want struct {
		rsp *fnv1beta1.RunFunctionResponse
		err error
	}

	cases := map[string]struct {
		reason string
		args   args
		want   want
	}{
		"AddTwoBuckets": {
			reason: "The Function should add two buckets to the desired composed resources",
			args: args{
				req: &fnv1beta1.RunFunctionRequest{
					Observed: &fnv1beta1.State{
						Composite: &fnv1beta1.Resource{
							// MustStructJSON is a handy way to provide mock
							// resources.
							Resource: resource.MustStructJSON(`{
								"apiVersion": "example.crossplane.io/v1alpha1",
								"kind": "XBuckets",
								"metadata": {
									"name": "test"
								},
								"spec": {
									"region": "us-east-2",
									"names": [
										"test-bucket-a",
										"test-bucket-b"
									]
								}
							}`),
						},
					},
				},
			},
			want: want{
				rsp: &fnv1beta1.RunFunctionResponse{
					Meta: &fnv1beta1.ResponseMeta{Ttl: durationpb.New(60 * time.Second)},
					Desired: &fnv1beta1.State{
						Resources: map[string]*fnv1beta1.Resource{
							"xbuckets-test-bucket-a": {Resource: resource.MustStructJSON(`{
								"apiVersion": "s3.aws.upbound.io/v1beta1",
								"kind": "Bucket",
								"metadata": {
									"annotations": {
										"crossplane.io/external-name": "test-bucket-a"
									}
								},
								"spec": {
									"forProvider": {
										"region": "us-east-2"
									}
								}
							}`)},
							"xbuckets-test-bucket-b": {Resource: resource.MustStructJSON(`{
								"apiVersion": "s3.aws.upbound.io/v1beta1",
								"kind": "Bucket",
								"metadata": {
									"annotations": {
										"crossplane.io/external-name": "test-bucket-b"
									}
								},
								"spec": {
									"forProvider": {
										"region": "us-east-2"
									}
								}
							}`)},
						},
					},
				},
			},
		},
	}

	for name, tc := range cases {
		t.Run(name, func(t *testing.T) {
			f := &Function{log: logging.NewNopLogger()}
			rsp, err := f.RunFunction(tc.args.ctx, tc.args.req)

			if diff := cmp.Diff(tc.want.rsp, rsp, protocmp.Transform()); diff != "" {
				t.Errorf("%s\nf.RunFunction(...): -want rsp, +got rsp:\n%s", tc.reason, diff)
			}

			if diff := cmp.Diff(tc.want.err, err, cmpopts.EquateErrors()); diff != "" {
				t.Errorf("%s\nf.RunFunction(...): -want err, +got err:\n%s", tc.reason, diff)
			}
		})
	}
}
```
{{</expand>}}

ユニットテストを `go test` コマンドを使用して実行します：

```shell
go test -v -cover .
=== RUN   TestRunFunction
=== RUN   TestRunFunction/AddTwoBuckets
--- PASS: TestRunFunction (0.00s)
    --- PASS: TestRunFunction/AddTwoBuckets (0.00s)
PASS
coverage: 52.6% of statements
ok      github.com/negz/function-xbuckets       0.016s  coverage: 52.6% of statements
```

この関数を使用するコンポジションの出力を、Crossplane CLIを使用してプレビューできます。これを行うためにCrossplaneコントロールプレーンは必要ありません。

`function-xbuckets` の下に `example` という名前のディレクトリを作成し、Composite Resource、Composition、およびFunctionのYAMLファイルを作成します。

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

`composition.yaml` ファイルには、コンポジットリソースをレンダリングするために使用するCompositionが含まれています：

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

`functions.yaml` ファイルには、Compositionがそのパイプラインステップで参照するFunctionsが含まれています：

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

`functions.yaml` のFunctionは、{{<hover label="development" line="6">}}Development{{</hover>}} ランタイムを使用しています。これは、`crossplane beta render` に対して、あなたの関数がローカルで実行されていることを示します。これは、Dockerを使用して関数をプルして実行するのではなく、ローカルで実行されている関数に接続します。

```yaml {label="development"}
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-xbuckets
  annotations:
    render.crossplane.io/runtime: Development
```

`go run` を使用して、ローカルで関数を実行します。

```shell {label="run"}
go run . --insecure --debug
```

{{<hint "warning">}}
{{<hover label="run" line="1">}}insecure{{</hover>}} フラグは、関数を暗号化や認証なしで実行するように指示します。テストや開発中のみ使用してください。
{{</hint>}}

別のターミナルで `crossplane beta render` を実行します。

```shell
crossplane beta render xr.yaml composition.yaml functions.yaml
```

このコマンドは、あなたの関数を呼び出します。関数が実行されているターミナルでは、次のようなログ出力が表示されるはずです：

```shell
go run . --insecure --debug
2023-10-31T16:17:32.158-0700    INFO    function-xbuckets/fn.go:29      Running Function        {"tag": ""}
2023-10-31T16:17:32.159-0700    INFO    function-xbuckets/fn.go:125     Added desired buckets   {"xr-version": "example.crossplane.io/v1", "xr-kind": "XBuckets", "xr-name": "example-buckets", "region": "us-east-2", "count": 3}
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

関数はデフォルトで `linux/amd64` のような単一のプラットフォームをサポートします。各プラットフォーム用にランタイムとパッケージをビルドし、すべてのパッケージをレジストリの単一のタグにプッシュすることで、複数のプラットフォームをサポートできます。

関数をレジストリにプッシュすることで、Crossplane コントロールプレーンで関数を使用できるようになります。関数をコントロールプレーンで使用する方法については、[構成関数のドキュメント]({{<ref "../concepts/composition-functions" >}})を参照してください。

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

Crossplane CLIを使用して、各プラットフォーム用のパッケージをビルドします。各パッケージはランタイムイメージを埋め込んでいます。

{{<hover label="build" line="2">}}--package-root{{</hover>}} フラグは、`crossplane.yaml` を含む `package` ディレクトリを指定します。これには、パッケージに関するメタデータが含まれています。


{{<hover label="build" line="3">}}--embed-runtime-image{{</hover>}} フラグは、Dockerを使用してビルドされたランタイムイメージタグを指定します。

{{<hover label="build" line="4">}}--package-file{{</hover>}} フラグは、パッケージファイルをディスクに書き込む場所を指定します。Crossplaneパッケージファイルは、拡張子 `.xpkg` を使用します。

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
Crossplaneパッケージは特別なOCIイメージです。パッケージについての詳細は、[パッケージのドキュメント]({{< ref "../concepts/packages" >}})をお読みください。
{{</hint>}}

両方のパッケージファイルをレジストリにプッシュします。両方のファイルをレジストリの1つのタグにプッシュすると、`linux/arm64` と `linux/amd64` ホストの両方で実行される
[multi-platform](https://docs.docker.com/build/building/multi-platform/)
パッケージが作成されます。

```shell
crossplane xpkg push \
  --package-files=function-amd64.xpkg,function-arm64.xpkg \
  negz/function-xbuckets:v0.1.0
```

{{<hint "tip">}}
関数をGitHubリポジトリにプッシュすると、テンプレートは自動的に
[GitHub Actions](https://github.com/features/actions)を使用して継続的インテグレーション（CI）を設定します。CIワークフローは、関数をリント、テスト、およびビルドします。テンプレートがCIをどのように設定しているかは、`.github/workflows/ci.yaml`を読むことで確認できます。

CIワークフローは自動的にパッケージを `xpkg.upbound.io` にプッシュできます。これを機能させるには、https://marketplace.upbound.io でリポジトリを作成する必要があります。APIトークンを作成し、[リポジトリに追加する](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository)ことで、CIワークフローにマーケットプレイスへのプッシュアクセスを与えます。
APIトークンアクセスIDを `XPKG_ACCESS_ID` という名前のシークレットとして保存し、APIトークンを `XPKG_TOKEN` という名前のシークレットとして保存します。
{{</hint>}}
