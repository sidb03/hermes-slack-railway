# Hermes Agent — Slack on Railway

Deploy [Hermes Agent](https://github.com/NousResearch/hermes-agent) to Railway as a Slack bot with persistent state.

This is a Slack-focused fork of [hermes-railway-template](https://github.com/lovexbytes/hermes-railway-template). Setup and configuration are done through Railway Variables — the container bootstraps Hermes automatically on first run.

## What you get

- Hermes gateway running as a Railway worker connected to Slack
- Socket Mode (no public URL required)
- First-boot bootstrap from environment variables
- Persistent Hermes state on a Railway volume at `/data`

## Slack app setup

Before deploying, create a Slack app and collect two tokens.

### 1. Create a Slack app

Go to [api.slack.com/apps](https://api.slack.com/apps) and click **Create New App** > **From scratch**. Give it a name and select your workspace.

### 2. Configure Bot Token Scopes

Navigate to **OAuth & Permissions** > **Bot Token Scopes** and add:

| Scope | Purpose |
|---|---|
| `app_mentions:read` | React when mentioned |
| `channels:history` | Read messages in public channels |
| `channels:read` | List public channels |
| `chat:write` | Send messages |
| `files:read` | Access shared files |
| `files:write` | Upload files |
| `groups:history` | Read messages in private channels |
| `groups:read` | List private channels |
| `im:history` | Read DMs |
| `im:read` | List DMs |
| `im:write` | Open DMs |
| `users:read` | Look up user info |

### 3. Enable Socket Mode

Go to **Socket Mode** and toggle it on. Create an **App-Level Token** with the `connections:write` scope. Copy the token — it starts with `xapp-`. This is your `SLACK_APP_TOKEN`.

### 4. Subscribe to events

Go to **Event Subscriptions** > toggle on. Under **Subscribe to bot events**, add:

- `message.channels`
- `message.groups`
- `message.im`
- `app_mention`

### 5. Install to workspace

Go to **Install App** and click **Install to Workspace**. Copy the **Bot User OAuth Token** — it starts with `xoxb-`. This is your `SLACK_BOT_TOKEN`.

### 6. Get allowed user IDs

In Slack, click a user's profile > **More** > **Copy member ID**. These go in `SLACK_ALLOWED_USERS`.

## Railway deploy

1. Add a volume mounted at `/data`.
2. Deploy as a worker service.
3. Set the required variables below.

## Required environment variables

```env
OPENROUTER_API_KEY=""
SLACK_BOT_TOKEN=""
SLACK_APP_TOKEN=""
SLACK_ALLOWED_USERS=""
```

### Inference provider (pick one)

| Variable | Example |
|---|---|
| `OPENROUTER_API_KEY` | `sk-or-v1-...` |
| `ANTHROPIC_API_KEY` | `sk-ant-...` |
| `OPENAI_BASE_URL` + `OPENAI_API_KEY` | Custom OpenAI-compatible endpoint |

If you set multiple provider keys, set `HERMES_INFERENCE_PROVIDER` (e.g. `openrouter`) to avoid auto-selection surprises.

### Slack variables

| Variable | Required | Description |
|---|---|---|
| `SLACK_BOT_TOKEN` | Yes | Bot User OAuth Token (`xoxb-...`) |
| `SLACK_APP_TOKEN` | Yes | App-Level Token (`xapp-...`) |
| `SLACK_ALLOWED_USERS` | Recommended | Comma-separated Slack Member IDs |
| `SLACK_HOME_CHANNEL` | No | Default channel for cron/scheduled messages |
| `SLACK_HOME_CHANNEL_NAME` | No | Display name for home channel |
| `SLACK_ALLOW_ALL_USERS` | No | Set `true` to skip allowlist (not recommended) |

Allowlist format — plain comma-separated, no brackets or quotes:

```
SLACK_ALLOWED_USERS=U01234ABCDE,U09876WXYZ
```

## Template defaults

Already set in `railway.toml`:

- `HERMES_HOME=/data/.hermes`
- `HOME=/data`
- `MESSAGING_CWD=/data/workspace`

## Usage

1. Invite the bot to a channel: `/invite @YourBotName`
2. Send a message or mention the bot.
3. Hermes responds via your configured model provider.

## Running Hermes commands manually

Use [Railway SSH](https://docs.railway.com/cli/ssh) to connect, then:

```bash
hermes status
hermes config
hermes model
```

## Runtime behavior

Entrypoint (`scripts/entrypoint.sh`):

1. Validates `SLACK_BOT_TOKEN` and `SLACK_APP_TOKEN` are set
2. Validates at least one inference provider is configured
3. Writes runtime env to `${HERMES_HOME}/.env`
4. Creates `${HERMES_HOME}/config.yaml` if missing
5. Starts `hermes gateway`

## Troubleshooting

- **`not_authed` or `invalid_auth`**: Regenerate your Bot Token and App Token in Slack app settings and update Railway variables.
- **Bot connected but no replies**: Check `SLACK_ALLOWED_USERS` contains the correct Member IDs.
- **`missing_scope`**: Add the scope under OAuth & Permissions and **reinstall** the app to your workspace.
- **Socket disconnects**: Usually network instability — Bolt auto-reconnects, but check Railway logs.
- **`401 Missing Authentication header`**: Provider key mismatch — check `HERMES_INFERENCE_PROVIDER` matches your key.
- **Data lost after redeploy**: Verify Railway volume is mounted at `/data`.

## Build pinning

Docker build arg `HERMES_GIT_REF` (default: `main`). Override in Railway to pin a tag or commit.

## Local smoke test

```bash
docker build -t hermes-slack-railway .

docker run --rm \
  -e OPENROUTER_API_KEY=sk-or-xxx \
  -e SLACK_BOT_TOKEN=xoxb-xxx \
  -e SLACK_APP_TOKEN=xapp-xxx \
  -e SLACK_ALLOWED_USERS=U01234ABCDE \
  -v "$(pwd)/.tmpdata:/data" \
  hermes-slack-railway
```

## Security

- Always set `SLACK_ALLOWED_USERS` — if unset, gateway denies all messages by default.
- Treat tokens as passwords. Store them only in Railway Variables.
- Socket Mode avoids exposing a public endpoint.
- Periodically rotate tokens via Slack app settings.

## Upstream docs

- [Hermes Agent](https://github.com/NousResearch/hermes-agent)
- [Hermes Slack Guide](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/messaging/slack.md)
