# Airbnb Cleaning Automation — Nick Mora

Automated cleaner dispatch, real-time communication, and complete job tracking for Airbnb hosts.

This project automates the manual workflow of notifying cleaners about upcoming checkouts and tracking their progress using n8n, Telegram, and Google Sheets.

## 🚀 Key Features

- **24/7 Calendar Monitoring:** Polls Airbnb iCal feeds every 30 minutes to detect guest checkouts.
- **Smart Deduplication:** Integrated n8n Data Table tracking ensures no checkout is ever re-dispatched to the cleaner accidentally.
- **Automated Telegram Dispatch:** Sends personalized property details, guest stay duration, and instructions directly to the cleaner.
- **Morning Duty Check:** Automatic 8 AM daily reminders for properties checking out today.
- **Two-Way Communication:** Cleaners reply via Telegram; the system automatically distinguishes between routine "Done" reports, maintenance issues, or questions.
- **Real-Time Alert Routing:** Routine completions are logged silently to the tracker; host is alerted immediately for issues or questions.
- **Full Transparency:** Every booking, dispatch, and response is logged automatically to Google Sheets.

## 🛠️ Tech Stack

- **Orchestration:** [n8n](https://n8n.io/) (Self-hosted on Ubuntu VPS)
- **Persistance:** n8n Data Tables (UID tracking)
- **Messaging:** Telegram Bot API (@NickCleanBot)
- **Database/Tracking:** Google Sheets
- **Calendar Integration:** Airbnb iCal

## 📐 Architecture

1. **Booking Monitor:** Polls the iCal feed, validates new checkouts, checks for duplicates via Data Table, and triggers the dispatcher.
2. **Cleaner Dispatcher:** Generates personalized AI-driven messages for today/tomorrow checkouts and handles morning reminders.
3. **Inbound Handler:** Polls the Telegram Bot API, acknowledges updates to prevent reprocessing, and routes responses based on classification.

## 📂 Repository Structure

- `workflow-*-live.json`: Production-ready n8n workflows for import.
- `PROJECT.md`: Comprehensive technical documentation, environment variables, and bug-fix history.
- `System Overview - Airbnb Cleaning Automation.pdf`: Premium client overview and sales proposal.
- `demo-calendar-server.js`: Local server for testing the end-to-end flow with mockup Airbnb data.

## 📝 Setup

1. Import the `.json` workflows into your n8n instance.
2. Add required environment variables (see `PROJECT.md` for the full list) to your n8n Docker container.
3. Set `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` to allow the custom logic to access configuration.
4. Link Google Sheets OAuth credentials and shared spreadsheet ID.

---
*Prepared by [Joe's Tech Solutions](https://joestechsolutions.com)*
