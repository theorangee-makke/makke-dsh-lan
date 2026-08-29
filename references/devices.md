# 各设备接入指南

## MacBook Air（唯一全自动设备）

- 自动化：LaunchAgent `com.makke.home-split-dns`（在 Air 上），每 45s 检测：
  在家（默认网关=192.168.31.1）→ DNS 指到 mini + `tailscale set --accept-dns=false`；
  出门 → Ali DNS + `--accept-dns=true`。切不成功会弹通知。
  脚本：`~/Library/Application Support/makke/home-split-dns.sh`（Air 上）；
  日志：`~/Library/Logs/makke-home-split-dns.log`。
- 前提：浏览器"安全 DNS/DoH"关；代理（Veee 等）已配 `*.makke.net` DIRECT。

## iPad / iPhone（手动两步，2026-08-29 起 iPad mini 已验证）

1. **关 iCloud 私人代理**：设置 → 顶部 Apple ID → iCloud → 私人代理 → 关闭。
   （头号杀手：它用 Apple 加密 DNS，无视一切手动 DNS 设置。没有这一项 = 没订阅，跳过）
2. **Wi-Fi DNS 手动**：设置 → Wi-Fi → 已连网络 ⓘ → DNS → 配置 DNS → 手动 →
   删掉原有（通常是 192.168.31.1）→ 添加 `192.168.31.12` → 存储。
3. 彻底重开 Safari（上滑杀掉再开，清 DNS 缓存）→ 打开 `https://dsh.makke.net`。
4. 代理类 App（Veee 等）内给 `*.makke.net` 设 DIRECT；浏览器关"安全 DNS"。
5. 出门无忧：DNS 设置绑定单个 Wi-Fi 网络，走蜂窝时自动用运营商 DNS → 域名照常打开（回落 CF 公网）。

**验证**（三个任选）：
- mini 上 `lsof -nP -iTCP:443 | grep ESTABLISHED` 出现该设备 IP（间歇连接用 watcher 循环采样，见 diagnostics.md）
- Safari 无痕标签打开，密码框 <0.5s 弹出 = 快路径
- 页面秒开

## 其它 Mac（如公司 MBP）

同 iPad 两步（macOS 系统设置 → WLAN → 详细信息 → TCP/IP → DNS 手动 192.168.31.12）；
出门建议优先 UU Remote 端口映射（本地端口 → mini 3080），Tailscale 地址 `:8443` 备用。

## Apple TV / 访客设备

无需配置。它们不访问 dsh.makke.net 就完全无感；DNS 转发链对普通网站透明。

## 设备 IP 对号方法（家里设备 IP 表保留在本地，不入库）

- `lsof` 抓到 192.168.31.x 后，到路由器管理页按主机名对号（Apple 设备 MAC 全部随机化，
  靠主机名识别；IP 可能因 DHCP 漂移，建议对常驻设备做 DHCP 保留）。
- 或 `arp -a` 辅助。Apple 设备 MAC 随机化后 `arp` 认不出型号。
