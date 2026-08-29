# 诊断命令手册

## 1. 判定设备走的哪条路（最常用）

```bash
# 快照：谁正连着 mini:443（出现 192.168.31.x = LAN 快路径）
lsof -nP -iTCP:443 | grep ESTABLISHED

# 设备连接是间歇的 → 后台 watcher 循环采样（示例 3 分钟，每秒一次）
rm -f /tmp/lan443-watch.log
for i in $(seq 1 180); do
  ts=$(date +%H:%M:%S)
  ips=$(lsof -nP -iTCP:443 2>/dev/null | grep ESTABLISHED \
        | grep -oE "192\.168\.31\.[0-9]+" | sort -u | tr '\n' ' ')
  [ -n "$ips" ] && echo "$ts $ips" >> /tmp/lan443-watch.log
  sleep 1
done
echo DONE >> /tmp/lan443-watch.log
# 用 run_in_background 跑，配合 ask_user_question 让用户配合在手机上打开页面
# 然后统计: awk '{$1="";print}' /tmp/lan443-watch.log | tr ' ' '\n' | grep ^192 | sort | uniq -c
```

设备 IP 对号：查用户路由器主机名表，或 `arp -a`（注意 Apple 设备 MAC 全随机化，认不出型号）。

## 2. 设备端零工具自测

Safari **无痕标签**打开 `https://dsh.makke.net`：回车 → 弹出账号密码框的时间。
（无痕保证 cookie 失效、每次都弹密码框；这个时间 ≈ 纯网络+TLS+网关耗时）
- <0.5s = LAN 快路径 ✅
- 1–3s 白屏 = CF 公网路径 ❌ → 按 devices.md 检查三杀手

## 3. mini 侧链路三连测

```bash
# DNS 分流是否在答
dig +short @192.168.31.12 dsh.makke.net A        # 期望: 192.168.31.12

# LAN 全链路（真证书，勿加 -k）：21ms 左右 + 401 = 全部正常
curl -o /dev/null -sS --resolve dsh.makke.net:443:192.168.31.12 \
  -w "http=%{http_code} total=%{time_total}s\n" -m 8 https://dsh.makke.net/

# 公网兜底对照：0.7–2s 属正常（CF 免费版物理极限）
curl -o /dev/null -sS -w "ttfb=%{time_starttransfer}s\n" -m 20 https://dsh.makke.net/

# 证书有效期
echo | openssl s_client -connect 192.168.31.12:443 -servername dsh.makke.net 2>/dev/null \
  | openssl x509 -noout -dates -subject

# 后端本体
curl -o /dev/null -sS -w "http=%{http_code} total=%{time_total}s\n" -m 5 http://127.0.0.1:3080/
```

基准参考（2026-08-29 实测）：DNS 应答即时 · LAN 全链路 21ms/401 · CF 公网 0.7–2s · 国内基线 ping 7ms · CF edge ping 均值 385ms 丢包 50%。

## 4. 服务健康

```bash
launchctl list | grep -iE "dsh|makke|cloudflared"   # 各服务 PID 与上次退出码
lsof -nP -iTCP:3080 -sTCP:LISTEN                    # 必须是 127.0.0.1（0.0.0.0 = 有人改坏了）
lsof -nP -iTCP:443 -sTCP:LISTEN; lsof -nP -iUDP:53  # tls-proxy 双端口
cat ~/.dsh/run/last-restart.json                    # 上次重启记录（state: failed 要警惕）
tail -20 ~/Library/Logs/dsh/web.error.log           # 启动报错（--host 拒绝信息在这）
tail -20 ~/Library/Logs/makke/tls-proxy.log         # 只记启动行，无请求日志
tail -30 ~/Library/Logs/dsh/gateway.log             # [gateway-retry] 是 keep-alive 噪音，非故障
```

## 5. 网关与认证（测试新网关实例的标准姿势）

```bash
# 网关完全由环境变量驱动，可零风险试跑（务必用临时端口 + 临时凭据文件）：
printf 'test-secret' > /tmp/t.secret
printf '%s' "$(printf 'test-pass' | shasum -a 256 | cut -d' ' -f1)" > /tmp/t.hash
cd /opt/homebrew/etc/dsh
DSH_GATEWAY_PORT=18080 DSH_GATEWAY_HOST=127.0.0.1 \
DSH_COOKIE_SECRET_FILE=/tmp/t.secret DSH_PASSWORD_HASH_FILE=/tmp/t.hash \
DSH_ALLOWED_ORIGINS=http://127.0.0.1:18080 \
nohup node auth-gateway.mjs > /tmp/t.log 2>&1 & sleep 1.5
# 期望：无认证 401（WWW-Authenticate: Basic realm="DSH"）；
#       -u "marco:test-pass" → 200（用户名必须 marco，这是踩过的坑）
kill %1; rm -f /tmp/t.secret /tmp/t.hash
```

## 6. 修改 launchd 服务的纪律（dsh-self-maintenance skill）

1. 改前时间戳备份：`cp xx.plist xx.plist.bak-$(date +%Y%m%d-%H%M%S)`
2. 最小改动 + 所有不需要重启的检查先做完（plutil -lint / 临时端口试跑）
3. 绝不在会话里 kill/bootout `com.marco.dsh-web`
4. 需要重启时，回合最后一个动作：`/Users/marco/.local/bin/dsh-request-restart --reason "..."`
5. 恢复后先读 `~/.dsh/run/last-restart.json` 验证再继续
