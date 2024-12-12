使用 Helm 安装和管理 Istio 是一种非常灵活的方式，适合需要自定义配置的用户。以下是通过 Helm 安装 Istio 并配置国内镜像的详细步骤：

---

### 1. **准备工作**
- 确保已经安装以下工具：
  - Helm (推荐版本 3.x 或更高)
  - Kubernetes 集群（建议 1.24 或更高）
  - kubectl 工具
- 如果你的集群在国内，需要设置镜像加速。

---

### 2. **添加 Istio Helm 仓库**
Istio 提供了官方 Helm Chart，可以直接添加：
```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

如果需要使用国内镜像，可以下载 Istio 的 Helm Chart，手动修改后部署。

---

### 3. **下载 Istio Helm Chart**
使用以下命令下载 Istio 的 Helm Chart：
```bash
helm pull istio/base --untar
helm pull istio/istiod --untar
helm pull istio/gateway --untar
```

这会分别下载以下组件：
- **base**: 基础 CRD 定义。
- **istiod**: Istio 控制平面。
- **gateway**: Istio 数据平面（网关）。

---

### 4. **修改 Helm Chart 使用国内镜像**
进入解压的 Chart 文件夹，找到 `values.yaml` 文件，并修改镜像前缀：

#### 示例修改：
将镜像源从默认的 `docker.io` 或 `gcr.io` 替换为国内源（如阿里云或中科大）：
```yaml
global:
  hub: registry.cn-hangzhou.aliyuncs.com/istio
  tag: 1.19.1
```

---

### 5. **安装 Istio**
先安装基础 CRD：
```bash
helm install istio-base ./base -n istio-system --create-namespace
```

然后安装控制平面：
```bash
helm install istiod ./istiod -n istio-system
```

最后安装入口网关：
```bash
helm install istio-ingress ./gateway -n istio-system
```

---

### 6. **验证安装**
检查 Pod 是否正常运行：
```bash
kubectl get pods -n istio-system
```

如果出现错误，可以检查日志：
```bash
kubectl logs -n istio-system <pod-name>
```

---

### 7. **配置国内镜像源**
如果未预先修改镜像源，可以使用 Helm 覆盖值的方式直接替换：
```bash
helm install istiod ./istiod -n istio-system \
  --set global.hub=registry.cn-hangzhou.aliyuncs.com/istio \
  --set global.tag=1.19.1
```

这样会在安装时直接指定国内镜像。

---

### 8. **清理安装**
如果需要重新安装或清理：
```bash
helm uninstall istio-base -n istio-system
helm uninstall istiod -n istio-system
helm uninstall istio-ingress -n istio-system
```

清理后可以重新安装或调整配置。

---

通过 Helm 安装 Istio 时，修改 `values.yaml` 或使用 `--set` 参数来设置国内镜像源，可以有效解决国内环境中网络限制的问题。