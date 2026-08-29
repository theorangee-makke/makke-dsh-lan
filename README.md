# makke-dsh-lan

DSH (DeepSeek Harness) 多设备接入与 makke-lan split-horizon 运维 Skill。

解决"在 Mac / iPad / iPhone / 远程网络下，怎样以最快且最稳的方式访问 `dsh.makke.net`"，
以及围绕它的 DNS 分流、TLS 证书、Cloudflare Tunnel、Tailscale、auth-gateway 的全部踩坑经验。

## 结构

- `SKILL.md` — 主文件：路径决策表、30 秒诊断流程、红线
- `references/architecture.md` — 完整拓扑、组件清单、设计原则
- `references/devices.md` — 各设备接入步骤（Air 自动 / iPad / iPhone 手动两步）
- `references/pitfalls.md` — 踩坑全录（含 2026-08-29 `--host 0.0.0.0` 崩溃循环事故复盘）
- `references/diagnostics.md` — 诊断命令手册（路径判定 / 三连测 / 网关试跑 / launchd 纪律）

## 安装

```bash
git clone https://github.com/theorangee-makke/makke-dsh-lan.git ~/.dsh/skills/makke-dsh-lan
```

源头文档：`~/WorkspaceAI/IT/makke-net/`（mac-mini 本地）
