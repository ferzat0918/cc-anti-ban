# cc-anti-ban

Agent Skill for the full **Claude residential-exit closed loop**.

It teaches coding agents how to **coach or build** a dual-layer setup so Claude sees a **US residential ISP IP**, while everyday overseas traffic can stay on the VPS datacenter and domestic / WeChat stay DIRECT. Soft router is optional and comes later.

Compatible with the open [Agent Skills](https://github.com/anthropics/skills) format and the [`skills`](https://www.npmjs.com/package/skills) CLI (Claude Code, Codex, Cursor, OpenCode, and many others).

> This repo ships **procedural knowledge only**. It does **not** include real IPs, passwords, panel tokens, or subscription URLs. Pull live secrets from your own vault / panel.

---

## Why this exists

Ordinary “airport” / datacenter nodes are often blocked or flagged by Claude. The stable pattern is:

1. Your PC connects to a **relay VPS** (VLESS Reality + Hysteria2).
2. On the VPS, **Xray** sends Claude / Anthropic / common AI / Cloudflare (+ STUN as configured) out through an **ISP / static residential SOCKS5**.
3. Everything else on those inbounds exits the **VPS direct**.
4. Only after three Claude IP check sites agree that the exit is residential do you **register and pay** on that same path.

Two layers, never one:

| Layer | Where | Job |
| --- | --- | --- |
| Clash (client) | Your PC | Which **door**: AI → VLESS, UDP/STUN → hy2, CN/WeChat → DIRECT |
| Xray (VPS) | Relay | After ingress: AI/CF/STUN → **socks**; remaining → **VPS direct** |

VLESS and hy2 normally **share** the same Xray rules. Do **not** assume “all hy2 → socks” unless your live `config.json` says so.

---

## Install

Requires [Node.js](https://nodejs.org/) **22.20+**.

```bash
# Claude Code + Codex (this project)
npx skills add ferzat0918/cc-anti-ban -a claude-code -a codex

# Same, user-global
npx skills add ferzat0918/cc-anti-ban -g -a claude-code -a codex

# Every agent the skills CLI supports
npx skills add ferzat0918/cc-anti-ban --all
```

Then start a **new** agent session. Invoke with `/cc-anti-ban`, or ask the agent to run the residential-exit closed loop / “cc anti-ban”.

List without installing:

```bash
npx skills add ferzat0918/cc-anti-ban --list
```

### Where files land

| Agent | Flag | Project path | Global path |
| --- | --- | --- | --- |
| Claude Code | `-a claude-code` | `.claude/skills/` | `~/.claude/skills/` |
| Codex | `-a codex` | `.agents/skills/` | `~/.codex/skills/` |

---

## Repo layout

```
cc-anti-ban/
├── README.md
├── LICENSE                 # MIT
└── skills/
    └── cc-anti-ban/
        └── SKILL.md        # name + description frontmatter + full recipe
```

---

## What the skill walks through

### 0. Buy list
- US-near **relay VPS** (prefer CN2 GIA or peer; city near the ISP; AlmaLinux 9 / Debian; ≥1G RAM; SSH keys only).
- **ISP / static residential SOCKS5** (e.g. IPRoyal **ISP**, not rotating Residential; US; Zero Fraud Score; whitelist **only** the VPS public IP).
- Desktop **Clash Verge / Clash Meta** first; soft router later.
- Aged Google account, phone OTP, trusted overseas friend for card (**no** commercial card farms).

### 1. VPS harden + Tunnel
SSH keys, firewall (SSH + TCP 443 / UDP 8443 typical), **panel not public**. Prefer Cloudflare Tunnel: `panel.<domain>` and `sub.<domain>` → localhost ports. China subscription updates must use the **domain**, never raw `IP:subPort`.

### 2. 3x-ui dual inbound + Clash sub
One client **subId** on both:
- TCP: VLESS + Reality (often `:443`)
- UDP: Hysteria2 **inside** 3x-ui (often `:8443`) — not a separate systemd hy2

Enable panel **Clash subscription** with **embedded routing/rules** so one URL brings DNS + groups + rules.

### 3. Buy ISP SOCKS5
Save host/port/user/pass privately; whitelist = VPS IP only.

### 4. Wire socks into Xray
Socks outbound + **same** routing on both inbounds: AI/Claude/Anthropic/Cloudflare (+ STUN ports if that is your live design) → socks; else → VPS direct. Prove with `curl` via socks: exit IP = residential ≠ VPS IP.

### 5. Clash on the PC
Prefer the **panel Clash sub** over hand-written YAML.
- Keep the **dns** block (fake-ip, respect-rules, CN DoH, panel/sub via CN resolve).
- Simple groups, detailed rules: node-select / AI (**VLESS**) / UDP accel (**hy2**, STUN **not REJECT**) / catch-all / DIRECT.
- WeChat: process match → DIRECT.
- `panel.<domain>` / `sub.<domain>` → **DIRECT** (anti-deadlock).

### 6. Verify (gate before signup)
Order matters; any fail → stop.

1. **VPS socks** — residential ≠ VPS IP  
2. **Clash** — AI→VLESS, UDP→hy2  
3. **Three sites** (same browser, AI path on):  
   - https://ip.net.coffee/claude/  
   - https://ipkk.com/claude-ip-check  
   - https://ipinfo.cv/claude-ai-check  
   All residential / usable, same country, not DC/VPN.  
   **Cloudflare caveat:** `cloudflare.com` (www, challenge) must also AI→socks or IPKK’s CF card lies.  
4. **Contrast** — Baidu/CN domestic; WeChat image OK.  
5. Optional — Google via node-select may show VPS DC IP (expected).

**Pass bar:** socks home IP + three sites agree + CN/WeChat DIRECT → step 7.

### 7. Register + pay (same exit)
Clean browser; AI group only; no mid-flow node switch; billing country = exit country; friend card on this path if possible; no card farms; don’t switch during 3DS.

### Soft router (later only)
e.g. Netcore N60 Pro + 2G + opw OpenWrt + OpenClash/mihomo. Same simple groups. Do **not** block signup waiting for it.

---

## FAQ (short)

| Symptom | Likely cause |
| --- | --- |
| Checkers still DC | AI not on VLESS, or VPS not routing AI/CF to socks |
| IPKK CF card wrong | `cloudflare.com` not on AI path |
| SOCKS fail | Whitelist / spaces in password |
| Sub update fail | panel/sub not DIRECT, or DNS stripped |
| Net works, Claude doesn’t | Traffic on node-select; AI rules miss |
| WebRTC weird | STUN REJECT’d |
| WeChat images fail | Need process match, not domain-only |
| Pay risk | Switched node, new mail, card farm, billing ≠ IP country |

---

## Agent behavior (for implementers)

When an agent loads this skill it should:

1. Prefer **live** panel / `config.json` / Clash sub over memory; correct stale “hy2-all-socks” claims.
2. Never put real IPs / passwords / sub tokens in public pages; placeholders only.
3. Coach in the user’s language; keep their live Clash group names when documenting *their* setup.
4. Keep soft-router digressions out of the main seven steps unless asked.

Full checklist lives in [`skills/cc-anti-ban/SKILL.md`](./skills/cc-anti-ban/SKILL.md).

---

## License

[MIT](./LICENSE)
