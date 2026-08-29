# makke-dsh-lan

DSH (DeepSeek Harness) 多设备接入与 split-horizon LAN 运维 Skill —— 让 agent 一句话回答「这台设备现在该用哪条路访问我的自托管服务，怎么验证，踩过哪些坑」。

解决的问题：在家 / 出门、Mac / iPad / iPhone / 远程网络下，怎样以最快且最稳的方式访问 `dsh.makke.net`，以及围绕它的 DNS 分流（split-horizon）、TLS 证书、Cloudflare Tunnel、Tailscale、auth-gateway 的全部踩坑经验。

## 安装

```bash
git clone https://github.com/theorangee-makke/makke-dsh-lan.git ~/.dsh/skills/makke-dsh-lan
```

> Claude Code / OpenClaw 类运行时同理：把目录放进各自的 skills 目录即可，入口是 `SKILL.md`。

## 结构

- `SKILL.md` — 主文件：路径决策表、30 秒诊断流程、红线
- `references/architecture.md` — 完整拓扑、组件清单、设计原则
- `references/devices.md` — 各设备接入步骤（Air 自动 / iPad / iPhone 手动两步）
- `references/pitfalls.md` — 踩坑全录（含 `--host 0.0.0.0` 崩溃循环事故复盘）
- `references/diagnostics.md` — 诊断命令手册（路径判定 / 三连测 / 网关试跑 / launchd 纪律）

## 隐私说明

本仓库为个人 homelab 经验的开源版本。域名与 RFC1918 内网段保留以保持文档可用；Tailscale 主机名、邮箱、家庭设备 IP 表已替换为占位符。真实值只存在于本地源头文档中。

## License

[MIT](LICENSE)
