# cc-anti-ban

给 Agent 用的 **Claude 家宽出口全闭环** Skill。

教编码 Agent（Claude Code、Codex、Cursor 等）配置两层分流。家里由 Clash / Mihomo 决定请求直连、经哪条协议进 VPS，或拒绝连接。到了 VPS，由 Xray 决定使用该订阅账号的住宅 SOCKS5，还是从 VPS 公网出口访问目标。

兼容开放 [Agent Skills](https://github.com/anthropics/skills) 格式与 [`skills`](https://www.npmjs.com/package/skills) CLI。

> 本仓库只提供**流程知识**，不含真实 IP、密码、面板 token、订阅 URL。密钥请放在你自己的密码库 / 面板里。

---

## 从家里到 VPS，再从 VPS 到网站

| 层 | 在哪里选 | 参考配置的行为 |
| --- | --- | --- |
| 家里 → VPS | OpenClash / Clash Verge 的规则模式 | Claude、ChatGPT、Persona 和已收录的 Cloudflare 域名命中 `AI 服务 → 入站协议`。可以自动或手选三条入口，整个选择链里都没有家庭 `DIRECT`。国内和微信按各自规则走家庭直连。 |
| VPS → 网站 | VPS 上的 Xray | 三种入口都按认证账号分流。通过拦截与专项规则后，默认走该账号的住宅出口。YouTube 播放、媒体与专用 API 走 VPS 出口，精确匹配的 `accounts.youtube.com` 仍走该账号家宽。 |

节点名里的 **REALITY 直连** 指家里直接连接 VPS。即使手选这条入口，Claude 也要经过 VPS，再按账号走住宅出口。换到 HY2 或 Cloudflare Tunnel 也沿用同一套账号规则。

---

## 务必结合视频使用 Skill

**只看文字不够。** 一边看油管教程一边让 Agent 按本 Skill 执行：

- 教程视频：https://www.youtube.com/watch?v=0zcKwIWatCE&t=704s
- 频道：https://www.youtube.com/@parhatFreddy
- 图文零基础：https://claude-d5bd.pages.grok.bot/

---

## 安装

需要 [Node.js](https://nodejs.org/) **22.20+**。

```bash
# 当前项目：Claude Code + Codex
npx skills add ferzat0918/cc-anti-ban -a claude-code -a codex

# 全局（所有项目）
npx skills add ferzat0918/cc-anti-ban -g -a claude-code -a codex

# 安装到 CLI 支持的全部 Agent
npx skills add ferzat0918/cc-anti-ban --all
```

装完后开**新的** Agent 会话。用 `/cc-anti-ban` 调用，或直接说「跑家宽出口闭环 / cc anti-ban」。

只查看不安装：

```bash
npx skills add ferzat0918/cc-anti-ban --list
```

### 文件落在哪

| Agent | 参数 | 项目路径 | 全局路径 |
| --- | --- | --- | --- |
| Claude Code | `-a claude-code` | `.claude/skills/` | `~/.claude/skills/` |
| Codex | `-a codex` | `.agents/skills/` | `~/.codex/skills/` |

---

## 仓库结构

```text
cc-anti-ban/
├── README.md
├── README.zh-CN.md
├── LICENSE
└── skills/cc-anti-ban/
    ├── SKILL.md                           # 给 Agent 的配置与验收步骤
    ├── clash-residential-template.yaml    # 从节点订阅读取三条入口的客户端配置
    └── references/
        ├── infrastructure.md             # 从新 VPS 部署、开账号、验证防火墙和恢复
        └── xray-routing-tail.json        # VPS 端路由末段示例
```

[Skill 正文（英文）](./skills/cc-anti-ban/SKILL.md) · [Mihomo 模板](./skills/cc-anti-ban/clash-residential-template.yaml) · [Xray 路由示例](./skills/cc-anti-ban/references/xray-routing-tail.json)

---

## Skill 会带你走完什么

从新 VPS 搭建或迁移时，可依次核对域名与证书、三个入口、面板订阅开关、家庭和 VPS 两侧的防火墙，以及备份恢复。[基建部署与运维手册](./skills/cc-anti-ban/references/infrastructure.md)

### 0. 购物清单
- 靠近美国、线路优先 CN2 GIA（或同类）的**中转 VPS**；城市靠近家宽 ISP；AlmaLinux 9 / Debian；内存 ≥1G；只用 SSH 密钥。
- **ISP / 静态住宅 SOCKS5**（如 IPRoyal 选 **ISP**，不要选轮换 Residential；美国；Zero Fraud Score；白名单**只**放 VPS 公网 IP）。
- 电脑先装 **Clash Verge / Clash Meta**；软路由以后再说。
- 养过的谷歌邮箱、能收验证码的手机、信得过的海外朋友协助绑卡（**不要**商业代付）。

### 1. VPS 加固 + Tunnel

先确认 SSH 密钥可用。参考部署的 REALITY 和 Hysteria2 对应 TCP 443、UDP 8443。启用 HY2 跳端口时，将 UDP 8443–8447 转发到监听端口 8443。Cloudflare Tunnel 经 Nginx 接到本机的 VLESS WebSocket，3x-ui 面板通过私有管理入口访问。

### 2. 3x-ui 入站与共享订阅

给每个订阅账号都绑定 REALITY、Hysteria2 和 Cloudflare Tunnel，后续新增账号也按这三条建立。一个人的三条记录使用自己的同一个 `subId`，并用该账号的凭据认证。按 REALITY、HY2、Cloudflare Tunnel 的顺序发布，核对节点名称、空格和图标。自动组按节点源中的实际顺序选线。

在 3x-ui 开启订阅、Clash 订阅和 Clash 专用的内嵌规则。参考版本 3.11.0 对应 `subEnable`、`subClashEnable` 和 `subClashEnableRouting`。将 DNS、策略组和规则写入共享模板，由面板为每份订阅注入该用户的三个节点。用户导入自己的**完整配置订阅**，以后刷新就能收到共享策略更新。

### 3. 买家宽 SOCKS5

私下保存 host、port 和账号密码。白名单使用 VPS 公网 IP。

### 4. Xray 按账号选出口

Xray 使用第一条命中的规则。先保留既有专项规则与拦截，再按以下顺序布置三种业务入站的路由末段。

1. `full:accounts.youtube.com` 按账号走住宅出口。对分开配置 TCP 和 UDP 的账号，两种流量各走对应出口。
2. 授权账号访问 YouTube 专用域名、播放和 API 时从 VPS 直出，例如 `googlevideo.com`、`ytimg.com`、`yt3.ggpht.com`、`youtubei.googleapis.com`。使用示例中的专用名单，Google 通用 API、共享头像域名继续按账号规则处理。
3. 各已绑定账号的其余流量走自己的住宅出口。最后拒绝从这三种入站进入、却未匹配到账号的流量。

参考部署另有一条账号专属例外，把 TCP/UDP 443 上的 `www.gstatic.com`、`ssl.gstatic.com` 精确送到该账号的住宅出口，排在它的 `geosite:cn` 拦截之前。合并路由末段时要保留前面的规则，同时核对面板保存的模板和 Xray 运行配置。[Xray 路由末段与占位账号](./skills/cc-anti-ban/references/xray-routing-tail.json)

### 5. 客户端策略组与规则

单 VPS 版保留 Aethersailor 标准版的应用分类，共 28 个策略组，取消国家和地区分组。

| 策略组 | 可选项（首项为新配置默认值） |
| --- | --- |
| 入站协议 | 自动切换、01 REALITY 直连、02 Hysteria2 备用、☁️ Cloudflare Tunnel |
| 自动切换 | REALITY → Hysteria2 → Cloudflare Tunnel，使用最高优先级的健康入口 |
| AI 服务、TikTok、YouTube、Netflix、DisneyPlus、HBO、PrimeVideo、国外媒体、谷歌服务 | 入站协议 |
| 即时通讯、社交媒体、GitHub、AppleTV+、Emby、Spotify、Bahamut、国外电商、谷歌FCM | 入站协议、全球直连 |
| 苹果服务、微软服务、游戏平台、Steam、测速工具 | 全球直连、入站协议 |
| 漏网之鱼 | 入站协议、全球直连、REJECT |
| 非标端口 | 漏网之鱼、全球直连 |
| 全球直连 | DIRECT |

**自动和手动可以共用。** 选「自动切换」时，每 15 秒通过三条入口分别用 `HEAD` 请求探测地址，超时 5 秒，以 HTTP 204 判定健康。首选失效后，新连接使用下一条健康入口。首选恢复后，新连接回到首选。手动点某个协议会一直固定在那里，直到用户重新选「自动切换」。已有连接需要应用重连，订阅刷新后客户端可能记住上次选择。[健康检查地址](https://cp.cloudflare.com/generate_204)

两种配置的组和规则可以共用。面板的两个入口组用 `include-all-proxies: true` 从当前账号的配置取节点。仓库的独立模板用 `use: [vps-sub]` 从节点源取节点，下载节点源走 `DIRECT`，节点源也启用同样的 204 检查。独立使用时需替换示例域名、源站 IP 和节点订阅地址。[独立 Mihomo 模板](./skills/cc-anti-ban/clash-residential-template.yaml)

公开模板含 1,571 条可移植规则，其中 128 条「AI 服务」规则与本次参考部署一致。它们合并了 Aethersailor 标准版、Claude 官方文档、CleanIP、Net.Coffee，并保留既有 Cloudflare 规则，覆盖 Persona 的 `withpersona.com` 和 `frompersona.com`。参考部署的完整订阅另外保留现场覆写，共 8,154 条。应用规则优先于通用 UDP 和兜底规则，DNS 保留 Fake-IP、`respect-rules` 和节点域名的独立解析。[规则来源与维护说明](./skills/cc-anti-ban/SKILL.md#5-configure-the-client-selectors-and-rule-order)

### 6. 验证两段路由

1. 用目标客户端的 Mihomo 内核检查每个账号的完整订阅，确认四个入口选项顺序正确，「AI 服务」只有「入站协议」一个选项。
2. 在隔离测试实例中先让 REALITY 不可用，再让 HY2 不可用。每次等待定时探测并新建请求，核对实际入口依次切到 HY2、CF。三条都断时，AI 新请求应失败。恢复首选后验证自动回切，再手选 CF，确认手动固定生效。
3. 在 Xray 核对同一账号分别经三个入口的匹配结果。单独验证 Claude / Cloudflare、普通境外请求、`accounts.youtube.com` 和 YouTube 播放。需要 UDP 的账号再测 UDP。Cloudflare 的 IP 回显只能证明对该回显站的出口。
4. 验证国内站点、微信和所需的 Google 登录步骤。

### 7. 同一出口注册与支付
干净浏览器；只用 AI 组；中途不换节点；账单国家与出口国家一致；尽量在该家宽环境填朋友的卡；不要商业代付；3DS 期间也不要切节点。

### 软路由（以后再说）
例如磊科 N60 Pro + 2G + opw OpenWrt + OpenClash/mihomo。同样：策略组简单、规则写细。**不要**为了等软路由耽误开户。

已使用 OpenWrt 时，可按两条 WAN 链分别验收路由器本机和 LAN 设备的流量。手册列出了 STUN、代理进程例外、国内直连与其余流量的判定顺序，以及三个入口对应的地址和端口。[路由器防火墙策略](./skills/cc-anti-ban/references/infrastructure.md#4-家庭防火墙与-openclash)

---

## 常见翻车

| 现象 | 多半原因 |
| --- | --- |
| 检测站仍是机房 | 核对命中的 Clash 规则、所选入口和该账号的 Xray 出站 |
| IPKK 的 CF 卡片不对 | `cloudflare.com` 没走 AI 家宽 |
| SOCKS 连不上 | 白名单 / 密码空格 |
| 订阅更新失败 | 核对子域名、DNS、provider 拉取路径和源站可达性 |
| 能上网但 Claude 不行 | 检查规则模式下实际命中的「AI 服务」、选中的入站和该账号的 Xray 出口 |
| 自动切换一直显示第一条 | 检查父组选中了「自动切换」、三个节点名称和各节点的 204 探测结果。旧 trace 地址对 HEAD 可返回 404。所有探测失败时内核也可能仍显示第一条。 |
| 只有 REALITY 连不上 | 手动固定 REALITY，分别核对 TCP 443、该账号的入站凭据、Vision flow 和客户端版本。OpenClash 若使用本机 Xray 桥接，再查回环监听与握手日志。 |
| 新用户只拿到 CF | 在面板补齐该用户在 REALITY、HY2 的绑定，并核对三条记录的 `subId` 和实际订阅内容 |
| WebRTC 异常 | 核对 STUN 规则、节点的 UDP 能力和账号 UDP 出站 |
| 微信发图失败 | 需要进程匹配，不只改域名 |
| 支付风控 | 中途换节点、新邮箱、代付、账单国家 ≠ IP 国家 |

---

## 给 Agent 执行时的约定

加载本 skill 时，Agent 应：

1. 对照**现网**面板、生成的 `config.json` 和 Clash 订阅，核对账号、入站、策略组与实际出口。
2. 绝不把真实 IP / 密码 / 订阅 token 写进公开页面；只用占位符。  
3. 用用户的语言辅导；记录用户配置时保留其真实策略组名。  
4. 软路由 / OpenClash 旁支不要塞进主七步，除非用户点名要。

完整可执行清单见 [`skills/cc-anti-ban/SKILL.md`](./skills/cc-anti-ban/SKILL.md)（英文）。

---

## 可选：下一步可加的脚本（尚未入库）

Skill 目录里还可以放脚本，让 Agent 少手敲，例如：

- `scripts/check-socks-exit.sh` — VPS 上经 socks 查出口 IP  
- `scripts/verify-checklist.md` — 人机共用的验收勾选  

需要的话再说一声，我可以加进仓库并在 `SKILL.md` 里引用。

---

## License

仓库原创内容采用 [MIT](./LICENSE)。Mihomo 模板改编自 Aethersailor 的标准版，调整了单 VPS 分组、协议选择和 AI 规则，改编模板沿用 CC BY-SA 4.0。[上游版本](https://github.com/Aethersailor/Custom_OpenClash_Rules/tree/8335eb2b8205ca2e3573cd7267d5b1e423637084) · [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)
