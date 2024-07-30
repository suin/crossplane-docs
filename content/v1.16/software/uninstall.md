---
title: Crossplaneのアンインストール
weight: 300
---

{{<hint "warning" >}}
Crossplaneがアンインストールされない限り、Crossplaneによって作成されたリソースは削除されません。

これにより、手動で削除する必要があるクラウドリソースが残る可能性があります。
{{< /hint >}}

## 順序付きCrossplaneアンインストール
ほとんどのCrossplaneリソースは、他のCrossplaneリソースに依存しています。

例えば、_マネージドリソース_は_プロバイダー_に依存しています。

Crossplaneリソースを順番に削除しないと、Crossplaneがプロビジョニングされた外部リソースを削除できなくなる場合があります。

Crossplaneリソースの削除は、以下の順序で行う必要があります：
1. すべての_コンポジットリソース定義_を削除する
2. 残りのすべての_マネージドリソース_を削除する
3. すべての_プロバイダー_を削除する

Crossplaneポッドを削除すると、_クレーム_などの残りのCrossplaneコンポーネントが削除されます。

{{<hint "tip" >}}
`kubectl get managed`を使用して、すべての外部リソースのインベントリを収集します。

Kubernetes APIサーバーのサイズやリソースの数によっては、このコマンドが返すまでに数分かかる場合があります。

{{<expand "An example kubectl get managed" >}}

```shell {copy-lines="1"}
kubectl get managed
NAME                                                 READY   SYNCED   EXTERNAL-NAME          AGE
securitygroup.ec2.aws.upbound.io/my-db-7mc7h-j84h8   True    True     sg-0da6e9c29113596b6   3m1s
securitygroup.ec2.aws.upbound.io/my-db-8bhr2-9wsx9   True    True     sg-02695166f010ec05b   2m26s

NAME                                         READY   SYNCED   EXTERNAL-NAME                       AGE
route.ec2.aws.upbound.io/my-db-7mc7h-vw985   True    True     r-rtb-05822b8df433e4e2b1080289494   3m1s
route.ec2.aws.upbound.io/my-db-8bhr2-7m2wq   False   True                                         2m26s

NAME                                                     READY   SYNCED   EXTERNAL-NAME      AGE
securitygrouprule.ec2.aws.upbound.io/my-db-7mc7h-mkd9s   True    True     sgrule-778063708   3m1s
securitygrouprule.ec2.aws.upbound.io/my-db-8bhr2-lzr89   False   True                        2m26s

NAME                                              READY   SYNCED   EXTERNAL-NAME           AGE
routetable.ec2.aws.upbound.io/my-db-7mc7h-mnqvm   True    True     rtb-05822b8df433e4e2b   3m1s
routetable.ec2.aws.upbound.io/my-db-8bhr2-dfhj6   True    True     rtb-02e875abd25658254   2m26s

NAME                                          READY   SYNCED   EXTERNAL-NAME              AGE
subnet.ec2.aws.upbound.io/my-db-7mc7h-7m49d   True    True     subnet-0c1ab32c5ec129dd1   3m2s
subnet.ec2.aws.upbound.io/my-db-7mc7h-9t64t   True    True     subnet-07075c17c7a72f79e   3m2s
subnet.ec2.aws.upbound.io/my-db-7mc7h-rs8t8   True    True     subnet-08e88e826a42e55b4   3m2s
subnet.ec2.aws.upbound.io/my-db-8bhr2-9sjpx   True    True     subnet-05d21c7b52f7ac8ca   2m26s
subnet.ec2.aws.upbound.io/my-db-8bhr2-dvrxf   True    True     subnet-0432310376b5d09de   2m26s
subnet.ec2.aws.upbound.io/my-db-8bhr2-t7dpr   True    True     subnet-0080fdcb6e9b70632   2m26s

NAME                                       READY   SYNCED   EXTERNAL-NAME           AGE
vpc.ec2.aws.upbound.io/my-db-7mc7h-ktbbh   True    True     vpc-08d7dd84e0c12f33e   3m3s
vpc.ec2.aws.upbound.io/my-db-8bhr2-mrh2x   True    True     vpc-06994bf323fc1daea   2m26s

NAME                                                   READY   SYNCED   EXTERNAL-NAME           AGE
internetgateway.ec2.aws.upbound.io/my-db-7mc7h-s2x4v   True    True     igw-0189c4da07a3142dc   3m1s
internetgateway.ec2.aws.upbound.io/my-db-8bhr2-q7dzl   True    True     igw-01bf2a1dbbebf6a27   2m26s

NAME                                                         READY   SYNCED   EXTERNAL-NAME                AGE
routetableassociation.ec2.aws.upbound.io/my-db-7mc7h-28qb4   True    True     rtbassoc-0718d680b5a0e68fe   3m1s
routetableassociation.ec2.aws.upbound.io/my-db-7mc7h-9hdlr   True    True     rtbassoc-0faaedb88c6e1518c   3m1s
routetableassociation.ec2.aws.upbound.io/my-db-7mc7h-txhmz   True    True     rtbassoc-0e5010724ca027864   3m1s
routetableassociation.ec2.aws.upbound.io/my-db-8bhr2-bvgkt   False   True                                  2m26s
routetableassociation.ec2.aws.upbound.io/my-db-8bhr2-d9gbg   False   True                                  2m26s
routetableassociation.ec2.aws.upbound.io/my-db-8bhr2-k6k8m   False   True                                  2m26s

NAME                                            READY   SYNCED   EXTERNAL-NAME       AGE
instance.rds.aws.upbound.io/my-db-7mc7h-5d6w4   False   True     my-db-7mc7h-5d6w4   3m1s
instance.rds.aws.upbound.io/my-db-8bhr2-tx9kf   False   True     my-db-8bhr2-tx9kf   2m26s

NAME                                               READY   SYNCED   EXTERNAL-NAME       AGE
subnetgroup.rds.aws.upbound.io/my-db-7mc7h-8c8n9   True    True     my-db-7mc7h-8c8n9   3m2s
subnetgroup.rds.aws.upbound.io/my-db-8bhr2-mc5ps   True    True     my-db-8bhr2-mc5ps   2m27s

NAME                                                   READY   SYNCED   EXTERNAL-NAME                 AGE
bucket.s3.aws.upbound.io/crossplane-bucket-867737b10   True    True
crossplane-bucket-867737b10   5m26s
```

{{</expand >}}
{{< /hint >}}

### コンポジットリソース定義の削除
インストールされた_コンポジットリソース定義_を削除すると、_コンポジットリソース定義_によって定義されたすべての_コンポジットリソース_と、それらが作成した_マネージドリソース_が削除されます。

インストールされた_コンポジットリソース定義_を`kubectl get xrd`で表示します。

```shell {copy-lines="1"}
kubectl get xrd
NAME                                                ESTABLISHED   OFFERED   AGE
compositepostgresqlinstances.database.example.org   True          True      40s
```

`kubectl delete xrd`を使用して_コンポジットリソース定義_を削除します。

```shell
kubectl delete xrd compositepostgresqlinstances.database.example.org
```

### マネージドリソースの削除

手動で作成した_マネージドリソース_を手動で削除します。

残りの_マネージドリソース_を表示するには、`kubectl get managed`を使用します。

```shell {copy-lines="1"}
kubectl get managed
NAME                                                   READY   SYNCED   EXTERNAL-NAME                 AGE
bucket.s3.aws.upbound.io/crossplane-bucket-867737b10   True    True     crossplane-bucket-867737b10   8h
```

リソースを削除するには `kubectl delete` を使用します。

```shell
kubectl delete bucket.s3.aws.upbound.io/crossplane-bucket-867737b10
```

### Crossplane プロバイダーの削除

インストールされている _プロバイダー_ を `kubectl get providers` でリストします。

```shell {copy-lines="1"}
kubectl get providers
NAME                   INSTALLED   HEALTHY   PACKAGE                                        AGE
upbound-provider-aws   True        True      xpkg.upbound.io/upbound/provider-aws:v0.27.0   8h
```

インストールされている _プロバイダー_ を `kubectl delete provider` で削除します。

```shell
kubectl delete provider upbound-provider-aws
```

## Crossplane デプロイメントのアンインストール

`helm uninstall` を使用して Helm で Crossplane をアンインストールします。

```shell
helm uninstall crossplane --namespace crossplane-system
```

`kubectl get pods` で Helm が Crossplane ポッドを削除したことを確認します。

```shell
kubectl get pods -n crossplane-system
No resources found in crossplane-system namespace.
```

## Crossplane ネームスペースの削除

Helm が Crossplane をインストールすると、`crossplane-system` ネームスペースが作成されます。Helm は `helm uninstall` でこのネームスペースをアンインストールしません。

`kubectl delete namespace` で Crossplane ネームスペースを手動で削除します。

```shell
kubectl delete namespace crossplane-system
```

`kubectl get namespaces` で Kubernetes がネームスペースを削除したことを確認します。

```shell
kubectl get namespace
NAME              STATUS   AGE
default           Active   2m45s
kube-flannel      Active   2m42s
kube-node-lease   Active   2m47s
kube-public       Active   2m47s
kube-system       Active   2m47s
```
