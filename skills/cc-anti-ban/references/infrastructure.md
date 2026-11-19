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

OpenClash 在规则模式下为已接管的连接选择家庭直连、VPS 或拒绝。独立的家庭防火墙约束哪些流量能直接出 WAN。VPS 的防火墙负责入口放行和 HY2 端口转发。进入 VPS 后，账号和目标站的出口选择由 Xray 完成。

- **选择一个客户端做验收。** 使用软路由时，确认测试设备以它为网关，查看 OpenClash 的活动配置和覆写后的策略组。使用 Clash Verge 时，确认应用流量实际进入桌面代理或 TUN。两者同时运行时，通过相应客户端的连接记录确认由谁接管。
- **核对真实出口接口。** 在 OpenWrt 上运行 `ubus call network.interface.wan status`，查看 `l3_device`，再用 `ip -4 route get <自有测试服务器的IP>` 核对实际出接口。检查 `nft -t list ruleset` 中绑定在 WAN 上的规则。PPPoE 的三层设备可能是 `pppoe-wan`，和下层以太网口不同。如匹配有误，先确认规则来自 fw4 的持久化配置还是独立服务，保留本地管理和回滚路径。在对应来源更新后，分别验证防火墙重载与 WAN 重拨后的匹配。
- **分开看两条链。** 参考防火墙在 `forward` 检查 LAN 转发，在 `output` 检查路由器本机流量，允许私网和国内 IPv4，并限制其他境外直连。代理进程的例外按实际外层地址、协议和端口配置。REALITY 与 HY2 连接源站，CF Tunnel 连接客户端实际使用的边缘地址。桌面客户端建立的隧道可能命中路由器的 `forward`，路由器本机代理则走 `output`。IPv6 启用时也要验证对应路径。
- **验证兜底。** 用自有境外测试服务器上已确认可达、又不在直连许可内的端口做对照。分别从路由器本机与受管 LAN 设备发起新连接，记录防火墙拒绝计数器的变化及服务端是否收到连接，再确认允许的代理入口可用。测试时明确透明代理是否接管了请求。这样可以区分「AI 组没有 DIRECT」与「全屋绕过代理也会被拦截」。

## 5. 发布、备份与回滚

- **发布前**，在私有位置保存 3x-ui 的一致性数据库备份。运行中的 SQLite 可用在线备份接口或 `.backup`，以包含已提交的 WAL 数据。保存面板中的 Xray 模板与 Clash 订阅模板、生成的 Xray 配置，以及这次涉及的 Nginx、Tunnel 路由、证书续期钩子、持久化防火墙和 OpenClash 覆写。记录当时的活动配置。备份含账号和密钥，应限制访问并保留一份加密的离机副本。
- **发布时**，用对应版本的 `xray run -test -c <候选完整配置>`、`mihomo -t -d <测试数据目录> -f <该账号完整订阅>` 及 `nginx -t` 分别验证。Mihomo 测试目录要有模板所用的 GeoIP、GeoSite 等数据。将变动写回对应的持久化来源，在维护窗口加载改动，再比较实际运行的路由与用户下载到的订阅。
- **恢复时**，优先恢复本次改动的模板、账号绑定或防火墙配置，保留其他账号随后发生的变更。整机恢复要使用与备份兼容的服务版本及一致的面板数据库。恢复 Tunnel 连接凭据、证书和私钥并检查文件权限。重新连接面板、下载一份完整订阅，复测三个入口、账号出口、YouTube 登录和播放、未匹配身份、家庭直连拦截。变更 VPS 公网 IP 时，同步更新 DNS、住宅 SOCKS5 的来源白名单、家庭防火墙的入口地址和订阅中的源站直连规则。
