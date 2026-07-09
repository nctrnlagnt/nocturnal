# Discord Integration

## What

An optional plugin that bridges Discord to Nocturnal agents. Once configured,
your agent responds to messages from whitelisted Discord users — conversations
happen in Discord, powered by the same agent backend you use in the TUI.

## How it works

The Discord plugin runs as a daemon alongside nocturnal. It connects to Discord
using a bot token, listens for messages from allowed users, and forwards them
to the agent. Responses stream back to Discord.

Only whitelisted users can interact with the bot. Messages from anyone else are
ignored.

## Configuration

### Enable the plugin

```
/plugins discord enable
```

### Set the bot token

```
/discord bot_token <token>
```

Or set the environment variable:

```
NOCTURNAL_DISCORD_BOT_TOKEN=<token>
```

### Whitelist users

```
/discord allowed_users <discord_user_id_1> <discord_user_id_2>
```

Or edit the config file directly at `~/.nocturnal/discord.yaml`:

```yaml
discord:
  whitelisted_users:
    - "123456789"
    - "987654321"
```

### Config file

All Discord settings are stored in `~/.nocturnal/discord.yaml`. You can edit this
file directly if you prefer.

## Getting a bot token


Prerequisites: A personal Discord account and a private server (recommended).

### 1. Create Discord Bot
1. Go to https://discord.com/developers/applications.
2. New Application → Name it → Create.
3. Bot tab → Add Bot → Copy token (store securely).
4. Enable Privileged Gateway Intents: Message Content, Server Members, Presence.
5. Installation tab → Set **Install Link** to **None** (fixes "Private application cannot have a default authorization link" error).

### 2. Invite Bot
- OAuth2 → URL Generator.
- Scopes: `bot` + `applications.commands`.
- Permissions: Send Messages, Read Message History, Use Slash Commands.
- Copy URL, open in browser, add to your private server.

### 3. Get Your Discord User ID
- Discord Settings → Advanced → Enable Developer Mode.
- Right-click your name → Copy User ID.

Test in DM or server channel.

If the bot connects but doesn't respond to messages, check that both Privileged
Intents are enabled. Discord requires these for bots that read message content.

## Bot Commands

The bot responds to text commands in any channel where it has permissions.
Commands can be prefixed with `/`, `!`, or used without a prefix.

| Command | Description |
| ------- | ----------- |
| `/start` | Display welcome message and available commands |
| `/help` | Show command reference |
| `/status` | Show current session ID for the channel/thread |
| `/clear` | Clear conversation history and start a fresh subsession |

## Session Tracking

Each Discord channel or thread maintains its own independent conversation session:

- **DMs and regular channels**: Session ID is `main:discord:{channel_id}`
- **Threads**: Session ID is `main:discord:{channel_id}:{thread_id}`

This means:
- Conversations in different channels are completely separate
- Each thread within a channel has its own conversation context
- Switching between threads preserves each thread's conversation history

## Streaming Responses

Agent responses stream to Discord in real-time:

1. **First chunk**: Sends a new message
2. **Subsequent chunks**: Edits the message in-place as content arrives
3. **Long responses**: When a message exceeds 1900 characters, a new message
   is sent and editing continues on that new message (Discord's limit is 2000)

This creates a smooth, live-updating experience where you can watch the agent's
response appear incrementally.

## User Configuration

### Per-User Settings

You can configure different agents, providers, or models for specific Discord
users by editing `~/.nocturnal/discord.yaml`:

```yaml
discord:
  bot_token: "your-token-here"
  whitelisted_users:
    - "123456789"
  default_agent: "coder"
  default_provider: "openai"
  default_model: "gpt-4"

users:
  "123456789":
    agent: "analyst"
    provider: "anthropic"
    model: "claude-3-opus"
```

### Authorization

- **Empty whitelist**: All users can interact with the bot
- **Non-empty whitelist**: Only listed user IDs can use the bot
- Unauthorized users receive a rejection message

## Features

### Typing Indicator

The bot shows a "typing..." indicator in Discord while the agent is processing
your message.

### Media Attachments

When you send attachments or embeds, the bot includes metadata in the message:
- Attachments: `[Attachment: filename=..., id=..., size=...]`
- Embeds: `[Embed]`

### Message Format

Messages sent to the agent include sender information:
```
from: discord:123456789 (username)
---
Your message content here
```
