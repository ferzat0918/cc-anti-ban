# cc-anti-ban

给 Agent 用的 **Claude 家宽出口全闭环** Skill。

教编码 Agent（Claude Code、Codex、Cursor 等）配置双层分流。Clash 按规则选择 VPS 入口，Xray 按账号、入站、目的地和协议选择住宅 SOCKS5 或 VPS 出口。国内与微信由客户端规则处理，软路由可选。

兼容开放 [Agent Skills](https://github.com/anthropics/skills) 格式与 [`skills`](https://www.npmjs.com/package/skills) CLI。

> 本仓库只提供**流程知识**，不含真实 IP、密码、面板 token、订阅 URL。密钥请放在你自己的密码库 / 面板里。

---

## 当前 VPS 与分流

| 层 | 位置 | 职责 |
| --- | --- | --- |
| Clash / Mihomo | 客户端 | 按目的地和协议选择 `🌐 默认代理`、`🤖 人工智能`、`📡 UDP 流量` 或 `DIRECT` |
| 3x-ui / Xray | 中转 VPS | 根据账号和入站匹配规则。选定的 AI、Cloudflare、Telegram、TikTok 和 UDP 流量走该账号的 SOCKS5，其余境外流量可以由 VPS 直出 |

参考部署有三个入口，分别是 `01 REALITY 直连`、`02 Hysteria2 备用`、`☁️ Cloudflare Tunnel`。具体订阅可根据账号发布其中的节点。客户端选择入口后，再核对该账号在对应入站上的出站规则。

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

```
cc-anti-ban/
├── README.md
├── LICENSE                 # MIT
└── skills/
    └── cc-anti-ban/
        ├── SKILL.md                            # 现网拓扑、路由与验收
        └── clash-residential-template.yaml     # 可替换域名和订阅地址的 Clash 模板
```

说明：`SKILL.md` 正文保持英文，方便跨 Agent 稳定执行；本 README 给人读，用中文。

---

## Skill 会带你走完什么

### 0. 购物清单
- 靠近美国、线路优先 CN2 GIA（或同类）的**中转 VPS**；城市靠近家宽 ISP；AlmaLinux 9 / Debian；内存 ≥1G；只用 SSH 密钥。
- **ISP / 静态住宅 SOCKS5**（如 IPRoyal 选 **ISP**，不要选轮换 Residential；美国；Zero Fraud Score；白名单**只**放 VPS 公网 IP）。
- 电脑先装 **Clash Verge / Clash Meta**；软路由以后再说。
- 养过的谷歌邮箱、能收验证码的手机、信得过的海外朋友协助绑卡（**不要**商业代付）。

### 1. VPS 加固 + Tunnel

先确认 SSH 密钥可用。参考部署的 REALITY 和 Hysteria2 对应 TCP 443、UDP 8443。Cloudflare Tunnel 经 Nginx 接到本机的 VLESS WebSocket，3x-ui 面板通过私有管理入口访问。

### 2. 3x-ui 入站与 Clash 订阅

按授权账号挂载 REALITY、Hysteria2 和 Cloudflare Tunnel 中的对应入站。同一份订阅的客户端条目可共用 `subId`。开启 Clash 订阅的内嵌规则，同时发布 DNS、策略组与分流规则。

### 3. 买家宽 SOCKS5

私下保存 host、port 和账号密码。白名单使用 VPS 公网 IP。

### 4. Xray 按账号选出口

核对 `user` 和 `inboundTag`，再检查域名、IP、UDP 端口和规则顺序。命中账号规则的流量走相应 SOCKS5。剩余流量按服务端拦截规则及 VPS 直出兜底处理。需要住宅 UDP 的账号应实测 STUN/NTP 和 UDP 出站。

当前 VPS 对受影响账号的 `www.gstatic.com` 与 `ssl.gstatic.com` 443 流量有精确例外，排在 `geosite:cn` 拦截前面，走 VPS 直出。排查 Google 登录时分别验证登录入口、静态资源和密码页「下一步」。

### 5. Clash 策略组与规则

| 策略组 | 选择 |
| --- | --- |
| `☁️ CDN 线路` | 隐藏组，按名称筛选 Cloudflare 节点 |
| `🔁 直连优先` | 按订阅顺序检查 REALITY、Hysteria2、Cloudflare 的可用性 |
| `🌐 默认代理` | CDN、直连优先、DIRECT |
| `🤖 人工智能`、`📡 UDP 流量` | CDN、直连优先、默认代理、DIRECT |
| `🐟 漏网之鱼` | 默认代理、CDN、直连优先、DIRECT、REJECT |

- Clash 使用规则模式。全局模式下由 `GLOBAL` 的选择决定入口。
- 保留 Fake-IP、`respect-rules` 和分流 DNS。当前订阅的面板和订阅域名命中 `🌐 默认代理`，VPS 源站 IP 直连规则位于前面。
- 微信按进程和域名走 DIRECT。国内 UDP 判断排在 STUN 前，剩余 UDP 由 `📡 UDP 流量` 接管。普通 Google 和 YouTube 规则走 `🌐 默认代理`，匹配到 AI 专项规则的域名走 `🤖 人工智能`。末尾保留国内 GEOIP 和 `🐟 漏网之鱼`。
- 优先导入 3x-ui 完整订阅。独立模板的 provider 通过 DIRECT 拉取，使用前替换示例域名、源站 IP 和订阅地址。[Clash 模板](./skills/cc-anti-ban/clash-residential-template.yaml)

### 6. 验证

1. 在 VPS 核对被选中的 SOCKS5 出口。选定的住宅路径应显示预期住宅 IP。
2. 在 Clash 核对实际命中的规则和节点，在 Xray 核对账号和入站。需要 UDP 的路径单独验证 UDP。
3. 可按技能正文使用同一浏览器验证 Claude 和 Cloudflare 的选定出口。按 Google 登录流程发起的新请求需要单独验证。
4. 对照国内站点、微信及普通境外站点，检查各自的预期出口。

### 7. 同一出口注册与支付
干净浏览器；只用 AI 组；中途不换节点；账单国家与出口国家一致；尽量在该家宽环境填朋友的卡；不要商业代付；3DS 期间也不要切节点。

### 软路由（以后再说）
例如磊科 N60 Pro + 2G + opw OpenWrt + OpenClash/mihomo。同样：策略组简单、规则写细。**不要**为了等软路由耽误开户。

---

## 常见翻车

| 现象 | 多半原因 |
| --- | --- |
| 检测站仍是机房 | 核对命中的 Clash 规则、所选入口和该账号的 Xray 出站 |
| IPKK 的 CF 卡片不对 | `cloudflare.com` 没走 AI 家宽 |
| SOCKS 连不上 | 白名单 / 密码空格 |
| 订阅更新失败 | 核对子域名、DNS、provider 拉取路径和源站可达性 |
| 能上网但 Claude 不行 | 核对 Clash 模式、人工智能组及 VPS 的账号规则 |
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

[MIT](./LICENSE)
