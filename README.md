Done by: https://github.com/OmarBr-5 
# Trading Bot

A Telegram-driven trading automation stack with a browser dashboard.
This is a personal side project built for practical Telegram-to-MT5 automation and operations control.

It listens to configured Telegram channels, parses trading signals, decides whether a signal is executable, and sends one or more orders to MetaTrader 5. The web dashboard gives you operator controls for bot lifecycle, auth prompts, channel routing, logs, and history.

## What This Bot Does 

### The idea
You receive trade calls in Telegram channels.
This bot watches only the channels you approve, reads incoming messages, extracts trade details, and can execute those trades in MT5.

### What happens when a message arrives
1. A message appears in a configured Telegram channel.
2. The bot parses it using the parser version assigned to that channel.
3. If the message is a valid trade signal, the bot builds MT5 order requests.
4. Depending on mode:
- Immediate ON: execute at market price.
- Immediate OFF: use range/pending-order logic.
5. The bot submits orders to MT5 (including multi-TP split logic).
6. Everything is logged to the dashboard and persisted to SQLite history.

### What you control from the dashboard
- Start/stop the bot process.
- Toggle immediate execution mode.
- Manually execute the latest valid parsed signal.
- Add/edit/remove monitored Telegram channels.
- Enable/disable individual channels while bot is running.
- Submit Telegram auth code / 2FA password when requested.
- Review runtime logs and trade history.
- Update runtime Telegram/MT5 connection settings.

## Architecture Overview

- `backend/`: FastAPI API + Telegram listener runtime + MT5 execution + storage.
- `frontend/`: React + TypeScript operator dashboard (single-page app).
- `tradingbot.db`: SQLite file for processed messages, bot events, channel metadata, and trade history.

## Backend (Technical)

### Main runtime pieces
- `backend/app/main.py`
- Loads environment, configures logging, creates `BotManager`, serves FastAPI on `127.0.0.1:8080`.
- Auto-starts listener on API startup only when required startup settings + at least one channel are configured.

- `backend/app/services/bot_manager.py`
- Spawns `telegram_listener.py` as a child process.
- Streams stdout into structured in-memory log events (also persisted to SQLite when storage is available).
- Handles commands sent to listener stdin:
- `execute`
- `immediate_on` / `immediate_off`
- `channel_on <id>` / `channel_off <id>`
- `exit`
- Detects auth prompts via sentinel lines:
- `<<<TELEGRAM_CODE_REQUIRED>>>`
- `<<<TELEGRAM_PASSWORD_REQUIRED>>>`

- `backend/app/services/telegram_listener.py`
- Connects to Telegram (Telethon) using persisted session (`fresh_session`).
- Registers per-channel handlers using assigned parser functions.
- Parses message -> validates executability -> builds and sends MT5 requests.
- Persists processed message + parsed signal + MT5 response + execution mode snapshot.
- Supports stdin command listener for runtime toggles and graceful shutdown.

- `backend/app/services/parsers.py`
- Includes multiple parser versions (`parse_trade_signal`, `parse_trade_signal_v2`, `parse_trade_signal_v3`).
- Handles flexible vs stricter templates.
- Detects informational/non-trade messages and marks incomplete strict signals.

- `backend/app/services/mt5_executor.py`
- Initializes/logs into MT5 terminal.
- Resolves broker symbol variants.
- Chooses market vs pending order type from side/range/price context.
- Supports optional dynamic risk lot sizing using SL distance and equity.
- Splits total volume across TP targets.
- Retries invalid MT5 filling mode with IOC/FOK/RETURN fallback sequence.

### Backend API surface
Defined in `backend/app/api/routes.py`.

- `GET /api/state`: bot status, PID, auth waiting mode, last exit code.
- `GET /api/channels`: list channel config.
- `POST /api/channels`: add channel (requires bot stopped).
- `PUT /api/channels/{channel_id}`: edit channel (requires bot stopped).
- `DELETE /api/channels/{channel_id}`: delete channel (requires bot stopped).
- `POST /api/bot/channels/enabled`: enable/disable a configured channel (hot-toggle while running).
- `GET /api/settings`: runtime settings + masked fields.
- `PUT /api/settings`: update settings (requires bot stopped).
- `POST /api/bot/start`: start listener subprocess.
- `POST /api/bot/stop`: stop listener subprocess.
- `POST /api/bot/execute`: execute last parsed signal.
- `POST /api/bot/immediate-execution`: toggle immediate mode.
- `POST /api/bot/auth/code`: submit Telegram login code.
- `POST /api/bot/auth/password`: submit Telegram 2FA password.
- `GET /api/logs`: polling logs (`limit`, optional `since_id`).
- `GET /api/trades/history`: recent trade history from SQLite.

### Settings and startup gating
Configured through settings service/config:
- Required for startup:
- `TELEGRAM_API_ID`
- `TELEGRAM_API_HASH`
- `TELEGRAM_PHONE`
- `MT5_PATH`
- `MT5_LOGIN`
- `MT5_PASSWORD`
- `MT5_SERVER`
- Optional/defaulted:
- `SQLITE_PATH` (defaults to `tradingbot.db`)

### Reliability and operations notes
- Telegram reconnect loop for transient network failures.
- SQLite lock retries around critical Telethon DB calls.
- Scheduled weekly cleanup of persisted bot events (Sunday 18:00 Asia/Beirut).
- Rotating backend logs configurable by env.

## Frontend (Technical)

### App structure
- `frontend/src/features/dashboard/components/DashboardPage.tsx`: composition root.
- `frontend/src/features/dashboard/hooks/useBotDashboard.ts`: dashboard state machine + polling.
- `frontend/src/shared/api/client.ts`: typed API client for backend endpoints.

### How frontend behaves
- Bootstraps state via parallel requests (`state`, `channels`, `settings`, `logs`).
- Polls every ~1.5s for:
- latest bot state
- incremental log events (`since_id`)
- Tracks stream connectivity status based on polling success/failure.
- Runs actions with optimistic busy-state + toast feedback.
- Opens auth dialog when backend reports `awaiting_auth_type`.

### Main UI blocks
- `StatusRail`: bot status, stream health, log count, settings launcher.
- `BotControls`: start, stop, execute, immediate execution toggle.
- `ChannelList`: CRUD + enable/disable for monitored channels.
- `LogConsole`: live log feed and clear action.
- `SettingsPanel`: Telegram/MT5 credential and terminal config updates.
- `AuthDialog`: code/password submission when auth is required.

## Local Development

### Backend
```bash
cd backend
pip install -r requirements.txt
python app/main.py
```

### Frontend
```bash
cd frontend
npm install
npm run dev
```

### One-click Windows launcher
```bash
start_dev.bat
```

## VPS Deployment (How It Is Served)

This project is typically served from a VPS with:
- Backend API running as a long-lived process (`python backend/app/main.py`) on port `8080`.
- Frontend built as static assets (`frontend/dist`) and served by a web server (usually Nginx).
- Reverse proxy routing:
- `/api/*` -> FastAPI backend (`127.0.0.1:8080`)
- `/` -> frontend static build

### Recommended production flow
1. Build frontend:
```bash
cd frontend
npm ci
npm run build
```
2. Start backend with process manager (`systemd`, `pm2`, or NSSM on Windows VPS) so it auto-restarts.
3. Put Nginx (or IIS/Caddy) in front to serve `frontend/dist` and proxy API requests.
4. Keep `backend/.env`, session files, and SQLite database on persistent disk/volume.

### Important VPS note
- MT5 integration requires a VPS environment where the MetaTrader 5 terminal is installed and accessible to the backend process. In practice, this is usually a Windows VPS.

## Testing

### Backend tests
```bash
cd backend
pip install -r requirements-dev.txt
pytest
```

### Frontend build validation
```bash
cd frontend
npm run build
```

## Data and Runtime Artifacts

Common runtime files you will see in `backend/` (or configured path):
- `.env`
- `fresh_session.session`
- `telegram_logs.txt`
- `crash_log.txt`
- SQLite DB (default `tradingbot.db` at repo root unless overridden)

## Security Notes

- This project handles real credentials (Telegram + MT5). Keep `.env` and DB files private.
- Do not commit secrets, session files, or local runtime artifacts.
- Treat immediate execution mode carefully in live accounts.

## Known Constraints

- MT5 execution depends on local terminal availability and broker symbol compatibility.
- Parser quality depends on signal format consistency per channel.
- Channel configuration edits are intentionally blocked while bot is running.
