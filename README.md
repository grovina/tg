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
# chat registry — GROUP backs `group`/`team`; UPPERCASE aliases resolve by name
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
tg send <to> <text|--file F>     [--topic name|--topic-id N] [--html] [--dry-run]
tg reply <msg_id> <text|--file F>  answer IN the originating chat (resolved from history) + thread it [--html]
tg sendvoice <to> <text>         ElevenLabs TTS → opus → voice note
tg senddoc <to> <path>           file attachment   [--caption] [--topic …]
tg sendphoto <to> <path>         inline image      [--caption] [--html] [--topic …]
tg sendpoll <q> <opt…> --to X    poll, 2-10 options [--quiz N] [--multi] [--public] [--topic …]
tg pollstatus [msg_id]           who voted each option + which options nobody picked [--poll-id ID] (needs --public)
tg react <msg_id> [emoji]        empty emoji clears; chat resolved from history [--to override]
tg reactions <to>                list the emoji this chat actually allows
tg delete <msg_id>               deleteMessage + synthetic 'deleted' history row; chat from history [--to override]
tg media                         idempotent backfill: download media + Scribe-transcribe
tg ids                           print chat ids seen in history
tg show <msg_id>                 full untruncated text of one inbound message (+ media / voice transcript)
tg pending                       chats/threads with inbound newer than your last reply or ack
tg ack <to> [--topic-id N]       mark a chat/thread SEEN (no reply needed) so it drops from `pending`
tg topics [list|set <name> <id>|rm <name>|resolve <name>]
```

**Target chat — recipient first, always explicit.** Every send command takes the
chat as its **required leading positional** (`to`): `group`/`team` for the group,
a `tg.conf` alias (`ALICE`), or a numeric id. `tg send GROUP "hi"`, `tg send alice
"hi"` — same shape for `sendvoice`, `senddoc`, `sendphoto`. Two rules earn their
keep:

- **Recipient first.** Like every messaging tool (`/msg nick text`, IRC
  `PRIVMSG <target> <text>`), the addressee leads and the free-form message
  trails — so a broken shell quote in the message can only swallow tokens *after*
  it, and the target sits safely in front, already parsed.
- **No group default.** Omitting the target is a **hard error**, never a silent
  broadcast. An absent target is almost always one a broken quote or unset `$VAR`
  ate; defaulting *that* to the group is how a private DM once leaked to the whole
  staff group. So `tg send "hi"` is refused (name the target); the group is reached
  only by naming it. An empty (`""`) or unknown target is likewise a hard error
  that lists your defined aliases.

`sendpoll` is the one exception to *positional*: its options are variadic, so the
target comes from `--to` (also required). `--to CHAT` is accepted everywhere as an
order-independent alternative to the positional.

**Long or quote-heavy text — use `--file`.** `send` and `reply` accept
`--file PATH` instead of inline text: write the message to a file and pass the
path. This sidesteps the shell-quoting trap where a stray `'` or `"` in the
message closes the shell string mid-sentence. With recipient-first order the
target is unambiguous: `tg send alice --file msg.txt`.

**`react` / `delete` need no target.** They're keyed by the message id and resolve
the originating chat from history (like `reply`): `tg react 156 👍`, `tg delete
156`. Pass `--to <chat>` only to override (or if the message predates your
history).

**Replying — never infer the target.** `watch` and `pending` name each inbound
channel by **identity and carry its chat_id**, so two same-type chats can't be
confused: `MSG [BACKOFFICE -5470784074] Jerome #156: …`. To answer, copy a token
straight off that line — `tg reply 156 "<text>"` (by the `#<id>` after the
sender; resolves the chat + forum thread from history and threads the reply in
place) or `tg send -5470784074 "<text>"` (by the chat_id). Don't route a reply
by a type word like `group` — that's how a Backoffice `group` and an OPS
`supergroup`, both once rendered `[group]`, got crossed.

**Reading a clipped message — `show`.** The live `watch` stream (and the Monitor
that surfaces it) renders one line per message and can clip a long one. That same
`#<id>` you'd reply to also fetches the whole thing: `tg show 156` prints the
full untruncated text — plus sender, chat, thread, date, and any media type or
voice transcript — straight from `history.jsonl`, so there's no hand-grepping the
log for the rest of a message. An edited message shows its current text, flagged
`(edited)`.

**Polls:** `tg sendpoll "Lunch?" Pizza Sushi Salad --to group` posts a regular,
**anonymous** poll to the group. `--multi` allows multiple answers; `--public`
makes votes non-anonymous; `--quiz N` turns it into a quiz with the 0-based
`N`-th option as the correct one. `--topic`/`--topic-id` target a forum thread
like the other send commands.

**Who voted — `pollstatus`.** The poller persists every vote (`poll_answer`) on a
poll the bot sent, so `tg pollstatus <msg_id>` tallies, per option, **who** picked
it and **which options nobody picked** — a checklist (`--multi --public`) whose
unticked items are the tasks still undone at end of service. The latest answer per
voter wins (a vote change replaces it; clearing all options retracts it). This
needs **`--public`**: Telegram delivers individual voters only for non-anonymous
polls — an anonymous poll yields no voters, and `pollstatus` says so.

**Reactions are a fixed set.** Telegram only lets a bot react with emoji from a
global allowed list, which a group's admins can narrow further — you can't react
with arbitrary emoji. `tg reactions <to>` prints what the target chat actually
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
