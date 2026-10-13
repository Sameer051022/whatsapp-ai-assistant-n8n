# WhatsApp AI Assistant for Salons (n8n)

An AI receptionist that runs on the official WhatsApp Cloud API. It answers customer questions about services and prices, understands voice notes and photos, and books, reschedules and cancels appointments in Google Calendar and Google Sheets. When a person is needed, it hands the chat over to the salon owner. Built in n8n with OpenAI, PostgreSQL and Google Workspace.

Built for salons, spas, clinics and other appointment-based businesses, especially in Pakistan, where customers mix English, Urdu and Roman Urdu.

## What it does

- **Answers on WhatsApp 24/7**, in English for English messages and in Roman Urdu for Urdu or Roman Urdu messages
- **Prices only from the salon's Google Sheet**, so it never invents a service or a price
- **Voice notes**: transcribes Urdu and English voice notes and replies to them
- **Photos**: looks at hair, skin, nail or reference photos (mehndi, hairstyle, makeup), suggests the matching services with prices, and flags medical concerns
- **Bookings**: checks real availability in Google Calendar (staff capacity and buffer time) and books only after the customer confirms
- **Reschedule and cancel**: customers can manage only their own bookings; Calendar and Sheet update automatically
- **Owner alerts by Gmail** for every booking, reschedule, cancellation and handoff
- **Human handoff**: the AI pauses for that customer, and the owner gets the summary and can reply through the bot
- **Owner commands** sent from the owner's WhatsApp: `#status`, `#on`, `#off`, `#reply`, `#help`
- **Multi-client**: one workflow serves many businesses through a Master Config sheet

## How it works

```
Customer (WhatsApp)
      │
      ▼
WhatsApp Cloud API ──► n8n Webhook ──► Parse + Dedupe (Postgres)
                                            │
                                  Client lookup (Master Config sheet)
                                            │
                      ┌─────────────────────┼──────────────────────┐
                      ▼                     ▼                      ▼
              Owner #command         Paused chat (handoff)     Normal chat
              (status/on/off/reply)  → email to owner          │
                                                               ▼
                                              Text / Voice (transcribe) / Photo (vision)
                                                               │
                                                  7s debounce (merge rapid messages)
                                                               │
                                       AI Agent (OpenAI + Postgres chat memory)
                                       + fresh menu from Google Sheet
                                                               │
                         Tools: check_availability · book_appointment · find_bookings
                                cancel_booking · reschedule_booking · transfer_to_human
                                                               │
                                       Google Calendar · Google Sheets · Gmail
                                                               │
                                                               ▼
                                                    WhatsApp reply to customer
```

1. Meta sends each incoming message to the n8n webhook. Duplicate deliveries are dropped with a Postgres check.
2. The `phone_number_id` identifies the business, and its settings load from the Master Config sheet.
3. Owner `#commands` and paused (handed-off) chats are routed away from the AI.
4. Voice notes are transcribed and photos are analysed, then everything goes through a short debounce so several quick messages get one reply.
5. The AI Agent gets the fresh service menu and the reply language, then answers or calls its tools.
6. The booking tools check slots against Calendar events, write to Calendar and Sheets, and email the owner.

## Multi-client by design

- One n8n workflow serves every client. Each client is one row in the **Master Config** sheet: number ID, business name, hours, working days, staff capacity, buffer, owner email and default language.
- Each client has its own Google Sheet (Services, Packages, Bookings) and Google Calendar.
- To onboard a new salon, copy the sheet template, add a Config row and connect their WhatsApp number.

## Tech stack

| Layer | Tool |
|---|---|
| Messaging | WhatsApp Cloud API (Meta Graph API) |
| Automation | n8n (self-hosted, Docker) |
| AI | OpenAI GPT-4o-mini (agent + vision), gpt-4o-mini-transcribe (voice) |
| Memory & state | PostgreSQL (chat memory, dedupe, debounce buffer, handoff flags) |
| Data | Google Sheets (config, services, packages, bookings) |
| Scheduling | Google Calendar |
| Alerts | Gmail |
| Tunnel (dev) | ngrok static domain |

## Setup

1. Run n8n and PostgreSQL in Docker, then create the `wa_app` database and its tables (`processed_messages`, `message_buffer`, `handoff_flags`).
2. In Meta for Developers, create an app with the WhatsApp product, generate a permanent System User token, and note the Phone Number ID and WABA ID.
3. In n8n, create the credentials: WhatsApp (Header Auth bearer token), Postgres, Google Sheets, Google Calendar, Gmail and OpenAI.
4. Import the workflow `.json` from this repo and assign the credentials to the nodes.
5. Create the Master Config sheet and the client sheet (Services, Packages, Bookings tabs) and a Google Calendar. Put their IDs in the Config row.
6. Publish the workflow and set the Meta webhook to `https://<your-domain>/webhook/wa-assistant`, subscribed to `messages`.
7. Send a WhatsApp message to the business number and test text, voice, photo, booking, cancel and `#status`.

> Keep tokens, API keys and sheet IDs out of the repository. The exported workflow references n8n credentials by name only.

## Author

**Sameer Faisal**, AI Developer & Automation Specialist (Lahore, Pakistan)

- LinkedIn: [linkedin.com/in/sameer-faisal-](https://www.linkedin.com/in/sameer-faisal-/)
- GitHub: [github.com/Sameer051022](https://github.com/Sameer051022)
- Portfolio: [sameer-faisal-portfolio.netlify.app](https://sameer-faisal-portfolio.netlify.app/)

Available for n8n automation, AI chatbot and WhatsApp assistant projects.
