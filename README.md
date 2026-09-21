# Home Assistant Legacy Docker Compatibility Lab

[![HA Compatibility Validation](https://github.com/ranran123987/home-assistant-docker-compat-lab/actions/workflows/ha-img-0.yml/badge.svg)](https://github.com/ranran123987/home-assistant-docker-compat-lab/actions/workflows/ha-img-0.yml)
[![License: MIT](https://img.shields.io/github/license/ranran123987/home-assistant-docker-compat-lab)](https://github.com/ranran123987/home-assistant-docker-compat-lab/blob/main/LICENSE)
[![Latest Release](https://img.shields.io/github/v/release/ranran123987/home-assistant-docker-compat-lab)](https://github.com/ranran123987/home-assistant-docker-compat-lab/releases/latest)

[English](#english) | [中文](#中文)

An unofficial GitHub Actions workflow for converting official Home Assistant container images into a format that can be tested and loaded on older Docker Engine 20.10.x environments.

> **Unofficial community project.**  
> This project is not affiliated with, maintained by, or endorsed by Home Assistant or the Open Home Foundation.

---

# English

## Overview

Newer Home Assistant container images use modern container image compression formats such as `zstd`.

Older Docker Engine releases, particularly Docker 20.10.x, may fail when directly pulling or loading images containing these layers.

A typical error may look like:

```text
archive/tar: invalid tar header
```

This project provides a reproducible GitHub Actions workflow that:

1. Downloads the official Home Assistant container image.
2. Resolves and pins the exact `linux/amd64` image digest.
3. Recompresses image layers to `gzip`.
4. Emits a Docker V2 Schema 2 compatible image.
5. Verifies that filesystem layer `diffIDs` remain unchanged.
6. Verifies important container configuration semantics.
7. Validates every generated gzip layer.
8. Tests the converted image using a real Docker Engine 20.10 environment.
9. Generates a Docker archive for offline loading.
10. Loads the archive back into Docker 20.10.
11. Boots a fresh Home Assistant container and verifies that its HTTP service becomes available.
12. Uploads validation evidence and the generated Docker archive as GitHub Actions artifacts.

The purpose of this project is **not to modify Home Assistant itself**.

The workflow changes the container image's compression and transport representation while attempting to preserve the uncompressed filesystem contents and runtime configuration.

---

## Why This Project Exists

Many systems still run older Docker versions, including:

- OpenWrt / iStoreOS routers
- NAS devices
- Home servers
- Embedded systems
- Vendor-managed appliances
- Systems where Docker cannot be upgraded independently without affecting other components

On these systems, upgrading Docker may also affect:

- containerd
- runc
- firewall integration
- container networking
- storage drivers
- existing containers
- vendor management interfaces
- other system services

For users who cannot safely or conveniently upgrade Docker immediately, this workflow provides a way to test whether a newer Home Assistant container can still run after its image layers are repackaged into an older-compatible compression format.

---

## What the Workflow Changes

Conceptually:

```text
Official Home Assistant image
        │
        │ zstd-compressed layers
        ▼
Layer recompression
        │
        │ gzip-compressed layers
        ▼
Docker V2 Schema 2 image
        │
        ▼
Docker 20.10 compatibility validation
```

The workflow does **not** intentionally modify Home Assistant application files.

Instead, it changes how container filesystem layers are compressed and packaged.

---

## Integrity Verification

The workflow compares the upstream and converted image `diffIDs`.

A `diffID` identifies the **uncompressed content** of a container filesystem layer.

Therefore:

```text
zstd-compressed layer
        ↓ decompress

filesystem contents
        ↑ compress

gzip-compressed layer
```

may have different compressed blob digests while still having the same `diffID`.

The workflow also compares important container configuration fields, including:

- Architecture
- Operating system
- Entrypoint
- Command
- Environment variables
- Working directory
- User
- Labels
- Exposed ports
- Volumes
- Stop signal
- Healthcheck

It additionally checks:

- manifest structure
- layer count
- Docker media types
- gzip integrity for every converted layer

---

## Current Validation Target

The current workflow validates:

```text
Platform:
linux/amd64

Compatibility test engine:
Docker Engine 20.10.24
```

This does **not** guarantee that every Docker 20.10.x installation will behave identically.

Real systems may differ in:

- Linux kernel
- containerd version
- runc version
- storage driver
- security profile
- cgroup configuration
- firewall implementation
- network stack
- vendor modifications

Always test carefully before replacing an existing Home Assistant installation.

---

## How to Use

### 1. Fork this repository

Fork this repository into your own GitHub account.

### 2. Open GitHub Actions

Navigate to:

```text
Actions
→ HA-IMG-0 compatibility validation
→ Run workflow
```

### 3. Enter a Home Assistant version

For example:

```text
2026.9.3
```

Then click:

```text
Run workflow
```

The Home Assistant version is supplied at runtime, so you do not need to edit the workflow YAML for every release.

---

## Workflow Stages

### Stage 1 — Prepare the environment

The workflow runs on a temporary Ubuntu GitHub-hosted runner and installs tools including:

- `jq`
- `curl`
- `gzip`

It also starts an isolated temporary Docker registry.

---

### Stage 2 — Resolve the official image

The workflow retrieves the official Home Assistant image and resolves the exact:

```text
linux/amd64
```

child manifest digest.

The image is then pinned by digest instead of relying only on a mutable tag.

---

### Stage 3 — Recompress image layers

Skopeo is used to generate an image using:

```text
Docker V2 Schema 2
+
gzip-compressed layers
```

The upstream image content is used as the source.

---

### Stage 4 — Verify integrity

The workflow verifies:

- upstream and converted `diffIDs`
- important container configuration semantics
- manifest structure
- layer count
- Docker media types
- gzip validity of every converted layer

---

### Stage 5 — Start a real Docker 20.10 engine

The workflow starts a temporary:

```text
Docker Engine 20.10.24
```

daemon inside the GitHub Actions environment.

This becomes the compatibility test target.

---

### Stage 6 — Pull and inspect

Docker 20.10 attempts to pull the converted image.

The workflow verifies that errors such as:

```text
invalid tar header
```

do not occur.

It also inspects the resulting image and validates its architecture and layer structure.

---

### Stage 7 — Generate a Docker archive

The workflow generates an offline Docker archive.

The resulting file is named similar to:

```text
homeassistant-compat-2026.9.3-gzip.tar
```

The workflow also handles repeated logical layer references so that physical layer files are packaged correctly in the archive.

---

### Stage 8 — Docker 20 load test

The generated archive is loaded using Docker Engine 20.10.

The workflow verifies:

- Architecture
- Operating system
- Entrypoint
- Working directory
- Layer count

This confirms that the generated archive can actually be processed by the older Docker engine used by the test environment.

---

### Stage 9 — Boot Home Assistant

The workflow starts a fresh Home Assistant container using the converted image.

It waits for the Home Assistant HTTP service on:

```text
8123
```

and verifies that:

- the container remains running
- HTTP becomes reachable
- `RestartCount` remains zero
- no obvious fatal runtime errors are detected during the observation period

This means the workflow tests more than just `docker pull` or `docker load`.

It performs an actual Home Assistant startup test.

---

### Stage 10 — Upload artifacts

When the workflow completes, it uploads two types of GitHub Actions artifacts.

#### Validation evidence

The evidence artifact may contain:

- upstream image digest
- converted image digest
- image manifests
- image configs
- diffID comparison
- gzip validation results
- Docker 20 version information
- Docker pull results
- Docker load results
- Home Assistant startup logs
- audit summary

#### Docker archive

The Docker archive artifact contains the generated compatibility image:

```text
homeassistant-compat-<VERSION>-gzip.tar
```

Artifacts are temporary GitHub Actions outputs.

The current workflow retains them for:

```text
7 days
```

---

## Loading the Generated Image

After downloading the generated archive to your target machine, it can normally be imported with:

```bash
docker load -i homeassistant-compat-<VERSION>-gzip.tar
```

After loading, inspect the image before replacing an existing Home Assistant container.

Do not immediately delete the previous working image or container.

---

## Recommended Upgrade Strategy

For an existing Home Assistant installation, a blue/green-style migration is safer than immediately replacing the current container.

Example:

```text
Existing Home Assistant
        │
        └── BLUE
            Keep unchanged for rollback

Converted new image
        │
        └── GREEN
            Test startup
            Test configuration
            Test database migration
            Test integrations
            Test devices
            Test network access
        │
        ▼
Switch only after validation
```

Before upgrading:

- back up your Home Assistant configuration
- preserve the old container
- preserve the old image
- prepare a rollback procedure

---

## What This Project Does Not Do

This project does **not**:

- make Docker 20.10 officially supported by Home Assistant
- patch Home Assistant source code
- upgrade Docker
- upgrade containerd
- upgrade runc
- modify the host kernel
- modify your production Home Assistant configuration
- automatically migrate an existing Home Assistant installation
- guarantee compatibility on every Docker 20.10 system
- replace the official Home Assistant container distribution

It provides a reproducible image conversion and validation workflow.

---

## Official Support vs. Technical Compatibility

There is an important difference between:

```text
Officially supported
```

and:

```text
Technically capable of running
```

A converted Home Assistant image may successfully run on an older Docker release even if that Docker release is outside Home Assistant's current official requirements.

Passing this workflow means:

> The converted image successfully passed the compatibility checks implemented by this project.

It does **not** mean:

> Home Assistant officially supports this Docker version.

Users who can safely upgrade to a currently supported Docker environment should generally prefer the official supported software stack.

---

## Security and Privacy

This workflow operates on public upstream Home Assistant container images inside temporary GitHub-hosted runners.

Normal usage does not require:

- Home Assistant passwords
- Home Assistant access tokens
- SSH private keys
- Tailscale keys
- access to your home network
- your production Home Assistant `/config`
- personal Home Assistant entity or device data

Do not add production credentials, passwords, tokens, private keys, or personal configuration data to this workflow.

---

## Artifact and Distribution Policy

This repository is intended primarily as a:

```text
conversion and compatibility validation tool
```

rather than an alternative permanent Home Assistant image distribution service.

The recommended model is:

```text
Official Home Assistant image
        ↓
User runs the workflow
        ↓
User generates a compatibility artifact
        ↓
User tests it on their own system
```

The official Home Assistant image remains the upstream source of truth.

---

## Troubleshooting

### `archive/tar: invalid tar header`

This is one of the compatibility problems this workflow is designed to address.

Older Docker engines may have difficulty processing newer image layer formats.

Run the compatibility workflow instead of directly forcing the original unsupported image representation into the older Docker engine.

---

### Docker archive fails to load

Check:

- Docker version
- CPU architecture
- artifact integrity
- free disk space
- Docker storage driver
- Docker daemon logs
- filesystem health

---

### Home Assistant starts but port 8123 does not respond

Inspect:

```bash
docker logs <container-name>
```

Successful image-format conversion does not guarantee that every Home Assistant integration, device, kernel feature, or host-specific dependency will work on every system.

---

### Workflow fails for a future Home Assistant release

Home Assistant's image structure or build process may change in the future.

If a future version fails:

1. identify the failed workflow stage
2. inspect the relevant error
3. open an issue if appropriate
4. include the Home Assistant version
5. include non-sensitive logs

Do **not** include:

- passwords
- tokens
- private keys
- private network details
- personal Home Assistant configuration

---

## Architecture Support

The current workflow is designed and validated for:

```text
linux/amd64
```

Other architectures such as:

```text
linux/arm64
```

are not currently validated at the same level.

Support for additional architectures may be added in the future.

---

## Contributing

Contributions are welcome.

Useful contributions may include:

- support for additional architectures
- validation against additional Docker versions
- improved integrity checks
- improved diagnostics
- additional reproducible test cases
- documentation improvements
- better compatibility reporting

Please do not commit or publish:

- credentials
- access tokens
- passwords
- SSH keys
- private network information
- personal Home Assistant configuration
- proprietary system data

---

## License

The original workflow code, scripts, and documentation in this repository may be distributed under the MIT License.

Home Assistant is a separate upstream project and remains subject to its own licenses, terms, and policies.

This repository does not relicense Home Assistant or third-party software contained in official Home Assistant container images.

Home Assistant names, logos, and related trademarks belong to their respective rights holders.

---

## Disclaimer

This project is provided **as is**, without warranty.

Container runtime compatibility depends on many host-specific factors.

Always maintain:

- backups
- the previous working image
- the previous working container
- a tested rollback path

before modifying a production Home Assistant installation.

A successful CI run does not guarantee compatibility with every real-world host environment.

---

## References

- Home Assistant: https://www.home-assistant.io/
- Home Assistant Linux installation documentation: https://www.home-assistant.io/installation/linux
- Home Assistant Core: https://github.com/home-assistant/core
- Docker Engine documentation: https://docs.docker.com/

---

# 中文

## 项目简介

这是一个非官方的 GitHub Actions Home Assistant 容器镜像兼容性转换与验证工具。

它主要用于将 Home Assistant 官方容器镜像转换为更适合旧版 Docker Engine 20.10.x 环境处理的镜像格式，并自动完成完整性验证、Docker 20 实际加载测试以及 Home Assistant 启动测试。

> **本项目属于非官方社区项目。**  
> 本项目与 Home Assistant 官方及 Open Home Foundation 不存在隶属、维护或官方背书关系。

---

## 为什么会有这个项目

新版 Home Assistant 官方容器镜像使用了包括 `zstd` 在内的现代容器镜像压缩方式。

部分旧版 Docker Engine，尤其是 Docker 20.10.x，在直接拉取或加载相关镜像时可能出现兼容性问题。

常见错误之一是：

```text
archive/tar: invalid tar header
```

目前仍有很多设备因为系统或厂商环境限制，无法轻易独立升级 Docker，例如：

- OpenWrt / iStoreOS 软路由
- NAS
- 家庭服务器
- 嵌入式设备
- 厂商定制系统
- Docker 与系统其他组件深度绑定的设备

在这些设备上，Docker 升级可能同时涉及：

- containerd
- runc
- 防火墙
- Docker 网络
- 存储驱动
- 已有容器
- 厂商管理界面
- 系统中的其他服务

因此，本项目提供一种相对独立的兼容性测试方案：

> 不修改 Home Assistant 本身，而是重新处理官方容器镜像的压缩和封装方式，并在真实 Docker 20.10 测试环境中验证其兼容性。

---

## 本项目实际上修改了什么

整体流程可以理解为：

```text
Home Assistant 官方镜像
        │
        │ zstd 压缩层
        ▼
重新压缩镜像层
        │
        │ gzip 压缩层
        ▼
Docker V2 Schema 2 镜像
        │
        ▼
Docker 20.10 兼容性验证
```

项目不会主动修改 Home Assistant 应用程序代码。

主要改变的是：

```text
容器镜像层的压缩与封装方式
```

而不是：

```text
镜像层解压后的实际文件系统内容
```

---

## 如何验证 Home Assistant 内容没有被改坏

工作流会比较转换前后的：

```text
diffID
```

`diffID` 可以理解为容器镜像某一层在**解压后的文件系统内容标识**。

因此：

```text
zstd 压缩层
      ↓ 解压

实际文件系统内容

      ↑ 压缩
gzip 压缩层
```

虽然压缩后的 blob digest 会发生变化，但如果解压后的内容完全一致，对应的 `diffID` 应保持一致。

除此之外，工作流还会比较重要的容器配置，包括：

- CPU 架构
- 操作系统
- Entrypoint
- Cmd
- 环境变量
- 工作目录
- User
- Labels
- ExposedPorts
- Volumes
- StopSignal
- Healthcheck

同时还会验证：

- Manifest 结构
- Layer 数量
- Docker media type
- 所有 gzip layer 的完整性

---

## 当前验证环境

目前 GitHub Actions 中的测试目标为：

```text
平台：
linux/amd64

兼容性测试 Docker：
Docker Engine 20.10.24
```

需要特别说明：

这并不意味着所有 Docker 20.10.x 环境都一定能够正常运行。

真实设备还可能存在不同的：

- Linux Kernel
- containerd
- runc
- 存储驱动
- 安全策略
- cgroup 配置
- 防火墙
- Docker 网络
- 厂商定制修改

因此，在正式替换现有 Home Assistant 之前，仍然建议在自己的设备上进行验证。

---

## 使用方法

### 第一步：Fork 本项目

将本项目 Fork 到自己的 GitHub 账号。

---

### 第二步：进入 GitHub Actions

依次进入：

```text
Actions
→ HA-IMG-0 compatibility validation
→ Run workflow
```

---

### 第三步：输入 Home Assistant 版本号

例如：

```text
2026.9.3
```

然后点击：

```text
Run workflow
```

Home Assistant 版本已经改成运行时输入参数。

以后处理新版本时，不再需要手动修改 YAML 文件中的版本号。

---

## Workflow 自动完成哪些工作

### 阶段 1 — 创建临时测试环境

GitHub Actions 会启动一台临时 Ubuntu Runner，并安装：

- `jq`
- `curl`
- `gzip`

同时创建一个临时 Docker Registry。

---

### 阶段 2 — 获取 Home Assistant 官方镜像

工作流会获取指定版本的 Home Assistant 官方镜像，并解析：

```text
linux/amd64
```

对应的真实 child manifest digest。

随后使用 digest 锁定具体镜像内容，而不是只依赖可能变化的版本 tag。

---

### 阶段 3 — 将镜像层重新压缩为 gzip

使用 Skopeo 将官方镜像转换为：

```text
Docker V2 Schema 2
+
gzip 压缩层
```

---

### 阶段 4 — 镜像完整性验证

工作流会检查：

- 转换前后 diffID 是否一致
- 重要容器配置是否一致
- Manifest 结构
- Layer 数量
- Docker media type
- 所有 gzip layer 是否完整

---

### 阶段 5 — 启动真实 Docker 20.10

GitHub Actions 中会额外启动：

```text
Docker Engine 20.10.24
```

作为真实兼容性测试目标。

---

### 阶段 6 — Docker 20 拉取测试

使用 Docker 20.10 实际拉取转换后的镜像。

工作流会检查是否出现：

```text
invalid tar header
```

等错误。

同时会检查转换后镜像的：

- CPU 架构
- Layer 数量
- 镜像结构

---

### 阶段 7 — 生成 Docker Archive

工作流会生成一个可用于离线导入的 Docker tar 包。

例如：

```text
homeassistant-compat-2026.9.3-gzip.tar
```

同时还会处理 Manifest 中可能存在的重复逻辑 Layer 引用，避免在最终 Docker Archive 中错误地重复保存同一个物理 Layer 文件。

---

### 阶段 8 — Docker 20 实际 load 验证

生成 tar 文件后，并不会直接认为转换成功。

工作流还会真正使用 Docker 20.10 执行：

```bash
docker load
```

并验证：

- Architecture
- Operating system
- Entrypoint
- Working directory
- Layer count

也就是说，它会确认最终生成的 tar 包确实能够被旧 Docker 正常读取。

---

### 阶段 9 — 实际启动 Home Assistant

本项目不仅验证镜像能否：

```text
docker pull
```

或者：

```text
docker load
```

还会真正启动一个全新的 Home Assistant 测试容器。

然后等待：

```text
8123
```

端口的 HTTP 服务正常响应。

同时检查：

- 容器是否保持 Running
- RestartCount 是否为 0
- HTTP 是否正常响应
- 观察期间是否出现明显 fatal 错误
- 经过一定时间后容器是否仍然稳定

因此，它属于实际启动验证，而不仅仅是镜像格式检测。

---

### 阶段 10 — 上传 Artifact

运行完成后，会生成两类 GitHub Actions Artifact。

#### Validation Evidence

其中可能包括：

- 官方镜像 digest
- 转换后镜像 digest
- Manifest
- Config
- diffID 比较结果
- gzip 完整性检查
- Docker 20 版本信息
- Docker pull 结果
- Docker load 结果
- Home Assistant 启动日志
- Audit Summary

#### Docker Archive

其中包含最终生成的兼容镜像：

```text
homeassistant-compat-<版本>-gzip.tar
```

Artifact 属于临时 GitHub Actions 输出。

当前工作流设置的保存时间为：

```text
7 天
```

---

## 如何把生成的镜像导入自己的设备

下载 Docker Archive Artifact 后，将 tar 文件复制到目标设备。

通常可以执行：

```bash
docker load -i homeassistant-compat-<版本>-gzip.tar
```

导入。

导入以后建议先检查新镜像。

不要立即删除原来的：

- Home Assistant 镜像
- Home Assistant 容器
- Home Assistant 配置备份

---

## 推荐升级方式

对于已经长期运行的 Home Assistant，推荐采用类似“蓝绿部署”的方式进行升级测试。

例如：

```text
现有 Home Assistant
        │
        └── BLUE
            保持不变
            用于快速回滚

新的兼容镜像
        │
        └── GREEN
            测试启动
            测试配置
            测试数据库迁移
            测试集成
            测试设备
            测试网络
        │
        ▼
确认稳定后再正式切换
```

进行生产环境升级之前，建议：

- 备份 Home Assistant 配置
- 保留旧镜像
- 保留旧容器
- 明确回滚方式

---

## 本项目不会做什么

本项目不会：

- 将 Docker 20.10 变成 Home Assistant 官方支持版本
- 修改 Home Assistant 源代码
- 自动升级 Docker
- 自动升级 containerd
- 自动升级 runc
- 修改 Linux Kernel
- 修改用户真实 Home Assistant 配置
- 自动迁移生产环境
- 保证所有 Docker 20.10 环境都能够兼容
- 替代 Home Assistant 官方镜像发行渠道

本项目所提供的是：

> 一个可重复执行、可验证、可审计的镜像格式转换和旧版 Docker 兼容性测试流程。

---

## “官方支持”和“技术上能够运行”不是一回事

需要特别区分：

```text
官方支持
```

和：

```text
技术上能够运行
```

即使经过转换后的 Home Assistant 可以在某个 Docker 20.10 环境中正常启动，也不能据此认为 Home Assistant 官方重新支持该 Docker 版本。

Workflow 测试通过只能说明：

> 该 Home Assistant 镜像经过本项目定义的转换和测试之后，通过了相应兼容性验证。

不能理解为：

> Home Assistant 官方保证这个 Docker 版本能够正常运行。

如果你的设备能够安全升级到 Home Assistant 当前官方支持的 Docker 环境，通常仍应优先采用官方支持的软件栈。

---

## 安全与隐私

正常使用本项目时，处理的是公开的 Home Assistant 官方容器镜像。

不需要提供：

- Home Assistant 密码
- Home Assistant Token
- SSH 私钥
- Tailscale 密钥
- 家庭网络访问权限
- 用户真实的 Home Assistant `/config`
- 用户个人实体信息
- 用户设备信息

不要把真实的：

- 密码
- Token
- API Key
- 私钥
- SSH 密钥
- 家庭配置

写入 Workflow。

---

## Artifact 与镜像分发原则

本项目的主要定位是：

```text
兼容性转换与验证工具
```

而不是：

```text
第三方 Home Assistant 镜像发行站
```

推荐使用模式：

```text
Home Assistant 官方镜像
        ↓
用户自己运行 GitHub Actions
        ↓
生成自己的兼容 Artifact
        ↓
用户在自己的设备中测试
```

Home Assistant 官方镜像仍然是本项目唯一的上游来源。

---

## 常见问题

### 为什么 Docker 20.10 会出现 `invalid tar header`？

部分较老的 Docker Engine 对较新的镜像压缩或封装格式支持有限。

本项目通过重新处理镜像层的压缩表示方式，并使用真实 Docker 20.10 环境进行验证，以解决或定位这一类兼容问题。

---

### 转成 gzip 会不会修改 Home Assistant？

本项目的目标不是修改 Home Assistant 内容。

工作流会检查转换前后的：

```text
diffID
```

以及关键容器配置，以验证解压后的文件系统内容和重要运行参数是否保持一致。

---

### 为什么不直接升级 Docker？

如果你的系统能够安全升级 Docker，当然可以。

如果能够使用 Home Assistant 当前官方支持的 Docker 版本，优先使用官方支持的软件栈通常更加合理。

本项目主要面向：

- 无法方便升级 Docker
- Docker 与系统或厂商固件深度绑定
- Docker 升级可能影响其他容器
- Docker 升级可能影响网络或防火墙
- 需要临时兼容方案
- 希望先验证新版 Home Assistant 是否能够运行

的用户。

---

### Workflow 成功是不是代表我的设备一定能运行？

不是。

GitHub Actions 成功只能说明：

```text
该镜像在本项目定义的测试环境中验证成功
```

你的真实设备还可能存在不同的：

- Kernel
- CPU
- containerd
- runc
- 存储驱动
- 内存
- cgroup
- 安全策略
- 网络
- 防火墙
- 厂商修改

所以仍然应该做好：

```text
备份 + 测试 + 回滚
```

---

### Home Assistant 启动了，但是 8123 无法访问怎么办？

查看：

```bash
docker logs <容器名称>
```

镜像格式兼容并不代表所有：

- Home Assistant 集成
- USB 设备
- 蓝牙设备
- 网络设备
- 内核功能
- Host 权限
- 第三方组件

都会在所有系统中自动兼容。

---

### 未来某个 Home Assistant 新版本运行失败怎么办？

未来 Home Assistant 的：

- 镜像结构
- Build Pipeline
- Container Runtime 要求
- Base Image
- Layer 格式

都可能继续变化。

如果未来出现问题，可以提交 Issue，并提供：

- Home Assistant 版本
- 失败的 Workflow 阶段
- 不包含敏感信息的错误日志

请不要提交：

- 密码
- Token
- 私钥
- 家庭网络敏感信息
- 用户真实 Home Assistant 配置

---

## 当前架构支持

当前 Workflow 主要针对：

```text
linux/amd64
```

进行完整验证。

例如：

```text
linux/arm64
```

等其他架构，目前还没有建立相同等级的自动化验证流程。

未来可以逐步扩展。

---

## 欢迎贡献

欢迎提交：

- 更多 CPU 架构支持
- 更多 Docker 版本测试
- 更严格的镜像完整性验证
- 更完善的错误诊断
- 更清晰的运行报告
- 可重复的问题案例
- README 与文档改进

请不要在：

- Commit
- Issue
- Pull Request
- Actions 日志

中公开：

- 密码
- Token
- API Key
- 私钥
- SSH Key
- 私人网络信息
- 真实 Home Assistant 配置
- 其他敏感数据

---

## 许可证

本仓库原创的 Workflow、脚本及文档可以按照 MIT License 进行授权和分发。

Home Assistant 本身属于独立的上游项目，并继续适用 Home Assistant 自身的软件许可证、使用条款及相关政策。

本项目不会对 Home Assistant 本身或官方容器镜像中的第三方软件重新授权。

Home Assistant 的名称、Logo、商标及相关权利归其各自权利人所有。

---

## 免责声明

本项目按“现状”提供，不作任何形式的兼容性保证。

容器运行环境是否兼容，还可能受到大量设备和系统因素影响。

对生产环境进行任何升级前，请务必保留：

- 配置备份
- 原镜像
- 原容器
- 明确可执行的回滚方案

GitHub Actions 测试成功，并不代表能够覆盖所有真实设备环境。

---

## 相关资料

- Home Assistant 官网：https://www.home-assistant.io/
- Home Assistant Linux 安装文档：https://www.home-assistant.io/installation/linux
- Home Assistant Core：https://github.com/home-assistant/core
- Docker Engine 文档：https://docs.docker.com/
