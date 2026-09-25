# ss2anytls-autodeploy

通过 sing-box 内核，一键在公网服务器 B（中转）和服务器 C（出口）之间部署 ss-anytls 隧道。
Autodeploy ss-anytls tunnel between server B (relay) &amp; C (exit) with sing-box core.

## 一键脚本

```bash
curl -O https://raw.githubusercontent.com/Cyli00/ss2anytls-autodeploy/refs/heads/main/autodeploy.sh
chmod +x autodeploy.sh
bash autodeploy.sh
```

## 模式说明

- **[1] B (Relay)** — 中转服务器：运行 SS-2022 入站，通过 AnyTLS 出站连接到服务器 C
- **[2] C (Exit)** — 出口服务器：运行 AnyTLS 入站，需要公网 IP

## 建议流程

1. **在服务器 C** 运行脚本 → 选择 `[2] (Exit)`，按提示设置端口和 Tag
2. 复制显示的 IP、Port、Password（以及 AnyTLS URI）
3. **在服务器 B** 运行脚本 → 选择 `[1] (Relay)`
4. 粘贴 C 的信息，设置本地端口和节点名称
5. 获得 SS URI 链接，分享给用户直接导入客户端

## 输出说明

- **C 端（Exit）**：输出 AnyTLS URI（`anytls://`），可直接导入支持 sing-box URI 的客户端
- **B 端（Relay）**：输出 Shadowsocks URI（`ss://`），用户可直接导入客户端使用

## 功能特性

- 支持多次运行添加多个入站/路由
- 自签 TLS 证书，无需域名
- 可选 SNI Server Name 配置
- 自动检测端口和 Tag 冲突，支持覆盖
- URI 一键导入链接输出


---

## 本 fork 的改动（相对上游 Cyli00/ss2anytls-autodeploy）

### 1. 修复「Sing-box 安装失败却显示成功」

上游是 `if curl -fsSL https://sing-box.app/install.sh | sh; then`：管道让 `if` 判断的是 `sh` 的退出码，
`sing-box.app` 连不上（`curl: (28) Failed to connect ... port 443`）时 `sh` 读到空输入仍返回 0，
于是照样打印 `[SUCCESS] Sing-box 安装完成`，然后继续改配置、要求输入 Server C 信息。

本 fork 改为 `install_sing_box()`：

- 从 **GitHub Release API** 取最新版本（不依赖 sing-box.app）；
- 按 CPU 架构（x86_64/amd64、aarch64/arm64、armv7l）选择安装包；
- 下载、解压、可执行文件、`sing-box version` **四重校验**，任一步失败直接 `exit 1`；
- 安装到 `/usr/local/bin/sing-box`；
- 自动创建 `/etc/systemd/system/sing-box.service`（上游依赖 sing-box.app 安装器创建，网络不通时就没有）。

### 2. 加密方式改为 ChaCha20

上游固定 `2022-blake3-aes-128-gcm`。实测在没有 AES-NI 的机器上（利群那类 CPU 型号被屏蔽的 KVM）：

| 算法 | 单核软件速度 |
| --- | --- |
| AES-128-GCM | ~106 MB/s |
| ChaCha20-Poly1305 | ~465 MB/s |

所以本 fork 全部改为 `2022-blake3-chacha20-poly1305`，SS-2022 的密钥长度相应由 16 字节改成 32 字节
（`openssl rand -base64 32`）。**协议与架构不变**，只是换了 AEAD。

### 3. 其它

- `autodeploy-reality.sh` 同样修复了安装判断；
- 三个脚本的 SS 密钥长度统一切到 32 字节；
- 上游当前的 `c_pass` / `c_server_name` 已是两个独立 `read`，不存在「密码写进 SNI 变量」的问题
  （我核对过三个脚本），如果你手上的版本有这个问题，直接在本 fork 基础上再跑一次即可。

用法与上游一致：先在 Server C 选 `[2]`，再到 Server B 选 `[1]`。
