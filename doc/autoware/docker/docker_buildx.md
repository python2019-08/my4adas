# 1.docker buildx
`buildx` 是Docker CLI插件，底层使用 **BuildKit** 构建引擎。
核心能力：
1. **多架构（multi‑platform）镜像构建**（amd64 / arm64 一命令产出）
2. 高级构建缓存、并行构建、多输出格式
3. 支持OCI镜像规范、manifest清单镜像

> 普通`docker build`是旧构建器；buildx全程使用BuildKit，不需要设置`DOCKER_BUILDKIT=1`环境变量。

## 1. 初始化builder（Ubuntu/Docker Engine）
Docker Desktop自带buildx；原生Docker Engine需要创建专用builder实例（driver=docker‑container，会跑一个buildkit容器）。
```bash
# 查看现有builder
docker buildx ls

# 创建builder实例，使用docker‑container驱动
docker buildx create --name mybuilder --driver docker-container --use

# 启动/引导builder，安装QEMU模拟器用于跨架构仿真
docker buildx inspect --bootstrap
```
> ⚠️默认`docker`驱动**不支持多平台push**，必须用`docker‑container`驱动。

## 2. 关键参数
- `--platform linux/amd64,linux/arm64`：指定目标架构列表
- `--push`：构建完成直接推送到镜像仓库（**多架构镜像必须push，不能存本地docker镜像库**）
- `--load`：把镜像加载到本地docker镜像，**仅支持单个platform**，多platform不能用‑‑load
- `--cache‑from` / `--cache‑to`：远程构建缓存（registry、local等）

## 3. 常用示例

### ① 单平台构建（替代docker build）
```bash
docker buildx build -t myapp:v1 -f Dockerfile . --load
```

### ② 多架构构建并推送到仓库（最常用）
```bash
# 同时构建 amd64 + arm64，生成manifest清单推送到registry
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t username/myapp:v1 \
  --push .
```
> 构建完本地`docker images`看不到该镜像，因为多架构manifest只存在远端仓库，本地docker库不支持存储多平台镜像列表。

### ③ Dockerfile内部获取构建/目标平台变量
Dockerfile内置ARG：`BUILDPLATFORM`（构建机架构）、`TARGETPLATFORM`（输出镜像架构），常用于多阶段交叉编译（Go/C++）
```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:1.23 AS builder
ARG TARGETOS
ARG TARGETARCH
ENV GOOS=$TARGETOS
ENV GOARCH=$TARGETARCH
RUN CGO_ENABLED=0 go build -o myapp ./main.go

FROM alpine
COPY --from=builder /go/myapp /usr/bin/
CMD ["myapp"]
```

## 4. driver驱动对比
|驱动|说明|多平台push|缓存能力|
|---|---|---|---|
|`docker`|docker daemon内置BuildKit|❌|有限|
|`docker‑container`|独立buildkit容器（推荐）|✅|完整支持cache‑to/cache‑from|

## 5. 高频踩坑点
1. **多架构不能用 `--load`**：`--load`只允许单个`--platform`；多架构只能`--push`输出到仓库，或者`--output type=local`输出到本地目录。
2. QEMU仿真很慢：C++编译、大型编译任务，QEMU用户态模拟性能差；优先使用**交叉编译（BUILDPLATFORM编译）**，不要在TARGETPLATFORM里编译代码。
3. buildkit容器**不继承宿主机docker镜像加速器**：daemon.json的registry‑mirrors对docker‑container驱动不生效，需要单独配置buildkitd.toml传入builder创建参数。
4. `exec format error`：QEMU未正确bootstrap，执行`docker buildx inspect --bootstrap`重新安装qemu。
5. 不要混用旧builder：执行`docker buildx use mybuilder`确认当前生效builder。

## 6. 其他子命令
```bash
docker buildx ls                # 列出builder实例
docker buildx inspect mybuilder # 查看builder信息，QEMU支持哪些架构
docker buildx prune             # 清理构建缓存
docker buildx bake              # 基于hcl/json批量构建多目标
docker buildx imagetools inspect username/myapp:v1 # 查看远端多架构manifest清单
```

## 7. 选型建议
- 只做单架构：`docker buildx build --load`，体验等价旧docker build，可享受BuildKit缓存优化。
- 需要amd64+arm64双架构镜像：**交叉编译优先（BUILDPLATFORM编译）**，避免QEMU仿真编译；输出必须`--push`到镜像仓库。

如果你需要，我可以写一个最小可运行C++交叉编译Dockerfile示例适配buildx多架构。

===================================

# 2.docker buildx bake
`buildx bake` 是 buildx 的批量构建工具，**从 HCL / JSON 文件读取构建配置，一次性构建多个镜像、多平台、多目标**，避免写很长的 shell 命令，替代一堆复制粘贴的 `docker buildx build`。

命令：
```bash
docker buildx bake -f docker/docker-bake.hcl
```
- `-f docker/docker-bake.hcl`：指定 bake 配置文件路径，不是 Dockerfile；Dockerfile 路径写在 hcl 内部。
- hcl：HashiCorp Configuration Language，人类可读配置格式。

> 类比理解：
> `docker‑bake.hcl` ≈ 构建领域的 Makefile / docker‑compose.yml，但用于**镜像构建**，不是运行容器。

## docker‑bake.hcl 核心结构示例
```hcl
// 设置全局默认
group "default" {
  targets = ["app", "app‑debug"]
}

target "app" {
  context = "."
  dockerfile = "Dockerfile"
  tags = ["myorg/myapp:latest"]
  platforms = ["linux/amd64", "linux/arm64"]
  push = true       // 是否push镜像
}

target "app‑debug" {
  context = "."
  dockerfile = "Dockerfile.debug"
  tags = ["myorg/myapp:debug"]
  platforms = ["linux/amd64"]
}
```

- `target`：一个构建目标，对应一个镜像构建任务，可以有自己的 Dockerfile、tag、platform、变量、缓存参数。
- `group`：把若干 target 打包成一组；`default` 组不指定 target 时默认执行。

### 常用运行变体
```bash
# 构建 default 组全部 target
docker buildx bake -f docker/docker-bake.hcl

# 只构建其中某几个target，不跑default组
docker buildx bake -f docker/docker-bake.hcl app

# 覆盖tag，命令行覆盖hcl内部tags
docker buildx bake -f docker/docker-bake.hcl --set app.tags=myorg/myapp:v1.2

# 不push，本地load（只适合单platform target）
docker buildx bake -f docker/docker-bake.hcl --load
```

## bake vs 直接 buildx build
1. **buildx build**：单次单任务，参数写在 shell 命令行；项目一多命令超长、难以维护。
2. **buildx bake**：把所有构建参数放到 hcl 文件，支持多镜像、继承、变量、分组；CI/项目里更整洁。

### hcl支持继承（很实用）
```hcl
target "base" {
  context = "."
  platforms = ["linux/amd64","linux/arm64"]
}

target "app" {
  inherits = ["base"]
  dockerfile = "Dockerfile"
  tags = ["xxx"]
}
```
`app` 自动继承 base 的 platforms、context，只需要改写差异项。

## 高频坑点
1. `--load` 只能用于**单平台 target**。如果 target 设置多平台 `platforms = ["linux/amd64","linux/arm64"]`，不能用 `--load`，只能 `push=true`。
2. hcl 里 `push=true` 会直接推镜像到 registry，本地不会出现镜像；测试阶段临时关闭：`docker buildx bake --set app.push=false`。
3. bake 只是 buildx 的上层封装，底层仍然依赖 buildx builder（必须是 `docker‑container` 驱动才能完整多平台）。
4. `-f` 指定的是 bake 配置文件，**不是 Dockerfile**，不要混淆。Dockerfile路径写在 hcl 的 `dockerfile=` 字段。
5. 支持 JSON 格式替代 hcl：`docker‑bake.json`，语法更啰嗦，hcl可读性更好。

## CI场景典型用法
> openpilot、很多C++/Go多架构项目会使用 bake：
- 一套配置维护 amd64/arm64 镜像；
- 区分 release / debug 两个 target；
- CI脚本只需要一行 `docker buildx bake -f docker/docker-bake.hcl`，不用维护一大串shell参数。

如果你需要，我可以给一份适配C++项目的最小完整 docker‑bake.hcl 模板。

===================================



===================================