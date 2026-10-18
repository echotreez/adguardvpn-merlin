# adguardvpn — Asuswrt-Merlin 域名分流 add-on

只有 `domains.list` 里的域名（含子域名）走 AdGuard VPN，其余流量保持 router 正常 NAT 出口。

## 原理

- dnsmasq `ipset=/domain/AGVPN_DOM` 把解析出的 IP 加入 ipset
- mangle PREROUTING 只对 NEW 连接打 fwmark，CONNMARK 保存，已建立的连接不会中途换路径
- `ip rule fwmark → table 1893 → default dev tun0`
- AdGuard CLI 固定为 `TUN` + `tun-routing-mode NONE` + `change-system-dns off`，不碰默认路由和 resolv.conf
- IPv6：默认 `block`（对白名单域名的 AAAA 目的地 REJECT，客户端回退 IPv4，避免泄漏）
- 可选 kill switch：tunnel 断开时白名单流量直接丢弃，不走 WAN

## 安装

在 Mac 上：

```sh
scp ~/Documents/config/network/adguardvpn/adguardvpn admin@<router-ip>:/jffs/scripts/adguardvpn
```

在 router 上（SSH）：

```sh
chmod a+rx /jffs/scripts/adguardvpn
/jffs/scripts/adguardvpn install
adguardvpn cli-install
adguardvpn login
adguardvpn connect -l "Tokyo"
adguardvpn domain add netflix.com openai.com
adguardvpn start
```

前提：Administration → System 里启用 JFFS custom scripts、SSH；已装 Entware（amtm → `ep`）。

## 命令

| 类别 | 命令 |
|---|---|
| 服务 | `install` `uninstall` `start` `stop` `restart` `status` `menu` `cli-install` `self-update` |
| 白名单 | `domain add/remove/list/clear/import/edit` `ip add/remove/list` `src add/remove/list` `reload` `warm` `test DOMAIN` |
| 设置 | `set [name value]`：iface table prio mark mask lan killswitch ipv6 dns home cli location fastest iponly timeout watchdog |
| CLI 对应 | `login` `logout` `license` `status` `list-locations [n]` `connect [-l LOC｜-f] [-4｜-6] [-y] [-v]` `disconnect` `config <sub>` `site-exclusions <sub>` `check-update` `update [-y] [-v]` `export-logs [-o] [-f]` `cli <任意参数>` |

无参数运行进入 amtm 风格菜单。`config set-tun-routing-mode` 非 NONE、`set-change-system-dns on`、`set-mode SOCKS` 会被拒绝（`AGVPN_FORCE=1` 可强制）。

`src add 192.168.50.0/24` 可限定只有这些 LAN 客户端参与分流；留空则全部 LAN。

`set dns 94.140.14.14`（可选）：白名单域名的 DNS 查询也走隧道，避免 DNS 泄漏。

## 安装后验证（按顺序）

```sh
adguardvpn status
ip -4 addr show tun0
ip rule
ip route show table 1893
ipset list AGVPN_DOM | head
iptables -t mangle -S AGVPN_MARK
adguardvpn test netflix.com
```

端到端测试：先 `adguardvpn domain add ifconfig.me`，然后在 LAN 客户端上 `curl ifconfig.me` 应显示 VPN 出口 IP；`curl ifconfig.co` 应仍显示本地 ISP 出口 IP。

## 需要实机确认的点

1. `tun0` 在 NONE 模式下是否有 IPv4 地址（MASQUERADE 需要）。AdGuard 文档没有明确说明。检查 `ip -4 addr show tun0`；如果没有 `inet`，把 `ip addr show tun0`、`adguardvpn-cli config show`、`/tmp/adguardvpn_connect.log` 的输出发回来再定方案。
2. fwmark `0x2000000` 是否与你的 VPN Director / QoS 冲突：`ip rule` 和 `iptables -t mangle -S` 里查；冲突就 `adguardvpn set mark 0x...` 和 `set mask 0x...`。
3. 客户端必须使用路由器的 dnsmasq 解析。浏览器 DoH、写死 DNS、AdGuardHome 接管 53 端口都会让 ipset 为空；用 DNS Director 强制重定向。
4. 白名单要覆盖站点用到的所有域名（CDN、API 域名）；用 `adguardvpn test <domain>` 逐个确认。
5. CLI 自带的 `site-exclusions` 只作用于 CLI 自身，不影响本方案的 LAN 转发分流。

## 文件

- `/jffs/scripts/adguardvpn`：主脚本
- `/jffs/addons/adguardvpn.d/`：`adguardvpn.conf` `domains.list` `ips.list` `sources.list` `enabled`
- Hook（带 `### adguardvpn start/end` 标记，`uninstall` 会清理）：services-start、firewall-start、nat-start、wan-event、post-mount、service-event、dnsmasq.postconf
- cron：`adguardvpn` 每分钟 watchdog（断线重连、规则补齐）

## 卸载

```sh
adguardvpn uninstall
```
