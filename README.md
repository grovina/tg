# tg

A global Telegram bot CLI — like `gh` or `gcloud`: **one binary, repo-local state.**

It consolidates a bash + python Telegram toolkit that had been copy-pasted (and
drifted) across several repos into a single stdlib-only Python CLI. Transport is
the Telegram **Bot API getUpdates long-poll** — the only transport any of them
used.

The code is shared and global; the **state and config are per-repo**, resolved
from whatever repo you're standing in (walks up from cwd to a `tg.conf`,
`var/telegram/`, or `.git` marker — same as `git`).

```
state   <repo>/var/telegram/   history.jsonl · offset · poller.pid · media/ · topics.json
config  <repo>/tg.conf         GROUP=… + UPPERCASE chat aliases + TG_TRANSCRIBE_LANG (KEY=VALUE)
        env / .env             secrets: TG_BOT_TOKEN · TG_ELEVENLABS_KEY (optional)
```

**Voice handling.** Incoming audio is **always downloaded** to `var/telegram/media/`.
Transcription (ElevenLabs **Scribe**) is layered on top **only when
`TG_ELEVENLABS_KEY` is set** — otherwise the bytes are kept and the message
still surfaces, just untranscribed. Language **auto-detects** per message unless
`TG_TRANSCRIBE_LANG` forces one, so a mixed de/en/pt-BR group transcribes
correctly. TTS (`sendvoice`) is optional and only needs `TG_ELEVENLABS_VOICE_ID`.

`history.jsonl` is an append-only in+out log with a stable schema — safe to
`tail` and parse.

## Install

One file, no dependencies — symlink it onto your `PATH`:

```bash
ln -s "$PWD/tg" /usr/local/bin/tg     # or any dir on your PATH
```

Stdlib only (Python 3.11+). `sendvoice` (TTS) additionally needs `ffmpeg`.

## Config (`tg.conf` at repo root)

```ini
# chat registry — GROUP is the default `send` target; aliases resolve by name
GROUP=-1001234567890
ALICE=10000001
# behavior (all optional)
TG_TRANSCRIBE_LANG=pt                  # omit for auto-detect (multilingual groups)
TG_WATCHED_CHATS=-1001234567890        # restrict `watch` to these chats
TG_ELEVENLABS_VOICE_ID=...             # only for sendvoice (TTS)
```

**Secrets** stay out of `tg.conf` (which is non-secret config — chat ids, voice
id, lang — and safe to commit):
- bot token: `TG_BOT_TOKEN` (env) or `var/telegram/token` (gitignored)
- ElevenLabs: `TG_ELEVENLABS_KEY` (env) — only needed for transcription/TTS.

How those env vars get set is the caller's business — a shell `export`, a
`.env`, or whatever your orchestrator injects. `tg` only reads them.

Config is read from `tg.conf` then `.env`, with environment variables winning.

## Commands

```
tg poll                          long-poll daemon (pid-guarded singleton, idempotent)
tg doctor                        (re)start the poller detached if dead
tg watch                         doctor + watchdog + follow history.jsonl → Monitor stream
tg send <text> [to]              [--topic name|--topic-id N] [--html] [--dry-run]
tg sendvoice <text> [to]         ElevenLabs TTS → opus → voice note
tg senddoc <path> [to]           file attachment   [--caption] [--topic …]
tg sendphoto <path> [to]         inline image      [--caption] [--html] [--topic …]
tg react <msg_id> [emoji] [to]   empty emoji clears the reaction
tg reactions [to]                list the emoji this chat actually allows
tg delete <msg_id> [to]          deleteMessage + synthetic 'deleted' history row
tg media                         idempotent backfill: download media + Scribe-transcribe
tg ids                           print chat ids seen in history
tg pending                       chats/threads with inbound newer than your last reply (reconcile gap)
tg topics [list|set <name> <id>|rm <name>|resolve <name>]
```

**Target chat — one rule everywhere.** Every outbound command takes the chat as
an **optional trailing positional** (`to`): a `tg.conf` alias (`ALICE`) or a
numeric id; **omit it for the group** (`GROUP`). So `tg send "hi"` goes to the
group and `tg send "hi" alice` to Alice — same shape for `react`, `senddoc`,
`sendphoto`, `delete`. (`--to` still works as a legacy alias, but the positional
is canonical.)

**Reactions are a fixed set.** Telegram only lets a bot react with emoji from a
global allowed list, which a group's admins can narrow further — you can't react
with arbitrary emoji. `tg reactions [to]` prints what the target chat actually
permits (via `getChat`), so pick from that list; a rejected `react` reports
Telegram's reason.

**Images:** `sendphoto` posts an *inline* image (with optional `--caption`);
`senddoc` sends the same file as a *download attachment*. Pick by how you want it
to appear.

The poller is a singleton: the `poller.pid` owner wins, so two `tg poll`
processes (or a host one racing a box one) never both consume `getUpdates`
(which Telegram would 409). `getFile`/media downloads are plain HTTPS and never
collide with the long-poll consumer.

## Running the poller continuously

`tg poll` is a foreground singleton; `tg doctor` (re)starts it detached. To keep
it alive under a supervisor, just run `tg poll` as your service command — it's
idempotent and pid-guarded, so a restart never double-consumes `getUpdates`.

## Not (yet) included

- **Per-client logs** — partitioning `history.jsonl` by chat (e.g.
  `clients/<chat_id>/`), toggled by a `storage=per-client` config knob.
- **Webhook transport** — only the getUpdates long-poll is implemented.
