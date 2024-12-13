要实现基于 x86 架构构建支持 x86 和 ARM 混合架构的 Docker 镜像，并确保不同架构的设备拉取匹配的镜像，可以使用 Docker 的 Buildx 工具进行多架构构建并推送到镜像仓库。

以下是具体实现步骤：

---

### 1. **准备环境**

#### **确保 Docker 支持 Buildx**
确保你的 Docker 版本支持 `Buildx`，推荐 Docker 版本 19.03+。

启用 `Buildx`：
```bash
docker buildx create --name multiarch-builder --use
docker buildx inspect --bootstrap
```

#### **安装 QEMU**
QEMU 可以在 x86 主机上模拟 ARM 架构：
```bash
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

验证 QEMU 安装：
```bash
docker run --rm -t arm64v8/ubuntu uname -m
```
如果输出是 `aarch64`，说明 ARM 模拟环境已经正确配置。

---

### 2. **编写 Dockerfile**
编写一个通用的 `Dockerfile`，无需为不同架构单独维护文件。例如：

```dockerfile
FROM ubuntu:20.04

# 安装一些通用的依赖
RUN apt-get update && apt-get install -y curl

CMD ["bash"]
```

---

### 3. **登录镜像仓库**
登录到目标镜像仓库（例如 Docker Hub、阿里云镜像仓库或华为云 SWR）：
```bash
docker login -u <用户名> -p <密码> <仓库地址>
```

---

### 4. **使用 Buildx 构建多架构镜像**
运行以下命令，构建并推送多架构镜像：

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t <仓库地址>/<命名空间>/<镜像名称>:<标签> \
  --push .
```

- **`--platform`**: 指定支持的目标架构。
  - `linux/amd64`: 适用于 x86 架构。
  - `linux/arm64`: 适用于 ARM 架构。
- **`-t`**: 指定镜像的名称和标签。
- **`--push`**: 构建完成后直接推送到镜像仓库。

例如，将镜像推送到华为云 SWR：

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t cn-north-4.myhuaweicloud.com/myproject/multiarch-image:latest \
  --push .
```

---

### 5. **验证镜像**
在镜像仓库中查看镜像是否成功推送，并验证支持的架构。

#### 查看支持的架构
可以使用以下命令检查镜像的架构信息：
```bash
docker manifest inspect <镜像地址>
```

输出中应包含：
```json
"platform": {
    "architecture": "amd64",
    "os": "linux"
},
"platform": {
    "architecture": "arm64",
    "os": "linux"
}
```

---

### 6. **拉取镜像时的架构匹配**
Docker 在拉取镜像时会根据客户端设备的架构自动选择匹配的镜像：
- 如果客户端是 x86（`linux/amd64`），会拉取 x86 的镜像。
- 如果客户端是 ARM（`linux/arm64`），会拉取 ARM 的镜像。

#### 测试拉取
在 x86 和 ARM 主机上分别运行：
```bash
docker pull <镜像地址>
```

然后验证镜像架构：
```bash
docker run --rm <镜像地址> uname -m
```

- x86 主机应返回 `x86_64`。
- ARM 主机应返回 `aarch64`。

---

### 示例总结

1. **创建多架构镜像并推送到仓库**
   ```bash
   docker buildx build --platform linux/amd64,linux/arm64 \
     -t cn-north-4.myhuaweicloud.com/demo/multiarch-image:latest \
     --push .
   ```

2. **验证镜像的多架构支持**
   ```bash
   docker manifest inspect cn-north-4.myhuaweicloud.com/demo/multiarch-image:latest
   ```

3. **拉取和运行测试**
   - 在 x86 主机：
     ```bash
     docker pull cn-north-4.myhuaweicloud.com/demo/multiarch-image:latest
     docker run --rm cn-north-4.myhuaweicloud.com/demo/multiarch-image:latest uname -m
     ```
   - 在 ARM 主机：
     ```bash
     docker pull cn-north-4.myhuaweicloud.com/demo/multiarch-image:latest
     docker run --rm cn-north-4.myhuaweicloud.com/demo/multiarch-image:latest uname -m
     ```

通过以上步骤，可以实现构建并推送多架构镜像，确保不同架构的设备能自动拉取到适配的镜像版本。