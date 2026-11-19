# 从新 VPS 部署和恢复中转服务

这份手册把三个入口、共享订阅、家庭防火墙和 VPS 出口接到一起。域名示例使用 `example.net`，端口号采用参考部署的公网服务端口。

## 1. 准备系统、域名和证书

1. 在 VPS 上用 SSH 密钥登录，确认供应商控制台可用于恢复。安装 Nginx、3x-ui 及其配套 Xray、cloudflared 和证书签发工具。参考面板版本是 3.11.0。记录本机的版本、服务名和配置路径，便于恢复。[3x-ui 参考版本](https://github.com/MHSanaei/3x-ui/releases/tag/v3.11.0) · [cloudflared 官方发行包](https://github.com/cloudflare/cloudflared/releases)
2. 给直达入口使用独立域名，例如 `relay.example.net`。它的 DNS A 记录直接指向 VPS 公网 IPv4，在 Cloudflare DNS 中选「仅 DNS」。给面板和订阅分别使用 `panel.example.net`、`sub.example.net`，将它们关联到同一条 Cloudflare Tunnel。填写 AAAA 前，先确认 IPv6 监听、路由和防火墙都已就绪。
3. 为直达入口准备匹配域名的有效证书和完整证书链。参考部署使用 ACME 自动续期。将证书和私钥部署到 Nginx 的本地 TLS 站点以及 HY2 配置引用的位置。TCP 443 由 REALITY 占用时可用 DNS-01 完成域名验证。续期后让用证书的服务加载新证书，分别检查本地 TLS 站点和 HY2 握手时看到的有效期。

## 2. 接通三个入口和管理面

| 用途 | 从客户端到 VPS 服务的路径 | 核对点 |
| --- | --- | --- |
| REALITY | `relay.example.net:443/TCP` → Xray VLESS + Vision + REALITY | Xray 占用公网 TCP 443。参考部署的 REALITY 目标为同机 Nginx 的回环 TLS 站点，域名、SNI 与证书对应。订阅发布该入口的公钥、short ID 和账号的 VLESS 凭据。 |
| Hysteria2 | 同一入口域名 → VPS 的 UDP 8443–8447 → Xray 的 UDP 8443 | 放行 UDP 8443，把 8444–8447 的 UDP 转到 8443。节点发布相同的跳端口范围，使用该账号的 HY2 凭据，并校验服务端证书。 |
| Cloudflare Tunnel | `sub.example.net:443` → Cloudflare 边缘 → cloudflared → 回环 Nginx → Xray 的 TLS WebSocket 入站 | cloudflared 主动连接 Cloudflare。Nginx 根据 WS 路径转发并传递 HTTP/1.1 Upgrade 与 Connection 头。客户端 Host、路径、服务端路径要一致。 |
| 订阅和面板 | 订阅域名 → 同一 Nginx → 3x-ui 的回环订阅监听。面板域名 → Tunnel → 3x-ui 的回环管理监听 | 配置两个主机名的 Tunnel 源站。Nginx 按订阅路径与 WS 路径分流。为管理入口配置身份验证，验证订阅客户端能收到 YAML。 |

在面板为每种入站填写客户端实际可达的发布地址、端口和 TLS 名称。CF 节点发布订阅域名的公网 443 与约定的 WS 路径，回环地址只作为源站。REALITY 和 HY2 发布直达域名。

在 VPS 用 `ss -lntup` 对照每个监听，用 `nginx -t` 校验配置。参考部署的 Tunnel 由 Cloudflare 远程管理，可在 Tunnel 的已发布应用路由中核对域名、源站协议和端口。VPS 使用 firewalld 时，核对实际网卡所属区域、运行和持久化的端口及转发规则。供应商另设云防火墙时，也要核对相同的公网服务端口。保留管理通道及同机其他业务的规则。最后从客户端分别固定三个入口做认证请求，HY2 按每个公布的端口验证。

## 3. 新增订阅账号

1. 在 3x-ui 创建账号并绑定三种入站。REALITY 使用该账号的 VLESS UUID 和 `xtls-rprx-vision`，WS 使用该账号的 VLESS UUID，HY2 使用该账号的认证值。三个绑定使用同一个人的 `subId`。核对三个入站都已启用、都参与订阅，按 REALITY → HY2 → CF 排序。
2. 在 Xray 建立共用的 `direct`（freedom）、`blocked`（blackhole），以及该账号专用的住宅 `socks` 出口。住宅提供商的来源白名单填写 VPS 的实际出口 IP。在三种 `inboundTag` 上按 Xray 收到的账号标识匹配。为新账号补齐 YouTube 登录例外、专用媒体直出名单和住宅兜底。有独立 UDP 出口的账号还要分别匹配 TCP 和 UDP。三种入站的最后一条规则拒绝未匹配身份。[路由顺序和示例](xray-routing-tail.json)
3. 面板 3.11.0 的 Clash 完整订阅用到 `subEnable`、`subClashEnable`、`subClashEnableRouting` 和共享 YAML `subClashRules`。在面板开启订阅、Clash 格式和 **Clash 专用**的内嵌路由。名称相近的 `subEnableRouting` 是独立设置，参考部署将它关闭。Xray 的持久化配置来自 `xrayTemplateConfig`，面板据此生成运行配置。
4. 用该账号的私人订阅地址重新下载完整 YAML，确认 HTTP 响应内容、三个节点名称与顺序、28 个策略组、`AI 服务 → 入站协议` 以及该账号的节点凭据。分别固定三个入口，新建请求，核对入口与账号出口。撤销账号时同时停用入站凭据和旧订阅，再用旧节点验证新连接被拒绝。只更换订阅地址时，客户端可能仍缓存有效凭据。

新设备首次拉取也要验证。独立模板通过 `vps-sub` 的 `DIRECT` 下载节点。如果家庭防火墙限制该下载路径，可从已有的授权路径取得本账号的完整配置，私下导入后再测试刷新。

三个入口共用该账号在 VPS 上的出口策略。若三条入口都能认证，但住宅路径均失败，从 VPS 单独测试该账号的 SOCKS5 认证和 TCP 出口。需要 UDP 时，用支持 UDP 的客户端单独测试映射。这样可以定位入站切换能处理的故障范围。

## 4. 家庭防火墙与 OpenClash

OpenClash 决定已接管的请求从家里直连、经哪个协议进 VPS，或拒绝连接。家庭防火墙检查实际要出 WAN 的报文。若 OpenClash 选择家庭直连，而防火墙拒绝该连接，请求就会失败。抵达 VPS 后，住宅出口和 VPS 出口仍由 Xray 按账号和目标站选择。

### WAN 上的判定顺序

下面两条链只处理匹配实际 WAN 出接口的流量。`forward` 看到从 LAN 设备转发的报文，`output` 看到路由器本机程序发出的报文。

| 顺序 | 命中条件 | 路由器动作 |
| --- | --- | --- |
| 1 | 两条链先检查外层 STUN。匹配现代 STUN 报头特征的 UDP，或发往 UDP 3478、3479、5349 及 TCP 3478、5349 的连接 | 拒绝。端口规则按目的端口生效，优先于国内地址和代理例外。日志限速，计数器记录命中。 |
| 2 | 仅 `output` 检查套接字 GID 和代理入口。参考规则要求 `meta skgid 65534`，且目的 IPv4 与目的端口同时命中对应的 TCP 或 UDP 白名单 | 允许该外层连接。`forward` 直接进入第 3 步。迁移时核对建立外层连接的实际进程及套接字 GID，包括可能使用的本机 Xray 桥接。 |
| 3 | 其余 WAN IPv6 | 拒绝。参考部署采用 IPv4 出网。 |
| 4 | 目的 IPv4 命中本地地址集 `overseas_private4`，或国内 IPv4 集 `overseas_cn4` | 允许。国内归属取决于这台路由器加载的 IP 前缀表。 |
| 5 | DHCPv4 客户端广播，目标 `255.255.255.255`，UDP 源端口 68、目的端口 67 | 允许。 |
| 6 | 其余直接出 WAN 的 IPv4 | 拒绝并计数。 |

参考 UDP 报头匹配检查长度至少 28 字节、STUN 类型的高两位为 0、magic cookie 为 `0x2112a442`。订阅和 VPS 里的 UDP 19302–19309 属于代理路由规则。

参考本地地址集包括 `10.0.0.0/8`、`100.64.0.0/10`、`127.0.0.0/8`、`169.254.0.0/16`、`172.16.0.0/12` 和 `192.168.0.0/16`。STUN 经已建立的加密代理隧道时，家庭 WAN 看到的是隧道外层。VPS 和住宅代理是否支持这段 UDP，要另行实测。

### 三个入口怎样放行

| 入站协议 | 路由器代理实际要连接的目标 |
| --- | --- |
| 01 REALITY 直连 | VPS 源站 IPv4 的 TCP 443，列入 `overseas_proxy_tcp4`。 |
| 02 Hysteria2 备用 | VPS 源站 IPv4 的 UDP 8443–8447，五个公网端口均列入 `overseas_proxy_udp4`。VPS 将 8444–8447 转到监听端口 8443。 |
| ☁️ Cloudflare Tunnel | 节点实际拨号的 Cloudflare 边缘 IPv4 与 TCP 端口，参考节点使用 443。按节点的真实解析或连接记录核对候选边缘，地址变化后复核。 |

仅含「VPS 源站 + 端口」的集合覆盖前两种入口。CF 备用入口需要有一条获准到达边缘的路径。若采用进程白名单，例外应限定到该进程使用的边缘地址和端口。Claude 域名和 204 探测地址是隧道内目的地，保留在订阅规则里。桌面 Clash Verge 自行拨号时，隧道外层对家庭路由器属于 `forward`，路由器本机的 GID 例外不适用于该桌面。

### 持久化、启动与验收

在这套 OpenWrt 参考部署中，fw4 从 `/etc/nftables.d/90-overseas-firewall.nft` 和 `/etc/overseas-firewall/cn4.nft` 加载策略。独立服务 `/etc/init.d/overseas-guard` 在启动早期加载 `inet overseas_guard` 表。两张表的 `forward` 和 `output` 钩子优先级分别为 10（fw4）和 20（独立表）。较早的表放行后，后面的表仍可以拒绝。

独立服务校验规则、国内地址表的 SHA-256 和候选规则语法。状态 `current` 表示当前配置已装入。更新失败时，已有表保留为 `retained`。冷启动时先建立 `minimal`，当前配置装入失败后尝试有效的 `last-good` 快照。参考最小策略的国内地址集为空，代理例外只含源站 TCP 443 与 UDP 8443。变更 WAN 或节点前保留本地管理和回滚路径。核对持久化规则、校验清单和内置最小策略，并在成功加载后核对新快照。

测试 OpenClash 时，先确认设备网关、活动配置及覆写后的策略组。测试 Clash Verge 时，确认应用流量已进入本机代理或 TUN。两者同时运行时，用客户端的连接记录确认接管位置。

在路由器上可用以下只读命令核对实际设备、钩子和拒绝计数。

```sh
ubus call network.interface.wan status | jsonfilter -e '@.l3_device'
/etc/init.d/overseas-guard status
for table in fw4 overseas_guard
do
    nft -n list chain inet "$table" overseas_forward
    nft -n list chain inet "$table" overseas_output
    nft -n list counter inet "$table" overseas_direct4_blocked
done
```

1. 用 `ip -4 route get <自有测试服务器的IP>` 查该目的地实际经过的设备，与两张表的 `oifname` 对照。PPPoE 的三层设备可能是 `pppoe-wan`。若规则仅匹配下层以太网口，走 PPPoE 设备的报文就不会命中这些限制。启用 IPv6 时也要核对实际 IPv6 出接口。防火墙重载、WAN 重拨或切换上联网口后重新核对。
2. 从路由器本机和一台受管 LAN 设备分别连接自有境外服务器上已确认可用、且不在直连许可内的端口。核对由哪个客户端接管、哪张表的拒绝计数增加，以及 WAN 上是否出现到目标端口的直接握手。拒绝测试以命中相应规则为准。
3. 再测一个应当直连的国内地址。通过 OpenClash 手动固定三个入口各发一个新请求，逐端口验证 HY2，单独验证 CF 边缘可达。用连接记录和 WAN 外层目的地址核对路径。这样能分别验收「AI 组只能选 VPS」和「普通设备的境外直连受到防火墙限制」。

## 5. 发布、备份与回滚

- **发布前**，在私有位置保存 3x-ui 的一致性数据库备份。运行中的 SQLite 可用在线备份接口或 `.backup`，以包含已提交的 WAL 数据。保存面板中的 Xray 模板与 Clash 订阅模板、生成的 Xray 配置，以及这次涉及的 Nginx、Tunnel 路由、证书续期钩子、持久化防火墙和 OpenClash 覆写。记录当时的活动配置。备份含账号和密钥，应限制访问并保留一份加密的离机副本。
- **发布时**，用对应版本的 `xray run -test -c <候选完整配置>`、`mihomo -t -d <测试数据目录> -f <该账号完整订阅>` 及 `nginx -t` 分别验证。Mihomo 测试目录要有模板所用的 GeoIP、GeoSite 等数据。将变动写回对应的持久化来源，在维护窗口加载改动，再比较实际运行的路由与用户下载到的订阅。
- **恢复时**，优先恢复本次改动的模板、账号绑定或防火墙配置，保留其他账号随后发生的变更。整机恢复要使用与备份兼容的服务版本及一致的面板数据库。恢复 Tunnel 连接凭据、证书和私钥并检查文件权限。重新连接面板、下载一份完整订阅，复测三个入口、账号出口、YouTube 登录和播放、未匹配身份、家庭直连拦截。变更 VPS 公网 IP 时，同步更新 DNS、住宅 SOCKS5 的来源白名单、家庭防火墙的入口地址和订阅中的源站直连规则。
