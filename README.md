# Telegram AI Assistant Bot

A production-ready, asynchronous Telegram AI chatbot built with **aiogram 3.x**, **OpenRouter**, and **SQLite** (via `aiosqlite`). Designed to run on **Render** with long polling.

## Features

- Natural chat — just send a message, no command required
- `/start`, `/help`, `/ai`, `/reset`, `/id`, `/about`
- Per-user conversation history (capped at `MAX_HISTORY` messages)
- Animated "thinking" status while the AI is generating a reply
- Telegram-safe HTML formatting (bold, italic, inline code, code blocks, links)
- Automatic splitting of long AI responses across multiple messages
- Protected owner identity (cannot be overridden by user prompts)
- Mandatory channel-join gate with live membership verification
- Per-user rate limiting
- Centralized custom/premium emoji mapping with safe Unicode fallback
- Robust error handling — no stack traces or secrets ever reach the user

---

## 1. Create your Telegram bot with BotFather

1. Open a chat with [@BotFather](https://t.me/BotFather) on Telegram.
2. Send `/newbot` and follow the prompts (choose a name and a unique username ending in `bot`).
3. BotFather will reply with your **bot token** — this is your `BOT_TOKEN`. Keep it secret.

## 2. Get your `OWNER_ID`

1. Message [@userinfobot](https://t.me/userinfobot) (or any similar "get my ID" bot) from the Telegram account that should be recognized as the owner.
2. It will reply with your numeric user ID — this is `OWNER_ID`.

This value is currently used for reference/configuration only (it does not unlock hidden commands), but keep it set for future use.

## 3. Create an OpenRouter API key

1. Sign up at [openrouter.ai](https://openrouter.ai).
2. Go to **Keys** in your dashboard and create a new API key.
3. Copy it — this is your `OPENROUTER_API_KEY`. Keep it secret.

## 4. Choose an OpenRouter model

1. Browse available models at [openrouter.ai/models](https://openrouter.ai/models).
2. Copy the model slug (for example `openai/gpt-4o-mini`, `anthropic/claude-3.5-haiku`, `meta-llama/llama-3.1-8b-instruct`).
3. Set it as `OPENROUTER_MODEL`. If you don't set one, the bot defaults to `openai/gpt-4o-mini`.

## 5. Set up the required channel and `REQUIRED_CHANNEL_ID`

The bot requires users to join this channel before using the AI:

```
https://t.me/+2Fxg6o4jEKAxOGQ1
```

To verify membership, the bot needs the channel's **numeric chat ID** (private invite links cannot be used directly as a `chat_id` in the Bot API):

1. Add your bot to the channel **as an administrator** (it needs permission to see the member list — "Add Members" or a general admin role is sufficient on most channel types).
2. Forward any message from the channel to [@userinfobot](https://t.me/userinfobot) or [@RawDataBot](https://t.me/RawDataBot) to reveal the channel's numeric ID. It will look like `-1001234567890`.
3. Set that value as `REQUIRED_CHANNEL_ID`.

If you ever change the invite link, update the `CHANNEL_INVITE_LINK` environment variable to match (it defaults to the link above).

## 6. Upload to GitHub

```bash
git init
git add .
git commit -m "Initial commit: Telegram AI assistant bot"
git branch -M main
git remote add origin https://github.com/<your-username>/<your-repo>.git
git push -u origin main
```

`.gitignore` already excludes `.env` and the SQLite database file, so secrets won't be committed.

## 7. Deploy on Render

1. Go to [render.com](https://render.com) → **New** → **Web Service** (or **Background Worker** — this bot uses long polling and doesn't need to accept HTTP traffic, so a Background Worker is the more accurate fit; a Web Service also works fine).
2. Connect your GitHub repository.
3. Configure:

   - **Environment:** Python 3
   - **Build Command:**
     ```
     pip install -r requirements.txt
     ```
   - **Start Command:**
     ```
     python main.py
     ```

4. Add the environment variables below under **Environment → Environment Variables**.
5. Deploy. Check the Render logs to confirm `Starting bot with model ...` appears with no errors.

### Render environment variables

| Variable | Required | Example | Notes |
|---|---|---|---|
| `BOT_TOKEN` | Yes | `123456:ABC-DEF...` | From BotFather |
| `OPENROUTER_API_KEY` | Yes | `sk-or-v1-...` | From OpenRouter |
| `OPENROUTER_MODEL` | No | `openai/gpt-4o-mini` | Defaults if unset |
| `REQUIRED_CHANNEL_ID` | Yes | `-1001234567890` | Numeric channel ID |
| `CHANNEL_INVITE_LINK` | No | `https://t.me/+2Fxg6o4jEKAxOGQ1` | Defaults to this exact link |
| `OWNER_ID` | No | `987654321` | Owner's numeric Telegram ID |
| `OWNER_USERNAME` | No | `X_NAGI7` | Defaults to `X_NAGI7` |
| `OWNER_NAME` | No | `MADARA UCHIHA` | Defaults to `MADARA UCHIHA` |
| `MAX_HISTORY` | No | `20` | Max stored messages per user |
| `RATE_LIMIT_SECONDS` | No | `2` | Minimum seconds between messages per user |
| `MAX_MESSAGE_LENGTH` | No | `4000` | Max characters accepted from a user |
| `DB_PATH` | No | `bot.db` | SQLite file path |

Never commit real values for these — set them only in Render's dashboard (or a local `.env` file, which is git-ignored).

---

## How mandatory channel verification works

1. Every command and every plain-text message first calls `ensure_access()`, which uses `bot.get_chat_member(REQUIRED_CHANNEL_ID, user_id)` — Telegram's official Bot API — to check the caller's status in the required channel.
2. A status of `member`, `administrator`, or `creator` is treated as "joined." Anything else (e.g. `left`, `kicked`) blocks access.
3. If the user hasn't joined, they see the **Access Restricted** screen with a **Join Channel** button (opens the invite link) and a **Verify** button.
4. Tapping **Verify** re-runs the exact same membership check. If it now passes, the user is marked verified in the database and shown the welcome screen. If not, they're told plainly that they still haven't joined — there is no fake or bypassable verification.
5. If Telegram returns a permission error (the bot isn't an admin in the channel, or the channel ID is wrong), users see a configuration-error screen instead of a silent failure, so an administrator knows to fix the bot's channel permissions.

## How the AI thinking animation works

1. When a user sends a message (and isn't asking a protected owner question), the bot immediately sends a status message with the first animation frame.
2. A background `asyncio` task then edits that same message on a fixed interval, cycling through a short sequence of frames (Thinking → Processing → Generating response → Almost ready), each using a custom emoji with a Unicode fallback.
3. The OpenRouter request runs concurrently. As soon as it completes (or fails), the animation task is cancelled — cleanly, with `asyncio.CancelledError` suppressed — so the loop never runs unbounded.
4. The same status message is then edited in place with the final, formatted AI response (or a clear error message if the request failed), avoiding extra message spam in the chat.

---

## Project structure

```
telegram-ai-bot/
├── main.py            # Complete bot application
├── requirements.txt
├── .python-version
├── .gitignore
└── README.md
```

## Local development

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env        # create this yourself, or export vars in your shell
python main.py
```

Example `.env` (do not commit real secrets):

```
BOT_TOKEN=your-telegram-bot-token
OPENROUTER_API_KEY=your-openrouter-key
OPENROUTER_MODEL=openai/gpt-4o-mini
REQUIRED_CHANNEL_ID=-1001234567890
OWNER_ID=your-numeric-telegram-id
OWNER_USERNAME=X_NAGI7
OWNER_NAME=MADARA UCHIHA
MAX_HISTORY=20
RATE_LIMIT_SECONDS=2
MAX_MESSAGE_LENGTH=4000
```
