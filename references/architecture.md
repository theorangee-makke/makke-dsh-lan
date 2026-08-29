# 架构：makke-lan split-horizon 与 DSH 接入拓扑

> 更新基准：2026-08-29。源头文档 `~/WorkspaceAI/IT/makke-net/`；线上配置 `/opt/homebrew/etc/{dsh,makke-lan,makke-preview}`。

## 流量拓扑图

```
【在家 · 已配 DNS 的设备】
  Safari 输入 https://dsh.makke.net
    → 设备 DNS = 192.168.31.12 (mini 上 tls-proxy 内置 DNS, UDP :53)
    → 只对 dsh/preview 两个名字应答（无 AAAA）；**split-response**：
      LAN 客户端（192.168.31.0/24）回局域网 IP，Tailnet 客户端（100.64/10）回 mini 的 Tailnet IP
      其它名字转发 192.168.31.1 → 223.5.5.5；*.ts.net → 100.100.100.100 (MagicDNS)
    → mini :443 (tls-proxy, node, com.makke.lan-tls-proxy)
    → Let's Encrypt 真证书（lego 签发，SAN: dsh.makke.net + preview.makke.net）
    → 127.0.0.1:8080 (auth-gateway, com.marco.dsh-auth-gateway，密码认证)
    → 127.0.0.1:3080 (dsh-web, com.marco.dsh-web，仅监听回环)
    实测全链路 ~21ms

【出门 · 公网兜底】
  https://dsh.makke.net
    → 运营商 DNS → Cloudflare anycast（免费版无国内节点）
    → CF Access 邮箱 OTP（theorangee@gmail.com，双层认证的外层，实测 302 跳转）
    → CF Tunnel (cloudflared --protocol quic, com.marco.cloudflared)
    → 127.0.0.1:8080 同一个 auth-gateway（内层 Basic） → 3080
    实测 TTFB 0.7–2s；edge RTT ~385ms / 丢包 50%（2026-08-29 实测）
    ⚠ 慢是物理性的：应用层无解，只能换路径

【出门 · Tailscale 备用】
  https://marcomac-mini.taildfba33.ts.net:8443 （必须带 :8443）
    → Tailscale Serve → mini
    ⚠ Tailscale 本身不稳定：controlplane.tailscale.com 被 SNI 阻断后守护进程会挂
      → 重启 Tailscale App 恢复。不要当唯一路径

【出门 · Mac 控制端最快路径】
  UU Remote 端口映射：控制端本地端口 → mini:3080（国内中继）
  ⚠ iOS 版 UU 没有端口映射功能 → iPhone/iPad 出门只能走上面两条
```

## 组件与文件清单

| 组件 | LaunchAgent | 监听 | 代码/配置 |
|---|---|---|---|
| tls-proxy（TLS+DNS+跳转 三合一） | `com.makke.lan-tls-proxy` | `*:443` / `*:53` / `*:80` | `/opt/homebrew/etc/makke-lan/tls-proxy.mjs` |
| auth-gateway（密码认证反代） | `com.marco.dsh-auth-gateway` | `127.0.0.1:8080` | `/opt/homebrew/etc/dsh/auth-gateway.mjs` |
| dsh-web（DSH 本体） | `com.marco.dsh-web` | `127.0.0.1:3080` | wrapper: `/opt/homebrew/etc/dsh/web-wrapper.mjs` |
| cloudflared | `com.marco.cloudflared` | 出站 QUIC | `~/.cloudflared/config.yml` |
| 证书周续期 | `com.makke.lego-renew` | — | `/opt/homebrew/etc/makke-lan/renew-lego.sh` |

- 证书位置：`~/.local/share/lego/certificates/dsh.makke.net.{crt,key}`；DNS-01 用 CF token（`~/.config/makke/cf-dns.token`），renew-lego.sh 有 `MIN_DAYS=20` 到期自检 + macOS 桌面告警。2026-08-29 时有效期至 **2026-11-20**。
- 认证：auth-gateway 用 SHA-256 口令哈希（`/opt/homebrew/etc/dsh/auth/pw.sha256`）+ HMAC 签名 cookie `dsh_auth`（HttpOnly SameSite=Strict，24h）+ Basic 兜底（**用户名必须是 `marco`**）。cookie secret 在 `/opt/homebrew/etc/dsh/auth/cookie.secret`。
- 网关 Origin 白名单（`DSH_ALLOWED_ORIGINS`）：ts.net:8443（带端口）、ts.net、dsh.makke.net。WS 升级靠它，缺了表现为"能登录但会话历史空白"。
- mini 网络前提：Wi-Fi IPv4 手动固定 `192.168.31.12`（en0 有线 + Wi-Fi 同 IP）+ 路由器 DHCP 对两个 MAC 都做保留 + 关闭私有无线地址。

## 设计原则（为什么这样）

1. **dsh-web 永远只绑回环**：web 界面本质能执行任意代码（RCE 级），DSH 官方拒绝 `--host 0.0.0.0` 启动。网络暴露面一律由 auth-gateway 承担（密码 + 限流 + 安全头）。
2. **一个域名走天下**：`dsh.makke.net` 在任何设备、任何网络都是同一个书签；在家/出门的差异只在 DNS 解析结果，设备无感切换。
3. **443 给 node 不给 nginx**：macOS 防火墙拦 nginx 入站，node 在白名单；绑具体 LAN IP `:443` 会 EACCES，通配 `0.0.0.0:443` 才行（tls-proxy 层有密码，安全）。
