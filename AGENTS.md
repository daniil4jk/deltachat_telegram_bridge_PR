# AGENTS.md — Delta Chat ↔ Telegram Bridge

## Architecture

Single-file bot (`bot.py`, ~4750 lines) plus `database.py` (SQLite wrapper). No tests, lint, or typecheck config. CI is a push-only webhook that notifies a changelog service — no test runs.

- **Delta Chat**: `deltabot-cli` (`BotCli`) runs in a background thread via `loop.run_in_executor`
- **Telegram**: `python-telegram-bot` `Application` with polling
- **Userbot (optional)**: Telethon client for bridging channels without admin permissions

## Key commands

```bash
# Run
python bot.py serve
docker compose run --rm bridge python bot.py serve   # Docker

# Init (all persistent in bridge.db)
python bot.py init dc <email> [password]       # DC account
python bot.py init tg [token]                  # TG token (interactive if omitted)
python bot.py init admin_tg <id>               # TG admin for error logs
python bot.py init admin_dc <email>            # DC admin
python bot.py init api_id <id>                 # Userbot MTProto API ID
python bot.py init api_hash <hash>             # Userbot MTProto API hash
python bot.py init userbot                     # Interactive userbot sign-in
python bot.py init transport <uri|addr>        # Add backup mail relay
python bot.py init transport <addr> <password>

# Running bot commands (Docker)
docker compose exec bridge python bot.py link
docker compose exec bridge python bot.py list
docker compose exec bridge python bot.py config
docker compose exec bridge python bot.py --account ID remove
docker compose exec bridge python bot.py admin
```

## Docker

```bash
docker compose up -d                          # Start
docker compose up -d --build                  # Rebuild & update
docker compose down                           # Stop
docker compose logs -f                        # Live logs
docker compose run --rm bridge python bot.py init ...   # One-off commands
```

The Dockerfile uses `python:3.11-slim` and builds with `HOST_UID`/`HOST_GID` from `.env`. Volumes: `bridge.db`, `userbot_session.session`, `~/.config/tgbridge`.

## Security

- `bridge.db` stores the TG bot token **in plaintext** (protect with `chmod 600`)
- `userbot_session.session` is a Telethon session file — a "master key" to the user's TG account. Never share.
- Setting an `admin_tg` or `admin_dc` locks management commands to owner-only ("Private mode")
- Each bridge also stores `created_by_tg_id` — sub-admins only see/edit their own bridges

## Key behaviors

| Behavior | Detail |
|---|---|
| Rate limits | 30 msg/min per chat, 60 msg/min global DC, 5 deletions/60s |
| Edit debounce | 60s debounce per (chat, msg_id) to suppress link-preview edits |
| Double-bridge dedup | In-memory `_processed_tg_msgs` dict (120s TTL) |
| History relay | Last 3 posts relayed on bridge creation & new member join (5min cooldown) |
| Group→supergroup migration | Auto-detected via `migrate_to_chat_id`, all DB refs updated |
| Transient error suppression | Specific network errors filtered from admin notifications |
| DC→TG deletion sync | Skipped for channel broadcasts; exempts bot-initiated deletes (edit replacements) |
| Userbot watchdog | Health check every 60s, auto-restarts on failure |

## Database (`bridge.db`)

Auto-created on import. Schema has migrations via `ALTER TABLE ADD COLUMN` with try/except. Tables: `bridges`, `config`, `message_map`, `polls`, `channels`, `admins`, `transport_stats`. Old message map rows pruned to 10K on init; periodic cleanup every hour.

## Notable constraints

- TG bot must have **Group Privacy disabled** in BotFather to read group messages
- For reaction bridging (TG→DC), bot must be group **administrator**
- Videos >20MB use lower-resolution fallback from Telegram's `qualities`
- Media >50MB are rejected from Userbot downloads
- `set_admin.py` is a standalone CLI script for managing DC admin credentials
- Requires Python 3.9+
