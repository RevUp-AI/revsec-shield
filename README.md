# 🛡️ RevSec Shield

**24/7 security monitoring for your OpenClaw agent — delivered to your WhatsApp in plain English.**

RevSec Shield watches your OpenClaw agent for prompt injection attacks, malicious skills, and unexpected configuration changes. When something suspicious is detected, it sends you a plain-English alert. It runs silently in the background — you only hear from it when something needs your attention.

[Homepage](https://revsec.revt2d.com) · [Free signup](https://revsec.revt2d.com/signup) · [Privacy policy](https://revsec.revt2d.com/privacy) · [Support](mailto:hello@revupai.com)

---

## What it does

- **Detects threats** — prompt injection, malicious skills, and unexpected config changes.
- **Alerts you instantly** — plain-English notifications via OpenClaw's WhatsApp integration.
- **Runs silently** — install once, and it monitors in the background every 5 minutes.
- **Keeps you in control** — RevSec detects and reports; you decide what happens next.

RevSec Shield is a **monitoring and alerting** tool. It never modifies your agent, settings, or skills.

---

## Quick start

### 1. Install

Ask your OpenClaw agent:

```
Use clawhub to install revsec-shield
```

Or via CLI:

```bash
npm i -g clawhub
clawhub install revsec-shield
```

### 2. Configure

Get your free API key at **https://revsec.revt2d.com/signup**, then add it to your OpenClaw environment settings (not a shell export):

```
REVSEC_API_KEY=rsk_personal_<your_key>
```

### 3. Activate

Ask your OpenClaw agent:

```
Run the revsec-shield setup
```

RevSec Shield confirms activation and starts monitoring immediately.

> 📱 **WhatsApp users:** After setup, send a message to OpenClaw from WhatsApp once to activate alert delivery. Without this first message, alerts can't reach you on WhatsApp.

---

## Requirements

| Requirement | Details |
|---|---|
| RevSec account | Free — [sign up here](https://revsec.revt2d.com/signup) |
| `REVSEC_API_KEY` | Set in OpenClaw's runtime environment |
| WhatsApp | Connected to OpenClaw (for alert delivery) |

---

## Environment variables

| Variable | Purpose | Required |
|---|---|---|
| `REVSEC_API_KEY` | RevSec API authentication key | Yes |

Set it via OpenClaw's environment settings UI or `.env` file — **never as a shell export**. No other environment variables are read by this skill.

---

## How it works

RevSec Shield operates in three modes:

1. **Setup** — first-time configuration; run once, never repeats.
2. **Alert poll** — a background cron job (`revsec:alert-poll`) that runs every 5 minutes, silently detecting config changes and polling for threats.
3. **Manual check** — ask your agent for a status update any time.

| Cron job | Schedule | Purpose |
|---|---|---|
| `revsec:alert-poll` | `*/5 * * * *` | Auto-detect config changes + poll for threats |

State is stored locally in `~/.openclaw/revsec-state.json` and persists identity and poll timing across sessions.

---

## Privacy & data

RevSec transmits **exactly four fields** to `revsec.revt2d.com` — nothing else:

| Field | What it is |
|---|---|
| `hostname` | Machine name (identifies the device) |
| `skills` | List of skill directory names only |
| `model` | OpenClaw model name from config |
| `openclaw_agent_id` | Locally-generated UUID (stable agent identity) |

**Never transmitted:** message content, conversation history, file contents, credential values, or any config beyond the model name.

- **WhatsApp:** RevSec has no access to your WhatsApp credentials, phone number, or message content. It sends alert text to OpenClaw, which delivers it to your connected channel.
- **API key:** Only sent as a Bearer token to `revsec.revt2d.com` — never logged, never stored in state.
- **Network:** The skill connects only to `revsec.revt2d.com`. No other outbound connections.
- **Read-only:** RevSec never modifies OpenClaw settings, agent configuration, or installed skills.

Full details: [Privacy policy](https://revsec.revt2d.com/privacy).

---

## What RevSec Shield does **not** do

- Change any OpenClaw settings or configuration
- Access data beyond skill directory names, hostname, and model name
- Store data locally beyond `~/.openclaw/revsec-state.json`
- Send prompts or conversation content to RevSec
- Block or modify LLM calls directly
- Read credential files or environment variables beyond `REVSEC_API_KEY`

---

## Troubleshooting

| Issue | Fix |
|---|---|
| `No API key found` | Set `REVSEC_API_KEY` in OpenClaw environment settings |
| `401 Unauthorized` | Check key starts with `rsk_personal_` and is complete (77 chars) |
| `429 Agent limit` | Free tier supports 1 agent — [upgrade](https://revsec.revt2d.com/upgrade) |
| Cron job errors | Ask your agent: "Run the revsec-shield setup" to recreate the cron job |
| No WhatsApp alerts | Send a message to OpenClaw from WhatsApp to establish a session |
| Setup re-runs every time | Check state file has `"setup_complete": true` |

---

## Links

- **Dashboard:** https://revsec.revt2d.com/personal
- **Free signup:** https://revsec.revt2d.com/signup
- **Privacy policy:** https://revsec.revt2d.com/privacy
- **Team plan:** https://revsec.revt2d.com/signup/team
- **Support:** hello@revupai.com

---

## Publisher

**RevUp AI, Inc.** — [@RevUp-AI](https://github.com/RevUp-AI)
Homepage: https://revsec.revt2d.com · Contact: hello@revupai.com
