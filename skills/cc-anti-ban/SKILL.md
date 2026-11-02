---
name: cc anti-ban
description: >-
  Use when guiding, building, or repairing the full Claude residential-exit path
  (cc anti-ban): buy VPS → 3x-ui dual inbound → ISP SOCKS5 → Xray
  AI/CF/STUN→socks → Clash Verge panel sub → verify three sites → register/pay.
  Also for the zero-base closed loop or redo of the whole stack—not verification
  alone.
---
# cc anti-ban (Claude residential exit closed loop)

End-to-end recipe: local Clash → VPS (VLESS Reality + Hysteria2) → Xray routes AI/Claude/Anthropic/Cloudflare (+ STUN) to residential SOCKS5; other inbound traffic exits the VPS direct. Soft router is optional later. Keep real IPs, passwords, sub tokens, and panel paths out of shared pages; pull live secrets from vault / private notes.

Canonical public walkthrough (scrubbed): the user’s beginner canvas if they have one. This skill is the agent checklist to execute or coach, not a dump of one person’s IPs.

## Goal in one line
Claude sees a US residential ISP IP. Everyday overseas can stay on the VPS datacenter. Domestic traffic and WeChat stay DIRECT.

## Two layers (never collapse into one)
1. **Clash (client):** which door — AI group → VLESS; UDP/STUN → hy2; CN/WeChat → DIRECT; rest → node-select / catch-all.
2. **Xray (VPS):** after ingress — AI/CF/STUN → residential socks; remaining on those inbounds → VPS **direct**. VLESS and hy2 normally **share** that rule set; do **not** assume “all hy2 → socks” unless live `config.json` says so and the user wants it.

---

## 0. Buy list
- US-near **relay VPS**: prefer CN2 GIA (or peer), location close to the ISP city; AlmaLinux 9 / Debian; ≥1G RAM; SSH key only.
- **ISP / static residential SOCKS5** (e.g. IPRoyal): entry **ISP** not rotating Residential; US; Zero Fraud Score; SOCKS5; whitelist **only** the VPS public IP.
- Desktop **Clash Verge / Clash Meta** first; soft router later.
- Aged Google account, phone for OTP, trusted overseas friend for card (no commercial card farms).

---

## 1. VPS harden + Tunnel
1. SSH key login; disable password login.
2. Firewall: SSH + business ports (common TCP 443, UDP 8443); **do not** expose the panel port publicly.
3. Cloudflare Tunnel (preferred): `panel.<domain>` → `127.0.0.1:<panelPort>`; `sub.<domain>` → `127.0.0.1:<subPort>`; origin localhost only; no public A record for the panel.
4. Until Tunnel is ready: SSH local port-forward to open the panel; subscription updates from China must use the **domain**, never raw VPS IP:subPort.

---

## 2. Install 3x-ui + dual inbound + Clash sub
1. Official 3x-ui one-click; strong password (never paste into public docs).
2. Two inbounds, **same client subId**:
   - TCP: VLESS + Reality (often :443)
   - UDP: Hysteria2 in 3x-ui (often :8443)—**not** a separate systemd hy2
3. Enable panel **Clash subscription** with **embedded routing/rules** so one URL brings DNS + groups + rules.
4. Sub URL shape: `https://sub.<domain>/…/<subId>` — keep path/token private.

---

## 3. Buy ISP SOCKS5
1. Provider ISP product (IPRoyal: ISP, not Residential).
2. Duration as needed; start with 1 IP; US city near the relay; Zero Fraud Score; SOCKS5.
3. Save host/port/user/pass privately; whitelist = VPS public IP only.

---

## 4. Wire socks into Xray
1. Add a socks outbound (custom tag ok) with ISP host/port/user/pass.
2. Attach the **same** routing to both VLESS and hy2 inbounds:
   - Claude / Anthropic / common AI + Cloudflare domains (and related IPs as needed) → socks
   - STUN UDP ports (3478/3479/5349/19302–19309 etc.) → socks when that is the live design
   - Remaining → VPS **direct**
3. On the VPS, `curl` via socks (or panel outbound test): exit IP = residential ≠ VPS public IP. Stop if not.

---

## 5. Clash on the PC
Prefer the **panel Clash sub** over hand-written YAML. A template YAML is backup only.

1. Install Verge/Meta; start with **system proxy** (TUN later).
2. Import the domain subscription URL.
3. **Keep the dns block**: fake-ip, respect-rules, CN DoH, panel/sub via CN resolve—deleting DNS causes leaks / update failure.
4. Keep groups simple, rules detailed. Typical names (adapt if the user’s config differs; Chinese labels are fine if that is their live config):
   - Node select — daily overseas
   - AI — **select VLESS**
   - UDP accel — **select Hysteria2**; STUN/WebRTC here, **not REJECT**
   - Catch-all — unmatched traffic
   - DIRECT
5. WeChat: `find-process-mode` (often always) + process/path → DIRECT + WeChat domains DIRECT.
6. `panel.<domain>` / `sub.<domain>` → **DIRECT** (anti-deadlock).
7. Rules copied from other configs must target **these** group names, not the source config’s.

---

## 6. Verify (gate before signup)
Order matters; any fail → stop.

**A. VPS socks** — residential IP ≠ VPS IP; else whitelist / creds / outbound.

**B. Clash** — both nodes present; AI→VLESS, UDP→hy2; optional log shows Claude/check domains hit the AI group.

**C. Three sites** (same browser, AI path on):
- https://ip.net.coffee/claude/
- https://ipkk.com/claude-ip-check
- https://ipinfo.cv/claude-ai-check  
All residential/usable, consistent country, not DC/VPN.  
**CF caveat:** cloudflare.com (www, challenge) must also AI→socks or IPKK’s CF card lies.

Fail order: AI group → VPS AI/CF→socks → whitelist → CF in AI rules → DNS block intact.

**D. Contrast** — Baidu/CN domestic; WeChat image ok.

**E. Optional** — Google via node-select may show VPS DC IP (expected); STUN not REJECT; leak sites secondary to the three Claude sites.

**Pass bar:** socks home IP + three sites agree + CN/WeChat DIRECT → step 7.

---

## 7. Register + pay (same exit)
1. Clean browser profile; AI group only; no node switch mid-flow.
2. Aged Google; timezone/language/billing country match exit country; store account + recovery codes in a password manager.
3. Friend’s card filled **on this residential path** if possible; billing country = IP country; no commercial card farms; don’t switch nodes during 3DS.
4. Sequence: three-site pass → register/verify email → bind card/subscribe → file receipt.

---

## Soft router (later only)
Whole-home later: e.g. Netcore N60 Pro + 2G + opw OpenWrt + OpenClash/mihomo. Same simple groups, detailed rules. Do **not** block signup waiting for a soft router or old dialer-proxy chains.

---

## Agent behavior when this skill runs
1. Prefer **live** panel / `config.json` / Clash sub over memory; correct stale “hy2-all-socks” claims.
2. Never put real IPs/passwords/sub tokens in public canvases; placeholders only.
3. Coach in the user’s language; keep their live Clash group names when documenting their setup.
4. Soft-router / OpenClash digressions stay out of the main seven steps unless asked.
5. After material live changes, offer to update the user’s beginner canvas + this skill if they drift.

## FAQ (quick)
- Still DC on checkers → AI not on VLESS or VPS not routing AI/CF to socks
- IPKK CF card wrong → cloudflare.com not on AI path
- SOCKS fail → whitelist / spaces in password
- Sub update fail → panel/sub not DIRECT or DNS stripped
- Net works, Claude doesn’t → traffic on node-select, AI rules miss
- WebRTC weird → STUN REJECT’d
- WeChat images fail → process match, not domain-only
- Pay risk → switched node, new mail, card farm, billing≠IP country
