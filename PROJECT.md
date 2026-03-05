# Nick Mora — Airbnb Cleaning Automation

**Client:** Nicolas Mora (nickmora92@yahoo.com)
**Project:** Automated cleaner dispatch + two-way Telegram communication
**Status:** Beta — End-to-end testing phase
**Pricing:** $8/mo flat retainer (no setup fee, unlimited properties)
**Started:** March 4, 2026

---

## Architecture Overview

```
  Airbnb iCal Feed
       |
       v
 [Booking Monitor]  ── polls every 30 min ──>  Parses checkouts
       |                                              |
       v                                              v
  Google Sheets                              [Cleaner Dispatch]
  (Bookings log)                                      |
                                                      v
                                          @NickCleanBot (Telegram)
                                                      |
                                              DM to cleaner
                                                      |
                                                      v
                                          Cleaner replies "Done"
                                                      |
                                                      v
                                           [Inbound Handler]
                                            Classifies reply:
                                           /       |       \
                                      "done"   "issue"   "question"
                                        |         |          |
                                        v         v          v
                                     Log to    Alert      Forward
                                     Sheets    Nick       to Nick
```

## Stack

| Component | Tech | Cost |
|---|---|---|
| Orchestration | N8N (self-hosted, Docker) | $0 |
| Hosting | OpenYogaClaw (Ubuntu VPS) | Already paid |
| Messaging | Telegram Bot API (@NickCleanBot) | $0 |
| Tracking | Google Sheets | $0 |
| Calendar | Airbnb iCal export (or demo server) | $0 |
| **Total infra cost** | | **$0/mo** |

## N8N Workflows (all ACTIVE)

### 1. Booking Monitor (`NLVOii3Hl4hTz8qc`)
**Trigger:** Schedule — every 30 minutes
**Flow:**
1. `Check Every 30 Minutes` — schedule trigger
2. `Fetch Airbnb Calendar` — HTTP GET to iCal URL
3. `Parse iCal & Find Checkouts` — extracts events with checkout in next 48 hours
4. `Has Checkouts?` — filter (skips if none)
5. `Log to Google Sheets` — appends to "Bookings" tab
6. `Prepare for Dispatch` — formats payload for Workflow 2

**iCal URL:** Set via `AIRBNB_ICAL_URL_NICK` env var (currently pointed at demo calendar on localhost:8199)

### 2. Cleaner Dispatch (`eJIAwYSzGEzIW51S`)
**Triggers:**
- Webhook POST to `/nick-cleaner-dispatch` (called by Booking Monitor)
- Schedule — 8 AM daily morning check

**Flow:**
1. `Dispatch Trigger` / `Morning Dispatch Check` — entry points
2. `AI Compose Cleaning Message` — builds personalized message:
   - Includes property address, guest stay duration, urgency (TODAY/TOMORROW)
   - Special instructions if set
   - Asks cleaner to reply DONE or report issues
3. `Send via Telegram Bot` — POST to `api.telegram.org/bot.../sendMessage`
   - `chat_id` = cleaner's Telegram ID
   - `parse_mode` = Markdown
4. `Log Dispatch to Sheets` — appends to "Dispatch Log" tab

### 3. Inbound Handler (`vFIvxqFZKZFGOUO7`)
**Trigger:** Schedule — polls Telegram every 1 minute
**Flow:**
1. `Poll Telegram Every Minute` — schedule trigger
2. `Get Telegram Updates` — GET `/getUpdates` from Bot API
3. `Process & Filter Updates` — deduplicates, extracts text, classifies:
   - **done/finished/complete/cleaned** → `job_complete`
   - **broken/damage/leak/missing** → `maintenance_issue`
   - **? / where / when / how** → `question`
   - anything else → `general`
4. `Route by Type` — switch node, branches to:

| Type | Action | Notify Nick? |
|---|---|---|
| `job_complete` | Log to Sheets + send "Thank you!" ack to cleaner | No (logged only) |
| `maintenance_issue` | Log to Sheets + alert Nick via Telegram | **Yes** (high priority) |
| `question` | Log to Sheets + forward to Nick via Telegram | **Yes** |
| `general` | Send generic ack to cleaner | No |

## Environment Variables (in N8N Docker)

| Variable | Value | Description |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | `8772704912:AAFA5uB-...` | @NickCleanBot token |
| `NICK_CLEANER_TELEGRAM_CHAT_ID` | `PLACEHOLDER_CHAT_ID` | Test cleaner (Joe) |
| `NICK_TRACKER_SHEET_ID` | `16kftTiwJs6VmNVr...` | Google Sheets tracker |
| `NICK_DEFAULT_PROPERTY_ADDRESS` | `YOUR_PROPERTY_ADDRESS, LA 90068` | Default property |
| `NICK_CLEANER_NAME` | `YOUR_CLEANER_NAME` | Test cleaner name |
| `AIRBNB_ICAL_URL_NICK` | `http://host.docker.internal:8199/demo-calendar.ics` | Demo calendar |

## Telegram Bot

| Field | Value |
|---|---|
| Bot name | NickCleanBot |
| Username | @NickCleanBot |
| Bot ID | 8772704912 |
| Token | Stored in N8N env `TELEGRAM_BOT_TOKEN` |
| Test cleaner | Joe (chat ID: PLACEHOLDER_CHAT_ID) |

## Google Sheets Tracker

**Sheet ID:** `16kftTiwJs6VmNVrdGag4L9vh7JAfCKMKeRbphlQGPnU`

**Tabs:**
- `Bookings` — all detected checkout events (from Booking Monitor)
- `Dispatch Log` — every message sent to cleaner (from Cleaner Dispatch)
- `Job Log` — cleaner replies classified as job_complete (from Inbound Handler)

## Demo Calendar (Test Data)

Served from `localhost:8199/demo-calendar.ics`

| Guest | Check-in | Check-out | Nights | Notes |
|---|---|---|---|---|
| John Smith (2 guests) | Mar 3 | **Mar 5** | 2 | — |
| Maria Garcia (1 guest) | Mar 5 | **Mar 6** | 1 | Pet-friendly |
| David Kim (4 guests) | Mar 6 | **Mar 9** | 3 | Family with kids |
| Sarah Johnson (2 guests) | Mar 10 | **Mar 12** | 2 | — |

## Test Results

| Test | Date | Result |
|---|---|---|
| Bot sends message to Joe | Mar 4 | PASS |
| Joe replies "Done" | Mar 4 | PASS — classified as `job_complete` |
| Full end-to-end flow | — | **PENDING** |

## What Happens in Production

1. Nick provides his Airbnb iCal URL (from Airbnb hosting dashboard)
2. We swap `AIRBNB_ICAL_URL_NICK` to the real URL
3. We swap `NICK_CLEANER_TELEGRAM_CHAT_ID` to the real cleaner's Telegram
4. We swap `NICK_CLEANER_NAME` to the real cleaner's name
5. Nick gets a Telegram account (or we use his existing one) for alerts
6. We set `NICK_TELEGRAM_CHAT_ID` to Nick's Telegram ID for alert routing
7. Google Sheets OAuth linked for persistent logging
8. Workflows are already active — it just starts working

## Files

| File | Location |
|---|---|
| Proposal PDF | `~/Projects/nick-cleaning-assistant/Airbnb Cleaning Automation.pdf` |
| This doc | `~/Projects/nick-cleaning-assistant/PROJECT.md` |
| Dispatch workflow (exported) | `~/Projects/nick-cleaning-assistant/workflow-cleaner-dispatch-telegram.json` |
| Inbound workflow (exported) | `~/Projects/nick-cleaning-assistant/workflow-inbound-handler-polling.json` |
| Demo calendar | `~/.openclaw/workspace/n8n-workflows/test-airbnb-calendar.ics` |
| N8N UI | `http://localhost:5678` |

## Pricing Model

| | Turno (competitor) | Our solution |
|---|---|---|
| Per unit | ~$8/property/mo | $8/mo flat (unlimited) |
| Setup fee | Varies | $0 |
| Messaging | Built-in (limited) | Telegram ($0) |
| Customization | None | Full (we own the code) |
| Hosting | Their cloud | Our VPS (we control it) |

## Next Steps

- [ ] Run full end-to-end test (checkout detected → bot message → reply → logged)
- [ ] Link Google Sheets OAuth credential in N8N UI
- [ ] Send proposal to Nick (nickmora92@yahoo.com)
- [ ] Demo for Nick (screen share or video walkthrough)
- [ ] Swap test data for real Airbnb iCal + real cleaner info
- [ ] Add duplicate dispatch prevention (don't re-notify for same checkout)
- [ ] Add photo upload support (cleaner sends photo → logged)
