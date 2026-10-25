# Missed Call Recovery

An n8n workflow for an HVAC business that follows up on missed calls. When a call is missed, the phone system sends the caller's details to a webhook. During business hours, the workflow drafts a short, friendly text with OpenAI and posts it to the team's Telegram chat. After hours, it sends a "we are closed, we will call first thing" message instead. Repeat calls from the same number within an hour are skipped. A Telegram chat with an AI assistant lets the team reply and keep a conversation going, and every outcome is logged to Google Sheets.

## Architecture

```
Part 1: Missed call (POST /missed-call)
Webhook → Respond to Webhook (empty XML reply)
   → Valid Call Data? (E.164 phone, name, business)
        ├── No  → Log Invalid Call
        └── Yes → Within Business Hours? (09:00 to 20:00)
                   ├── Yes → Get Call History → Check Duplicate (1hr)
                   │          → Is New Call?
                   │               ├── Yes → Generate Recovery SMS (gpt-5-mini)
                   │               │            ├── success → Send Telegram Message
                   │               │            │              ├── Log Sent
                   │               │            │              ├── Chat Memory Manager (save to AI memory)
                   │               │            │              └── error → Log Telegram Failed → Send Failure Alert
                   │               │            └── error   → Log AI Failed → Send Failure Alert
                   │               └── No  → Log Duplicate Skipped
                   └── No  → Basic LLM Chain (after-hours text, gpt-5-mini)
                              → Send After Hours Reply → Log After Hours

Part 2: Team chat (Telegram Trigger)
Telegram Trigger → AI Agent (Mia, gpt-5-mini, Redis memory) → Send AI Reply
```

## Features

* Accepts missed-call data from a phone system webhook and replies with an empty XML response right away
* Checks the caller's number is in international format (`+` followed by 10 to 15 digits) and that a name and business are given
* Sends a different message during business hours (09:00 to 20:00) and after hours
* Prevents repeat messages: a second call from the same number within one hour is logged as `duplicate_skipped`
* Writes casual, short text messages with OpenAI. The prompt forbids em dashes, sign-offs, and bullet points so the text reads like a real person
* Keeps the AI's messages in Redis memory, so replies from the team chat continue the same conversation
* Logs every outcome to Google Sheets: `sent`, `failed`, `after_hours`, `duplicate_skipped`, or `invalid`
* Telegram alert when the message cannot be drafted or sent

## Tech Stack

* n8n 2.x (workflow automation, LangChain AI Agent)
* OpenAI `gpt-5-mini` (recovery and after-hours text, team chat assistant)
* Redis (chat memory)
* Google Sheets (call log)
* Telegram Bot API (team chat and alerts)

## Setup

1. Create a Google Sheet with a tab named `Sheet1` and these headers in row 1:

```
caller_name | business | missed_at | logged_at | caller_phone | status | message
```

2. Set up a Redis instance (local or hosted) and add its credential in n8n.
3. Import `missed_call_recovery.json` into n8n. The workflow timezone is set to `Asia/Karachi`. Change it in the workflow settings if you are in another time zone.
4. Add credentials: OpenAI, Google Sheets OAuth2, Telegram, and Redis. Assign them to the matching nodes.
5. Replace these placeholders:
   * `YOUR_CALL_LOG_SHEET_ID`: your call log sheet ID (all Google Sheets nodes)
   * `YOUR_TELEGRAM_CHAT_ID`: your team's chat ID (Telegram nodes, and the Redis session key in Chat Memory Manager)
   * `YOUR_BUSINESS_NAME`: your business name (the AI prompts)
6. Point your phone system's missed-call webhook at `https://YOUR_N8N_DOMAIN/webhook/missed-call` and send the fields below.
7. Activate the workflow.

## Webhook Payload

```
{
  "caller_name": "Ali Raza",
  "caller_phone": "+923111222333",
  "business": "Your Business Name",
  "missed_at": "2026-11-05T14:20:00+05:00"
}
```

`caller_phone` must start with `+` and contain 10 to 15 digits.

## Test

Send the same payload twice within an hour. The first run should post a draft to the team chat, and the second should be logged as `duplicate_skipped`.

## Known Limits

* The drafted text is posted to the team's Telegram chat. This workflow does not send SMS to the customer. To text the customer directly, add an SMS provider node (for example Twilio) after Generate Recovery SMS.
* Duplicate checking reads the whole call log sheet on each call. Move to a database if the log grows large.

## Project Notes

* Built as part of my AI Automation portfolio.
* The business-hours check runs in the workflow timezone, so set it correctly before going live.
* All AI nodes use `gpt-5-mini`. Duplicate and business-hours decisions are made in code, not by the model.
