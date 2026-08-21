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
state   <repo>/var/telegram/   history.jsonl · offset · poller.pid · media/<chat_id>/ · topics.json
config  <repo>/tg.conf         GROUP=… + UPPERCASE chat aliases + TG_TRANSCRIBE_* (KEY=VALUE)
        env / .env             secrets: TG_BOT_TOKEN · TG_ELEVENLABS_KEY (optional)
```

**A message is `(chat_id, message_id)`.** A Telegram message id is unique only
*within its chat*, so `#1746` in the group and `#1746` in a DM are two unrelated
messages. Everything here is keyed on the pair, and refs are written
`<chat>:<id>` — `GROUP:1746`, `-1004412570945:1746`. That's what every surface
prints, so a ref copied out of `watch` or `pending` stays correct days later. A
bare `1746` still works and is **resolved**: unique ⇒ fine; ambiguous ⇒ writes
(`reply`/`react`/`delete`) refuse and list the candidates, while `show` prints
them all. See [Message refs](#message-refs--chat-id-msg-id).

**Voice handling.** Incoming audio is **always downloaded** to
`var/telegram/media/<chat_id>/`. Transcription (ElevenLabs **Scribe**) is layered
on top **only when `TG_ELEVENLABS_KEY` is set** — otherwise the bytes are kept and
the message still surfaces, just untranscribed. Language **auto-detects** per
message unless `TG_TRANSCRIBE_LANG` forces one — globally, or per chat as
`TG_TRANSCRIBE_LANG_<ALIAS>` — so a mixed de/en/pt-BR group transcribes correctly
while a one-language DM can be pinned. Transcripts keep the engine's own language
+ confidence and are **flagged, never dropped**, when the detected language or
writing system can't be right for the chat. TTS (`sendvoice`) is optional and only
needs `TG_ELEVENLABS_VOICE_ID`.

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
TG_TRANSCRIBE_LANG_ALICE=fr            # per-chat override — beats auto-detect on short clips
TG_TRANSCRIBE_EXPECT=fr,pt,en          # what this repo can plausibly be; flags anything else
TG_WATCHED_CHATS=-1001234567890        # restrict `watch` to these chats
TG_ELEVENLABS_VOICE_ID=...             # only for sendvoice (TTS)
TG_GATE=tools/my-gate                  # veto every outbound message (see below)
TG_GATE_TIMEOUT=30                     # seconds the gate gets to answer
```

**Secrets** stay out of `tg.conf` (which is non-secret config — chat ids, voice
id, lang — and safe to commit):
- bot token: `TG_BOT_TOKEN` (env) or `var/telegram/token` (gitignored)
- ElevenLabs: `TG_ELEVENLABS_KEY` (env) — only needed for transcription/TTS.

How those env vars get set is the caller's business — a shell `export`, a
`.env`, or whatever your orchestrator injects. `tg` only reads them.

Config is read from `tg.conf` then `.env`, with environment variables winning.
The one exception is `TG_GATE`: the environment can *arm* a gate but never
*disarm* the one `tg.conf` declares, because `TG_GATE= tg send …` would be a
bypass of exactly the kind the gate exists to close.

## The outbound gate (optional)

Set `TG_GATE` to an executable and **nothing outbound reaches Telegram until it
exits 0** — `send`, `reply`, `senddoc`, `sendphoto`, `sendvoice`, `sendpoll`, and
any `send*` added later. The check sits in the HTTP layer, not in the command
parsers, so there is one door and no shell syntax leads around it.

That last property is the whole point. The usual place to mount such a guard is
an agent harness's pre-tool hook, which inspects the *shell command* — and so is
defeated, silently, by `bash -c "tg send …"`, by `T=tg; $T send …`, and by a
heredoc feeding a compound command. Silently is the part that hurts: an ungated
send leaves nothing behind, so you cannot even count what you lost. A hook is
still a fine *pre-filter* (it sees intent earlier and can explain itself better);
it just cannot be the guarantee. This can.

**Contract.** The gate is run with no shell, cwd at the repo root, and receives
one JSON object on stdin:

```json
{"schema": 1, "iso": "2026-08-22T…", "method": "sendMessage", "command": "send",
 "argv": ["send", "GROUP", "…"], "chat": {"id": "-1001234567890", "alias": "GROUP"},
 "text": "the message as a human reads it", "thread_id": null,
 "reply_to_message_id": null, "files": [], "fields": {…verbatim API fields…},
 "root": "/path/to/repo"}
```

Exit `0` sends. Any non-zero exit refuses, and whatever the gate printed on
stdout becomes the refusal message (`tg` then exits **6**). The gate's stderr is
always forwarded, so it can warn without refusing.

**It fails closed.** A `TG_GATE` that is missing, not executable, crashes, or
exceeds `TG_GATE_TIMEOUT` refuses the send (exit **7**). A guard that cannot run
must never read as a guard that passed — that equivalence is how gates die
unnoticed. `tg doctor` prints whether the gate is armed, for the same reason.

**No recursion, no bypass.** The gate runs with `TG_GATE_DEPTH=1`; a send
attempted from inside a gate decision is refused rather than exempted, so the
variable is useless as an escape hatch — setting it can only block you. Reading
(`show`, `pending`, `ids`) is never gated.

**Side effect worth more than the veto:** if your gate journals its decisions, it
becomes the authoritative send log — *"a message with no journal entry"* goes
from undetectable to impossible.

## Commands

```
tg poll                          long-poll daemon (pid-guarded singleton, idempotent)
tg doctor                        (re)start the poller detached if dead
tg watch                         doctor + watchdog + follow history.jsonl → Monitor stream
tg send <to> <text|--file F>     [--topic name|--topic-id N] [--html] [--dry-run]
tg reply <ref> <text|--file F>   answer IN the originating chat (resolved from the ref) + thread it [--html]
tg sendvoice <to> <text>         ElevenLabs TTS → opus → voice note
tg senddoc <to> <path>           file attachment   [--caption] [--topic …]
tg sendphoto <to> <path>         inline image      [--caption] [--html] [--topic …]
tg sendpoll <q> <opt…> --to X    poll, 2-10 options [--quiz N] [--multi] [--public] [--topic …]
tg pollstatus [ref]              who voted each option + which options nobody picked [--poll-id ID] (needs --public)
tg react <ref> [emoji]           empty emoji clears; chat from the ref/history [--to override]
tg reactions <to>                list the emoji this chat actually allows
tg delete <ref>                  deleteMessage + synthetic 'deleted' history row [--to override]
tg media                         idempotent backfill: download media + Scribe-transcribe
tg migrate [--dry-run]           move legacy media/<id>.<ext> → media/<chat_id>/<id>.<ext>
tg ids                           print chat ids seen in history
tg show <ref>                    full untruncated text of one inbound message (+ media / voice transcript)
tg pending                       chats/threads with inbound newer than your last reply or ack
tg ack <to> [--topic-id N]       mark a chat/thread SEEN (no reply needed) so it drops from `pending`
                                 [--all-threads] clears every open thread in the chat
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

**`react` / `delete` need no target.** They're keyed by the message ref and
resolve the originating chat from it (like `reply`): `tg react BACKOFFICE:156 👍`,
`tg delete BACKOFFICE:156`. Pass `--to <chat>` only to override (or if the message
predates your history).

**Replying — never infer the target.** `watch` and `pending` name each inbound
channel by **identity and carry its chat_id**, so two same-type chats can't be
confused: `MSG [BACKOFFICE -5470784074] Jerome #BACKOFFICE:156: …`. To answer,
copy a token straight off that line — `tg reply BACKOFFICE:156 "<text>"` (by the
ref; resolves the chat + forum thread from history and threads the reply in
place) or `tg send -5470784074 "<text>"` (by the chat_id). Don't route a reply
by a type word like `group` — that's how a Backoffice `group` and an OPS
`supergroup`, both once rendered `[group]`, got crossed.

## Message refs — `<chat>:<msg_id>`

A Telegram `message_id` is unique **per chat**, not globally. `#1746` in the staff
group and `#1746` in a DM are different messages that happen to share a number,
and in a busy repo that is common, not exotic — one shop had 71 of 401 ids
appearing in more than one chat.

So a message is identified by the **pair**, written `<chat>:<msg_id>`:

```bash
tg show GROUP:1746            # alias-qualified (preferred — short and stable)
tg show -1004412570945:1746   # id-qualified
tg show 1746                  # bare: resolved against history
```

`thread_id` is deliberately *not* part of the identity — a forum thread is an
attribute of a message, not a namespace for its id. (Threads *are* the scope for
`pending`/`ack`, which is why `ack` is per-thread.)

**A bare id is resolved, never guessed.** Unique ⇒ it just works. Ambiguous:

* `reply` / `react` / `delete` / `pollstatus` **refuse** and list the candidates.
  A write into the wrong chat can't be taken back, so this fails closed. "Pick the
  newest and mention it in the output" is not an option: the caller is usually a
  model, and a caveat printed next to plausible-looking content gets skimmed past.
* `show` prints **every** message carrying the id, each under its own header with
  its own transcript. Reading is safe to disambiguate by showing more.

**Media follows the same key:** `var/telegram/media/<chat_id>/<msg_id>.<ext>`. A
chat *directory* rather than a `<chat>_<msg>` filename prefix, because chat ids are
negative and a file named `-1004412570945_1746.ogg` starts with a dash — every
bare `cat`/`rm` of it would parse as flags.

Repos created before this layout keep working; run **`tg migrate`** (with
`--dry-run` first) to move existing files into it. `tg doctor` warns while any
remain. Migration also lists the messages whose media was lost to a past collision
— their `file_id`s are still in history, so `tg media` re-fetches them.

**Reading a clipped message — `show`.** The live `watch` stream (and the Monitor
that surfaces it) renders one line per message and can clip a long one. That same
ref you'd reply to also fetches the whole thing: `tg show BACKOFFICE:156` prints the
full untruncated text — plus sender, chat, thread, date, and any media type or
voice transcript — straight from `history.jsonl`, so there's no hand-grepping the
log for the rest of a message. An edited message shows its current text, flagged
`(edited)`. Given a bare id that several chats share, it prints them all, clearly
separated, each with its own transcript attached — a transcript is never rendered
under another message's header.

**Reconciling — `pending` and `ack`.** `pending` lists every (chat, thread) whose
newest inbound is newer than your newest reply **or** ack there. A thread is its
own scope, so `tg ack <chat>` clears the chat's main timeline and *not* its
threads — correct, but easy to read as a broken tool, so an ack that leaves
anything open in the same chat now names it and the command that finishes it.
`--all-threads` does the whole chat at once.

Note what counts as a "thread". In a supergroup, replying to a message makes
Telegram open a message thread rooted on it, so `thread_id` is often a **reply
chain**, not a forum topic — an active chat can therefore accumulate many small
pending scopes, each needing its own ack. That is deliberate (a reply chain is a
conversation you may still owe an answer to), and `--all-threads` is the bulk
clear when it isn't.

A reply sent *into* a thread does clear it: Telegram returns `message_thread_id`
on the sent message and the outbound row records it, so the watermark lands in the
right scope. If a thread ever looks stuck despite a reply, check that first —
it means the outbound row landed in the `(chat, None)` scope instead.

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
