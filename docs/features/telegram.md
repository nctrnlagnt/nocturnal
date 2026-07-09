# Telegram Integration

## What

An optional plugin that bridges Telegram chats to Nocturnal agents. Telegram
users can send messages to a configured bot and receive agent responses
directly in their Telegram conversation.

## How it works

The plugin runs a Telegram bot that listens for incoming messages. When a
whitelisted user sends a message, the plugin creates or resumes a session on
the Nocturnal server and forwards the message. The agent's response is sent
back to the Telegram chat.

Each Telegram user gets their own session. User-specific settings (agent,
provider, model) can be configured independently.

## Setup

### 1. Enable the plugin

```
/plugins telegram enable
```

### 2. Create a Telegram bot

1. Open Telegram and start a chat with [@BotFather](https://t.me/BotFather)
2. Send `/newbot` and follow the prompts
3. Copy the bot token you receive

### 3. Configure the bot token

You can set the bot token in two ways:

**Via the config file** (`~/.nocturnal/telegram.yaml`):

```yaml
telegram:
  bot_token: "123456789:ABCdefGHIjklMNOpqrSTUvwxYZ"
```

**Or via environment variable:**

```
NOCTURNAL_TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrSTUvwxYZ
```

### 4. Whitelist users

Restrict access to specific Telegram users by ID or username:

```yaml
telegram:
  whitelisted_users:
    - "123456789"    # by user ID
    - "username"     # by username (with or without @)
```

Leave `whitelisted_users` empty to allow all users.

### 5. Per-user settings (optional)

Configure different agents, providers, or models per user:

```yaml
users:
  "123456789":
    agent: "coder"
    provider: "openai"
    model: "gpt-4"
  "987654321":
    agent: "reviewer"
    provider: "anthropic"
    model: "claude-3"
```

## Configuration

All Telegram settings are stored in `~/.nocturnal/telegram.yaml`. The plugin
binary lives at `~/.nocturnal/plugins/telegram/`.

### Management commands

| Command | Description |
|---------|-------------|
| `/plugins list` | Show all plugins with status |
| `/plugins telegram enable` | Enable and start the Telegram plugin |
| `/plugins telegram disable` | Stop and disable the Telegram plugin |
| `/plugins telegram status` | Detailed Telegram plugin info |

### Config reference

```yaml
telegram:
  bot_token: "YOUR_BOT_TOKEN_HERE"      # or set NOCTURNAL_TELEGRAM_BOT_TOKEN env var
  whitelisted_users:                     # restrict access by user ID or username
    - "123456789"
  default_agent: "coder"                 # fallback for users without overrides
  default_provider: "openai"
  default_model: "gpt-4"

users:                                    # optional per-user overrides
  "123456789":
    agent: "coder"
    provider: "openai"
    model: "gpt-4"
```

## Advanced Features

### Thread/Topic Support (Forum Groups)

The bot supports Telegram's forum (topics) feature in supergroups. When added to a
forum-enabled supergroup, each topic maintains its own isolated conversation
session. This allows multiple independent conversations within the same group.

- **Forum detection**: Automatically detects forum-enabled supergroups
- **Session isolation**: Each topic gets its own session (`main:telegram:{chat_id}:{thread_id}`)
- **Reply threading**: Responses are posted to the correct topic thread

In regular (non-forum) groups, all messages share a single session.

### Media Support

The bot can receive and process media messages:

| Media Type | Format Sent to Agent |
|------------|---------------------|
| Photos | `[Photo: file_id=..., width=..., height=...]` (largest variant) |
| Documents | `[Document: filename=..., file_id=..., size=...]` |
| Voice messages | `[Voice: duration=...s, file_id=...]` |

Media captions are preserved and appended to the media description.

### Message Formatting

Agent responses are formatted using Telegram's MarkdownV2 syntax:

- Special characters are automatically escaped: `_ * [ ] ( ) ~ \` > # + - = | { } . ! \`
- If MarkdownV2 parsing fails, the bot falls back to plain text
- Long messages are automatically truncated to fit Telegram's 4000-byte limit

### Streaming Responses

Agent responses stream in real-time:

- Messages are edited in-place as content arrives
- When a message exceeds 4000 characters, a new message is sent
- Typing indicators are shown while the agent processes

### Group Chat Support

In group chats, the bot:

- Attributes messages to the sender (`telegram:{user_id} (@username)`)
- Includes chat context in the message (`chat: {chat_id}`)
- Shares a single session per group (or per topic in forums)
