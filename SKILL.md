---
name: makke-dsh-lan
description: Use when troubleshooting, improving, or asking about access to dsh.makke.net / DSH web from any device (MacBook Air, iPad, iPhone, other Macs, remote networks) — covers slowness, DNS split-horizon, TLS, certificates, Cloudflare tunnel, Tailscale, and connectivity paths on the mac-mini. 触发词：dsh.makke.net 慢/打不开、局域网直连、split-horizon、tls-proxy、设备连接 DSH。
---

# makke-dsh-lan：DSH 多设备接入与 makke-lan split-horizon 运维

mini（192.168.31.12）上的 DSH web 只监听 `127.0.0.1:3080`（**官方安全设计，永远不要改**）。
所有外部访问都经过 `auth-gateway:8080`（密码认证）。有三条入口路径，本 skill 告诉你
"谁在什么场景该走哪条、怎么验证、踩过哪些坑"。

## 路径决策表（先看这个）

| 场景 | 用什么地址 | 预期速度 | 前提 |
|---|---|---|---|
| 在家 · MacBook Air | `https://dsh.makke.net` | ~60ms | 自动（home-split-dns agent 每 45s 切 DNS） |
| 在家 · iPad / iPhone | `https://dsh.makke.net` | ~21ms | **手动两步**：Wi-Fi DNS=`192.168.31.12` ＋ 关 iCloud 私人代理 |
| 在家 · 其它 Mac | `https://dsh.makke.net` | ~21ms | 手动设 DNS 同上 |
| 出门 · 有 Tailscale | `https://marcomac-mini.taildfba33.ts.net:8443` | 快 | **必须带 :8443**；Tailscale 本身不稳，别当唯一路径 |
| 出门 · 没有 Tailscale | `https://dsh.makke.net` | 0.7–2s | CF Tunnel 兜底，慢是物理性的，别试图优化 |
| 出门 · Mac 控制端 | UU Remote 端口映射 → mini 3080 | 最快 | **iOS 的 UU 没有端口映射功能** |

## 30 秒诊断流程

1. **判定设备走的哪条路**（mini 上执行）：
   `lsof -nP -iTCP:443 | grep ESTABLISHED` —— 设备 IP（192.168.31.x）出现 = 走 LAN 快路径；
   不出现 = 还在绕 Cloudflare 公网。需要抓间歇性连接时用后台 watcher 循环采样（见 references/diagnostics.md）。
2. **设备端自测（零工具）**：Safari 无痕标签打开域名，从回车到**弹出密码框**的时间：
   <0.5s = 快路径 ✅；白屏 1–3s = 还在走 CF ❌。
3. **慢路径 → 按顺序查设备设置**：① iCloud 私人代理关了没（头号杀手，会无视手动 DNS）
   ② Wi-Fi DNS 是否手动 `192.168.31.12` ③ 代理 App 给 `*.makke.net` 设 DIRECT ④ 浏览器 DoH 关。
4. **快路径仍慢** → 查 mini 侧：`references/diagnostics.md` 的网关/证书/后端三连测。

## 红线（碰了会出事故）

- **绝不**把 dsh-web 的 `--host` 改成 `0.0.0.0`：DSH 官方拒绝（RCE 安全设计），
  会导致 KeepAlive 崩溃循环直到手动恢复（2026-08-29 事故）。LAN 直连需求一律走 gateway 方案。
- **绝不**在活跃 DSH 会话里 kill/bootout `com.marco.dsh-web`；重启走
  `/Users/marco/.local/bin/dsh-request-restart --reason "..."`，且改 launchd 前必须备份 + 临时端口试跑验证。
- 不要把 `makke.net` 加进 Tailscale Split DNS；不要开 Tailscale "Override local DNS"；
  不要复活 Tailscale Funnel（曾占用 443）。
- 不要启用 brew 的 dnsmasq 服务（会抢 53 端口；旧 dnsmasq.conf 里的 .210 是历史遗留）。

## 深入阅读（按需加载）

- `references/architecture.md` — 完整拓扑：组件链、端口、证书、LaunchAgent 清单、流量走向
- `references/devices.md` — 每类设备的接入配置步骤与验证方法
- `references/pitfalls.md` — 踩坑全录（含事故复盘与"为什么这么设计"）
- `references/diagnostics.md` — 诊断命令手册（路径判定、网关三连测、日志解读）

## 相关资产

- 源头文档：`~/WorkspaceAI/IT/makke-net/`（文档与副本）；线上真实配置 `/opt/homebrew/etc/{dsh,makke-lan,makke-preview}`
- 本 skill 的 Git 仓库：`https://github.com/theorangee-makke/makke-dsh-lan`（改动后记得 push）
