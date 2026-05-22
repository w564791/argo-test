# argo-test:多集群 Rollout 演示

ArgoCD ApplicationSet 把同一份 Rollout 部署到 3 个 sub-cluster,replica 各 3 / 2 / 1。
canary 步骤由各集群本地的 argo-rollouts controller 执行。

## 结构

```
apps/                     # 一个目录一个应用
  gateway/
    base/                 # Rollout + Service (image: nginx:1.25, replicas: 1)
    overlays/sub-{1,2,3}/ # Kustomize 覆盖 replicas (3/2/1) 与 CLUSTER_NAME env
argocd/                   # 控制层,在 control 集群 apply
  appset-gateway.yaml     # ApplicationSet,生成 3 个 Application
```

加新应用时:复制 `apps/gateway` 为 `apps/<name>`,再加一份 `argocd/appset-<name>.yaml`,互不干扰。

## 部署

```bash
# 1. push 本仓库到 GitHub(分支 main)

# 2. 在 control 集群创建 ApplicationSet
kubectl --context=kind-argo-control apply -f argocd/appset-gateway.yaml

# 3. 观察生成的 3 个 Application
argocd app list
argocd app get gateway-sub-1

# 4. 查看每个集群里的 Rollout 状态
for n in 1 2 3; do
  echo "=== sub-cluster-$n ==="
  kubectl --context=kind-argo-sub-cluster-$n -n gateway get rollout,svc,pod
done
```

## 触发 canary

改 `apps/gateway/base/rollout.yaml` 的 `image: nginx:1.25` → `nginx:1.27`,提交并 push:

```bash
git commit -am "bump nginx to 1.27"
git push
```

ArgoCD 同步后,3 个集群各自走 25% → 50% → 75% → 100%(每步 30s pause)。

```bash
# 用 kubectl-argo-rollouts 插件实时观察(单集群)
kubectl argo rollouts --context=kind-argo-sub-cluster-1 -n gateway get rollout gateway --watch
```
