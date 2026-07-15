# Whoop MCP Server

A Model Context Protocol (MCP) server that connects your Whoop health data to Claude. Designed to be hosted remotely and used as a custom connector in Claude.ai.

Built using the [Whoop Developer API v2](https://developer.whoop.com/docs/introduction).

## Features

- **Recovery Data**: Daily recovery scores, HRV, resting heart rate, SpO2, skin temperature
- **Sleep Analysis**: Sleep duration, stages, efficiency, performance, respiratory rate
- **Strain Tracking**: Daily strain scores, calories burned, heart rate zones
- **Workout History**: All logged workouts with detailed metrics
- **Auto-Sync**: Automatically keeps data fresh with smart sync logic
- **90-Day History**: Maintains local cache of your health data for trend analysis

## MCP Tools

| Tool | Description |
|------|-------------|
| `get_today` | Morning briefing with recovery, sleep, and strain |
| `get_recovery_trends` | Recovery patterns over time with HRV/RHR |
| `get_sleep_analysis` | Sleep quality trends and stage breakdowns |
| `get_strain_history` | Training load and calorie trends |
| `sync_data` | Manually trigger a data sync |
| `get_auth_url` | Get authorization URL for Whoop connection |
| `send_email` | Send an HTML email (e.g. a daily briefing) to a fixed, pre-configured recipient via Resend |

## Setup

### 1. Create a Whoop Developer App

1. Go to [developer.whoop.com](https://developer.whoop.com)
2. Create a new application
3. Note your **Client ID** and **Client Secret**
4. Set the redirect URI to your deployed server's callback URL (e.g., `https://your-app.railway.app/callback`)

### 2. Set up Email Delivery (Resend)

1. Go to [Resend.com](https://resend.com) and create an account
2. Create an API key in the dashboard
3. Verify the sender domain/email address you'll use
4. Note your API key and sender email

### 3. Deploy to Railway

1. Fork/push this repo to GitHub
2. Create a new project on [Railway](https://railway.app)
3. Connect your GitHub repo
4. Add environment variables:
   - `WHOOP_CLIENT_ID`: Your Whoop app client ID
   - `WHOOP_CLIENT_SECRET`: Your Whoop app client secret
   - `WHOOP_REDIRECT_URI`: `https://your-app.railway.app/callback`
   - `RESEND_API_KEY`: Your Resend API key
   - `EMAIL_FROM`: Your verified Resend sender email
   - `EMAIL_FROM_NAME`: Display name for sender
   - `EMAIL_TO`: Email address to send briefings to
5. Add a volume mounted at `/data` for persistent SQLite storage
6. Deploy!

### 4. Authorize with Whoop

1. Visit `https://your-app.railway.app/health` to verify it's running
2. The first time you use the `get_auth_url` tool in Claude, it will provide an authorization link
3. Visit the link, log in to Whoop, and authorize the app
4. You'll be redirected back and the initial 90-day sync will begin

### 5. Connect to Claude

1. Go to Claude.ai settings → Connectors
2. Click "Add custom connector"
3. Enter:
   - **Name**: Whoop
   - **Remote MCP server URL**: `https://your-app.railway.app/mcp`
4. Use it in any chat!

## Local Development

```bash
# Install dependencies
npm install

# Create .env file
cat > .env << EOF
WHOOP_CLIENT_ID=your_client_id
WHOOP_CLIENT_SECRET=your_client_secret
WHOOP_REDIRECT_URI=http://localhost:3000/callback
MCP_MODE=http
RESEND_API_KEY=your_resend_api_key
EMAIL_FROM=noreply@yourdomain.com
EMAIL_FROM_NAME=Daily Health Briefing
EMAIL_TO=your-email@example.com
EOF

# Run in development mode
npm run dev
```

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `WHOOP_CLIENT_ID` | Whoop OAuth client ID | Required |
| `WHOOP_CLIENT_SECRET` | Whoop OAuth client secret | Required |
| `WHOOP_REDIRECT_URI` | OAuth callback URL | `http://localhost:3000/callback` |
| `DB_PATH` | SQLite database path | `./whoop.db` |
| `PORT` | HTTP server port | `3000` |
| `MCP_MODE` | `http` for remote, `stdio` for local | `http` |
| `RESEND_API_KEY` | Resend API key for sending emails | Required for `send_email` |
| `EMAIL_FROM` | Email address to send from (must be verified in Resend) | `onboarding@resend.dev` |
| `EMAIL_FROM_NAME` | Sender display name | `RC's Daily Health Briefing` |
| `EMAIL_TO` | Fixed recipient email address for `send_email` | `raghavchadha@gmail.com` |

### Sending email (`send_email`)

The Resend API requires authentication via Bearer token, which client-side web-fetch
tools (including the one available in Claude Routines) cannot reliably attach.
`send_email` moves that authenticated call server-side: it always sends to the single
recipient configured via `EMAIL_TO` (the caller only supplies `subject` and
`htmlContent`), so the endpoint can't be used as an open relay to arbitrary addresses.
Set `RESEND_API_KEY` as a Railway environment variable — never put it in routine/prompt
text, since that text isn't a secrets store.

## Architecture

```
┌─────────────────────────────────────────────────┐
│              Whoop MCP Server                   │
│                                                 │
│  ┌─────────────┐      ┌──────────────────┐    │
│  │ MCP Server  │◄────►│  SQLite Database │    │
│  │ (HTTP)      │      │  - cycles        │    │
│  └─────────────┘      │  - recovery      │    │
│         │             │  - sleep         │    │
│         │             │  - workouts      │    │
│         ▼             │  - tokens        │    │
│  ┌─────────────┐      └──────────────────┘    │
│  │ Whoop API   │                               │
│  │ Client      │                               │
│  └─────────────┘                               │
└─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│  Claude.ai (Custom Connector)                   │
│  "Hey, what's my recovery today?"               │
└─────────────────────────────────────────────────┘
```

## API Endpoints Used

This server uses the following Whoop API v2 endpoints:

- `GET /v2/user/profile/basic` - User profile
- `GET /v2/user/measurement/body` - Body measurements
- `GET /v2/cycle` - Physiological cycles (strain data)
- `GET /v2/recovery` - Recovery scores
- `GET /v2/activity/sleep` - Sleep records
- `GET /v2/activity/workout` - Workout records

## License

MIT - See [LICENSE](LICENSE) for details.
