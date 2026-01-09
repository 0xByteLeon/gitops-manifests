# Dev 集群部署指南

本文档详细说明如何使用 k3d 在本地创建 dev 集群，并部署完整的 GitOps 基础设施和 OTEL LGTM 可观测性栈。

## 目录

- [前提条件](#前提条件)
- [一、安装必要工具](#一安装必要工具)
- [二、创建 k3d 集群](#二创建-k3d-集群)
- [三、安装 ArgoCD](#三安装-argocd)
- [四、配置 Git 仓库](#四配置-git-仓库)
- [五、部署应用](#五部署应用)
- [六、验证部署](#六验证部署)
- [七、访问服务](#七访问服务)
- [八、故障排除](#八故障排除)
- [九、清理环境](#九清理环境)

---

## 前提条件

| 工具 | 最低版本 | 用途 |
|------|---------|------|
| Docker | 20.10+ | 容器运行时 |
| k3d | 5.0+ | 本地 Kubernetes 集群 |
| kubectl | 1.25+ | Kubernetes CLI |
| kustomize | 5.0+ | 配置管理 |
| helm | 3.12+ | Helm Chart 渲染 |
| argocd CLI | 3.0+ | ArgoCD 命令行工具 |

---

## 一、安装必要工具

### 1.1 安装 Docker

**macOS:**
```bash
# 使用 Homebrew 安装 Docker Desktop
brew install --cask docker

# 启动 Docker Desktop
open /Applications/Docker.app

# 验证安装
docker --version
docker info
```

**Linux (Ubuntu/Debian):**
```bash
# 安装 Docker
curl -fsSL https://get.docker.com | sh

# 将当前用户添加到 docker 组
sudo usermod -aG docker $USER
newgrp docker

# 启动 Docker
sudo systemctl start docker
sudo systemctl enable docker

# 验证安装
docker --version
```

### 1.2 安装 k3d

```bash
# macOS
brew install k3d

# Linux
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# 验证安装
k3d version
```

### 1.3 安装 kubectl

```bash
# macOS
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# 验证安装
kubectl version --client
```

### 1.4 安装 Kustomize

```bash
# macOS
brew install kustomize

# Linux
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/

# 验证安装
kustomize version
```

### 1.5 安装 Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 验证安装
helm version
```

### 1.6 安装 ArgoCD CLI

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# 验证安装
argocd version --client
```

---

## 二、创建 k3d 集群

### 2.1 使用项目配置文件创建集群

项目根目录已包含 k3d 集群配置文件 `k3d-dev-config.yaml`:

```yaml
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: dev-cluster
servers: 1
agents: 2
image: rancher/k3s:v1.31.4-k3s1
ports:
  # Traefik Ingress HTTP
  - port: 80:80
    nodeFilters:
      - loadbalancer
  # Traefik Ingress HTTPS
  - port: 443:443
    nodeFilters:
      - loadbalancer
options:
  k3d:
    wait: true
    timeout: "120s"
  kubeconfig:
    updateDefaultKubeconfig: true
    switchCurrentContext: true
registries:
  create:
    name: dev-registry
    host: "0.0.0.0"
    hostPort: "5000"
```

> **说明**: k3s 默认包含 Traefik Ingress Controller，本配置保留 Traefik 以支持 Ingress 资源。同时创建本地容器镜像仓库。

### 2.2 创建集群

```bash
# 进入项目目录
cd /path/to/gitops-manifests

# 创建 k3d 集群
k3d cluster create --config k3d-dev-config.yaml

# 等待集群完全就绪（约 1-2 分钟）
echo "等待集群节点就绪..."
kubectl wait --for=condition=Ready nodes --all --timeout=300s
```

### 2.3 验证集群创建

```bash
# 检查集群状态
k3d cluster list

# 预期输出:
# NAME          SERVERS   AGENTS   LOADBALANCER
# dev-cluster   1/1       2/2      true

# 检查节点状态
kubectl get nodes

# 预期输出:
# NAME                       STATUS   ROLES                  AGE   VERSION
# k3d-dev-cluster-server-0   Ready    control-plane,master   1m    v1.31.4+k3s1
# k3d-dev-cluster-agent-0    Ready    <none>                 1m    v1.31.4+k3s1
# k3d-dev-cluster-agent-1    Ready    <none>                 1m    v1.31.4+k3s1

# 检查所有系统 Pod 运行状态 (包括 Traefik)
kubectl get pods -n kube-system

# 验证 Traefik Ingress Controller 运行中
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik

# 验证 kubectl context
kubectl config current-context
# 预期输出: k3d-dev-cluster
```

---

## 三、安装 ArgoCD

### 3.1 使用 Kustomize 安装 ArgoCD

本项目使用 ArgoCD v3.2.3，具有以下特性：
- 资源跟踪方法使用 annotation（v3.x 默认）
- 启用服务器端差异比对以提升性能
- Kustomize 构建选项启用 Helm 支持

```bash
# 进入项目目录
cd /path/to/gitops-manifests

# 使用 dev overlay 安装 ArgoCD
kubectl apply -k argocd/overlays/dev

# 等待 ArgoCD 组件就绪
echo "等待 ArgoCD 组件启动..."
kubectl wait --for=condition=Available deployment --all -n argocd --timeout=300s
```

### 3.2 验证 ArgoCD 安装

```bash
# 检查 ArgoCD 命名空间中的 Pod
kubectl get pods -n argocd

# 预期输出 (所有 Pod 应为 Running 状态):
# NAME                                                READY   STATUS    RESTARTS   AGE
# argocd-application-controller-0                    1/1     Running   0          2m
# argocd-applicationset-controller-xxx               1/1     Running   0          2m
# argocd-dex-server-xxx                              1/1     Running   0          2m
# argocd-notifications-controller-xxx                1/1     Running   0          2m
# argocd-redis-xxx                                   1/1     Running   0          2m
# argocd-repo-server-xxx                             1/1     Running   0          2m
# argocd-server-xxx                                  1/1     Running   0          2m

# 检查 ArgoCD 服务
kubectl get svc -n argocd

# 检查 ArgoCD Ingress
kubectl get ingress -n argocd
```

### 3.3 获取 ArgoCD 初始管理员密码

```bash
# 获取初始管理员密码
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD 初始密码: $ARGOCD_PASSWORD"

# 建议：记录此密码或立即修改
```

### 3.4 配置 ArgoCD CLI

```bash
# 方式一：通过端口转发访问
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
sleep 3
argocd login localhost:8080 \
  --username admin \
  --password $ARGOCD_PASSWORD \
  --insecure

# 方式二：通过 Ingress 访问（需要配置 hosts）
# 先添加 hosts 映射
echo "127.0.0.1 argocd.dev.local" | sudo tee -a /etc/hosts

# 然后登录
argocd login argocd.dev.local \
  --username admin \
  --password $ARGOCD_PASSWORD \
  --insecure

# 验证登录
argocd account list
```

---

## 四、配置 Git 仓库

### 4.1 确认仓库 URL

项目已配置使用以下 Git 仓库地址：

```
https://github.com/0xByteLeon/gitops-manifests.git
```

> **注意**: 如果需要修改仓库地址，可以使用以下命令批量替换：
> ```bash
> find argocd/apps -name "*.yaml" -exec sed -i 's|0xByteLeon/gitops-manifests|YOUR_ORG/YOUR_REPO|g' {} \;
> ```

### 4.2 添加 Git 仓库到 ArgoCD

**方式一：使用 SSH（推荐）**
```bash
# 使用 SSH 密钥添加仓库
argocd repo add git@github.com:0xByteLeon/gitops-manifests.git \
  --ssh-private-key-path ~/.ssh/id_rsa
```

**方式二：使用 HTTPS + Token**
```bash
argocd repo add https://github.com/0xByteLeon/gitops-manifests.git \
  --username git \
  --password <your-github-token>
```

**方式三：使用 HTTPS（公开仓库）**
```bash
argocd repo add https://github.com/0xByteLeon/gitops-manifests.git
```

### 4.3 添加 Helm 仓库

```bash
# 添加 Grafana Helm 仓库
argocd repo add https://grafana.github.io/helm-charts --type helm --name grafana

# 添加 Prometheus Helm 仓库
argocd repo add https://prometheus-community.github.io/helm-charts --type helm --name prometheus-community

# 添加 OpenTelemetry Helm 仓库
argocd repo add https://open-telemetry.github.io/opentelemetry-helm-charts --type helm --name open-telemetry

# 验证仓库添加成功
argocd repo list
```

---

## 五、部署应用

### 5.1 项目结构说明

本项目使用 **App-of-Apps** 模式和 **ApplicationSet** 实现自动化部署：

```
argocd/apps/
├── root-app.yaml        # 根应用 - 引导所有其他应用
├── argocd-self.yaml     # ArgoCD 自管理应用
├── dev-apps.yaml        # Dev 环境 ApplicationSet
├── qa-apps.yaml         # QA 环境 ApplicationSet
└── prod-apps.yaml       # Prod 环境 ApplicationSet
```

**AppProject 定义**：
- `platform`: 平台基础设施组件（LGTM 可观测性栈）
- `apps`: 业务应用

### 5.2 部署 Root Application

```bash
# 应用 root-app，这将自动引导所有其他应用
kubectl apply -f argocd/apps/root-app.yaml

# 验证 root-app 创建成功
argocd app get root-app
```

### 5.3 ApplicationSet 自动发现机制

Dev 环境使用 ApplicationSet 自动发现和部署应用：

**业务应用 (dev-apps)**:
- 自动扫描 `apps/*/overlays/dev` 目录
- 为每个发现的应用创建 ArgoCD Application
- 部署到 `dev` 命名空间

**平台组件 (dev-platform)**:
- 自动扫描 `platform/*/overlays/dev` 目录
- 每个组件部署到独立命名空间（prometheus, grafana, loki, tempo, opentelemetry）

### 5.4 手动同步应用（可选）

```bash
# 同步 root-app（将触发所有子应用同步）
argocd app sync root-app

# 或者单独同步特定应用
argocd app sync my-app-dev
argocd app sync prometheus-dev
argocd app sync grafana-dev
argocd app sync loki-dev
argocd app sync tempo-dev
argocd app sync otel-collector-dev
```

### 5.5 等待同步完成

```bash
# 等待所有应用同步完成
argocd app wait root-app --sync --timeout 600

# 查看所有应用状态
argocd app list

# 预期输出:
# NAME                CLUSTER                         NAMESPACE      PROJECT   STATUS  HEALTH   SYNCPOLICY
# root-app            https://kubernetes.default.svc  argocd         default   Synced  Healthy  Auto-Prune
# argocd              https://kubernetes.default.svc  argocd         default   Synced  Healthy  Auto-Prune
# my-app-dev          https://kubernetes.default.svc  dev            apps      Synced  Healthy  Auto-Prune
# prometheus-dev      https://kubernetes.default.svc  prometheus     platform  Synced  Healthy  Auto-Prune
# grafana-dev         https://kubernetes.default.svc  grafana        platform  Synced  Healthy  Auto-Prune
# loki-dev            https://kubernetes.default.svc  loki           platform  Synced  Healthy  Auto-Prune
# tempo-dev           https://kubernetes.default.svc  tempo          platform  Synced  Healthy  Auto-Prune
# otel-collector-dev  https://kubernetes.default.svc  opentelemetry  platform  Synced  Healthy  Auto-Prune
```

### 5.6 ArgoCD 自管理（可选）

项目包含 `argocd-self.yaml`，支持 ArgoCD 自管理模式：

```bash
# ArgoCD 自管理已通过 root-app 自动部署
# 验证 ArgoCD 自管理应用
argocd app get argocd

# 自管理的优势：
# - ArgoCD 配置变更通过 GitOps 流程管理
# - 版本升级可通过 Git 提交控制
# - 配置变更可追溯和回滚
```

---

## 六、验证部署

### 6.1 检查命名空间

```bash
# 查看所有命名空间
kubectl get namespaces

# 预期应包含以下命名空间:
# - argocd        (ArgoCD 组件)
# - dev           (业务应用)
# - prometheus    (Prometheus 监控)
# - grafana       (Grafana 可视化)
# - loki          (日志存储)
# - tempo         (链路追踪)
# - opentelemetry (OTel Collector)
```

### 6.2 检查 Dev 命名空间中的工作负载

```bash
# 检查 dev 命名空间中的所有资源
kubectl get all -n dev

# 检查 Pod 状态
kubectl get pods -n dev -o wide

# 检查 Deployment
kubectl get deployments -n dev

# 检查 Service
kubectl get svc -n dev
```

### 6.3 检查各平台组件命名空间中的工作负载

```bash
# 检查 Prometheus 命名空间
kubectl get pods -n prometheus
kubectl get pvc -n prometheus

# 检查 Grafana 命名空间
kubectl get pods -n grafana

# 检查 Loki 命名空间
kubectl get pods -n loki
kubectl get pvc -n loki

# 检查 Tempo 命名空间
kubectl get pods -n tempo
kubectl get pvc -n tempo

# 检查 OpenTelemetry 命名空间
kubectl get pods -n opentelemetry

# 快速检查所有平台组件 Pod 状态
for ns in prometheus grafana loki tempo opentelemetry; do
  echo "=== $ns ==="
  kubectl get pods -n $ns
done
```

### 6.4 验证 OTEL 可观测性栈

```bash
# 验证 Prometheus
kubectl get pods -n prometheus
kubectl logs -n prometheus -l app.kubernetes.io/name=prometheus --tail=20

# 验证 Grafana
kubectl get pods -n grafana
kubectl logs -n grafana -l app.kubernetes.io/name=grafana --tail=20

# 验证 Loki
kubectl get pods -n loki
kubectl logs -n loki -l app.kubernetes.io/name=loki --tail=20

# 验证 Tempo
kubectl get pods -n tempo
kubectl logs -n tempo -l app.kubernetes.io/name=tempo --tail=20

# 验证 OpenTelemetry Collector
kubectl get pods -n opentelemetry
kubectl logs -n opentelemetry -l app.kubernetes.io/name=opentelemetry-collector --tail=20
```

### 6.5 端到端验证

```bash
# 检查所有命名空间中是否有失败的 Pod
kubectl get pods -A | grep -v Running | grep -v Completed

# 如果有问题，查看 Pod 事件
kubectl describe pod <pod-name> -n <namespace>

# 检查 ArgoCD 应用健康状态
argocd app list --output wide
```

---

## 七、访问服务

### 7.1 ArgoCD UI

**方式一：通过 Ingress 访问（推荐）**

```bash
# 添加 hosts 映射（如果尚未添加）
echo "127.0.0.1 argocd.dev.local" | sudo tee -a /etc/hosts

# 访问: http://argocd.dev.local
# 用户名: admin
# 密码: 使用之前获取的 ARGOCD_PASSWORD
```

**方式二：端口转发**

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# 访问: https://localhost:8080
# 用户名: admin
# 密码: 使用之前获取的 ARGOCD_PASSWORD
```

### 7.2 Grafana

```bash
# 端口转发 Grafana
kubectl port-forward svc/grafana -n grafana 3000:80 &

# 访问: http://localhost:3000
# 默认用户名: admin
# 获取密码:
kubectl get secret grafana -n grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

### 7.3 Prometheus

```bash
# 端口转发 Prometheus
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n prometheus 9090:9090 &

# 访问: http://localhost:9090
```

### 7.4 Loki

```bash
# 端口转发 Loki (用于调试)
kubectl port-forward svc/loki -n loki 3100:3100 &

# 访问: http://localhost:3100/ready
```

### 7.5 Tempo

```bash
# 端口转发 Tempo (用于调试)
kubectl port-forward svc/tempo -n tempo 3200:3200 &

# 访问: http://localhost:3200/ready
```

### 7.6 应用服务

```bash
# 端口转发应用服务 (my-app 使用 nginx 镜像)
kubectl port-forward svc/my-app -n dev 8888:80 &

# 访问: http://localhost:8888

# 或通过 Ingress 访问 (需要配置 hosts 文件)
# 添加到 /etc/hosts: 127.0.0.1 my-app.dev.local
# 然后访问: http://my-app.dev.local
```

### 7.7 创建便捷访问脚本

创建一个脚本来启动所有端口转发：

```bash
cat > start-port-forwards.sh << 'EOF'
#!/bin/bash

echo "停止现有端口转发..."
pkill -f "kubectl port-forward" || true
sleep 2

echo "启动 ArgoCD 端口转发 (8080)..."
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

echo "启动 Grafana 端口转发 (3000)..."
kubectl port-forward svc/grafana -n grafana 3000:80 &

echo "启动 Prometheus 端口转发 (9090)..."
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n prometheus 9090:9090 &

echo "启动 Loki 端口转发 (3100)..."
kubectl port-forward svc/loki -n loki 3100:3100 &

echo "启动 Tempo 端口转发 (3200)..."
kubectl port-forward svc/tempo -n tempo 3200:3200 &

echo "启动 my-app 端口转发 (8888)..."
kubectl port-forward svc/my-app -n dev 8888:80 &

echo ""
echo "==========================================="
echo "服务访问地址:"
echo "==========================================="
echo "ArgoCD:     https://localhost:8080 或 http://argocd.dev.local"
echo "Grafana:    http://localhost:3000"
echo "Prometheus: http://localhost:9090"
echo "Loki:       http://localhost:3100/ready"
echo "Tempo:      http://localhost:3200/ready"
echo "my-app:     http://localhost:8888"
echo "==========================================="
echo ""
echo "按 Ctrl+C 停止所有端口转发"

wait
EOF

chmod +x start-port-forwards.sh
```

---

## 八、故障排除

### 8.1 常见问题

#### 问题 1: Pod 处于 Pending 状态

```bash
# 检查 Pod 事件
kubectl describe pod <pod-name> -n <namespace>

# 常见原因和解决方案:
# - 资源不足: 增加 k3d agent 节点或减少资源请求
# - PVC 无法绑定: 检查 StorageClass 配置
```

#### 问题 2: ArgoCD 应用同步失败

```bash
# 查看应用详细状态
argocd app get <app-name> --show-params

# 查看同步日志
argocd app logs <app-name>

# 强制同步
argocd app sync <app-name> --force

# 刷新应用状态
argocd app refresh <app-name>
```

#### 问题 3: Helm Chart 渲染失败

```bash
# 本地测试 Kustomize 渲染（需要启用 Helm）
cd apps/my-app/overlays/dev
kustomize build --enable-helm --load-restrictor=LoadRestrictionsNone .

# 检查 Helm values 文件语法
helm template my-app ../../base -f values-dev.yaml
```

#### 问题 4: 无法访问 Git 仓库

```bash
# 检查仓库连接状态
argocd repo list

# 测试仓库连接
argocd repo get <repo-url>

# 重新添加仓库凭据
argocd repo rm <repo-url>
argocd repo add <repo-url> --username <user> --password <token>
```

#### 问题 5: ApplicationSet 未生成应用

```bash
# 检查 ApplicationSet 状态
kubectl get applicationset -n argocd

# 查看 ApplicationSet 详情
kubectl describe applicationset dev-apps -n argocd
kubectl describe applicationset dev-platform -n argocd

# 检查 ApplicationSet Controller 日志
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-applicationset-controller --tail=50
```

### 8.2 日志查看

```bash
# ArgoCD 组件日志
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server --tail=100
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=100
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server --tail=100
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-applicationset-controller --tail=100

# 应用日志
kubectl logs -n dev -l app.kubernetes.io/name=my-app --tail=100

# 监控组件日志
kubectl logs -n prometheus -l app.kubernetes.io/name=prometheus --tail=100
kubectl logs -n grafana -l app.kubernetes.io/name=grafana --tail=100
kubectl logs -n loki -l app.kubernetes.io/name=loki --tail=100
kubectl logs -n tempo -l app.kubernetes.io/name=tempo --tail=100
kubectl logs -n opentelemetry -l app.kubernetes.io/name=opentelemetry-collector --tail=100
```

### 8.3 重置环境

```bash
# 删除并重新创建单个应用
argocd app delete <app-name> --cascade
kubectl apply -f argocd/apps/<app-file>.yaml

# 完全重置 ArgoCD
kubectl delete -k argocd/overlays/dev
kubectl apply -k argocd/overlays/dev
```

---

## 九、清理环境

### 9.1 删除所有 ArgoCD 应用

```bash
# 删除 root-app (将级联删除所有子应用)
argocd app delete root-app --cascade

# 或者强制删除
kubectl delete application --all -n argocd
kubectl delete applicationset --all -n argocd
```

### 9.2 删除 ArgoCD

```bash
kubectl delete -k argocd/overlays/dev
kubectl delete namespace argocd
```

### 9.3 删除 k3d 集群

```bash
# 删除集群
k3d cluster delete dev-cluster

# 验证删除
k3d cluster list

# 清理 kubeconfig
kubectl config delete-context k3d-dev-cluster
kubectl config delete-cluster k3d-dev-cluster
```

### 9.4 清理 Docker 资源

```bash
# 删除未使用的镜像
docker image prune -a

# 删除所有未使用的资源
docker system prune -a
```

---

## 附录

### A. 组件版本参考

| 组件 | Helm Chart 版本 | 命名空间 | 说明 |
|------|----------------|----------|------|
| ArgoCD | v3.2.3 | argocd | GitOps 引擎 |
| Prometheus Stack | 80.13.0 | prometheus | 监控指标收集 |
| Prometheus Operator CRDs | 25.0.1 | prometheus | CRD 定义 |
| Grafana | 10.5.3 | grafana | 可视化仪表板 |
| Loki | 6.49.0 | loki | 日志聚合 |
| Tempo | 1.24.1 | tempo | 分布式追踪 |
| OpenTelemetry Collector | 0.142.2 | opentelemetry | 遥测数据收集 |

### B. 端口映射参考

| 服务 | 本地端口 | 命名空间 | 用途 |
|------|---------|----------|------|
| ArgoCD Server | 8080 | argocd | ArgoCD UI |
| Grafana | 3000 | grafana | 可视化仪表板 |
| Prometheus | 9090 | prometheus | 指标查询 |
| Loki | 3100 | loki | 日志查询 |
| Tempo | 3200 | tempo | 链路追踪 |
| my-app | 8888 | dev | 业务应用 |

### C. Ingress 配置

Dev 环境使用 k3s 默认的 Traefik Ingress Controller。

| 服务 | 域名 | 备注 |
|------|------|------|
| ArgoCD | argocd.dev.local | HTTP 访问（dev 环境启用 insecure 模式）|
| my-app | my-app.dev.local | 需要添加 hosts 配置 |

```bash
# 添加本地 hosts 映射
echo "127.0.0.1 argocd.dev.local my-app.dev.local" | sudo tee -a /etc/hosts
```

### D. 有用的命令速查

```bash
# 查看集群信息
kubectl cluster-info

# 查看所有资源
kubectl get all -A

# 查看事件
kubectl get events -A --sort-by='.lastTimestamp'

# 资源使用情况
kubectl top nodes
kubectl top pods -A

# ArgoCD 应用状态
argocd app list
argocd app get <app-name>

# 刷新所有应用
argocd app list -o name | xargs -I {} argocd app refresh {}

# 快速检查所有平台组件状态
for ns in argocd dev prometheus grafana loki tempo opentelemetry; do
  echo "=== $ns ===" && kubectl get pods -n $ns --no-headers 2>/dev/null || echo "命名空间不存在"
done
```

### E. OTEL 数据流配置

应用通过 OpenTelemetry Collector 发送遥测数据：

```
应用 (dev namespace)
    │
    │ OTLP (gRPC :4317)
    ▼
OTel Collector (opentelemetry namespace)
    │
    ├──metrics──► Prometheus (prometheus namespace)
    ├──logs────► Loki (loki namespace)
    └──traces──► Tempo (tempo namespace)
                    │
                    ▼
               Grafana (grafana namespace)
```

应用中配置的 OTEL endpoint：
```
http://otel-collector-opentelemetry-collector.opentelemetry.svc:4317
```

### F. ArgoCD 配置说明

本项目 ArgoCD 配置特点：

1. **Kustomize 构建选项**:
   ```yaml
   kustomize.buildOptions: --enable-helm --load-restrictor=LoadRestrictionsNone
   ```
   - `--enable-helm`: 允许 Kustomize 处理 Helm Chart
   - `--load-restrictor=LoadRestrictionsNone`: 允许加载 overlay 目录外的文件

2. **资源跟踪方法**: 使用 annotation（v3.x 默认）

3. **服务器端差异比对**: 启用以提升大规模部署的性能

4. **自管理模式**: 通过 `argocd-self.yaml` 实现 ArgoCD 自身的 GitOps 管理

### G. AppProject 权限说明

| Project | 源仓库 | 目标命名空间 | 用途 |
|---------|-------|-------------|------|
| platform | gitops-manifests, helm charts | prometheus, grafana, loki, tempo, opentelemetry | 平台基础设施 |
| apps | gitops-manifests | dev, qa, prod | 业务应用 |

---

## 更新日志

| 日期 | 版本 | 更新内容 |
|------|------|---------|
| 2026-01-09 | 1.2.0 | ArgoCD 升级至 v3.2.3、添加 ApplicationSet 说明、更新 ArgoCD 自管理配置、更新所有组件版本 |
| 2026-01-08 | 1.1.0 | 更新命名空间结构（每组件独立命名空间）、更新 Helm Chart 版本、添加 Ingress 配置说明 |
| 2026-01-07 | 1.0.0 | 初始版本 |
