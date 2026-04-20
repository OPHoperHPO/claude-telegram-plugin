# Telegram

Connect a Telegram bot to your Claude Code with an MCP server.

The MCP server logs into Telegram as a bot and provides tools to Claude to reply, react, or edit messages. When you message the bot, the server forwards the message to your Claude Code session.

## Prerequisites

- [Bun](https://bun.sh) — the MCP server runs on Bun. Install with `curl -fsSL https://bun.sh/install | bash`.

## Quick Setup
> Default pairing flow for a single-user DM bot. See [ACCESS.md](./ACCESS.md) for groups and multi-user setups.

**1. Create a bot with BotFather.**

Open a chat with [@BotFather](https://t.me/BotFather) on Telegram and send `/newbot`. BotFather asks for two things:

- **Name** — the display name shown in chat headers (anything, can contain spaces)
- **Username** — a unique handle ending in `bot` (e.g. `my_assistant_bot`). This becomes your bot's link: `t.me/my_assistant_bot`.

BotFather replies with a token that looks like `123456789:AAHfiqksKZ8...` — that's the whole token, copy it including the leading number and colon.

**2. Install the plugin.**

These are Claude Code commands — run `claude` to start a session first.

Install the plugin:
```
/plugin install telegram@telegram-plugin
/reload-plugins
```

**3. Give the server the token.**

```
/telegram:configure 123456789:AAHfiqksKZ8...
```

Writes `TELEGRAM_BOT_TOKEN=...` to `~/.claude/channels/telegram/.env`. You can also write that file by hand, or set the variable in your shell environment — shell takes precedence.

> To run multiple bots on one machine (different tokens, separate allowlists), point `TELEGRAM_STATE_DIR` at a different directory per instance.

**4. Relaunch with the channel flag.**

The server won't connect without this — exit your session and start a new one:

```sh
claude --channels plugin:telegram@telegram-plugin
```

**5. Pair.**

With Claude Code running from the previous step, DM your bot on Telegram — it replies with a 6-character pairing code. If the bot doesn't respond, make sure your session is running with `--channels`. In your Claude Code session:

```
/telegram:access pair <code>
```

Your next DM reaches the assistant.

> Unlike Discord, there's no server invite step — Telegram bots accept DMs immediately. Pairing handles the user-ID lookup so you never touch numeric IDs.

**6. Lock it down.**

Pairing is for capturing IDs. Once you're in, switch to `allowlist` so strangers don't get pairing-code replies. Ask Claude to do it, or `/telegram:access policy allowlist` directly.

## Access control

See **[ACCESS.md](./ACCESS.md)** for DM policies, groups, mention detection, delivery config, skill commands, and the `access.json` schema.

Quick reference: IDs are **numeric user IDs** (get yours from [@userinfobot](https://t.me/userinfobot)). Default policy is `pairing`. `ackReaction` only accepts Telegram's fixed emoji whitelist.

## Tools exposed to the assistant

| Tool | Purpose |
| --- | --- |
| `reply` | Send to a chat. Takes `chat_id` + `text`; optional `reply_to` (message ID for native threading), `message_thread_id` (forum topic), `files` (absolute paths — images ship as photos, other types as documents, 50MB cap), `format` (`text` / `markdownv2` / `html`). Auto-chunks long text. Returns sent message IDs. |
| `react` | Add an emoji reaction to a message by ID. **Only Telegram's fixed whitelist** is accepted (👍 👎 ❤ 🔥 👀 etc). |
| `edit_message` | Edit a message the bot previously sent. Supports `format` (`text` / `markdownv2` / `html`). Edits don't trigger push notifications. |
| `pin_message` | Pin a message (e.g. final result). Requires pin permissions in the target chat. Optional `disable_notification`. |
| `download_attachment` | Fetch a file attachment by `file_id` (from inbound `attachment_file_id` meta) into the local inbox. Capped at 20MB by the Bot API. |

Inbound messages trigger a typing indicator automatically — Telegram shows
"botname is typing…" while the assistant works on a response.

## Photos & albums

Inbound photos land in `~/.claude/channels/telegram/inbox/` and the local path
rides on the `<channel>` notification as `image_path`. Albums (multiple photos
in one send) are debounced and delivered as a single notification with all
paths in `image_paths` (comma-separated). Telegram compresses photos — for the
original bytes, send as a document (long-press → Send as File).

## Voice transcription (optional)

If `GEMINI_API_KEY` is set, inbound voice notes, audio files, and video notes
are transcribed via Gemini. The transcript replaces the `(voice message)`
placeholder and meta carries `transcribed: "true"`. Leave the env var unset to
skip transcription — payload passes through as a plain placeholder.

| Variable | Purpose |
| --- | --- |
| `GEMINI_API_KEY` | Enables transcription. Unset = disabled, no external calls. |
| `GEMINI_BASE_URL` | Optional custom endpoint (e.g. a proxy or compatible gateway). |
| `GEMINI_MODEL` | Model override. Default `gemini-3-flash-preview`. |

Put these in `~/.claude/channels/telegram/.env` alongside `TELEGRAM_BOT_TOKEN`,
or in the shell environment (shell wins). Files ≥20MB are skipped (Bot API
download cap).

## Inbound context

Meta that may arrive on the `<channel>` block:

- `message_thread_id` — forum topic ID; pass back on `reply` to stay in-topic.
- `reply_to_message_id`, `reply_to_user`, `reply_to_text` — sender quoted an earlier message.
- `quote_text` — partial text quote (Telegram's quote-reply feature).
- `forward_source`, `forward_from` — message was forwarded from there.
- `image_path` / `image_paths` — local paths for photos.
- `attachment_file_id` + `attachment_kind` / `attachment_mime` / `attachment_name` / `attachment_size` — non-photo attachments. Call `download_attachment` to fetch.
- `transcribed: "true"` — content already contains the Gemini transcript.

## No history or search

Telegram's Bot API exposes **neither** message history nor search. The bot
only sees messages as they arrive. If the assistant needs earlier context, it
will ask you to paste or summarize. Photos are downloaded eagerly on arrival
since there's no way to fetch them later; non-photo attachments stay
addressable via `download_attachment` for as long as Telegram keeps the
`file_id` valid (typically hours to days).
