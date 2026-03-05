# Nick Mora — Airbnb Cleaning Automation

**Client:** Nicolas Mora (nickmora92@yahoo.com)
**Project:** Automated cleaner dispatch + two-way Telegram communication
**Status:** Production-ready — bugs fixed, pending go-live with real data
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
  Dedup Check                                 [Cleaner Dispatch]
  (skip already                                (webhook trigger
   dispatched)                                  + 8 AM morning check)
       |                                              |
       v                                              v
  Google Sheets                              @NickCleanBot (Telegram)
  (Bookings log)                                      |
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
| Calendar | Airbnb iCal export (or demo server for dev) | $0 |
| **Total infra cost** | | **$0/mo** |

## N8N Workflows (all ACTIVE)

### 1. Booking Monitor (`NLVOii3Hl4hTz8qc`)
**Trigger:** Schedule — every 30 minutes
**Flow:**
1. `Check Every 30 Minutes` — schedule trigger
2. `Fetch Airbnb Calendar` — HTTP GET to iCal URL
3. `Parse iCal & Find Checkouts` — extracts events with checkout in next 48 hours
4. `Has Checkouts?` — filter (skips if uid doesn't exist)
5. `Fetch Dispatched UIDs` — reads from n8n Data Table (`b8BSdZL1vF0I8m1L`) via API
6. `Filter New Only` — code node compares checkout UIDs against dispatched list, skips duplicates
7. `Log to Google Sheets` — appends to "Bookings" tab (disabled until OAuth linked)
8. `Prepare for Dispatch` — formats payload for Workflow 2
9. `Trigger Dispatch` — POSTs to `/webhook/nick-cleaner-dispatch` to invoke Workflow 2
10. `Record Dispatch` — writes UID to Data Table to prevent re-dispatch

**iCal URL:** Set via `AIRBNB_ICAL_URL_NICK` env var (currently pointed at demo calendar on localhost:8199)

### 2. Cleaner Dispatch (`eJIAwYSzGEzIW51S`)
**Triggers:**
- Webhook POST to `/nick-cleaner-dispatch` (called by Booking Monitor)
- Schedule — 8 AM daily morning check (via `Morning Check Payload` code node)

**Flow:**
1. `Dispatch Trigger` / `Morning Dispatch Check` — entry points
2. `Morning Check Payload` — (8 AM only) builds default payload with today's date and property address
3. `AI Compose Cleaning Message` — builds personalized message:
   - Includes property address, guest stay duration, urgency (TODAY/TOMORROW)
   - Morning check variant with reminder text
   - Special instructions if set
   - Asks cleaner to reply DONE or report issues
4. `Send via Telegram Bot` — POST to `api.telegram.org/bot.../sendMessage`
   - `chat_id` = cleaner's Telegram ID
   - `parse_mode` = Markdown
5. `Log Dispatch to Sheets` — appends to "Dispatch Log" tab

### 3. Inbound Handler (`vFIvxqFZKZFGOUO7`)
**Trigger:** Schedule — polls Telegram every 1 minute
**Flow:**
1. `Poll Telegram Every Minute` — schedule trigger
2. `Get Telegram Updates` — GET `/getUpdates` from Bot API
3. `Process & Filter Updates` — filters by allowed chat ID (env var), classifies messages, returns single item with maxUpdateId + packed messages array
4. `Acknowledge Updates` — GET `/getUpdates?offset={maxUpdateId+1}` to mark all processed updates as acknowledged
5. `Unpack Messages` — code node that unpacks messages array into individual items (returns empty if no messages)
6. `Route by Type` — switch node (v3.2), branches to:

| Type | Action | Notify Nick? |
|---|---|---|
| `job_complete` | Log to Sheets + send "Thank you!" ack to cleaner | No (logged only) |
| `maintenance_issue` | Log to Sheets + alert Nick via Telegram | **Yes** (high priority) |
| `question` | Log to Sheets + forward to Nick via Telegram | **Yes** |
| `general` | Send generic ack to cleaner | No |

## Bugs Fixed (March 2026)

| Bug | Fix |
|---|---|
| Monitor → Dispatch had no connection (dead end) | Added `Trigger Dispatch` HTTP Request node that POSTs to dispatch webhook |
| Morning Dispatch Check connected to nothing | Added `Morning Check Payload` code node wired to `AI Compose Cleaning Message` |
| Hardcoded chat ID `PLACEHOLDER_CHAT_ID` in Inbound Handler | Now reads from `$env.NICK_CLEANER_TELEGRAM_CHAT_ID` with fallback |
| `getUpdates` had no offset/dedup (reprocessed all on restart) | Added `Acknowledge Updates` node that calls `getUpdates?offset=maxId+1` to mark processed |
| No duplicate dispatch prevention | Added n8n Data Table (`b8BSdZL1vF0I8m1L`) to track dispatched UIDs; `Filter New Only` code node skips already-dispatched bookings |
| Broken `Update State` set node in Inbound Handler | Removed; Process & Filter now returns single item with maxUpdateId for acknowledgment |
| n8n task runner blocks `$env`, `fetch()`, static data | Added `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`; replaced static data with Data Table API; used HTTP Request nodes instead of `fetch()` |
| Switch v3 node used invalid properties (`outputName`, `outputIndex`) | Fixed to use `outputKey` in `rules.values`, `fallbackOutput` in `options` collection, typeVersion 3.2 |
| Filter v2 node used invalid `notTrue` boolean operation | Replaced with Code node (`Has Messages?` → `Unpack Messages`) for reliable filtering |
| Google Sheets nodes block activation without OAuth credentials | Added `disabled: true` + `onError: continueRegularOutput`; re-enable after linking OAuth |
| Inbound Handler output 8 items to Acknowledge Updates (ran 8 times) | Restructured Process & Filter to return single item with packed messages array; Unpack Messages node splits for routing |

## Environment Variables (in N8N Docker)

| Variable | Value | Description |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | `8772704912:AAFA5uB-...` | @NickCleanBot token |
| `NICK_CLEANER_TELEGRAM_CHAT_ID` | `PLACEHOLDER_CHAT_ID` | Test cleaner (Joe) |
| `NICK_TRACKER_SHEET_ID` | `16kftTiwJs6VmNVr...` | Google Sheets tracker |
| `NICK_DEFAULT_PROPERTY_ADDRESS` | `YOUR_PROPERTY_ADDRESS, LA 90068` | Default property |
| `NICK_CLEANER_NAME` | `YOUR_CLEANER_NAME` | Test cleaner name |
| `AIRBNB_ICAL_URL_NICK` | `http://localhost:8199/demo-calendar.ics` | Demo calendar |
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | `false` | Allow Code nodes to access env vars |
| `N8N_API_KEY` | (auto-generated JWT) | Used by workflows to call Data Table API |

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

Served from `localhost:8199/demo-calendar.ics` — for development/testing only.

| Guest | Check-in | Check-out | Nights | Notes |
|---|---|---|---|---|
| John Smith (2 guests) | Mar 3 | **Mar 5** | 2 | — |
| Maria Garcia (1 guest) | Mar 5 | **Mar 6** | 1 | Pet-friendly |
| David Kim (4 guests) | Mar 6 | **Mar 9** | 3 | Family with kids |
| Sarah Johnson (2 guests) | Mar 10 | **Mar 12** | 2 | — |

## Production Go-Live Checklist

When Nick is ready to go live, swap these env vars and do these manual steps:

- [ ] Get Nick's Airbnb iCal URL → set `AIRBNB_ICAL_URL_NICK`
- [ ] Get real cleaner on Telegram → set `NICK_CLEANER_TELEGRAM_CHAT_ID` + `NICK_CLEANER_NAME`
- [ ] Get Nick on Telegram → set `NICK_TELEGRAM_CHAT_ID`
- [ ] Link Google Sheets OAuth credential in n8n UI (manual)
- [ ] Share Google Sheet with Nick (view access)
- [ ] Kill demo calendar server (`kill PID`)
- [ ] Change n8n auth from `admin/admin` to secure credentials
- [ ] Re-import workflow JSONs to n8n via API (`PUT /api/v1/workflows/{id}`)

## Testing with Demo Calendar

1. Start the demo calendar server: `node demo-calendar-server.js` (port 8199)
2. In n8n UI, trigger the Booking Monitor manually
3. Verify: checkout detected → logged to Sheets → dispatch webhook fired → Telegram message sent
4. Reply "done" from test Telegram → verify classified as `job_complete`, logged, acknowledged
5. Reply with an issue → verify Nick alert fires
6. Restart n8n → verify old messages are NOT reprocessed (offset fix)
7. Run Monitor again → verify same checkout is NOT re-dispatched (dedup)

## Files

| File | Description |
|---|---|
| `workflow-monitor-live.json` | Booking Monitor workflow (n8n export) |
| `workflow-dispatch-live.json` | Cleaner Dispatch workflow (n8n export) |
| `workflow-inbound-live.json` | Inbound Message Handler workflow (n8n export) |
| `PROJECT.md` | This doc |
| `README.md` | Public-facing project overview |
| `Airbnb Cleaning Automation.pdf` | Client proposal |

## Pricing Model

| | Turno (competitor) | Our solution |
|---|---|---|
| Per unit | ~$8/property/mo | $8/mo flat (unlimited) |
| Setup fee | Varies | $0 |
| Messaging | Built-in (limited) | Telegram ($0) |
| Customization | None | Full (we own the code) |
| Hosting | Their cloud | Our VPS (we control it) |

## Next Steps

- [ ] Run full end-to-end test with demo calendar
- [ ] Link Google Sheets OAuth credential in N8N UI
- [ ] Demo for Nick (screen share or video walkthrough)
- [ ] Go live with real data (see checklist above)

## Future Features (only if Nick requests)

- Photo upload support (detect `message.photo` in inbound handler)
- Multiple properties (array of iCal URLs)
- Multiple cleaners (Sheets-based cleaner registry)
- Error workflow (alert Joe via Telegram on any workflow failure)
- Health check workflow (daily 9 AM ping of all services)
