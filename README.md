# infochan

A single-file Telegram AI chatbot that runs on **Cloudflare Workers** (free tier
works). One JS file, one D1 database, many AI providers — drop it in, fill in
keys, deploy.

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/darkreaperboy/infochan)

**One-click deploy.** The button above does everything for you:

1. Forks this repo into your GitHub
2. Creates the Worker on your Cloudflare account
3. Auto-creates the D1 database from `wrangler.toml`
4. **Prompts you for every credential on a web form** — Telegram token,
   admin user IDs, at least one provider key — no terminal, no editing code
5. Deploys

After deploy, open `https://<your-worker>.workers.dev/?action=init` once in a
browser to register the Telegram webhook. Done.

You can edit the credentials anytime at: **Cloudflare Dashboard → Workers &
Pages → your-worker → Settings → Variables**.

> **Security note:** deploy-button vars are plaintext. For production, open a
> terminal once and run `wrangler secret put TELEGRAM_TOKENS` (etc.) to promote
> them to encrypted secrets. The code reads both identically.

---

## What it does

- **Multi-provider LLM chat** with automatic key rotation across 19 providers
  (Cloudflare Workers AI, Gemini, Groq, OpenRouter, GitHub Models, Mistral, Poe,
  Cerebras, NVIDIA NIM, Vercel AI Gateway, and more). Per-user provider/model
  selection via an inline settings menu.
- **Streaming replies** that live-edit the Telegram message as the model types
  (three modes: `off` / `edit` / `draft`).
- **Personas** (7 built-in) and **jailbreak / system-prompt presets** (7) and
  **prefills** (6). All per-user.
- **Character cards** — import SillyTavern / Chub.ai character JSON via
  `/addchar`. Embedded lorebooks are parsed and injected with a keyword trigger
  system (2000-token budget by default).
- **Image generation** via Cloudflare Workers AI (FLUX, SDXL, etc.).
- **Text-to-speech** via Cloudflare (MeloTTS, Deepgram Aura) — sends the reply
  as a Telegram voice message.
- **Speech-to-text** — send the bot a voice note, it transcribes and replies.
- **Vision** — attach an image to a message; supported providers read it
  directly.
- **Web search** via Serper (Google results), togglable per user.
- **Webhook deduplication** (in-memory + D1 fallback) — Telegram's retries won't
  produce duplicate replies.

---

## Repository layout

```
infochan/
├── main.js          # the Worker. READ THE TOP OF THIS FILE FIRST — full setup guide lives there
├── wrangler.toml    # Cloudflare Workers config; paste your D1 database_id here
├── .gitignore       # keeps local secrets / editor junk out of the repo
└── README.md        # you are here
```

`main.js` is the file you deploy. It has a ~200-line setup guide at the
very top and per-credential docstrings throughout `CONFIG`. If anything in this
README is ambiguous, **the docstrings in `main.js` are canonical.**

---

## Quick start

Full, beginner-grade instructions live at the top of `main.js`. In brief:

1. **Telegram**: talk to `@BotFather` → `/newbot` → save the token. Then
   `@userinfobot` → save your numeric user ID.
2. **Cloudflare**: make a free account at https://dash.cloudflare.com/. Note
   your Account ID. Create an API token with `Workers AI: Read + Run`.
3. **Tools**: install Node.js 20+, then `npm install -g wrangler` and
   `wrangler login`.
4. **D1 database**: `wrangler d1 create infochan-bot` → copy the printed
   `database_id` into `wrangler.toml`.
5. **Credentials**: open `main.js`, replace every `YOUR_..._HERE` with
   real values. Each has a docstring above it explaining where to get it.
   Minimum: `TELEGRAM_TOKENS`, `ADMIN_USER_IDS`, `CLOUDFLARE_ACCOUNT_ID`,
   `CLOUDFLARE_API_TOKEN`, and at least one provider key in `PROVIDERS[]`.
6. **Deploy**: `wrangler deploy` — prints your bot URL.
7. **Initialize**: open `https://<your-worker>.workers.dev/?action=init` once
   in a browser. You'll see green "Active" / "Updated" badges.
8. **Chat**: find your bot on Telegram, send `/system` or `/new`.

---

## Commands

| Command    | What it does                                            |
| ---------- | ------------------------------------------------------- |
| `/system`  | Settings panel: provider, persona, jailbreak, prefill, stream mode, TTS/STT, web search, image gen |
| `/new`     | Start a fresh conversation (clears history)             |
| `/redo`    | Regenerate the last response                            |
| `/del`     | Delete the last Q&A pair from history                   |
| `/img`     | Generate an image from a prompt                         |
| `/addchar` | Import a character card (Chub.ai URL or JSON paste)     |
| `/setname` | Set the name the bot calls you in roleplay              |
| `/history` | View the current conversation history                   |

---

## Admin endpoints

| URL              | Purpose                                                         |
| ---------------- | --------------------------------------------------------------- |
| `/`              | HTML dashboard: active streams, provider count, admin count     |
| `/?action=init`  | Create D1 tables, register webhook, push command menu. Run once per deployment or whenever `TELEGRAM_TOKENS` changes. |
| `/?action=wipe`  | **DESTRUCTIVE.** Truncates every table. No auth check — keep the URL private. |
| `/webhook/<tok>` | Telegram webhook target (not called directly by humans)         |

---

## How the code is structured

Everything is in `main.js`. The notable pieces:

- `CONFIG` — all tunable settings at the top of the file (providers, personas,
  jailbreaks, prefills, TTS/STT, feature flags).
- `class TelegramAPI` — thin wrapper over the Bot API (sendMessage,
  editMessageText, setWebhook, setMyCommands, sendVoice, sendPhoto…).
- `class AIProvider` — unified LLM client with branches for OpenAI-compatible,
  Gemini (routed through Cloudflare AI Gateway), and Cloudflare Workers AI.
- `class D1Store` — persistence layer. Seven tables: `user_settings`,
  `user_history`, `character_sessions`, `user_characters`, `key_health`,
  `update_dedup`, `locks`.
- `class RequestLimiter` — caps concurrent in-flight requests (default 10).
- `export default { fetch }` — router at the bottom of the file.

---

## Limits worth knowing

- **Cloudflare free plan**: 50 outbound subrequests per Worker invocation.
  `CONFIG.SETTINGS.maxSubrequests` is set to 45 to stay under this. If you hit
  the ceiling mid-reply, shorten the conversation (`/new`) or upgrade to the
  paid plan.
- **D1 free plan**: 5 GB storage, 5M reads/day, 100k writes/day. Plenty for a
  personal bot.
- **History cap**: `CONFIG.SETTINGS.maxHistoryLength = 50` messages. Older
  messages are dropped. Raise at your own risk (token usage scales linearly).
- **Gemini routing**: Gemini calls go through Cloudflare AI Gateway. You must
  set up a gateway (Dashboard → AI → AI Gateway) and paste its slug into
  `CONFIG.CLOUDFLARE_GATEWAY_ID`, or Gemini requests will 404. Other providers
  are unaffected.

---

## Security notes

- `main.js` is the sanitized template. `main.js` is your working copy
  with real keys — **do not commit it to a public repo.** Add it to
  `.gitignore`.
- `?action=wipe` has no authentication. Anyone who guesses your Worker URL
  can wipe your DB. Either keep the URL secret, or add an admin check in the
  router before making this repo public.
- API keys in the file are plaintext. For stronger isolation, use
  `wrangler secret put NAME` and read them from `env.*` inside the `fetch`
  handler. The provided code does not currently do this — patch as needed.

---

## Troubleshooting

The full list lives at the bottom of the setup guide in `main.js`.
Top three:

- **"D1 database binding not found"** — `wrangler.toml` binding name must be
  exactly `d1_db`. Code reads `env.d1_db`.
- **Webhook "Failed" on `/?action=init`** — bot token has a typo. Check for
  trailing whitespace.
- **"Too many subrequests"** — you hit the CF free-plan cap of 50. Start a
  `/new` conversation or upgrade.

---

## License

Personal project, no license attached — treat as "all rights reserved" unless
the author adds one. If you fork it, rotate the keys first.
