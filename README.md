# cc-anti-ban

Agent Skill for the full **Claude residential-exit closed loop**: relay VPS → 3x-ui (VLESS + Hysteria2) → ISP SOCKS5 → Clash → verify three check sites → register/pay.

Teaches coding agents (Claude Code, Codex, Cursor, and others) how to coach or build that stack. No real IPs, passwords, or subscription tokens.

## Install

Requires [Node.js](https://nodejs.org/) 22.20+.

```bash
# Claude Code + Codex (project-local)
npx skills add ferzat0918/cc-anti-ban -a claude-code -a codex

# Global (all projects)
npx skills add ferzat0918/cc-anti-ban -g -a claude-code -a codex
```

Or install for every supported agent:

```bash
npx skills add ferzat0918/cc-anti-ban --all
```

Then start a **new** agent session. Invoke with `/cc-anti-ban` or ask the agent to run the residential-exit closed loop.

## Layout

```
skills/cc-anti-ban/SKILL.md
```

Standard [Agent Skills](https://github.com/anthropics/skills) format (`name` + `description` frontmatter + markdown body).

## What it covers

1. Buy list (VPS, ISP SOCKS5, Clash client)
2. VPS harden + Cloudflare Tunnel
3. 3x-ui dual inbound + Clash subscription
4. Wire residential SOCKS into Xray (AI/CF/STUN → socks; else VPS direct)
5. Clash groups, DNS, WeChat DIRECT, STUN via UDP
6. Verify before signup (VPS socks → Clash → three Claude IP sites → CN/WeChat contrast)
7. Register and pay on the same exit

## License

MIT
