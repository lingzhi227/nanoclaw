# ClawBot

Personal Claude assistant running securely in containers. Forked from [NanoClaw](https://github.com/qwibitai/nanoclaw) and extended with Telegram-first design.

## What's Different from Upstream

| Feature | Upstream NanoClaw | This Fork |
|---------|------------------|-----------|
| Primary channel | WhatsApp | **Telegram** |
| Activity streaming | No | **Real-time** (💭 thinking, 🔧 tools, ✅ results) |
| Forum Topics | No | **Auto-registered isolated sessions** |
| Agent model | Sonnet 4.6 | **Opus 4.6** |
| Container SSH | No | **openssh-client included** |

## Key Features

**Telegram-native.** Uses grammy for polling. Supports groups, private chats, and Forum Topics. Send `/chatid` anywhere to get the registration JID.

**Forum Topics = isolated conversations.** Each Telegram Forum Topic auto-registers as its own group with a fresh session and memory. Create a new topic to start a completely isolated conversation — no history from other topics.

**Real-time activity stream.** While the agent works, you see its thinking process and tool calls as they happen. No waiting in silence.

**Claude Opus 4.6.** The agent runs on Opus 4.6 via Claude Agent SDK (Claude Code).

**Container isolation.** Agents run in Linux containers (Apple Container on macOS). Each group has its own mounted filesystem. SSH client is available inside the container for remote access (e.g. HPC clusters).

## Architecture

```
Telegram (grammy) --> SQLite --> Polling loop --> Container (Claude Agent SDK) --> Response
                                                        |
                                              Real-time stream events
                                           (thinking, tools, results)
```

Single Node.js process. Agents execute in isolated Linux containers with mounted directories. Per-group message queue with concurrency control. IPC via filesystem.

Key files:
- `src/index.ts` — Orchestrator: state, message loop, agent invocation
- `src/channels/telegram.ts` — Telegram connection, forum topics, send/receive
- `src/channels/whatsapp.ts` — WhatsApp (optional, disabled in TELEGRAM_ONLY mode)
- `src/ipc.ts` — IPC watcher and task processing
- `src/router.ts` — Message formatting and outbound routing
- `src/group-queue.ts` — Per-group queue with global concurrency limit
- `src/container-runner.ts` — Spawns streaming agent containers
- `src/task-scheduler.ts` — Runs scheduled tasks
- `src/db.ts` — SQLite operations
- `groups/*/CLAUDE.md` — Per-group memory (isolated per container)

## Setup

```bash
git clone https://github.com/lingzhi227/nanoclaw.git
cd nanoclaw
npm install
npm run build
```

Configure `.env`:
```
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_ONLY=true
ANTHROPIC_API_KEY=your_api_key
ASSISTANT_NAME=Claw
```

Register your Telegram group: add the bot, send `/chatid` to get the JID, then register it via the main channel agent.

## Forum Topics Usage

1. Create a Telegram group → enable **Topics** → add bot as admin
2. Register the parent group once via the main channel
3. Create a new Topic → send any message → bot auto-registers it and responds
4. Each topic is a fully isolated session with its own memory

## Requirements

- macOS (Apple Container) or Linux (Docker)
- Node.js 20+
- [Claude Code](https://claude.ai/download)
- Telegram bot token (from [@BotFather](https://t.me/botfather))

## License

MIT
