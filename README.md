# MetalLB Gateway Helm Chart

一个用于在 Kubernetes 集群上部署 MetalLB 和 Traefik Gateway 的 Helm Chart，支持完整的负载均衡和 ingress 功能，并提供离线安装支持。

## 功能特性

- **MetalLB**: 为裸机 Kubernetes 集群提供 LoadBalancer 类型的服务
- **Traefik**: 现代化的 HTTP 反向代理和负载均衡器
- **Gateway API**: 支持 Kubernetes Gateway API 标准（v1.4.1）
- **Layer2 模式**: MetalLB 默认使用 Layer2 模式，无需额外路由配置
- **离线安装**: 支持完全离线环境部署
- **开箱即用**: 预配置合理的默认值

## 架构组件

| 组件        | 版本    | 说明                            |
| ----------- | ------- | ------------------------------- |
| MetalLB     | v0.15.3 | 负载均衡器，提供 IP 地址管理    |
| Traefik     | v3.6.5  | Gateway API 实现，HTTP 反向代理 |
| FRR         | 10.4.1  | BGP 路由支持（可选）            |
| Gateway API | v1.4.1  | Kubernetes Gateway API CRDs     |

## 前置要求

- Kubernetes 1.25+
- Helm 3.0+
- 集群中未安装其他 LoadBalancer 实现（如 MetalLB、OpenShift Router 等）

## 安装

### 1. 准备 Chart 依赖

```bash
# 克隆或下载此 chart
cd metallb-gateway

# 更新依赖（下载本地 charts）
helm dependency update
```

### 2. 配置 IP 地址池

编辑 `values.yaml`，修改 MetalLB IP 地址池配置：

```yaml
metallb:
  ipAddressPools:
    - name: default-pool
      protocol: layer2
      addresses:
        - 192.168.1.100-192.168.1.200 # 修改为你的网络环境可用 IP
```

⚠️ **重要**: 确保 IP 地址在你的网络环境中未被使用，且与你的节点在同一网段。

### 3. 安装 Chart

```bash
# 安装到自定义命名空间
helm upgrade --install metallb-gateway . -n metallb-system --create-namespace -f values.yaml
```

### 4. 验证安装

```bash
# 检查 pods 状态
kubectl get pods -n metallb-system
kubectl get pods -n traefik-system

# 检查 MetalLB 组件
kubectl get IPAddressPool
kubectl get L2Advertisement

# 检查 GatewayClass
kubectl get gatewayclass
```

## 快速开始

### 方式 1: 使用内置 Whoami 示例（推荐）

安装时启用 whoami 示例应用，快速验证 MetalLB 和 Gateway API：

```bash
helm upgrade --install metallb-gateway . -n metallb-system --set whoami.enabled=true
```

等待几分钟让所有 Pod 启动，然后验证：

#### 验证 MetalLB LoadBalancer

```bash
# 1. 获取 whoami 的外部 IP
export WHOAMI_IP=$(kubectl get svc whoami -n metallb-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# 2. 访问应用（多次调用可以看到不同 Pod 响应）
curl http://${WHOAMI_IP}/
# 输出示例:
# hostname: whoami-7d6f8c5b9d-abc123
# ...

# 3. 在浏览器中访问
echo "访问: http://${WHOAMI_IP}/"
```

#### 验证 Gateway API

```bash
# 1. 获取 Traefik Gateway 外部 IP
export GATEWAY_IP=$(kubectl get svc -n metallb-system metallb-gateway-traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# 2. 通过 Gateway API 访问（使用 Host 头）
curl -H "Host: whoami.local" http://${GATEWAY_IP}/

# 3. 配置本地 hosts（可选，用于浏览器访问）
echo "${GATEWAY_IP} whoami.local" | sudo tee -a /etc/hosts
# 然后在浏览器访问: http://whoami.local/
```

#### 查看所有资源

```bash
kubectl get all,gateway,httproute -n metallb-system | grep whoami
```

#### 清理示例应用

```bash
# 方式 1: 通过 helm 禁用
helm upgrade metallb-gateway . --set whoami.enabled=false

# 方式 2: 手动删除
kubectl delete deployment,service,gateway,httproute -l app=whoami -n metallb-system
```

### 方式 2: 手动创建 LoadBalancer 服务

```bash
# 部署测试应用
kubectl create deployment nginx --image=nginx

# 暴露为 LoadBalancer
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# 获取外部 IP
kubectl get svc nginx
```

### 方式 3: 手动创建 Gateway API 资源

创建 `example-gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: metallb-gateway-traefik
  listeners:
    - name: web
      port: 80
      protocol: HTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
    - name: my-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: nginx
          port: 80
```

```bash
kubectl apply -f example-gateway.yaml
```

## 配置说明

### MetalLB 配置

#### Layer2 模式（默认）

```yaml
metallb:
  ipAddressPools:
    - name: default-pool
      protocol: layer2
      addresses:
        - 10.0.0.100-10.0.0.150
      autoAssign: true

  l2Advertisements:
    - name: default
      ipAddressPoolSelectors:
        - matchLabels:
            metallb.io/address-pool: default-pool
```

#### BGP 模式

```yaml
metallb:
  ipAddressPools:
    - name: bgp-pool
      protocol: bgp
      addresses:
        - 10.0.100.0-10.0.100.255

  bgp:
    peers:
      - my-asn: 64500
        peer-asn: 64501
        peer-address: 192.168.1.1
        source-address: 192.168.1.2
```

### Traefik 配置

#### 启用 HTTPS

```yaml
traefik:
  gateway:
    listeners:
      websecure:
        port: 443
        protocol: HTTPS
        mode: Terminate

  ports:
    websecure:
      port: 443
      expose:
        default: true
      exposedPort: 443
      protocol: TCP
```

#### 使用 Let's Encrypt

```yaml
traefik:
  certificatesResolvers:
    letsencrypt:
      acme:
        email: your-email@example.com
        storage: /acme.json
        httpChallenge:
          entryPoint: web
```

### 离线安装配置

对于离线环境，配置私有镜像仓库：

```yaml
global:
  imageRegistry: "dockerhub.kubekey.local"
  imagePullSecrets: []
```

推送镜像到私有仓库（如果需要）：

```bash
# 导出镜像
docker save quay.io/metallb/controller:v0.15.3 -o metallb-controller.tar
docker save quay.io/metallb/speaker:v0.15.3 -o metallb-speaker.tar
docker save quay.io/frrouting/frr:10.4.1 -o frr.tar
docker save traefik:v3.6.5 -o traefik.tar

# 导入到离线仓库
docker load -i metallb-controller.tar
docker tag quay.io/metallb/controller:v0.15.3 harbor.example.com/metallb/controller:v0.15.3
docker push harbor.example.com/metallb/controller:v0.15.3
# ... 其他镜像类似
```

## 访问 Traefik Dashboard

### 方式 1: Port Forward

```bash
kubectl port-forward -n traefik-system deployment/metallb-gateway-traefik 9000:9000
```

访问: http://localhost:9000/dashboard/

### 方式 2: 通过 LoadBalancer

```bash
# 获取 Traefik 外部 IP
kubectl get svc -n traefik-system metallb-gateway-traefik

# 访问 Dashboard（注意：需要确保安全配置）
curl http://<EXTERNAL-IP>:9000/dashboard/
```

⚠️ **安全提示**: 默认情况下 Dashboard 是不安全的，仅用于开发/测试环境。生产环境请配置认证。

## 故障排查

### MetalLB 无法分配 IP

```bash
# 检查 IP 地址池
kubectl get IPAddressPool -o yaml

# 检查 Speaker 日志
kubectl logs -n metallb-system -l app.kubernetes.io/component=speaker

# 检查是否有 IP 冲突
kubectl describe ipaddresspool default-pool
```

### LoadBalancer 服务一直处于 Pending 状态

```bash
# 检查 MetalLB Controller
kubectl logs -n metallb-system deployment/metallb-gateway-metallb-controller

# 检查 IPAddressPool 配置
kubectl get IPAddressPool

# 确认 IP 地址范围与节点在同一网段
kubectl get nodes -o wide
```

### Traefik Gateway 无法访问

```bash
# 检查 GatewayClass
kubectl get gatewayclass

# 检查 Gateway 状态
kubectl get gateway
kubectl describe gateway <gateway-name>

# 检查 Traefik 日志
kubectl logs -n traefik-system deployment/metallb-gateway-traefik
```

## 升级

```bash
# 升级 chart
helm upgrade metallb-gateway . -f values.yaml

# 或指定新的 values 文件
helm upgrade metallb-gateway . -f values-new.yaml --reuse-values
```

## 卸载

```bash
# 卸载 chart
helm uninstall metallb-gateway

# 删除命名空间（可选）
kubectl delete namespace metallb-system
kubectl delete namespace traefik-system

# 删除 Gateway API CRDs（如果需要）
kubectl delete crd -l gateway.networking.k8s.io/bundle-version=v1.4.1
```

## 配置参考

### 关键参数

| 参数                     | 默认值         | 说明                |
| ------------------------ | -------------- | ------------------- |
| `global.imageRegistry`   | `""`           | 全局镜像仓库地址    |
| `metallb.enabled`        | `true`         | 启用 MetalLB        |
| `metallb.ipAddressPools` | 见 values.yaml | IP 地址池配置       |
| `traefik.enabled`        | `true`         | 启用 Traefik        |
| `gatewayAPI.enabled`     | `true`         | 启用 Gateway API    |
| `gatewayAPI.installCRDs` | `false`        | 通过 Helm 安装 CRDs |
| `whoami.enabled`         | `false`        | 启用示例应用        |

### Whoami 示例应用配置

| 参数                     | 默认值            | 说明                  |
| ------------------------ | ----------------- | --------------------- |
| `whoami.enabled`         | `false`           | 启用示例应用          |
| `whoami.service.type`    | `LoadBalancer`    | 服务类型              |
| `whoami.deployment.replicas` | `2`          | 副本数                |
| `whoami.gateway.enabled` | `true`            | 创建 Gateway 资源     |
| `whoami.gateway.host`    | `whoami.local`    | Gateway 主机名        |

完整配置选项请查看 [values.yaml](./values.yaml)。

## 常见问题

### Q: 我应该使用 Layer2 还是 BGP 模式？

**A**:

- **Layer2**: 适合小型集群、简单网络环境，无需路由器配置，但单点故障
- **BGP**: 适合生产环境、需要高可用性，需要路由器支持 BGP

### Q: 可以更改已分配的 IP 地址吗？

**A**: 可以，但需要重启服务：

```bash
kubectl delete svc <service-name>
# 重新创建服务或等待自动重新分配
```

### Q: 支持多网卡环境吗？

**A**: MetalLB Speaker 会自动检测所有网卡。如果需要指定网卡，配置 `speaker.speakerMetricsContainer` 的 `--address-pool` 参数。

### Q: 如何监控 MetalLB 和 Traefik？

**A**: 启用 Prometheus 监控：

```yaml
metallb:
  prometheus:
    serviceMonitor:
      enabled: true

traefik:
  serviceMonitor:
    enabled: true
```

## 项目结构

```
metallb-gateway/
├── Chart.yaml              # Helm Chart 元数据
├── values.yaml             # 默认配置值
├── charts/                 # 依赖的 charts（离线安装）
│   ├── metallb/
│   └── traefik/
├── crds/                   # Gateway API CRDs
│   └── gateway-api-v1.4.1.yaml
├── templates/              # Helm 模板
│   ├── NOTES.txt           # 安装后提示信息
│   ├── _helpers.tpl        # 模板助手
│   ├── namespaces.yaml     # 命名空间创建
│   ├── gateway-api/        # Gateway API 相关
│   └── metallb/            # MetalLB 资源配置
└── README.md               # 本文档
```

## 相关资源

- [MetalLB 官方文档](https://metallb.universe.tf/)
- [Traefik 官方文档](https://doc.traefik.io/traefik/)
- [Gateway API 规范](https://gateway-api.sigs.k8s.io/)
- [Helm 文档](https://helm.sh/docs/)
