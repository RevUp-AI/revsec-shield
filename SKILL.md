---
name: revsec-shield
description: >
  24/7 security monitoring for your OpenClaw agent. Detects prompt injection
  attacks, malicious skills, and data exfiltration attempts. Delivers plain-English
  alerts to your WhatsApp when something suspicious is detected. Use when the user
  asks about agent security, wants to check for threats, asks what RevSec detected,
  or when running setup for the first time. Also handles the 5-minute background
  security poll (cron job revsec:alert-poll).
homepage: https://revsec.revt2d.com
privacy: https://revsec.revt2d.com/privacy
metadata:
  {
    "openclaw":
      {
        "emoji": "🛡️",
        "requires": { "env": ["REVSEC_API_KEY"] },
        "primaryEnv": "REVSEC_API_KEY",
      },
  }
---

# RevSec Shield

One-line description: 24/7 security monitoring for your OpenClaw agent —
delivered to your WhatsApp in plain English.

## Overview

- **What it does:** Monitors your OpenClaw agent for prompt injection attacks,
  malicious skills, and unexpected configuration changes. Sends WhatsApp alerts
  when something suspicious is detected.
- **When to use it:** Install once. It runs silently in the background. You only
  hear from it when something needs your attention.
- **Requirements:** A free RevSec account (revsec.revt2d.com/signup), WhatsApp
  connected to OpenClaw, and a `REVSEC_API_KEY` environment variable.
- **Publisher:** RevUp AI, Inc. — hello@revupai.com — revsec.revt2d.com

RevSec Shield is a **monitoring and alerting** tool. It detects threats and
puts them in front of you immediately. You stay in control of what happens next.

**Three operating modes:**

1. **Setup** — first-time configuration, run once, never needs repeating
2. **Alert poll** — background cron job, runs every 5 minutes silently
3. **Manual check** — user asks for a status update

---

## Quick Start

### Install
Ask your OpenClaw agent:
```
Use clawhub to install revsec-shield
```

Or via CLI:
```bash
npm i -g clawhub
clawhub install revsec-shield
```

### Configure
Add your API key in OpenClaw's environment settings (not shell exports):
```
REVSEC_API_KEY=rsk_personal_<your_key>
```

Get your free key at: https://revsec.revt2d.com/signup

### Activate
Ask your OpenClaw agent:
```
Run the revsec-shield setup
```

That's it. RevSec Shield will confirm activation and start monitoring immediately.

---

## Environment Variable Contract

| Variable | Purpose | Required | Where to set |
|---|---|---|---|
| `REVSEC_API_KEY` | RevSec API authentication key | Yes | OpenClaw runtime environment |

Set via OpenClaw's environment settings UI or `.env` file — never as a shell export.
No other environment variables are read by this skill.

---

## Data collected and transmitted

This skill transmits exactly the following fields to `revsec.revt2d.com` — nothing else:

| Field | What it is | Why it's needed |
|---|---|---|
| `hostname` | Machine name (e.g. `Nehas-iMac.local`) | Identifies which device the agent runs on |
| `skills` | List of skill directory names only | Detects new integrations |
| `model` | OpenClaw model name from config | Tracks model changes |
| `openclaw_agent_id` | Locally-generated UUID, never changes | Stable agent identity |

**Never transmitted:** message content, conversation history, file contents,
credential values, environment variables beyond presence-check of REVSEC_API_KEY,
or any other config file contents beyond the model name field.

**WhatsApp:** Alerts are delivered via OpenClaw's existing WhatsApp integration.
RevSec does not have access to WhatsApp credentials, your phone number, or
message content. RevSec sends alert text to OpenClaw; OpenClaw delivers it
to your connected channel.

---

## Core rules (always follow)

- Never ask the user to do something manually that can be done via curl or shell.
- RevSec detects first, reports second. Never ask the user "should I flag this?" —
  report what was already detected.
- Keep alert messages plain and human. No JSON, no confidence scores, no
  technical jargon unless the user asks for detail.
- State file is `~/.openclaw/revsec-state.json`. Read before every action.
  Write after every action that changes state.
- If the state file is missing, run Setup before anything else.
- If `REVSEC_API_KEY` is not set, prompt the user to get one and stop.
- All curl calls must include `-s` (silent) and `-f` (fail on HTTP errors).
- On any curl failure: log the error to state, stay silent to the user unless
  it is a setup step or they explicitly asked for a status check.
- Never re-run setup if it has already completed. Use the existing
  `openclaw_agent_id` from state for all subsequent API calls.

---

## State file — `~/.openclaw/revsec-state.json`

Read and write this file to persist identity and poll state across sessions.

**Schema:**
```json
{
  "openclaw_agent_id": "<uuid — generated once, never changes>",
  "agent_id": "<uuid — returned by RevSec on registration>",
  "org_id": "<uuid — returned by RevSec on registration>",
  "last_poll_at": "<ISO 8601 timestamp — advanced after each successful poll>",
  "registered_at": "<ISO 8601 timestamp>",
  "registered_skills": ["skill1", "skill2"],
  "skill_version": "1.0.0",
  "setup_complete": true
}
```

**Read state:**
```bash
cat ~/.openclaw/revsec-state.json 2>/dev/null || echo "{}"
```

**Write state** (always write the full object, never partial):
```bash
cat > ~/.openclaw/revsec-state.json << 'EOF'
{ <full updated state object> }
EOF
```

---

## Workflow 1 — Setup (run when setup_complete is missing or false)

**BEFORE ANYTHING ELSE — Check if already set up:**

Read state file:
```bash
cat ~/.openclaw/revsec-state.json 2>/dev/null || echo "{}"
```

If `setup_complete` is `true`:
1. Check if cron job exists: `openclaw cron list`
2. If `revsec:alert-poll` is missing → create it (Step 5 only)
3. Tell user: "RevSec Shield is already active and protecting your agent."
4. **STOP — do not proceed with Steps 1–4. Do not re-register.**

Only run Steps 1–6 if `setup_complete` is missing or false.

### Step 1 — Check for API key

Check if the key is present without exposing its value:
```bash
[ -n "$REVSEC_API_KEY" ] && echo "✓ API key is set" || echo "✗ API key not set — see setup instructions"
```

If not set, stop and tell the user:

> To activate RevSec Shield, get your free API key at
> https://revsec.revt2d.com/signup
> Then add it to your OpenClaw environment settings:
> `REVSEC_API_KEY=rsk_personal_<your_key>`

Do not proceed until the key is set.

### Step 2 — Generate stable agent identity

Generate a UUID. This is the `openclaw_agent_id` — it never changes.

```bash
python3 -c "import uuid; print(uuid.uuid4())"
```

If python3 is unavailable:
```bash
cat /proc/sys/kernel/random/uuid 2>/dev/null || uuidgen
```

Store this value. It will be written to state in Step 4.

### Step 3 — Collect exactly three values from the local environment

Collect ONLY: hostname, model name, and skill directory names.
Do not read any other files or environment variables. Do not ask the user
questions that can be inferred from these three values.

```bash
# 1. Hostname only — no other system info
hostname

# 2. Model name from OpenClaw config — model field only
cat ~/.openclaw/openclaw.json 2>/dev/null | python3 -c "
import json,sys
d=json.load(sys.stdin)
print(d.get('model','unknown'))
" 2>/dev/null || echo "unknown"

# 3. Skill directory names only — not file contents
ls ~/.openclaw/skills/ 2>/dev/null || ls ~/clawd/skills/ 2>/dev/null || echo ""
```

Capture the skills list — this becomes the baseline for change detection.

### Step 4 — Register agent with RevSec

Send only the three collected values plus the generated UUID:

```bash
curl -sf -X POST \
  "https://revsec.revt2d.com/fcc/api/v1/personal/register-agent" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $REVSEC_API_KEY" \
  -d '{
    "openclaw_agent_id": "<uuid from Step 2>",
    "skills": [<list of skill directory names as strings>],
    "model": "<model name from Step 3>",
    "channels": ["openclaw"],
    "integrations": [],
    "hostname": "<hostname from Step 3>",
    "skill_version": "1.0.0"
  }'
```

On success the API returns `{ "agent_id": "...", "action": "registered" }`.

Write the full state file including the skills baseline:

```bash
cat > ~/.openclaw/revsec-state.json << 'EOF'
{
  "openclaw_agent_id": "<uuid from Step 2>",
  "agent_id": "<agent_id from API response>",
  "org_id": "<org_id from API response if present, else empty string>",
  "last_poll_at": "<current UTC timestamp in ISO 8601>",
  "registered_at": "<current UTC timestamp in ISO 8601>",
  "registered_skills": [<same skills list sent to API>],
  "skill_version": "1.0.0",
  "setup_complete": true
}
EOF
```

### Step 5 — Schedule the background alert poll

Check if the cron job already exists:

```bash
openclaw cron list
```

Look for a job named `revsec:alert-poll`. If it exists, skip creation.

If it does not exist, create it:

```bash
openclaw cron add \
  --name "revsec:alert-poll" \
  --model "google/gemini-2.0-flash" \
  --schedule "*/5 * * * *" \
  --prompt "Run the revsec-shield skill in alert poll mode. Do these two things silently: 1) Check if any new skills have been installed since last registration by comparing the current skills list against the registered_skills array in the state file — if changed, call register-agent automatically with the updated skills list and update registered_skills in state. 2) Poll for new security alerts from RevSec since the last poll timestamp and deliver any alerts to the user. Be completely silent if there are no changes and no alerts."
```

### Step 6 — Confirm setup to the user

Tell the user exactly this (keep it short, match the tone):

> ✅ RevSec Shield is active. Your agent is registered and protected.
>
> I'll check for threats every 5 minutes and alert you here if anything
> is detected. You won't hear from me unless something happens.
>
> Your dashboard: https://revsec.revt2d.com/personal
>
> 📱 **If you use WhatsApp:** Send me a message from WhatsApp now to
> activate alert delivery. Without this first message, I cannot reach
> you on WhatsApp when threats are detected. Just say "Hi" from the
> number you connected during setup.

---

## Workflow 2 — Alert poll (triggered by cron job revsec:alert-poll)

This workflow runs every 5 minutes automatically. It must be fast and silent
when there are no changes and no alerts. Never output anything to the user
unless there is something they need to know.

### Step 1 — Read state

```bash
cat ~/.openclaw/revsec-state.json
```

If state is missing or `setup_complete` is not true, output nothing and stop.
Do not trigger setup from a cron context.

### Step 1b — Check for config changes (automatic, silent)

Get current skill directory names only:

```bash
ls ~/.openclaw/skills/ 2>/dev/null || ls ~/clawd/skills/ 2>/dev/null || echo ""
```

**If `registered_skills` is missing from state file:**
Treat it as an empty list. All current skills are "new". Re-register once to
set the baseline, then write `registered_skills` to state. Stay silent —
this is a one-time migration for users upgrading from an older version.

**If the skills list has changed (new skill added or skill removed):**

1. Re-register the agent using the existing `openclaw_agent_id` from state:

```bash
curl -sf -X POST \
  "https://revsec.revt2d.com/fcc/api/v1/personal/register-agent" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $REVSEC_API_KEY" \
  -d '{
    "openclaw_agent_id": "<openclaw_agent_id from state — never changes>",
    "skills": [<current skill directory names>],
    "model": "<model from openclaw.json or unknown>",
    "channels": ["openclaw"],
    "integrations": [],
    "hostname": "<hostname>",
    "skill_version": "1.0.0"
  }'
```

2. Update `registered_skills` in the state file with the new skills list.
   Keep all other state values unchanged.

3. Stay completely silent — this is automatic background maintenance.
   The RevSec policy engine will evaluate the updated config and create
   violations if any new skills are suspicious. Those violations will be
   picked up in Step 2 on this same poll cycle.

**If the skills list has not changed:** skip to Step 2.

### Step 2 — Poll for new alerts

```bash
curl -sf \
  "https://revsec.revt2d.com/fcc/api/v1/personal/alerts?since=<last_poll_at from state>" \
  -H "Authorization: Bearer $REVSEC_API_KEY"
```

If curl fails (network error, timeout, 5xx): update nothing, stay silent, stop.
Do not advance `last_poll_at` on failure — the same window will be retried
on the next poll.

If the response is an empty array `[]`: stay completely silent. Advance
`last_poll_at` to now and write state. Stop.

### Step 3 — Deliver alerts (only if response is non-empty)

For each alert in the response, use the `message` field from the API response
directly — it is already formatted for human delivery. Do not add your own
headers or reformat it.

If there are multiple alerts, deliver them as a single message grouped by
severity — critical first, then high, then medium. Use a separator line
between groups only if there are multiple severity levels.

### Step 4 — Advance poll timestamp

After successful delivery (or confirmed empty response), update `last_poll_at`
to the current UTC time and write the full state file.

```bash
python3 -c "from datetime import datetime,timezone; print(datetime.now(timezone.utc).isoformat())"
```

---

## Workflow 3 — Manual security check

### Step 1 — Read state

Check `~/.openclaw/revsec-state.json`. If setup is not complete, run
Workflow 1 first.

### Step 1b — Check cron health (self-healing)

```bash
openclaw cron list
```

If `revsec:alert-poll` is missing from the list — recreate it silently
using Step 5 from Workflow 1, then tell the user:

> Background monitoring was inactive — I've restarted it. Your protection
> is now fully active again.

### Step 2 — Fetch dashboard summary

```bash
curl -sf \
  "https://revsec.revt2d.com/fcc/api/v1/personal/summary" \
  -H "Authorization: Bearer $REVSEC_API_KEY"
```

### Step 3 — Report to user

```
🛡️ RevSec Shield — last 7 days

Threats detected: <threats_blocked_7d>
Flagged for review: <threats_flagged_7d>
Agent: <agent.name> (<agent.agent_framework>)
Status: <agent.status>
Policies active: <active_policies>

Full details: https://revsec.revt2d.com/personal
```

If both counts are 0:

```
🛡️ RevSec Shield — no threats detected in the last 7 days.
Your agent is clean. Protection is active.
```

---

## Workflow 4 — Config update (automatic via cron, or triggered manually)

The user never needs to run this manually — it happens automatically.

### Step 1 — Read current state

Get `openclaw_agent_id` from `~/.openclaw/revsec-state.json`.

### Step 2 — Collect current config (three values only)

```bash
# Skill directory names only
ls ~/.openclaw/skills/ 2>/dev/null || echo ""

# Model name from config
cat ~/.openclaw/openclaw.json 2>/dev/null | python3 -c "
import json,sys
d=json.load(sys.stdin)
print(d.get('model','unknown'))
" 2>/dev/null || echo "unknown"

# Hostname
hostname
```

### Step 3 — Re-register with updated config

```bash
curl -sf -X POST \
  "https://revsec.revt2d.com/fcc/api/v1/personal/register-agent" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $REVSEC_API_KEY" \
  -d '{
    "openclaw_agent_id": "<openclaw_agent_id from state — never changes>",
    "skills": [<current skill directory names>],
    "model": "<current model>",
    "channels": ["openclaw"],
    "integrations": [],
    "hostname": "<hostname>",
    "skill_version": "1.0.0"
  }'
```

### Step 4 — Update state file

Update `registered_skills` in the state file with the new skills list.
Keep all other state values unchanged — especially `openclaw_agent_id`.

If triggered manually by the user, confirm:

> ✅ RevSec updated — your agent profile is now in sync.

If triggered automatically by cron, stay silent.

---

## Security & Guardrails

- **No message content sent:** RevSec only receives agent metadata — skill names,
  hostname, model. Your conversations and message content are never sent to RevSec.
- **Exact data sent to revsec.revt2d.com:** Only these four fields are transmitted:
  hostname (machine name), list of installed skill directory names, OpenClaw model
  name, and a locally-generated UUID. Nothing else. No file contents, no config
  values, no environment variables beyond confirming REVSEC_API_KEY is set.
- **API key handling:** Setup checks that REVSEC_API_KEY is present without echoing
  its value. The key is only transmitted as a Bearer token in Authorization headers
  to revsec.revt2d.com — never logged, never stored in the state file.
- **WhatsApp delivery:** Alerts are delivered via OpenClaw's existing WhatsApp
  integration. RevSec does not have access to WhatsApp credentials, your phone
  number, or message content. RevSec sends alert text to OpenClaw; OpenClaw
  delivers it to your connected channel.
- **Credentials:** API key stored in OpenClaw runtime environment only.
  Never logged or transmitted beyond the RevSec API.
- **Network access:** Only connects to `revsec.revt2d.com`. No other outbound
  connections are made by this skill.
- **Local storage:** State stored only in `~/.openclaw/revsec-state.json`.
  No other local files are created or modified.
- **Fail-closed on API errors:** If RevSec API is unreachable, skill stays
  silent and retries on next poll. Agent behaviour is never modified.
- **No pipe-to-interpreter:** No `curl | bash`, `curl | python`, or similar
  patterns are used anywhere in this skill.
- **No credential harvesting:** This skill does not read environment variables
  beyond `REVSEC_API_KEY`, does not access credential files, and does not
  transmit system information beyond hostname and skill directory names.
- **Read-only agent monitoring:** This skill never modifies OpenClaw settings,
  agent configuration, or installed skills.
- **Privacy policy:** https://revsec.revt2d.com/privacy — documents what data
  is collected, retained, and how it is used.
- **Company:** RevUp AI, Inc. Contact: hello@revupai.com

---

## Error handling

| Situation | Action |
|---|---|
| `REVSEC_API_KEY` not set | Stop, prompt user to get key at signup URL |
| State file missing during cron poll | Stay silent, stop |
| curl timeout during cron poll | Stay silent, do not advance timestamp |
| HTTP 429 from API | Stay silent during cron. On manual check: "You've hit the daily scan limit. Upgrade at https://revsec.revt2d.com/upgrade" |
| HTTP 401 from API | Tell user their API key may have expired. Direct to: https://revsec.revt2d.com/personal |
| HTTP 5xx from API | Stay silent during cron. On manual check: "RevSec is temporarily unavailable — protection resumes automatically." |
| Registration fails | Tell user registration failed, show the curl error, ask them to try again |
| New skill detected but re-registration fails | Stay silent, retry on next cron run |
| Cron job missing | Recreate silently during next manual check (Workflow 3 Step 1b) |
| WhatsApp alerts not arriving | Remind user to send first message from WhatsApp to establish session |

---

## Troubleshooting

| Error | Fix |
|---|---|
| `No API key found` | Set `REVSEC_API_KEY` in OpenClaw environment settings |
| `401 Unauthorized` | Check key starts with `rsk_personal_` and is complete (77 chars) |
| `429 Agent limit` | Free tier supports 1 agent — upgrade at revsec.revt2d.com/upgrade |
| Cron job errors | Ask agent: "Run the revsec-shield setup" to recreate cron job |
| No WhatsApp alerts | Send a message to OpenClaw from WhatsApp first to establish session |
| `registered_skills` missing | Cron will auto-fix on next run — no action needed |
| Setup re-runs every time | Check state file has `"setup_complete": true` |

---

## What RevSec Shield does NOT do

- It does not change any OpenClaw settings or configuration
- It does not access any data beyond skill directory names, hostname, and model name
- It does not store any data locally beyond `~/.openclaw/revsec-state.json`
- It does not send prompts or conversation content to RevSec — only metadata
- It does not block or modify LLM calls directly — it monitors agent
  configuration and behaviour via the RevSec API
- It does not initiate WhatsApp sessions — user must send first message
- It does not read credential files or access environment variables beyond REVSEC_API_KEY

---

## Cron job reference

| Job name | Schedule | Purpose |
|---|---|---|
| `revsec:alert-poll` | `*/5 * * * *` | Auto-detect config changes + poll for threats |

Use `openclaw cron list` to verify the job is registered.
Use `openclaw cron runs revsec:alert-poll` to see recent run history.

---

## Release notes

**v1.0.2** — Security transparency update
- Added explicit data transmission table showing exactly what is sent to API
- API key check no longer echoes key value
- Narrowed Step 3 language to specify exactly three collected values
- Added WhatsApp clarification — RevSec does not access WhatsApp credentials
- Added privacy policy link and company identity
- Added "What RevSec Shield does NOT do" section

**v1.0.1** — Patch
- Improved Security & Guardrails section

**v1.0.0** — Initial public release
- Agent registration and monitoring
- 5-minute background polling
- WhatsApp alert delivery
- Automatic config change detection
- New integration alerts
- Personal security dashboard at revsec.revt2d.com/personal

---

## Links

- Dashboard: https://revsec.revt2d.com/personal
- Free signup: https://revsec.revt2d.com/signup
- Privacy policy: https://revsec.revt2d.com/privacy
- Team plan: https://revsec.revt2d.com/signup/team
- Support: hello@revupai.com

---

## Publisher

- **Publisher:** @RevUp-AI
- **Company:** RevUp AI, Inc.
- **Homepage:** https://revsec.revt2d.com
- **Support:** hello@revupai.com
- **GitHub:** https://github.com/RevUp-AI/revsec-shield