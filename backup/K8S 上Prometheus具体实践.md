在 Kubernetes 上部署 Prometheus 时，推荐使用 **Prometheus Operator**，它可以极大地简化 Prometheus 的配置和管理。以下是具体的操作步骤和配置指南：

---

## **1. 准备环境**

1. **确保 Kubernetes 集群可用**：
   - 检查 Kubernetes 的版本是否支持 Operator（推荐 v1.20+）。
   - 配置 `kubectl` 与集群通信。

2. **安装 Helm**（推荐用于部署 Operator 和相关资源）：
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
   ```

3. **创建命名空间**：
   ```bash
   kubectl create namespace monitoring
   ```

---

## **2. 安装 Prometheus Operator**

### **使用 Helm 安装**
1. 添加 `kube-prometheus-stack` Helm 仓库：
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

2. 安装 `kube-prometheus-stack`：
   ```bash
   helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
   ```

3. 验证安装：
   ```bash
   kubectl get pods -n monitoring
   ```
   确保 Prometheus、Alertmanager、Grafana 等 Pod 都处于 Running 状态。

---

## **3. 配置 Prometheus Operator 资源**

Prometheus Operator 提供了多种 CRD（自定义资源定义），以下是常用资源的配置示例。

### **(1) 配置 ServiceMonitor**
`ServiceMonitor` 用于定义 Prometheus 如何抓取目标服务的指标。

#### 示例：监控命名空间 `default` 中的服务
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-service-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: example-app # 服务的标签选择器
  namespaceSelector:
    matchNames:
      - default          # 目标命名空间
  endpoints:
    - port: metrics      # 服务暴露的端口名称
      interval: 15s      # 抓取间隔
      path: /metrics     # 指标路径（默认是 /metrics）
```
- 部署后，Prometheus 自动发现符合条件的服务。

#### 应用配置：
```bash
kubectl apply -f service-monitor.yaml
```

---

### **(2) 配置 PrometheusRule**
`PrometheusRule` 用于定义告警规则。

#### 示例：节点 CPU 使用率告警规则
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cpu-alert-rule
  namespace: monitoring
spec:
  groups:
    - name: example-alerts
      rules:
        - alert: HighCpuUsage
          expr: 100 * (1 - avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) > 80
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "Instance {{ $labels.instance }} has high CPU usage"
            description: "CPU usage is above 80% for {{ $labels.instance }} for more than 1 minute."
```

#### 应用配置：
```bash
kubectl apply -f prometheus-rule.yaml
```

---

### **(3) 配置 PodMonitor**
`PodMonitor` 用于直接监控 Pod，而非通过 Service。

#### 示例：监控具有特定标签的 Pod
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: example-pod-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: example-app
  namespaceSelector:
    matchNames:
      - default
  podMetricsEndpoints:
    - path: /metrics
      port: metrics
      interval: 30s
```

#### 应用配置：
```bash
kubectl apply -f pod-monitor.yaml
```

---

## **4. 自定义 Prometheus 配置**

### **(1) 修改抓取配置**
`Prometheus` 的实例配置由 `Prometheus` CRD 定义，支持自定义参数。

#### 示例：修改 Prometheus 抓取配置
```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus-instance
  namespace: monitoring
spec:
  serviceMonitorSelector: {}
  podMonitorSelector: {}
  retention: 10d
  resources:
    requests:
      memory: 2Gi
    limits:
      memory: 4Gi
  replicas: 2  # 高可用配置
```

#### 应用配置：
```bash
kubectl apply -f prometheus-instance.yaml
```

---

## **5. 暴露 Prometheus 服务**

### **(1) 使用 NodePort 暴露服务**
编辑 `prometheus` 服务：
```bash
kubectl edit svc prometheus-operated -n monitoring
```
将 `type` 修改为 `NodePort`：
```yaml
spec:
  type: NodePort
```

获取 NodePort 和集群 IP：
```bash
kubectl get svc prometheus-operated -n monitoring
```

访问 Prometheus UI：
```
http://<NodeIP>:<NodePort>
```

### **(2) 使用 Ingress 暴露服务**
如果集群中已部署 Ingress Controller，可以通过 Ingress 暴露 Prometheus 服务。

#### 示例：创建 Ingress 规则
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: prometheus.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-operated
                port:
                  number: 9090
```

应用配置：
```bash
kubectl apply -f ingress.yaml
```

确保 `DNS` 或 `/etc/hosts` 配置正确，访问 `http://prometheus.example.com`。

---

## **6. 配置 Alertmanager**

`Alertmanager` 是 Prometheus 的告警管理组件，支持发送告警到邮件、Slack、Webhook 等。

### **(1) 配置 Alertmanager**
编辑 Alertmanager 配置：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
    route:
      receiver: 'default'
    receivers:
      - name: 'default'
        email_configs:
          - to: 'alert@example.com'
            from: 'prometheus@example.com'
            smarthost: 'smtp.example.com:587'
            auth_username: 'username'
            auth_password: 'password'
```

应用配置：
```bash
kubectl apply -f alertmanager-config.yaml
```

---

## **7. 集成 Grafana**

Grafana 是可视化 Prometheus 数据的最佳工具。

1. **访问 Grafana**
   Grafana 在 `kube-prometheus-stack` 中默认部署，查找服务名称：
   ```bash
   kubectl get svc -n monitoring
   ```

2. **登录 Grafana**
   默认用户名和密码：
   ```
   用户名: admin
   密码: prom-operator
   ```

3. **添加 Prometheus 数据源**
   - URL: `http://prometheus-operated:9090`
   - 数据源类型: Prometheus

4. **导入仪表盘**
   - Grafana 提供社区仪表盘，推荐导入 ID 为 `1860` 的 Kubernetes 监控模板。

---

通过上述配置，您可以在 Kubernetes 上成功部署和运行 Prometheus，并实现全面的监控与告警管理。在 Kubernetes 上部署 Prometheus 时，推荐使用 **Prometheus Operator**，它可以极大地简化 Prometheus 的配置和管理。以下是具体的操作步骤和配置指南：

---

## **1. 准备环境**

1. **确保 Kubernetes 集群可用**：
   - 检查 Kubernetes 的版本是否支持 Operator（推荐 v1.20+）。
   - 配置 `kubectl` 与集群通信。

2. **安装 Helm**（推荐用于部署 Operator 和相关资源）：
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
   ```

3. **创建命名空间**：
   ```bash
   kubectl create namespace monitoring
   ```

---

## **2. 安装 Prometheus Operator**

### **使用 Helm 安装**
1. 添加 `kube-prometheus-stack` Helm 仓库：
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

2. 安装 `kube-prometheus-stack`：
   ```bash
   helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
   ```

3. 验证安装：
   ```bash
   kubectl get pods -n monitoring
   ```
   确保 Prometheus、Alertmanager、Grafana 等 Pod 都处于 Running 状态。

---

## **3. 配置 Prometheus Operator 资源**

Prometheus Operator 提供了多种 CRD（自定义资源定义），以下是常用资源的配置示例。

### **(1) 配置 ServiceMonitor**
`ServiceMonitor` 用于定义 Prometheus 如何抓取目标服务的指标。

#### 示例：监控命名空间 `default` 中的服务
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-service-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: example-app # 服务的标签选择器
  namespaceSelector:
    matchNames:
      - default          # 目标命名空间
  endpoints:
    - port: metrics      # 服务暴露的端口名称
      interval: 15s      # 抓取间隔
      path: /metrics     # 指标路径（默认是 /metrics）
```
- 部署后，Prometheus 自动发现符合条件的服务。

#### 应用配置：
```bash
kubectl apply -f service-monitor.yaml
```

---

### **(2) 配置 PrometheusRule**
`PrometheusRule` 用于定义告警规则。

#### 示例：节点 CPU 使用率告警规则
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cpu-alert-rule
  namespace: monitoring
spec:
  groups:
    - name: example-alerts
      rules:
        - alert: HighCpuUsage
          expr: 100 * (1 - avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))) > 80
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "Instance {{ $labels.instance }} has high CPU usage"
            description: "CPU usage is above 80% for {{ $labels.instance }} for more than 1 minute."
```

#### 应用配置：
```bash
kubectl apply -f prometheus-rule.yaml
```

---

### **(3) 配置 PodMonitor**
`PodMonitor` 用于直接监控 Pod，而非通过 Service。

#### 示例：监控具有特定标签的 Pod
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: example-pod-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: example-app
  namespaceSelector:
    matchNames:
      - default
  podMetricsEndpoints:
    - path: /metrics
      port: metrics
      interval: 30s
```

#### 应用配置：
```bash
kubectl apply -f pod-monitor.yaml
```

---

## **4. 自定义 Prometheus 配置**

### **(1) 修改抓取配置**
`Prometheus` 的实例配置由 `Prometheus` CRD 定义，支持自定义参数。

#### 示例：修改 Prometheus 抓取配置
```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus-instance
  namespace: monitoring
spec:
  serviceMonitorSelector: {}
  podMonitorSelector: {}
  retention: 10d
  resources:
    requests:
      memory: 2Gi
    limits:
      memory: 4Gi
  replicas: 2  # 高可用配置
```

#### 应用配置：
```bash
kubectl apply -f prometheus-instance.yaml
```

---

## **5. 暴露 Prometheus 服务**

### **(1) 使用 NodePort 暴露服务**
编辑 `prometheus` 服务：
```bash
kubectl edit svc prometheus-operated -n monitoring
```
将 `type` 修改为 `NodePort`：
```yaml
spec:
  type: NodePort
```

获取 NodePort 和集群 IP：
```bash
kubectl get svc prometheus-operated -n monitoring
```

访问 Prometheus UI：
```
http://<NodeIP>:<NodePort>
```

### **(2) 使用 Ingress 暴露服务**
如果集群中已部署 Ingress Controller，可以通过 Ingress 暴露 Prometheus 服务。

#### 示例：创建 Ingress 规则
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: prometheus.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-operated
                port:
                  number: 9090
```

应用配置：
```bash
kubectl apply -f ingress.yaml
```

确保 `DNS` 或 `/etc/hosts` 配置正确，访问 `http://prometheus.example.com`。

---

## **6. 配置 Alertmanager**

`Alertmanager` 是 Prometheus 的告警管理组件，支持发送告警到邮件、Slack、Webhook 等。

### **(1) 配置 Alertmanager**
编辑 Alertmanager 配置：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
    route:
      receiver: 'default'
    receivers:
      - name: 'default'
        email_configs:
          - to: 'alert@example.com'
            from: 'prometheus@example.com'
            smarthost: 'smtp.example.com:587'
            auth_username: 'username'
            auth_password: 'password'
```

应用配置：
```bash
kubectl apply -f alertmanager-config.yaml
```

---

## **7. 集成 Grafana**

Grafana 是可视化 Prometheus 数据的最佳工具。

1. **访问 Grafana**
   Grafana 在 `kube-prometheus-stack` 中默认部署，查找服务名称：
   ```bash
   kubectl get svc -n monitoring
   ```

2. **登录 Grafana**
   默认用户名和密码：
   ```
   用户名: admin
   密码: prom-operator
   ```

3. **添加 Prometheus 数据源**
   - URL: `http://prometheus-operated:9090`
   - 数据源类型: Prometheus

4. **导入仪表盘**
   - Grafana 提供社区仪表盘，推荐导入 ID 为 `1860` 的 Kubernetes 监控模板。

---

通过上述配置，您可以在 Kubernetes 上成功部署和运行 Prometheus，并实现全面的监控与告警管理。