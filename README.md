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

在 router 上通过 SSH 运行一条命令（推荐）：

```sh
curl -fsSL https://raw.githubusercontent.com/echotreez/adguardvpn-merlin/main/adguardvpn -o /jffs/scripts/adguardvpn && chmod a+rx /jffs/scripts/adguardvpn && /jffs/scripts/adguardvpn install
```

或者手动：在客户端（下载了脚本的电脑）上进入脚本所在目录运行下面任一命令（Merlin 的 dropbear 没有 sftp-server，所以 scp 要加 `-O` 强制走旧协议）：

```sh
scp -O adguardvpn admin@<router-ip>:/jffs/scripts/adguardvpn
ssh admin@<router-ip> "cat > /jffs/scripts/adguardvpn" < adguardvpn
```

再在 router 上：

```sh
chmod a+rx /jffs/scripts/adguardvpn
/jffs/scripts/adguardvpn install
```

`install` 会按顺序完成整个流程：检查环境、安装 hook、通过 Entware 安装 adguardvpn-cli、登录、选择 location、录入初始域名、启动隧道并输出状态。中途失败后可随时重新运行 `adguardvpn setup` 继续。只想安装 add-on 本身用 `adguardvpn install --no-setup`。

升级：`adguardvpn self-update`（从本仓库 main 分支拉取并做语法检查后替换）。

前提：Administration → System 里启用 JFFS custom scripts、SSH；已装 Entware（amtm → `ep`）。

## 白名单（allow list）

`install` 会自动在 `/jffs/addons/adguardvpn.d/` 下生成以下文件，可以直接用 `vi` 手动编辑：

| 文件 | 作用 | 初始内容 |
|---|---|---|
| `domains.list` | 走 VPN 的域名白名单（核心文件） | 一行注释 |
| `ips.list` | 可选：固定 IPv4 / CIDR，不依赖 DNS，直接走 VPN | 一行注释 |
| `sources.list` | 可选：限定哪些 LAN 客户端参与分流，留空 = 全部 LAN 设备 | 一行注释 |
| `adguardvpn.conf` | 设置（`adguardvpn set` 写入），初始为空，使用默认值 | 空 |
| `enabled` | 服务开关标记，`start` 创建、`stop` 删除 | `start` 后才出现 |

### domains.list 格式

每行一个域名，`#` 之后是注释。子域名自动包含：写 `netflix.com` 同时覆盖 `www.netflix.com`、`api.netflix.com` 等。

```
# streaming
netflix.com
nflxvideo.net

openai.com   # 行内注释
https://www.github.com/foo   # 带协议或路径会自动取域名
*.example.org                # 开头的 *. 会自动去掉
```

- 大小写不敏感，统一转小写
- 不合法的行（如没有点的 `foo`）被忽略，重复项自动去重
- 只写主站域名通常不够，CDN / API 域名也要加（如 Netflix 的 `nflxvideo.net`），加完用 `adguardvpn test <域名>` 检查

### ips.list / sources.list 格式

每行一个 IPv4 地址或 CIDR，`#` 为注释，不支持 IPv6。

```
# ips.list
1.2.3.0/24
8.8.8.8

# sources.list
192.168.50.0/24
192.168.50.20
```

### 修改后如何生效

- 推荐用命令，会自动重启 dnsmasq 并预热：
  `adguardvpn domain add|remove|list|clear|import FILE|edit`、`adguardvpn ip add|remove|list`、`adguardvpn src add|remove|list`
- 手动编辑 `domains.list` 后必须执行 `adguardvpn reload`（dnsmasq 重启后才会读到新域名）
- 手动编辑 `ips.list` / `sources.list` 后执行 `adguardvpn restart`

## AdGuard CLI 连接时的两个提示

官方 Merlin 教程是为“整个路由器全部流量走 VPN”设计的，和本 add-on 的目标相反：

| 提示 | 官方教程的答案 | 实际效果 | 本 add-on 的做法 |
|---|---|---|---|
| `Would you like to set default routes in TUN mode?` | yes | `Tunnel routing mode: auto`，CLI 往路由表写默认路由，LAN 所有流量都被劫持进隧道 | 固定为 `none`：只建 tun 接口，不动路由表；分流由 ipset + fwmark + `ip rule` 完成 |
| `Would you allow AdGuard VPN to set system DNS?` | 教程未提及（常见做法 no） | `Set system DNS: on` 时 CLI 会改系统 DNS，可能覆盖路由器 dnsmasq 的上游 | 固定为 `off`：路由器 DNS 保持原样，dnsmasq 才能把白名单域名解析结果写入 ipset |

注意：`connect -y` 会对所有提示自动回答 yes，等于选了 auto。脚本因此不再使用 `-y`，连接前先把配置设成 `none/off`，并用 `config show` 校验，不符合就断开重连。连接后还会清理 CLI 可能写进主路由表的 `0.0.0.0/1`、`128.0.0.0/1` 劫持路由。

不要手工用官方教程的方式 `connect`（回答 yes），否则会变成全局 VPN。

## IPv6（例如 tunnelbroker 的 6in4 隧道）

路由器有 IPv6 时，浏览器会优先用 AAAA，直接走 IPv6 隧道，绕过 VPN。默认 `ipv6=block`：

- dnsmasq 把白名单域名的 AAAA 地址写入 `AGVPN_DOM6`
- ip6tables 在 FORWARD 里对这些地址 REJECT，客户端立刻回退到 IPv4，走 VPN
- 用 `adguardvpn test <域名>` 检查：应显示 AAAA 地址在 `AGVPN_DOM6` 里，且提示 “IPv6 to this domain is rejected”
- 局限：客户端要先解析过域名（地址进了 set）才会被拦；首次连接可能有一个短暂回退过程。客户端用 DoH 或写死 DNS 时 set 为空，IPv4 和 IPv6 都会绕过（用 DNS Director 强制走路由器 dnsmasq）

## 排障：`adguardvpn doctor`

一条命令输出所有关键信息：CLI 配置、tun 接口地址、主路由表里 tun0 的残留路由、`ip rule`、table 内容、各条 iptables/ip6tables 链、ipset 条目数、dnsmasq 规则数、`resolv.conf`、路由器自身经 WAN 与经 tun0 的出口 IP。分流不生效时把输出贴出来即可定位。

## 命令

| 类别 | 命令 |
|---|---|
| 服务 | `install` `uninstall` `start` `stop` `restart` `status` `menu` `cli-install` `self-update` |
| 白名单 | `domain add/remove/list/clear/import/edit` `ip add/remove/list` `src add/remove/list` `reload` `warm` `test DOMAIN` `doctor` |
| 设置 | `set [name value]`：iface table prio mark mask lan killswitch ipv6 dns home cli location fastest iponly timeout watchdog |
| CLI 对应 | `login` `logout` `license` `status` `list-locations [n]` `connect [-l LOC｜-f] [-4｜-6] [-v]`（`-y` 被忽略） `disconnect` `config <sub>` `site-exclusions <sub>` `check-update` `update [-y] [-v]` `export-logs [-o] [-f]` `cli <任意参数>` |

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

## License

MIT
