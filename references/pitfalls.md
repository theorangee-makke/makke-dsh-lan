# 踩坑全录（含事故复盘）

按"血泪程度"排序。每条都是真实发生过的事。

## 💀 1. dsh-web 改 `--host 0.0.0.0` → 崩溃循环（2026-08-29 事故）

- **经过**：为让 iPad 局域网直连，把 `com.marco.dsh-web.plist` 的 `--host 127.0.0.1` 改成 `0.0.0.0`，
  未做预启动验证就请求重启。DSH 官方直接拒绝：
  `error: --host 0.0.0.0 is intentionally not supported yet for safety: it would expose remote code execution to the network`
  → web 界面本质是 RCE，绝不允许裸奔到网络。KeepAlive 以 ~5s 间隔崩溃循环 6 次，服务全断，靠 Marco 手动恢复。
- **教训**：①改 launchd 前必须先在临时端口试跑完整验证（这次试跑纪律是事后补的）
  ②LAN 直连需求一律走 auth-gateway 方案（密码认证承担暴露面），不碰 dsh-web 绑定
  ③`--trusted-host 192.168.31.12` 早就在 plist 里预埋，说明此需求早有规划——动手前先搜现有配置。

## 🔪 2. 手机"设了 DNS 还是慢"——三个隐形杀手

- **iCloud 私人代理**（设置 → Apple ID → iCloud → 私人代理）：Apple 加密 DNS，无视一切手动 DNS。
  2026-08-29 实测：iPad mini DNS 已改对，代理没关 → 3 分钟监视器 180 次采样零命中。
- **浏览器 DoH**（Chrome"安全 DNS"等）：同理绕过系统 DNS。家里必关。
- **代理 App**（Veee 等）全局接管：流量根本不过系统 DNS 路由。`*.makke.net` 必须 DIRECT。
- 通用症状：`dsh.makke.net` 解析到 Cloudflare anycast（104.21.x / 172.67.x）而非 192.168.31.12。

## ⚠ 3. ts.net 地址的端口与 Origin 陷阱（2026-08-23 事故）

- `https://<mini-hostname>.<tailnet>.ts.net` **不带 ：8443** → 证书对不上（443 被 tls-proxy 的 dsh 证书占着）。
- 网关 Origin 白名单必须有**带端口**的版本 `https://<mini-hostname>.<tailnet>.ts.net:8443`，
  缺了 = 能登录但会话历史空白（WebSocket 403）。
- 不要把 makke.net 加进 Tailscale Split DNS；不要开"Override local DNS"；Tailscale Funnel 之前占过 443，已移走别复活。

## ⚠ 4. mini DHCP 搬家 → 全家 split-horizon 静默失效（2026-08-23 前科）

- 旧 DNS 配置硬编码 `192.168.31.210`，mini 换到 .12 后 LAN 直连全断。
- 现状：tls-proxy.mjs 启动时自动探测 LAN IP（正则 192.168.31.x，排除 .1），每 30s 复查。
- 防回归三件套：mini Wi-Fi IPv4 手动固定 .12；路由器对**两个 MAC**（随机化 + 硬件）都做 DHCP 保留；关闭私有无线地址。
- 旧 `dnsmasq.conf` 里的 .210 是历史遗留，brew 的 dnsmasq 服务别再启用（会抢 53 端口）。

## ⚠ 5. nginx 进不了场

- macOS 应用防火墙拦 nginx 入站连接；node 已在白名单（例外 #10）。
- 另：绑**具体** LAN IP 的 ：443 会 EACCES，通配 `0.0.0.0:443` 反而可以（node）。

## ℹ 6. 噪音 vs 真故障（别误诊）

- `gateway.log` 里大量 `[gateway-retry] POST /api/host.describe stale socket (ECONNREFUSED)`：
  keep-alive 连接复用失败后立即重试，每次代价 ~1ms，**不是故障**（2026-08-29 计 482 条，全程无感）。
- cloudflared 日志 `stream N canceled by remote with error code 0`：浏览器取消请求（页面刷新掐掉在途 XHR），良性。

## ⚠ 7. makke-net AGENTS.md 红线（完整版见 ~/WorkspaceAI/IT/makke-net/AGENTS.md）

- **tls-proxy.mjs 同时是全家 DNS 服务器**：重启它 = 家里所有设备 DNS 断一下。改它之前想清楚。
- tls-proxy DNS 的 split-response 改动（`splitResponse`/`isAllowedClient`）必须同时顾
  LAN 回内网 IP、Tailnet 回 Tailnet IP 两条路径。
- **preview.makke.net 的 LAN 路径无认证**，直接实时读 `~/WorkspaceAI`（no-store）——
  往 Games/Products/3D/Travel bucket 放东西前想想会不会被家里网络看见。
- 证书 DNS-01 token 在 `~/.config/makke/cf-dns.token`，**永远不要写进任何会被提交的文件**。
- 改线上文件前 cp 一份到 `IT/makke-net/config/` 留档；mini 上不要再堆 `.bak-YYYYMMDD` 后缀备份。
- mini 的 ssh 别名：在家 `ssh mini`，在外 `ssh mini-ts`。

## ℹ 8. 排查口诀

- **在家所有网站都打不开**（不只 DSH）→ 先看 mini 在不在线（整个 LAN 的 DNS 都指它）
- **DSH 打不开但网正常** → 无痕标签开域名，看密码框弹出快慢判定走的哪条路
- **远程会话"卡住"** → 多半是 mini 本机屏幕上有 GUI 弹窗在等确认（sudo/osascript 原生对话框远程不可见；
  DSH web 的审批弹窗远程可见）。无人值守场景避免 sudo。
- **证书告警** → renew-lego.sh 的 MIN_DAYS=20 自检会发 macOS 通知；手动跑：
  `/opt/homebrew/etc/makke-lan/renew-lego.sh`（CF token 在 ~/.config/makke/cf-dns.token）
