# Claude Code + Poe.com via LiteLLM Proxy

This project enables Claude Code to use Poe.com's API by running a local LiteLLM proxy that translates between Anthropic and OpenAI API formats.

## Why This Is Needed

Claude Code uses Anthropic's API format, while Poe.com uses OpenAI's format. They differ in:
- **Authentication**: Claude Code sends `x-api-key` header, Poe expects `Authorization: Bearer`
- **Request format**: Anthropic Messages API vs OpenAI Chat Completions API
- **Response structure**: Different JSON structures

LiteLLM handles all of this translation automatically.

## Prerequisites

- Docker and Docker Compose installed
- Poe API key from https://poe.com/api_key
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
```

### 2. Start the Proxy

```bash
docker compose up -d
```

### 3. Configure Claude Code

Add these environment variables to your `~/.zshrc`:

```bash
export ANTHROPIC_BASE_URL=http://localhost:4000
export ANTHROPIC_AUTH_TOKEN=sk-your_actual_master_key_here
```

Replace `sk-your_actual_master_key_here` with your actual `LITELLM_MASTER_KEY` from `.env`. Adjust the URL if using a different port or host.

Then reload your shell:

```bash
source ~/.zshrc
```

### 4. Test It

```bash
claude --verbose
```

You should see Claude Code connecting to your proxy URL, which then forwards requests to Poe.

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
  "anthropicBaseUrl": "http://localhost:4000",
  "anthropicAuthToken": "sk-your_actual_master_key_here"
}
```

### Custom Port

Edit `LITELLM_PORT` in `.env`, then restart:

```bash
docker compose down && docker compose up -d
```

### Remote Deployment

For production deployments:

1. Deploy the container on your server
2. Use HTTPS with a reverse proxy (nginx, Caddy, Traefik)
3. Bind to localhost: `127.0.0.1:${LITELLM_PORT}:4000` in `docker-compose.yml`
4. Configure Claude Code to use your server URL:
   ```bash
   export ANTHROPIC_BASE_URL=https://your-server.com
   export ANTHROPIC_AUTH_TOKEN=sk-your_master_key
   ```

## Management Commands

```bash
# Start
docker compose up -d

# View logs
docker compose logs -f litellm

# Check status
docker compose ps
curl http://localhost:4000/health -H "Authorization: Bearer sk-your_master_key"

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
├── docker-compose.yml      # Docker configuration
├── litellm-config.yaml     # Model routing configuration
├── .env.template          # Environment template
├── .env                    # Your secrets (gitignored)
└── README.md
```

## How It Works

1. Claude Code sends requests to your LiteLLM proxy (e.g., `http://localhost:4000`)
2. LiteLLM receives Anthropic Messages format request
3. LiteLLM translates to OpenAI Chat Completions format
4. LiteLLM forwards to Poe.com with Bearer token authentication
5. Poe processes request and returns OpenAI format response
6. LiteLLM translates back to Anthropic format
7. Claude Code receives Anthropic format response

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

- **Never commit `.env`** - contains sensitive keys
- **Master key** - required for all proxy requests
- **Network binding** - default is all interfaces (`0.0.0.0`). For local-only, use `127.0.0.1:${LITELLM_PORT}:4000`
- **Production** - use HTTPS with reverse proxy (nginx, Caddy)
- **Rate limits** - Poe API: 500 requests/minute

## Links

- [Poe API Docs](https://creator.poe.com/docs/external-applications/openai-compatible-api)
- [LiteLLM Docs](https://docs.litellm.ai/)
- [Claude Code Docs](https://code.claude.com/docs)
