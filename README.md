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

Add these environment variables to your `~/.zshrc` (or run before each Claude Code session):

```bash
export ANTHROPIC_BASE_URL=http://localhost:4000
export ANTHROPIC_AUTH_TOKEN=sk-your_actual_master_key_here
```

**Important:**
- Replace `sk-your_actual_master_key_here` with the actual `LITELLM_MASTER_KEY` value from your `.env` file
- Replace `localhost:4000` with your proxy's actual host and port:
  - **Local**: `http://localhost:4000` (default)
  - **Remote server**: `http://your-server.com:4000`
  - **Custom port**: `http://localhost:8080` (if you changed the port mapping)

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

## Configuration Options

### Project-Specific Configuration

Instead of global environment variables, you can configure Claude Code per-project using `.claude/settings.local.json`:

```json
{
  "anthropicBaseUrl": "http://localhost:4000",
  "anthropicAuthToken": "sk-your_actual_master_key_here"
}
```

**Important:**
- Replace `sk-your_actual_master_key_here` with the actual `LITELLM_MASTER_KEY` value from your `.env` file
- Replace the URL with your proxy's actual location (e.g., `http://your-server.com:4000` for remote servers)

### Remote Deployment

To run the proxy on a remote server:

1. **Deploy the Docker container** on your server (same setup as local)
2. **Expose the port** through your firewall (if needed)
3. **Configure Claude Code** to use the remote URL:
   ```bash
   export ANTHROPIC_BASE_URL=http://your-server.com:4000
   export ANTHROPIC_AUTH_TOKEN=sk-your_master_key
   ```

**Production considerations:**
- Use HTTPS with a reverse proxy (nginx, Caddy, Traefik)
- Bind to localhost only: change `docker-compose.yml` ports to `127.0.0.1:4000:4000`
- Let the reverse proxy handle SSL and external exposure
- Consider rate limiting and monitoring

## Commands

### Start the proxy
```bash
docker compose up -d
```

### View logs
```bash
docker compose logs -f litellm
```

### Check proxy status
```bash
docker compose ps
curl http://localhost:4000/health -H "Authorization: Bearer sk-your_master_key"
```

Replace `localhost:4000` with your actual proxy host and port.

### Stop the proxy
```bash
docker compose down
```

### Restart after config changes
```bash
docker compose restart
```

## Troubleshooting

### Changing the Port

To use a different port, edit `.env`:

```bash
LITELLM_PORT=8080
```

Then restart the container:
```bash
docker compose down && docker compose up -d
```

Update your Claude Code configuration:
```bash
export ANTHROPIC_BASE_URL=http://localhost:8080
```

Or in `.claude/settings.local.json`:
```json
{
  "anthropicBaseUrl": "http://localhost:8080",
  "anthropicAuthToken": "sk-your_actual_master_key_here"
}
```

### Authentication errors
1. Verify your Poe API key in `.env` is correct
2. Check LiteLLM logs: `docker compose logs -f litellm`
3. Test Poe API directly:
   ```bash
   curl -H "Authorization: Bearer YOUR_POE_KEY" \
        https://api.poe.com/v1/models
   ```

### Model not found
Verify the model names in `litellm-config.yaml` match Poe's exact model identifiers. Current supported models:
- `claude-sonnet-4.5`
- `claude-opus-4.5`

### Connection refused
Ensure the Docker container is running:
```bash
docker compose ps
```

If it's not running, check logs for errors:
```bash
docker compose logs
```

### Claude Code debug mode
Run with verbose flag to see detailed request/response info:
```bash
claude --verbose
```

## File Structure

```
/Users/tholtz/code/claude-poe/
├── docker-compose.yml      # Docker Compose configuration
├── litellm-config.yaml     # LiteLLM proxy configuration
├── .env.template          # Environment variable template
├── .env                    # Your API keys (DO NOT COMMIT)
├── .gitignore             # Ignore sensitive files
└── README.md              # This file
```

## How It Works

1. Claude Code sends requests to your LiteLLM proxy (e.g., `http://localhost:4000`)
2. LiteLLM receives Anthropic Messages format request
3. LiteLLM translates to OpenAI Chat Completions format
4. LiteLLM forwards to Poe.com with Bearer token authentication
5. Poe processes request and returns OpenAI format response
6. LiteLLM translates back to Anthropic format
7. Claude Code receives Anthropic format response

## Adding More Models

To add more Poe models, edit `litellm-config.yaml`:

```yaml
model_list:
  - model_name: claude-sonnet-4-5-20250929
    litellm_params:
      model: claude-sonnet-4.5
      api_base: https://api.poe.com/v1
      api_key: os.environ/POE_API_KEY
      custom_llm_provider: openai

  # Add more models here
  - model_name: gpt-4o
    litellm_params:
      model: gpt-4o
      api_base: https://api.poe.com/v1
      api_key: os.environ/POE_API_KEY
      custom_llm_provider: openai
```

See Poe's full model list at: https://creator.poe.com/docs/external-applications/openai-compatible-api

## Security Notes

- The `.env` file contains sensitive keys (Poe API key and master key) - never commit it to version control
- The master key (`LITELLM_MASTER_KEY`) is required for all proxy requests - keep it secure
- By default, the proxy binds to all interfaces (`0.0.0.0:4000`). For local-only access, change the port mapping in `docker-compose.yml` to `127.0.0.1:4000:4000`
- If exposing the proxy on a network, use HTTPS with a reverse proxy (nginx, Caddy, etc.)
- Poe API has a rate limit of 500 requests per minute

## Resources

- [Poe API Documentation](https://creator.poe.com/docs/external-applications/openai-compatible-api)
- [LiteLLM Documentation](https://docs.litellm.ai/)
- [Claude Code Documentation](https://code.claude.com/docs)
