---  
title: CrossplaneのArgo CDの設定
weight: 270
---  

[Argo CD](https://argoproj.github.io/cd/) と [Crossplane](https://crossplane.io) は素晴らしい組み合わせです。Argo CDはGitOpsを提供し、Crossplaneは任意のKubernetesクラスターをすべてのリソースのためのユニバーサルコントロールプレーンに変えます。両者が適切に連携するためには、いくつかの設定が必要です。このドキュメントは、これらの要件を理解するのに役立ちます。Crossplaneと共に使用する場合は、Argo CDのバージョン2.4.8以降を使用することをお勧めします。

Argo CDは、Gitリポジトリに保存されたKubernetesリソースマニフェストをKubernetesクラスターで実行されているものと同期します（GitOps）。Argo CDがリソースを追跡する方法を設定するには、いくつかの方法があります。Crossplaneを使用する場合、Argo CDをアノテーションベースのリソース追跡を使用するように設定する必要があります。詳細については、[Argo CDのドキュメント](https://argo-cd.readthedocs.io/en/latest/user-guide/resource_tracking/)を参照してください。

### CrossplaneとのArgo CDの設定

#### リソース追跡方法の設定

Argo CDがCrossplane関連のオブジェクトを含むアプリケーションリソースを正しく追跡するためには、アノテーションメカニズムを使用するように設定する必要があります。

設定するには、`argocd` `Namespace`内の`argocd-cm` `ConfigMap`を次のように編集します：
```yaml
apiVersion: v1
kind: ConfigMap
data:
  application.resourceTrackingMethod: annotation
```

#### 健康状態の設定

Argo CDにはKubernetesリソースのための組み込みの健康評価があります。一部のチェックは、Argoの[リポジトリ](https://github.com/argoproj/argo-cd/tree/master/resource_customizations)でコミュニティによって直接サポートされています。例えば、`pkg.crossplane.io`の`Provider`はすでに宣言されているため、追加の設定は必要ありません。

Argo CDは、インスタンスごとにこれらのチェックをカスタマイズすることも可能で、これがProviderのCRDをサポートするために使用されるメカニズムです。

設定するには、`argocd` `Namespace`内の`argocd-cm` `ConfigMap`を編集します。
{{<hint "note">}}
{{<hover label="argocfg" line="22">}} ProviderConfig{{</hover>}}にはステータスまたは`status.users`フィールドがない場合があります。
{{</hint>}}
```yaml {label="argocfg"}
apiVersion: v1
kind: ConfigMap
data:
  application.resourceTrackingMethod: annotation
  resource.customizations: |
    "*.upbound.io/*":
      health.lua: |
        health_status = {
          status = "Progressing",
          message = "Provisioning ..."
        }

        local function contains (table, val)
          for i, v in ipairs(table) do
            if v == val then
              return true
            end
          end
          return false
        end

        local has_no_status = {
          "ProviderConfig",
          "ProviderConfigUsage"
        }

        if obj.status == nil or next(obj.status) == nil and contains(has_no_status, obj.kind) then
          health_status.status = "Healthy"
          health_status.message = "Resource is up-to-date."
          return health_status
        end

        if obj.status == nil or next(obj.status) == nil or obj.status.conditions == nil then
          if obj.kind == "ProviderConfig" and obj.status.users ~= nil then
            health_status.status = "Healthy"
            health_status.message = "Resource is in use."
            return health_status
          end
          return health_status
        end

        for i, condition in ipairs(obj.status.conditions) do
          if condition.type == "LastAsyncOperation" then
            if condition.status == "False" then
              health_status.status = "Degraded"
              health_status.message = condition.message
              return health_status
            end
          end

          if condition.type == "Synced" then
            if condition.status == "False" then
              health_status.status = "Degraded"
              health_status.message = condition.message
              return health_status
            end
          end

          if condition.type == "Ready" then
            if condition.status == "True" then
              health_status.status = "Healthy"
              health_status.message = "Resource is up-to-date."
              return health_status
            end
          end
        end

        return health_status

    "*.crossplane.io/*":
      health.lua: |
        health_status = {
          status = "Progressing",
          message = "Provisioning ..."
        }

        local function contains (table, val)
          for i, v in ipairs(table) do
            if v == val then
              return true
            end
          end
          return false
        end

        local has_no_status = {
          "Composition",
          "CompositionRevision",
          "DeploymentRuntimeConfig",
          "ControllerConfig",
          "ProviderConfig",
          "ProviderConfigUsage"
        }
        if obj.status == nil or next(obj.status) == nil and contains(has_no_status, obj.kind) then
            health_status.status = "Healthy"
            health_status.message = "Resource is up-to-date."
          return health_status
        end

        if obj.status == nil or next(obj.status) == nil or obj.status.conditions == nil then
          if obj.kind == "ProviderConfig" and obj.status.users ~= nil then
            health_status.status = "Healthy"
            health_status.message = "Resource is in use."
            return health_status
          end
          return health_status
        end

        for i, condition in ipairs(obj.status.conditions) do
          if condition.type == "LastAsyncOperation" then
            if condition.status == "False" then
              health_status.status = "Degraded"
              health_status.message = condition.message
              return health_status
            end
          end

          if condition.type == "Synced" then
            if condition.status == "False" then
              health_status.status = "Degraded"
              health_status.message = condition.message
              return health_status
            end
          end

          if contains({"Ready", "Healthy", "Offered", "Established"}, condition.type) then
            if condition.status == "True" then
              health_status.status = "Healthy"
              health_status.message = "Resource is up-to-date."
              return health_status
            end
          end
        end

        return health_status
```

#### リソース除外の設定

Crossplane プロバイダーは、管理リソース (MR) ごとに `ProviderConfigUsage` を生成します。このリソースは、MR と ProviderConfig の関係を表現できるようにし、コントローラーが ProviderConfig が削除されたときにファイナライザーとして使用できるようにします。Crossplane のエンドユーザーは、このリソースと対話することは期待されていません。

リソースとタイプの数が増えると、Argo CD UI の反応性に影響を与える可能性があります。この数を低く保つために、すべての `ProviderConfigUsage` リソースを Argo CD UI から隠すことをお勧めします。

リソース除外を設定するには、`argocd` 名前空間の `argocd-cm` `ConfigMap` を次のように編集します：
```yaml
apiVersion: v1
kind: ConfigMap
data:
    resource.exclusions: |
      - apiGroups:
        - "*"
        kinds:
        - ProviderConfigUsage
```

`"*"` を apiGroups として使用することで、すべての Crossplane プロバイダーに対してメカニズムが有効になります。

#### K8s クライアント QPS の増加

コントロールプレーン上の CRD の数が増えると、Argo CD アプリケーションコントローラーが Kubernetes API に送信する必要があるクエリの量が増加します。この場合、Argo CD Kubernetes クライアントのレート制限を増加させることができます。

多数の CRD との互換性を向上させるために、環境変数 `ARGOCD_K8S_CLIENT_QPS` を `300` に設定します。

`ARGOCD_K8S_CLIENT_QPS` のデフォルト値は 50 であり、この値を変更すると `ARGOCD_K8S_CLIENT_BURST` も更新されます。これはデフォルトで `ARGOCD_K8S_CLIENT_QPS` x 2 です。
