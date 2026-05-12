# Bouy -- CSI Fall Detection

A real-time fall detection platform built at HackDavis. Bouy uses Wi-Fi Channel State Information (CSI) to passively detect falls without cameras or wearables, then alerts caregivers through a live dashboard and WhatsApp messages.

## How it works

Four ESP32 boards continuously measure how Wi-Fi signals change as people move through a room. A trained ensemble model running on a CSI receiver board classifies these changes in real time. When a fall is detected, the system pages the person at risk first, then notifies their care circle if there is no response, and escalates further if no caregiver acknowledges.

## Repository layout

```
apps/
  server/       Express + SQLite backend (deployed on Railway)
  web/          Next.js PWA frontend (deployed on Vercel)
fall-detection-training/
  firmware/     ESP32 firmware for TX and RX boards
  collection/   Session recording scripts
  labeling/     Label splitting tools
  training/     LSTM and CNN training scripts
  evaluation/   LOOCV and ensemble evaluation
  model/        Trained TorchScript ensemble weights and config
FontendInspo/   Design reference components
```

## Apps

### Frontend (apps/web)

Two installable PWAs built with Next.js 16 App Router:

**Bouy Care** -- the recipient-facing app (`/household/[code]/at-risk`)
- Shows monitoring status (CSI sensor or on-device motion)
- Full-screen alarm with countdown when a fall is detected
- Responds with "I'm OK" or "I need help"
- Can request a monitoring pause, which requires caregiver approval

**Bouy Dashboard** -- the caregiver-facing app (`/household/[code]/contact`)
- Real-time sensor status and floor plan
- Receives alerts only after the at-risk user's response window closes
- "I'm Responding" acknowledges the alert and sends a WhatsApp to the recipient
- Monitoring pause approval and resume controls
- Full incident history

### Backend (apps/server)

Express server with SQLite (better-sqlite3).

Key pieces:
- `src/state-machine.ts` -- incident lifecycle with in-process timers (T1 user response window, T2 contact ack window, T3 before 911 prompt)
- `src/routes/incidents.ts` -- fall detection intake, user respond, contact ack/resolve, 911 trigger
- `src/routes/households.ts` -- household data, caregiver management, monitoring pause flow
- `src/routes/sensors.ts` -- telemetry ingestion from CSI board
- `src/routes/devices.ts` -- per-device state and staleness check

Real-time updates use Pusher Channels. WhatsApp alerts use Twilio.

## Incident state machine

```
DETECTED
  -> AWAITING_USER_RESPONSE   (T1 starts, SMS sent to recipient)
  -> CONTACTS_NOTIFIED        (T1 expired or user requests help, SMS to caregivers)
  -> CONTACT_RESPONDING       (caregiver acknowledges)
  -> RESOLVED                 (user or caregiver confirms OK)

CONTACTS_NOTIFIED
  -> ESCALATION_AVAILABLE     (T2 expired, no caregiver responded)
  -> ESCALATED                (caregiver triggers 911)
```

## Fall detection model

The shipped model lives in `fall-detection-training/model/fall_impact_seq9_ensemble/`. It is a TorchScript ensemble of a CNN spectrogram encoder, an LSTM, and a Transformer encoder trained on 7 recorded sessions using leave-one-session-out cross-validation.

- Input: 9 stacked 6-second windows (14-second receptive field)
- Band-spectrogram tensor shape: `[1, 9, 32, 49, 21]` (9 windows x 32 channels x 49 freq bins x 21 time steps)
- Classes: binary -- `FALL_IMPACT` vs everything else
- Window-level macro-F1: 0.81, FALL_IMPACT recall: 91% at threshold 0.50
- Event-level F1: 0.90 at threshold 0.50 (90% precision, 90% recall on held-out session)

See `fall-detection-training/README.md` for the full training pipeline.

## Setup

### Prerequisites

- Node.js 20+
- Python 3.12+
- A Pusher Channels app
- A Twilio account with WhatsApp messaging enabled

### Environment variables

Copy `.env.example` to `.env` and fill in the values:

```
T1_MS=30000
T2_MS=45000
T3_MS=60000

PUSHER_APP_ID=
PUSHER_KEY=
PUSHER_SECRET=
PUSHER_CLUSTER=us2
NEXT_PUBLIC_PUSHER_KEY=
NEXT_PUBLIC_PUSHER_CLUSTER=us2

TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_FROM=
TWILIO_MODE=whatsapp

BASE_URL=http://localhost:3000
PORT=3001
```

### Run locally

```bash
# Install root dependencies
npm install

# Start the backend
cd apps/server
npm install
npm run dev

# Start the frontend (separate terminal)
cd apps/web
npm install
npm run dev
```

The dashboard is at `http://localhost:3000/household/483920/contact`.
The recipient app is at `http://localhost:3000/household/483920/at-risk`.

### Deploy

The backend is configured for Railway (`apps/server/railway.toml`).
The frontend is configured for Vercel (`apps/web`).

Set all environment variables in each platform's dashboard. The backend needs `BASE_URL` set to its public Railway URL so SMS links resolve correctly.

## Hardware

- 1 ESP32 TX board (transmitter, raised above shoulder height)
- 4 ESP32 RX boards (receivers placed around the room)
- Wi-Fi channel 6, 921600 baud USB serial
- 192 subcarriers per packet at approximately 70 Hz

Flash instructions and firmware source are in `fall-detection-training/firmware/`.

## License

MIT
