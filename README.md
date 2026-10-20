# slick-ai-bot

An AI as a bot in Slick. One bot can answer with **Muse**, **OpenCode** and
**Claude** — one at a time (`tey muse`, `tey opencode`, `tey claude`), or
all three in one bot (`tey all`), where each conversation chooses which AI
answers it with `/model`.

It replaces three bots that were one per provider, and it no longer polls:
Slick tells it what happens as it happens (`/api/events`), and it looks at
one conversation at a time. When Slick cannot tell it — an older Slick, or
a network in the way — it falls back to looking, the way the old bots
always did.

## Running

```
SLICK_TOKEN=slick_bot_… tey all       # one bot, every AI
MUSE_TOKEN=slick_bot_…    tey muse    # a bot that asks Muse
OPENCODE_TOKEN=slick_bot_… tey opencode
CLAUDE_TOKEN=slick_bot_…  tey claude
```

`SLICK_TOKEN` works for every command; a command of one AI's name also
takes that AI's token (`MUSE_TOKEN` and so on), so the same token as its
old bot can be kept.

### Operators

An AI has no tools unless operators are named — the people whose answer to
"Permission for Bash: Allow | Deny" counts, and the only ones who may
choose the mode, the model, or hand a session over.

```
SLICK_OPERATORS=u1791…,u4823… tey all
```

`BOT_OPERATORS` reads the same, for a `.env` written for the old bots.

### Choosing what answers

In a conversation an operator writes:

- `/model` — pick an AI (muse, opencode, claude, or default), then a model.
  OpenCode's models are listed; Muse's and Claude's are typed.
- `/mode` — how tools are let through: *manual* (ask before each),
  *auto* (the AI judges), *plan* (change nothing).
- `/adopt <session id>` — hand a session begun at the command line to this
  conversation, so it goes on from where that one left off; `/adopt <ai>
  <session id>` hands it over and switches the AI too.

A conversation is a **whole channel** (or a group chat, or a private chat),
not a thread: everything said in one channel shares one session, so a
channel can be created for one piece of work, and a session adopted into it
serves every thread of it. A session belongs to one AI — a conversation that
switches AI keeps each AI's session apart, so no work carries from one
provider into another. Several bots — different AIs — can share a channel,
each with a session of its own, for a second opinion.

## How it reads Slick

On start, and whenever Slick says it has fallen too far behind to be told
what it missed, the bot takes one full look — every channel, group chat and
private chat it is in — and notes what is there without answering it. From
then on it is told, over Slick's `/api/events` WebSocket, what happens:

- a **message**, to a bot that may read the conversation it is in;
- an **answer** to a form, to the one bot whose form it was;
- a **command**, to the one bot it was used for.

Each telling is only *where* — the bot fetches the words through the API as
ever, which decides again what it may read. Every telling carries a cursor;
the bot brings its cursor on each connection and Slick replays what it
missed, or tells it to start again with a full look.

How much a bot reads of a channel is the channel's own setting in Slick —
*none*, *mentions*, or *all*:

- **mentions** — the bot is told only what names it and the threads it is
  in, and answers those (the safe default).
- **all** — the bot reads the whole channel and takes part naturally: it
  judges each message and stays quiet when it has nothing to add, rather
  than waiting to be named.
- **none** — the bot reads and hears nothing of it.

The bot's private chats are always its own to read, whatever a channel's
setting.

## Environment

`src/main.kex` documents every variable; the important ones:

```
SLICK_URL          where Slick is; http://127.0.0.1:4000 unless set
SLICK_TOKEN        the bot's token, from Slick's Bots dialog
SLICK_OPERATORS    who may allow a tool (see above)
SLICK_POLL_SECONDS how often Slick is looked at when it cannot tell the
                   bot what happens; every 2 seconds unless set
PROVIDERS          with `tey all`, which AIs the bot offers; all unless set
```

Each AI is configured as its old bot was — `MUSE_*`, `OPENCODE_*`,
`CLAUDE_*`, and `ANTHROPIC_API_KEY` for Claude over the API. A provider's
tools are its `*_TOOLS` (Bash, Write, Web), and an AI has tools at all only
when operators are named.

## Layout

```
src/main.kex            the entrypoint, the environment, and the stream
src/slick_ai/wire.kex   the one way out to the network (HTTP + WebSocket)
src/slick_ai/slick.kex  Slick's bot API, as much as the bot uses
src/slick_ai/mind.kex   which AI is asked, in one shape whatever it is
src/slick_ai/muse.kex   the local Muse
src/slick_ai/opencode.kex  the local OpenCode
src/slick_ai/claude.kex    Claude Code, or Anthropic's API
src/slick_ai/bot.kex    what the bot does with what it reads and is told
```

The bot is a plain Kex package (`package.kex`), and it is tested with
`tey test`.
