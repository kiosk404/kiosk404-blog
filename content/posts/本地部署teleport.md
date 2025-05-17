---
title: 本地部署 teleport
author: kiosk
tags: 
  - devops
categories:
  - Linux
date : 2025-05-05 08:50:00
---





由于最近装了好多虚拟机，希望有一个统一的入口，那么就需要一个开源的跳板机了，调研了一下 teleport（远程传送）真的是个不错的选择。这样就可以有一个统一的 跳板入口了。



<!--more-->

Teleport 是由 Gravitational 打造的一体化访问控制平面（Access Plane），通过统一的身份验证与授权机制，为 SSH、Kubernetes、数据库、内部应用以及桌面等多种资源提供安全、可审计的访问能力。它支持零信任架构理念，能够集成多种 SSO（单点登录）提供商，生成短期证书与令牌，并对会话进行录制与审计，满足安全合规和运维效率需求。下面，我们将从 Teleport 的定义、核心功能模块、典型应用场景等方面进行详细介绍。

<img src="https://img1.kiosk007.top/static/images/blog/20250517110030-teleport.png" />

## Teleport 是什么？

Teleport 是一个开源的基础设施访问安全平台（Infrastructure Access Platform），它通过“访问代理”模式（Proxy Service）在公网上暴露单一入口，同时将实际资源置于私有网络中，有效降低网络暴露面和攻击面 [Teleport](https://goteleport.com/docs/core-concepts/?utm_source=chatgpt.com)。它整合了认证（Auth Service）、代理（Proxy Service）和多种 Agent 服务，用于管理和保护 SSH、Kubernetes、数据库、内部 Web 应用、远程桌面等多种类型的访问 [Teleport](https://goteleport.com/docs/core-concepts/?utm_source=chatgpt.com)[Teleport](https://goteleport.com/docs/?utm_source=chatgpt.com)。

<br/>

Teleport 的两大支柱是身份安全（Identity Security）和统一访问控制（Unified Access Plane）。在身份安全方面，它能够检测常驻权限（standing privileges）并消除影子访问（shadow access）与盲点（blind spots），帮助团队可视化并强化 RBAC（基于角色的访问控制）策略 [Teleport](https://goteleport.com/docs/?utm_source=chatgpt.com)。在统一访问控制方面，所有访问请求均通过 Proxy Service，使用 Teleport 证书进行加密，并可灵活配置多因素认证（MFA）与设备信任（Device Trust） [Teleport](https://goteleport.com/docs/?utm_source=chatgpt.com)。

## 核心功能模块



### SSH 与终端访问

Teleport 最初以 SSH 访问安全著称，通过 Teleport SSH Service 替代传统的 SSH Daemon，使用短期证书代替静态密钥，消除了长生命周期秘钥带来的风险，并支持记录和回放终端会话，满足审计合规要求 [Teleport](https://goteleport.com/docs/core-concepts/?utm_source=chatgpt.com)。

----



### Kubernetes 访问

通过 Teleport Kubernetes Service，用户可以使用同一套 SSO 凭据访问多个 Kubernetes 集群，且可针对不同集群或资源级别实施细粒度 RBAC 策略，并将 `kubectl` 会话录制到审计日志中 [Teleport](https://goteleport.com/docs/enroll-resources/kubernetes-access/introduction/?utm_source=chatgpt.com)。

----



### 数据库访问

Teleport Database Service 支持对 PostgreSQL、MySQL 等常见数据库的访问代理，无需在客户端配置静态凭据，所有数据库会话均通过 Teleport Proxy，并可启用 SQL 查询审计 [Teleport](https://goteleport.com/docs/core-concepts/?utm_source=chatgpt.com)。

----



### 内部应用与远程桌面

Teleport Application Service 代理 HTTP/TCP 流量，可安全发布内部 Web 应用和如 AWS 控制台之类的私有后台；Teleport Desktop Service 则代理 RDP 流量，实现对 Windows 桌面的安全访问 [Teleport](https://goteleport.com/docs/core-concepts/?utm_source=chatgpt.com)。

----



### 工作负载身份（Workload Identity）

Teleport Workload Identity（兼容 SPIFFE 标准）为非人类实体发放短期证书和 JWT，用于微服务间的相互认证，支持细粒度的工作负载鉴权与自动续期 [Teleport](https://goteleport.com/docs/enroll-resources/workload-identity/introduction/?utm_source=chatgpt.com)。

<br/>

## 典型应用场景

- **跨团队运维与审计**：集中管理运维权限，记录所有 SSH、Kubernetes、数据库等会话，便于事后审计与问题溯源 [Teleport](https://goteleport.com/docs/enroll-resources/kubernetes-access/introduction/?utm_source=chatgpt.com)。
- **零信任安全架构**：将 Teleport 作为统一访问网关，结合 MFA、设备信任，确保每次访问都经过身份验证和授权 [Teleport](https://goteleport.com/docs/?utm_source=chatgpt.com)。
- **云上与混合云环境**：无需 VPN，通过 Proxy Service 的 443 端口暴露入口，私有网络中的服务通过反向隧道接入，简化网络配置与安全防护 [Teleport](https://goteleport.com/docs/core-concepts/?utm_source=chatgpt.com)。
- **开发与 CI/CD 集成**：借助 Teleport API 与 Terraform 提供的 gRPC 接口，可将访问配置纳入基础设施即代码流程，自动化管理 [Teleport](https://goteleport.com/docs/reference/architecture/api-architecture/?utm_source=chatgpt.com)。

<br/>

## Docker 部署
> 注意： 在生产环境中部署 Teleport 需要更详细的规划和配置，包括持久化存储、TLS 证书、高可用性设置等。本文档提供的是一个基础的 Docker 部署和用户创建示例。

- **前提条件**

已安装 Docker。

对 Docker 命令有基本了解。

防火墙已允许访问 Teleport 所需的端口（默认通常是 3023, 3025, 3080）。

<br/>

### 生成 Teleport 配置文件

在运行 Teleport 容器之前，建议生成一个基础的配置文件 `teleport.yaml`。这将帮助您更好地控制 Teleport 的行为并持久化配置。

创建一个用于存放 Teleport 配置和数据的本地目录，例如 `~/teleport`：

```bash
mkdir -p teleport/data teleport/config teleport/certs
```

使用 Teleport 镜像生成默认配置文件，并将其保存到您创建的配置目录中：

```bash
docker run --hostname localhost --rm \
  --entrypoint=/usr/local/bin/teleport \
  public.ecr.aws/gravitational/teleport-distroless:17.4.8 configure --roles=proxy,auth > ~/teleport/config/teleport.yaml
```

- `--hostname localhost`: 设置容器的主机名为 `localhost`，这对于使用自签名证书进行本地测试很有用。
- `--rm`: 容器停止后自动删除。
- `--entrypoint=/usr/local/bin/teleport`: 指定容器的入口点为 teleport 可执行文件。
- `configure`: 运行 Teleport 的配置生成命令。
- `--roles=proxy,auth`: 指定 Teleport 实例同时运行认证服务（auth）和代理服务（proxy）。
- `> ~/teleport/config/teleport.yaml`: 将生成的配置输出重定向到本地文件。

您可以根据需要编辑 `~/teleport/config/teleport.yaml` 文件来修改端口、域名等配置。例如，您可能需要修改 `public_addr` 来匹配您的服务器域名或 IP 地址，以便通过 Web UI 访问。

<br/>

这里我们对配置文件做一些修改：

```yaml
proxy_service:
  enabled: "yes"
  web_listen_addr: 0.0.0.0:3080
  tunnel_listen_addr: 0.0.0.0:3024
  listen_addr: 0.0.0.0:3023
  public_addr:
    - localhost:3080
  https_key_file: /etc/teleport/certs/server-key.pem
  https_cert_file: /etc/teleport/certs/server.pem
  https_keypairs: []
  https_keypairs_reload_interval: 0s
  acme: {}

```

这里提到了证书，如果需要的话填上，不需要的话不填，证书的生成可以使用 [cfssl](https://github.com/cloudflare/cfssl) 参考 [教程](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/certificates/#cfssl)生成

<br/>

### 运行 Teleport Docker 容器

现在，使用以下命令运行 Teleport 容器，并将配置、数据和证书目录都挂载进去：

```bash
docker run -d --restart=always \
  --hostname localhost --name teleport \
  -v ~/Applications/teleport/config:/etc/teleport \
  -v ~/Applications/teleport/data:/var/lib/teleport \
  -v ~/Applications/teleport/certs:/etc/teleport/certs \
  -p 3025:3025 -p 3080:3080 -p 3023:3023 \
  public.ecr.aws/gravitational/teleport-distroless:17.4.8
```

`-v ~/teleport/certs:/etc/teleport/certs`: 将主机上的证书目录挂载到容器内 `/etc/teleport/certs`，这样 Teleport 才能找到证书文件。

其他参数与之前运行容器的命令类似，用于映射端口、持久化数据和配置、设置重启策略等。

`start -c /etc/teleport/teleport.yaml`: 启动 Teleport 服务并加载指定的配置文件。

<br/>

### 验证 Teleport 集群状态

确认容器正在运行，并且 Teleport 服务健康：

```bash
docker ps | grep teleport
docker exec teleport tctl status
```

确保 Auth Service 和 Proxy Service 都显示为 Healthy 或 Running。

<br/>

### 访问 Teleport Web UI

现在可以通过 HTTPS 访问 Teleport Web UI 了。在浏览器中输入：

```bash
https://<Teleport服务器IP或域名>:3080
```

由于使用的是自签名证书，浏览器会显示安全警告（例如 "Your connection is not private"）。这是正常的，因为证书不是由受信任的 CA 颁发的。

- **在测试环境中，可以选择忽略警告并继续访问。** 在大多数浏览器中，可以点击 "Advanced"（高级）或 "Details"（详情），然后选择 "Proceed to..." 或 "Add Exception..."。
- **理解风险：** 忽略警告意味着您信任连接的目标是自己的 Teleport 服务器，并且接受中间人攻击的风险（尽管在知道自己在连接什么的情况下风险较低）。

<br/>

### 创建第一个 Teleport 用户

首次部署 Teleport 集群时，需要创建第一个用户。通过 `docker exec` 在容器内运行 `tctl users add` 命令：

```bash
docker exec teleport tctl users add admin --roles=editor,access --logins=root,ubuntu # 根据需要调整 --logins
```

将 `admin` 替换为您想要的用户名。

`--roles=editor,access`: 授予用户编辑 Teleport 资源和访问节点的权限。

`--logins=root,ubuntu`: 允许该用户以 root 或 ubuntu 系统用户身份登录目标节点

执行命令后，终端会输出一个用户邀请链接。复制这个链接，并在浏览器中访问它（请注意，访问链接时使用的域名或 IP 必须与您生成证书时的 CN/SANs 匹配，否则浏览器会再次显示证书错误）。通过邀请链接，您可以设置用户的密码和多因素认证 (MFA)。

<br/>

>  如果希望删除用户的话，可以使用 `docker exec teleport tctl users rm admin` 命令。

<br/>

### 安装 Teleport 客户端 (`tsh`)

在您希望连接到 Teleport 的本地机器上，下载并安装 Teleport 客户端工具 `tsh`。访问 [Teleport 下载页面](https://goteleport.com/download/) 获取适合您操作系统的版本。

<br/>

### 登录 Teleport 并连接节点

在本地机器的终端中：

1. **登录 Teleport 集群：**

   Bash

   ```bash
   tsh login --proxy=<您的Teleport服务器IP或域名>:3080 --user=admin # 使用您创建的用户名
   ```

   根据提示输入密码和 MFA 验证码。成功后，您会获得一个临时的本地证书。

2. **连接到节点：** 首先需要有节点加入集群。假设您已经录入了一个名为 `node-a` 的节点，并且您的用户被授权以 `ubuntu` 用户身份登录：

   Bash

   ```bash
   tsh ssh ubuntu@node-a
   ```

至此，您已经完成了使用 Docker 部署带自签证书的 Teleport 集群、创建用户并通过客户端连接的基本流程。



## 录入其他服务器节点

录入过程主要是在**目标节点**上安装和配置 Teleport 代理（节点服务），使其连接到 Teleport 集群。

**目标：** 使远程服务器 ("目标节点") 出现在 Teleport Web UI 的列表中，并能够通过 Teleport 客户端 (`tsh`) 连接访问它。

**前期准备：**

1.  Teleport 集群（Auth 和 Proxy 服务）已经在运行中（如我们之前使用 Docker 部署的那样）。
2. 需要有权限访问要录入的“目标节点”，并且具备在其上安装软件和运行命令的权限（通常需要 `sudo` 权限）。
3. “目标节点”需要能够通过网络连接到 Teleport 集群的 Auth Service 地址和端口（默认是 3025）。

**详细步骤**

- **步骤 1：在 Teleport 集群上获取加入令牌 (Join Token) 和安装脚本**

这个令牌和脚本包含了让目标节点知道如何连接到集群所需的所有信息。

**推荐方式：通过 Teleport Web UI 获取脚本** 这是最简单直观的方法。

1. 登录到你的 Teleport Web UI (通常是 `https://<Teleport服务器IP或域名>:3080`)。

2. 导航到页面上添加资源的选项。在主页上右上角会有 "Enroll New Resource" 或类似的按钮。

3. 选择 "Server" 或 "SSH Server" 作为要添加的资源类型。

4. Teleport Web UI 会根据集群信息生成一个定制化的安装脚本命令。这个命令通常看起来像这样：

   ```bash
   sudo bash -c "$(curl -kfsSL https://<Teleport服务器IP或域名>:3080/scripts/<一个长令牌>/install-node.sh)"
   ```

5. 仔细复制这个完整的命令。请注意，这个链接中的令牌是**一次性**且**有时效性**的。



- **步骤 2：在目标节点上执行安装脚本**

好的，我们来详细讲解如何将其他服务器节点（假设称为 "目标节点"）录入到您的 Teleport 集群，以便通过 Teleport 进行安全访问。

这个过程主要是在**目标节点**上安装和配置 Teleport 代理（节点服务），使其连接到您的 Teleport 集群。

**目标：** 使您的远程服务器 ("目标节点") 出现在 Teleport Web UI 的列表中，并能够通过 Teleport 客户端 (`tsh`) 连接访问它。

**前期准备：**

1. 您的 Teleport 集群（Auth 和 Proxy 服务）已经在运行中（如我们之前使用 Docker 部署的那样）。
2. 您需要有权限访问要录入的“目标节点”，并且具备在其上安装软件和运行命令的权限（通常需要 `sudo` 权限）。
3. “目标节点”需要能够通过网络连接到您的 Teleport 集群的 Auth Service 地址和端口（默认是 3025）。

**详细步骤：**

**步骤 1：在 Teleport 集群上获取加入令牌 (Join Token) 和安装脚本**

这个令牌和脚本包含了让目标节点知道如何连接到您的集群所需的所有信息。

- **推荐方式：通过 Teleport Web UI 获取脚本** 这是最简单直观的方法。

  1. 登录到您的 Teleport Web UI (通常是 `https://<您的Teleport服务器IP或域名>:3080`)。

  2. 导航到页面上添加资源的选项。通常在主页上会有 "Add Server" 或类似的按钮，或者在导航菜单中有相关选项。

  3. 选择 "Server" 或 "SSH Server" 作为您要添加的资源类型。

  4. Teleport Web UI 会根据您的集群信息生成一个定制化的安装脚本命令。这个命令通常看起来像这样：

     Bash

     ```
     sudo bash -c "$(curl -kfsSL https://<您的Teleport服务器IP或域名>:3080/scripts/<一个长令牌>/install-node.sh)"
     ```

  5. 仔细复制这个完整的命令。请注意，这个链接中的令牌是**一次性**且**有时效性**的。

- **替代方式：通过 `tctl` 命令生成令牌（如果无法访问 Web UI）** 如果您无法访问 Web UI，可以在运行 Teleport Auth 服务的 Docker 容器内使用 `tctl` 命令生成一个令牌。

  Bash

  ```
  docker exec teleport tctl nodes add --ttl=5m --roles=node --format=json
  ```

  - `--ttl=5m`: 设置令牌的有效时间，请确保在目标节点上操作期间令牌不会过期。
  - `--roles=node`: 指定这个令牌是用于加入一个节点代理。
  - `--format=json`: 以 JSON 格式输出，方便提取信息。 命令输出会包含一个 `token` 字段和一个 `auth_servers` 列表。您需要记下 `token` 的值和 `auth_servers` 的地址。您需要手动使用这些信息去安装和配置 Teleport 代理。**使用 Web UI 脚本通常更容易。**

**步骤 2：在目标节点上执行安装脚本**

在希望加入 Teleport 的“目标节点”服务器上执行以下操作：

1. 确保节点 A 可以执行 `curl` 和 `bash`： 大多数 Linux 系统都默认安装了，如果缺少，请先安装。

2. 粘贴并运行在步骤 1 中获取的安装脚本命令：

   直接在目标节点的终端中粘贴并运行从 Teleport Web UI 复制的完整命令。例如：

   Bash

   ```bash
   sudo bash -c "$(curl -kfsSL https://192.168.120.100:3080/scripts/e4662d25348effdef1847175e9ec6264/install-node.sh)"
   ```

   - `sudo bash -c "..."`: 使用 root 权限执行后面的 bash 命令。

   - ```
     curl -kfsSL ...
     ```

     - `-k`: 忽略不安全的连接（用于自签名证书）。如果在生产环境中使用正式证书，可以移除此标志。
     - `-f`: 使 curl 在 HTTP 错误时失败。
     - `-s`: 静默模式，不显示进度条。
     - `-S`: 在静默模式下显示错误信息。
     - `-L`: 如果遇到重定向，跟踪重定向。

   - `$(...)`: 命令替换，先执行括号内的 `curl` 命令下载脚本，然后将脚本内容作为参数传递给 `bash -c` 执行。

这个脚本会自动完成以下工作：

- 检测操作系统和架构。
- 下载适合的 Teleport 二进制文件。
- 将其放置到正确的系统路径。
- 生成一个基础的 `teleport.yaml` 配置文件，其中已经包含了集群的 Auth Server 地址和加入令牌。
- 将 Teleport 配置为系统服务（例如使用 systemd），并启动它。

**步骤3**：验证 Teleport 代理服务在目标节点上运行

脚本执行完成后，检查 Teleport 代理服务是否成功启动。

- 对于使用 systemd 的系统（如 Ubuntu 15.04+, CentOS 7+, Debian 8+）：

```bash
sudo systemctl status teleport
```



最后就可以看到，Web UI 中存在了一个 资源可以被 SSH 连上去：

<img src="https://img1.kiosk007.top/static/images/blog/20250517143851-teleport-connect-vm00.png" />
