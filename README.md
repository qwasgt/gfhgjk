# Webui_Railway_API_IntelDeploy


## 🚀 一键关联 Railway 部署

<a href="https://railway.com/new">
  <img src="https://railway.com/button.svg" alt="Deploy on Railway" height="36">
</a>

> 如果上面的 Railway 按钮无法直接识别本仓库，可在 Railway 中选择 **Deploy from GitHub Repo**，然后选择本仓库。

### 部署流程（**具体操作看图片教程**）

**1. GitHub**
- 将本项目导入自己的 GitHub Repository。

**2. Railway**
- 点击上方 **Deploy on Railway**，或进入 Railway → **New Project** → **Deploy from GitHub Repo**。
- 选择本仓库并完成服务创建。

**3. Persistent Volume**
- 在 Railway 服务中添加 Persistent Volume。
- 挂载路径必须为：
  `/data`
- **不要删除 `/data` Volume**，否则持久化身份将无法继续复用。

**4. Railway Token**
- 如果需要程序自动检查/创建 Railway Public Domain 与 TCP Proxy，在 Railway Variables 中设置：
  `RAILWAY_TOKEN`
- Token 只需配置一次；程序支持兼容的 Railway Token 认证方式。

**5. Deploy**
- 点击 **Deploy** / **Redeploy**。
- 首次启动：初始化 `/data` 中的节点身份。
- 后续 Deploy / Redeploy / Restart：自动复用已有身份，不重新生成 UUID、Reality 密钥、Short ID 或 Subscription Token。

**6. 部署完成**节点身份只初始化一次，之后永久复用；运行时网络 endpoint 随当前 Deployment 重新发现。**

## 1. 部署要求

1. 将 GitHub 仓库连接到 Railway Service。
2. Service 使用仓库中的 `Dockerfile`。
3. 在 **Service → Volumes** 创建 Persistent Volume，并将 Mount Path 固定为 `/data`。
4. 在 **Service → Variables** 设置 `RAILWAY_TOKEN`。
5. Deploy。
6. 首次启动验证 `/data` 为真实 Persistent Volume；仅当 Volume 全新且为空时初始化 UUID、REALITY key、3 个 Short IDs 和 subscription token，并原子写入 `/data`。
7. 程序自动检查/创建当前 Deployment 所需的 Public Domain 与 TCP Proxy（Target `8080`）；仅执行非破坏性检查、复用或创建，不删除现有网络资源。
8. 后续 Restart / Redeploy / Container recreation 从 `/data` 读取并复用同一套节点身份；runtime manifest、订阅内容和 Railway endpoint 根据当前 Deployment 环境重新生成。

**Persistent Volume 必须在首次正式 Deploy 前完成挂载。** 缺少 Volume、身份不完整或身份完整性校验失败时，程序拒绝生成临时/替代身份。

## 2. 当前节点架构

正常运行时为 5 个节点：

| 节点 | 协议 | 传输 | 安全 | Endpoint 来源 |
|---|---|---|---|---|
| Node 01 | VLESS | XHTTP | TLS | Railway Public Domain :443 |
| Node 02 | VLESS | WebSocket | TLS | Railway Public Domain :443 |
| Node 03 | VLESS | RAW TCP | REALITY + Vision | Railway TCP Proxy |
| Node 04 | VLESS | XHTTP | REALITY | Railway TCP Proxy |
| Node 05 | VLESS | gRPC | REALITY | Railway TCP Proxy |

Cloudflare Tunnel + VLESS XHTTP TLS 保留为**可选 Node 06**；未启用时不计入运行节点数量。

当前 Gateway 监听 `:8080`，TCP Proxy Target 固定为 `8080`，并按 REALITY 节点将流量路由到 Xray 内部端口 `10087`、`10088`、`10089`。

## 3. Subscription Token 与 URL

Subscription Token 持久保存在 `/data/subscription_token.txt`，不会因为 Redeploy、Restart 或 Railway endpoint 变化而自动更换。

订阅 URL 使用当前 Railway Public Domain 与持久 Token 组合生成：

```text
https://<current-public-domain>/sub/<subscription-token>
```

`/data/subscription_url.txt` 是当前运行时生成的派生文件，不是身份来源。Railway Public Domain 变化后，程序重新生成 URL；Token 保持不变。

如需**主动轮换 Token**，使用环境变量：

```text
SUBSCRIPTION_TOKEN_ROTATE_ID=YYYYMMDD-NNN
```

例如：`20260904-001`。相同 Rotation ID 不会重复轮换；非法 Rotation ID 直接 fail closed，且不修改身份。轮换只替换 subscription token，并重新封印身份完整性；UUID、REALITY key、Short IDs 保持不变。

## 4. 启动与故障恢复

主进程由 `boot.sh` 直接运行。启动顺序：

```text
identity preflight
→ Gateway :8080
→ Railway networking reconciliation
→ runtime discovery
→ runtime / subscription generation
→ Xray
→ optional Cloudflare Tunnel
→ readiness
```

Gateway 或 Xray 主进程异常退出时，容器以失败状态结束，由 Railway `ON_FAILURE` 负责容器级恢复。重新启动后仍从 `/data` 复用同一身份。

Railway healthcheck 使用 `/health` 作为早期 liveness 检查；`boot.sh` 在完成 runtime、subscription 和 Xray readiness 检查后保持主进程运行。


⚠️ 使用说明

本项目仅供学习、研究和合法的网络技术测试使用。

请遵守所在地区法律法规、Railway、Cloudflare 及相关服务的使用条款。
