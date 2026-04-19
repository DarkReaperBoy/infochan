# infochan

A single-file Telegram AI chatbot that runs on **Cloudflare Workers** (free tier
works). One JS file, one D1 database, many AI providers — drop it in, fill in
keys, deploy.

## 📋 Before you click Deploy — get these ready

The Cloudflare deploy form is a flat list of variable names with no help text
(that's a Cloudflare limitation, not something this repo can fix). Open the
links below **in other tabs**, collect the values, then come back and paste.

**Required (bot won't boot without these):**

| Field in deploy form       | What to paste                                                                 | Where to get it |
| -------------------------- | ----------------------------------------------------------------------------- | --------------- |
| `TELEGRAM_TOKENS`          | One or more bot tokens, comma-separated. Format: `<id>:<secret>`.             | [@BotFather](https://t.me/BotFather) → `/newbot` |
| `ADMIN_USER_IDS`           | Your numeric Telegram user ID(s), comma-separated.                            | [@userinfobot](https://t.me/userinfobot) |
| `CLOUDFLARE_ACCOUNT_ID`    | 32-char hex. Same Cloudflare account you're deploying to.                     | [dash.cloudflare.com](https://dash.cloudflare.com/) → Workers & Pages → sidebar shows Account ID |
| `CLOUDFLARE_API_TOKEN`     | Token with **Workers AI: Read + Run**.                                        | [API Tokens → Create](https://dash.cloudflare.com/profile/api-tokens) → "Custom token" |
| At least one provider key  | See the provider table below.                                                 | Pick one: Gemini / Groq are free and easiest. |

**Conditional:**

| Field                      | Fill in only if…                                                              | Where to get it |
| -------------------------- | ----------------------------------------------------------------------------- | --------------- |
| `CLOUDFLARE_GATEWAY_ID`    | …you plan to use the `gemini` provider. Any slug works.                       | Dashboard → AI → AI Gateway → Create |
| `SERPER_API_KEY`           | …you want `/system → Web Search` enabled.                                     | [serper.dev](https://serper.dev/) (2,500 free queries) |

**Provider keys — fill in whichever you want (comma-separated for rotation), leave the rest blank:**

All 19 providers are prompted on the deploy form. You only need **one** working
key total. The ⭐ rows are the easiest free options to start with.

| Field                     | Format           | Signup / docs |
| ------------------------- | ---------------- | ------------- |
| ⭐ `GEMINI_KEYS`          | `AIza...`        | [aistudio.google.com/app/apikey](https://aistudio.google.com/app/apikey) — free |
| ⭐ `GROQ_KEYS`            | `gsk_...`        | [console.groq.com](https://console.groq.com/) — free |
| `OPENROUTER_KEYS`         | `sk-or-v1-...`   | [openrouter.ai/keys](https://openrouter.ai/keys) — free models available |
| `MISTRAL_KEYS`            | (plain)          | [console.mistral.ai](https://console.mistral.ai/api-keys) |
| `GITHUB_KEYS`             | `github_pat_...` | [Create PAT](https://github.com/settings/personal-access-tokens) — free for GitHub Models |
| `POE_KEYS`                | (plain)          | [poe.com/api_key](https://poe.com/api_key) — free daily points |
| `CEREBRAS_KEYS`           | `csk-...`        | [cloud.cerebras.ai](https://cloud.cerebras.ai/) |
| `NVIDIA_KEYS`             | `nvapi-...`      | [build.nvidia.com](https://build.nvidia.com/) |
| `KIVEST_KEYS`             | `sk-...`         | [ai.ezif.in](https://ai.ezif.in/) |
| `G4F_KEYS`                | `g4f_u_...`      | [g4f.space](https://g4f.space/) — free proxy |
| `G4F_OLLAMA_KEYS`         | `g4f_u_...`      | same — ollama endpoint |
| `G4F_GEMINI_KEYS`         | `g4f_u_...`      | same — gemini endpoint |
| `MNNAI_KEYS`              | `mnn-key-...`    | [mnnai.ru](https://mnnai.ru/) |
| `NAGA_KEYS`               | `ng-...`         | [naga.ac](https://naga.ac/) |
| `NAVY_KEYS`               | `sk-navy-...`    | [api.navy](https://api.navy/) |
| `POIXE_KEYS`              | `sk-...`         | [poixe.com](https://poixe.com/) — has free tier |
| `VERCEL_SHARE_YOURS_KEYS` | `vck_...`        | [vercel.com](https://vercel.com/) → AI Gateway → API Keys |
| `VOIDAI_KEYS`             | `sk-voidai-...`  | [voidai.app](https://voidai.app/) |
| `CHATANYWHERE_KEYS`       | `sk-...`         | [api.chatanywhere.tech](https://api.chatanywhere.tech/) |

**Fields you can ignore on the form:**

- **Project name** — anything, e.g. `infochan-bot`. Becomes part of your URL.
- **Select D1 database** → click **"Create new"**, name it anything (`infochan-db` is fine). Location hint: pick the region nearest you.
- **Build command / Deploy command** — leave empty. This is a single-file Worker, no build step.

---

## 🚀 Deploy

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/darkreaperboy/infochan) — *Cloudflare's form won't let fields stay empty. For **provider keys you won't use**, typing junk (`-`, `none`, `asdf`) is fine — you'll never select that provider in `/system`, so the bot never tries the bogus key. Do **not** put junk in required fields (Telegram token, admin ID, Cloudflare account/token) — those are actually read on every request.*

After the deploy finishes:

1. Note the Worker URL Cloudflare shows you (e.g. `https://infochan-bot.your-subdomain.workers.dev`).
2. Open `https://<that-url>/?action=init` **once** in a browser. You should see green "Active" / "Updated" badges.
3. Find your bot on Telegram and send `/new`.

Edit credentials anytime at **Dashboard → Workers & Pages → your-worker → Settings → Variables**.

> **Security:** deploy-form values are stored as plaintext env vars. For production, open a terminal once and run `wrangler secret put TELEGRAM_TOKENS` (etc.) to promote them to encrypted secrets. The code reads both identically.

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

## Adding or removing a provider

Providers live in the `CONFIG.PROVIDERS = [ ... ]` array near the top of
`main.js`. Each entry is a plain object — no classes to subclass, no registry
to touch. The `AIProvider` class at runtime reads `baseURL`, `keys`, and the
flags below to decide how to talk to the endpoint.

### Add a new OpenAI-compatible provider

Most providers (Groq, OpenRouter, Mistral, DeepSeek, Together, Fireworks,
Anyscale, local LM Studio / Ollama with `openai` mode, etc.) speak the OpenAI
chat-completions dialect. Drop in a block like this anywhere inside the
`PROVIDERS` array:

```js
/**
 * deepseek — reasoning + chat models, cheap.
 * How to get a key: https://platform.deepseek.com/api_keys
 */
{
  name: "deepseek",                        // shown in /system menu. lowercase, short.
  baseURL: "https://api.deepseek.com/v1",  // must be OpenAI-compatible (ends in /v1 usually)
  keys: ["YOUR_DEEPSEEK_API_KEY"],         // array — add more to enable auto-rotation
  defaultModel: "deepseek-chat",           // model id picked until user overrides in /system
  supportsImages: false,                   // true if the provider accepts image inputs
  useHardcodedModels: false,               // true = skip /v1/models fetch, use a fixed list
},
```

That's it. Redeploy and the new provider shows up in `/system → Providers`.

**To also prompt for its key on the Cloudflare deploy form**, add one line to
`wrangler.toml` `[vars]`:

```toml
DEEPSEEK_KEYS = ""   # comma-separated for key rotation
```

The hydration code in `main.js` auto-derives the env-var name from the provider
name: uppercase it, replace non-alphanumerics with `_`, append `_KEYS`. So
`deepseek` → `DEEPSEEK_KEYS`, `my cool ai` → `MY_COOL_AI_KEYS`. No code change
needed for hydration.

### Flags cheat-sheet

| Flag                 | When to set                                                              |
| -------------------- | ------------------------------------------------------------------------ |
| `supportsImages`     | Provider accepts image parts in chat messages (GPT-4o, Gemini, etc.)     |
| `useHardcodedModels` | `true` if `/v1/models` is missing or returns garbage. Then edit the hardcoded list in `AIProvider`. |
| `isGemini`           | Google AI Studio dialect (not OpenAI). Routed through CF AI Gateway.     |
| `isCloudflareAI`     | Cloudflare Workers AI (`@cf/...` model IDs). Uses account/token instead of a keys array the normal way. |
| `isCustom`           | Per-user baseURL + key set via `/system → Custom provider`. Don't reuse. |

For anything that isn't OpenAI-compatible or one of the three dialects above,
you'll need to add a new branch in the `AIProvider` class's `buildRequest` /
`parseResponse` methods. That's a bigger job — grep for `isGemini` to see the
pattern.

### Remove a provider

Two places:

1. **`main.js`** — delete the whole `{ name: "foo", ... }` block from
   `CONFIG.PROVIDERS`. If the provider had a docstring comment above it, delete
   that too.
2. **`wrangler.toml`** — delete its `FOO_KEYS = ""` line (if present).

Optional third step: **`README.md`** — drop its row from the provider table
above so the deploy-form docs don't lie.

Any user who had that provider selected in their `/system` settings will fall
back to the default provider on their next message. No database migration
needed; the per-user settings are re-read every request and unknown provider
names are ignored.

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

- `main.js` in this repo is the sanitized template (every credential is
  `YOUR_..._HERE`). If you paste real keys into it instead of using the
  deploy-form `[vars]` / `wrangler secret put`, **do not commit that copy
  back to a public repo.** Rotate any key that has been pushed.
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
