# Claude Code + Poe.com via LiteLLM Proxy

This project enables Claude Code to use Poe.com's API by running a local LiteLLM proxy that translates between Anthropic and OpenAI API formats.

## Why This Is Needed

Claude Code uses Anthropic's API format, while Poe.com uses OpenAI's format. LiteLLM handles the translation automatically:
- **Authentication**: `x-api-key` → `Authorization: Bearer`
- **Request format**: Anthropic Messages API → OpenAI Chat Completions API
- **Response structure**: Different JSON structures

Tailscale provides secure remote access without exposing the proxy to the internet.

## Prerequisites

- Docker and Docker Compose installed
- Poe API key from https://poe.com/api_key
- Tailscale account (free) - https://tailscale.com
- Claude Code installed

## Quick Start

### 1. Configure Environment Variables

Copy the template and add your keys:

```bash
cp .env.template .env
```

Edit `.env` and add your keys:

```bash
# Get your Poe API key from https://poe.com/api_key
POE_API_KEY=your_actual_poe_api_key_here

# Generate a secure master key
LITELLM_MASTER_KEY=sk-$(openssl rand -hex 32)

# Port (default: 4000)
LITELLM_PORT=4000

# Get Tailscale auth key from https://login.tailscale.com/admin/settings/keys
TAILSCALE_AUTHKEY=tskey-auth-xxxxx
```

### 2. Start the Proxy

```bash
docker compose up -d
```

### 3. Get Tailscale Machine Name

On the server where the proxy is running, get the Tailscale machine name:

```bash
tailscale status
```

Look for the machine name (e.g., `litellm-proxy.tail1234.ts.net`) or IP (e.g., `100.x.y.z`).

### 4. Configure Claude Code

On your client machine (where Claude Code runs), install Tailscale:

```bash
# Install and connect to Tailscale
tailscale up
```

Add these environment variables to your `~/.zshrc`:

```bash
# Use full Tailscale hostname or IP
export ANTHROPIC_BASE_URL=http://litellm-proxy.tail1234.ts.net:4000
export ANTHROPIC_AUTH_TOKEN=sk-your_actual_master_key_here
```

**Important:**
- Replace `litellm-proxy.tail1234.ts.net` with your actual Tailscale hostname from step 3
- Or use the Tailscale IP: `http://100.x.y.z:4000`
- Replace `sk-your_actual_master_key_here` with your `LITELLM_MASTER_KEY` from `.env`

Then reload your shell:

```bash
source ~/.zshrc
```

### 5. Test It

```bash
claude --verbose
```

You should see Claude Code connecting to your Tailscale URL, which then forwards requests to Poe.

## Available Models

This configuration supports:
- **Claude Sonnet 4.5** (`claude-sonnet-4-5-20250929`)
- **Claude Opus 4.5** (`claude-opus-4-5-20251101`)
- **Claude Haiku 4.5** (`claude-haiku-4-5-20251001`)

## Advanced Configuration

### Per-Project Settings

Configure Claude Code per-project using `.claude/settings.local.json`:

```json
{
  "anthropicBaseUrl": "http://litellm-proxy.tail1234.ts.net:4000",
  "anthropicAuthToken": "sk-your_actual_master_key_here"
}
```

Replace with your actual Tailscale hostname or IP.

### Accessing from Multiple Devices

All devices on your Tailscale network can access the proxy:

1. Install Tailscale on each device
2. Connect to your Tailscale network: `tailscale up`
3. Use the Tailscale hostname: `http://litellm-proxy.tail1234.ts.net:4000`

### Custom Port

Edit `LITELLM_PORT` in `.env`, then restart:

```bash
docker compose down && docker compose up -d
```

## Management Commands

```bash
# Start
docker compose up -d

# View logs
docker compose logs -f litellm

# Check status (from server)
docker compose ps
curl http://localhost:4000/health -H "Authorization: Bearer sk-your_master_key"

# Check status (from client via Tailscale)
curl http://litellm-proxy.tail1234.ts.net:4000/health -H "Authorization: Bearer sk-your_master_key"

# Restart
docker compose restart

# Stop
docker compose down
```

## Troubleshooting

**Authentication errors:**
- Verify `POE_API_KEY` in `.env` is correct
- Check `LITELLM_MASTER_KEY` matches between `.env` and your Claude Code config
- View logs: `docker compose logs -f litellm`

**Connection refused:**
- Ensure Tailscale is running on both server and client: `tailscale status`
- Verify you're using the correct Tailscale hostname or IP
- Ensure container is running: `docker compose ps`
- Check correct port: verify `LITELLM_PORT` in `.env`
- View logs: `docker compose logs`

**Debug mode:**
Run Claude Code with verbose output:
```bash
claude --verbose
```

## Project Structure

```
├── docker-compose.yml      # Docker configuration with Tailscale
├── litellm-config.yaml    # Model routing configuration
├── .env.template          # Environment template
├── .env                    # Your secrets (gitignored)
└── README.md
```

## How It Works

1. Claude Code connects to `http://litellm-proxy.tail1234.ts.net:4000` via Tailscale (encrypted tunnel)
2. LiteLLM receives Anthropic Messages format request
3. LiteLLM translates to OpenAI Chat Completions format
4. LiteLLM forwards to Poe.com with Bearer token authentication
5. Poe processes request and returns OpenAI format response
6. LiteLLM translates back to Anthropic format
7. Claude Code receives Anthropic format response

**Security:** The proxy is only accessible via localhost (on server) or Tailscale network. It's never exposed to the public internet.

## Adding Models

Add more models to `litellm-config.yaml`:

```yaml
- model_name: gpt-4o
  litellm_params:
    model: openai/gpt-4o
    api_base: https://api.poe.com/v1
    api_key: os.environ/POE_API_KEY
```

See [Poe's model list](https://creator.poe.com/docs/external-applications/openai-compatible-api) for available models.

## Security

- **Never commit `.env`** - contains sensitive keys (Poe API, master key, Tailscale auth key)
- **Network isolation** - proxy only accessible via localhost and Tailscale network
- **Encrypted connections** - Tailscale provides WireGuard encryption
- **No public exposure** - proxy never exposed to the internet
- **Master key** - required for all proxy requests
- **Rate limits** - Poe API: 500 requests/minute

## Links

- [Poe API Docs](https://creator.poe.com/docs/external-applications/openai-compatible-api)
- [LiteLLM Docs](https://docs.litellm.ai/)
- [Claude Code Docs](https://code.claude.com/docs)
- [Tailscale](https://tailscale.com) - Secure remote access
