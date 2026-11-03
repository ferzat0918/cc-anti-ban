# cc-anti-ban

给 Agent 用的 **Claude 家宽出口全闭环** Skill。

教编码 Agent（Claude Code、Codex、Cursor 等）如何**辅导或搭建**双层分流：让 Claude 看到**美国住宅 ISP IP**；日常境外可继续走 VPS 机房；国内与微信保持 DIRECT。软路由是可选项，开户后再搞。

兼容开放 [Agent Skills](https://github.com/anthropics/skills) 格式与 [`skills`](https://www.npmjs.com/package/skills) CLI。

> 本仓库只提供**流程知识**，不含真实 IP、密码、面板 token、订阅 URL。密钥请放在你自己的密码库 / 面板里。

---

## 为什么要家宽出口

普通机场 / 机房节点很容易被 Claude 拦或标风险。稳妥做法是：

1. 电脑先连**中转 VPS**（VLESS Reality + Hysteria2）。
2. 在 VPS 上，**Xray** 把 Claude / Anthropic / 常见 AI / Cloudflare（以及按需的 STUN）转到 **ISP / 静态住宅 SOCKS5**。
3. 这两个入站上的其余流量仍从 **VPS 机房直出**。
4. 三个 Claude IP 检测站都确认是住宅后，再在**同一出口**注册、绑卡、付钱。

两层，不要合成一层：

| 层 | 位置 | 职责 |
| --- | --- | --- |
| Clash（客户端） | 电脑 | 进哪扇门：AI → VLESS，UDP/STUN → hy2，国内/微信 → DIRECT |
| Xray（服务端） | 中转 VPS | 进门后：AI/CF/STUN → **socks**；其余 → **VPS direct** |

VLESS 与 hy2 通常**共用**同一套 Xray 规则。除非现网 `config.json` 明确如此，否则不要默认「hy2 整路过家宽」。

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
        └── SKILL.md        # frontmatter（name/description）+ 完整英文配方（给 Agent 执行）
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
密钥登录、防火墙（常见放行 SSH、TCP 443、UDP 8443），**面板不要对公网裸奔**。推荐 Cloudflare Tunnel：`panel.<域名>`、`sub.<域名>` → 本机端口。国内更新订阅必须用**域名**，不要直拉 `IP:订阅端口`。

### 2. 3x-ui 双入站 + Clash 订阅
同一客户端 **subId** 挂两个协议：
- TCP：VLESS + Reality（常挂 `:443`）
- UDP：Hysteria2 **做在 3x-ui 里**（常挂 `:8443`）——不要另起一套 systemd hy2

打开面板 **Clash 订阅**，并开启**内嵌路由/规则**，一条 URL 带上 DNS、策略组与规则。

### 3. 买家宽 SOCKS5
私下保存 host/port/账号密码；白名单 = 仅 VPS IP。

### 4. 接到 Xray
新建 socks 出站；两个入站挂**同一套**规则：AI/Claude/Anthropic/Cloudflare（以及按现网设计的 STUN 端口）→ socks；其余 → VPS direct。在 VPS 上用 socks `curl` 验 IP：必须是家宽，不能等于 VPS 公网 IP。

### 5. 电脑 Clash
优先导入**面板 Clash 订阅**，别从零手写 YAML。
- **保留 dns 段**（fake-ip、respect-rules、国内 DoH、panel/sub 走国内解析）。
- 策略组简单、规则写细：节点选择 / 人工智能（选 **VLESS**）/ UDP 加速（选 **hy2**，STUN **不要 REJECT**）/ 漏网之鱼 / DIRECT。
- 微信：进程匹配 → DIRECT。
- `panel.<域名>` / `sub.<域名>` → **DIRECT**（防订阅死锁）。

### 6. 验证（注册前闸门）
顺序固定；任一失败先停。

1. **VPS socks** — 家宽 IP ≠ VPS IP  
2. **Clash** — AI→VLESS，UDP→hy2  
3. **三站**（同一浏览器、AI 链路开着）：  
   - https://ip.net.coffee/claude/  
   - https://ipkk.com/claude-ip-check  
   - https://ipinfo.cv/claude-ai-check  
   都要住宅 / 可用、国家一致、不是机房或 VPN。  
   **Cloudflare 注意：** `cloudflare.com`（含 www、challenge）也必须走 AI→家宽，否则 IPKK 上 CF 卡片会显示错出口。  
4. **对照** — 百度等国内仍是国内网；微信发图正常。  
5. 可选 — 经「节点选择」访问 Google 等，出口可以是 VPS 机房 IP（说明没有整网绑家宽）。

**通过标准：** socks 已是家宽 + 三站一致 + 国内/微信直连 → 再进第 7 步。

### 7. 同一出口注册与支付
干净浏览器；只用 AI 组；中途不换节点；账单国家与出口国家一致；尽量在该家宽环境填朋友的卡；不要商业代付；3DS 期间也不要切节点。

### 软路由（以后再说）
例如磊科 N60 Pro + 2G + opw OpenWrt + OpenClash/mihomo。同样：策略组简单、规则写细。**不要**为了等软路由耽误开户。

---

## 常见翻车

| 现象 | 多半原因 |
| --- | --- |
| 检测站仍是机房 | 人工智能没选 VLESS，或 VPS 没把 AI/CF 指到 socks |
| IPKK 的 CF 卡片不对 | `cloudflare.com` 没走 AI 家宽 |
| SOCKS 连不上 | 白名单 / 密码空格 |
| 订阅更新失败 | panel/sub 没 DIRECT，或 DNS 段被删 |
| 能上网但 Claude 不行 | 流量走了节点选择，AI 规则没命中 |
| WebRTC 异常 | STUN 被 REJECT |
| 微信发图失败 | 需要进程匹配，不只改域名 |
| 支付风控 | 中途换节点、新邮箱、代付、账单国家 ≠ IP 国家 |

---

## 给 Agent 执行时的约定

加载本 skill 时，Agent 应：

1. 以**现网**面板 / `config.json` / Clash 订阅为准，纠正过时的「hy2 整路过 socks」说法。  
2. 绝不把真实 IP / 密码 / 订阅 token 写进公开页面；只用占位符。  
3. 用用户的语言辅导；记录用户配置时保留其真实策略组名。  
4. 软路由 / OpenClash 旁支不要塞进主七步，除非用户点名要。

完整可执行清单见 [`skills/cc-anti-ban/SKILL.md`](./skills/cc-anti-ban/SKILL.md)（英文）。

---

## 可选：下一步可加的脚本（尚未入库）

Skill 目录里还可以放脚本，让 Agent 少手敲，例如：

- `scripts/check-socks-exit.sh` — VPS 上经 socks 查出口 IP  
- `scripts/verify-checklist.md` — 人机共用的验收勾选  
- `templates/clash-groups.snippet.yaml` — 策略组命名对照  

需要的话再说一声，我可以加进仓库并在 `SKILL.md` 里引用。

---

## License

[MIT](./LICENSE)
