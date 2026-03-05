# Airbnb Cleaning Automation — Nick Mora

Automated cleaner dispatch, real-time communication, and complete job tracking for Airbnb hosts.

This project automates the manual workflow of notifying cleaners about upcoming checkouts and tracking their progress using n8n, Telegram, and Google Sheets.

## 🚀 Features

- **24/7 Calendar Monitoring:** Polls Airbnb iCal feeds every 30 minutes to detect guest checkouts.
- **Automated Telegram Dispatch:** Sends personalized property details and instructions to cleaners.
- **Two-Way Communication:** Cleaners reply via Telegram; the system understands "Done", maintenance issues, or questions.
- **Real-Time Alert Routing:** Routine completions are logged silently; issues and questions reach the host immediately.
- **Full Transparency:** Every booking, dispatch, and response is logged automatically to Google Sheets.

## 🛠️ Tech Stack

- **Orchestration:** [n8n](https://n8n.io/) (Self-hosted)
- **Messaging:** Telegram Bot API (@NickCleanBot)
- **Database/Tracking:** Google Sheets
- **Calendar Integration:** Airbnb iCal

## 📐 Architecture

1. **Booking Monitor:** Detects checkouts and logs them to a "Bookings" sheet.
2. **Cleaner Dispatcher:** Composes and sends Telegram messages to the cleaner.
3. **Inbound Handler:** Classifies cleaner replies and routes them to the host or the logs.

## 📂 Repository Structure

- `workflow-*.json`: Exported n8n workflows for easy import.
- `PROJECT.md`: Technical documentation and configuration details.
- `System Overview - Airbnb Cleaning Automation.pdf`: Premium presentation for clients.

## 📝 Setup

1. Import the `.json` workflows into your n8n instance.
2. Configure environment variables for `TELEGRAM_BOT_TOKEN`, `NICK_TRACKER_SHEET_ID`, and `AIRBNB_ICAL_URL_NICK`.
3. Link Google Sheets OAuth credentials in n8n.

---
*Prepared by [Joe's Tech Solutions](https://joestechsolutions.com)*
