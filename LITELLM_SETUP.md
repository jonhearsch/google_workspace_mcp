# LiteLLM Integration Guide

## Overview

This guide explains how to use the Google Workspace MCP server with LiteLLM as a personal MCP proxy. LiteLLM provides a unified interface to multiple LLM providers while supporting MCP tool integration.

## Why LiteLLM + Google Workspace MCP?

**Perfect for personal use:**
- ✅ Single unified API for all LLM providers (OpenAI, Anthropic, Ollama, etc.)
- ✅ Built-in MCP support
- ✅ Cost tracking and rate limiting
- ✅ Caching and fallback providers
- ✅ Simple authentication model (personal access)

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Your Application / Client                                │
│   ↓                                                      │
│ LiteLLM Proxy (Port 4000)                               │
│   ├─ Routes requests to: OpenAI, Claude, Ollama, etc.  │
│   └─ Provides MCP tools from Google Workspace MCP      │
│       ↓                                                  │
│   Google Workspace MCP Server (Port 8000)               │
│       └─ Authenticates with YOUR Google account         │
│           └─ Accesses YOUR Gmail, Drive, Calendar       │
└─────────────────────────────────────────────────────────┘
```

## Quick Start

### Prerequisites

- Docker and Docker Compose
- Google OAuth credentials ([setup guide](../README.md#google-cloud-setup))
- LLM API keys (OpenAI, Anthropic, etc.) or local Ollama

### Step 1: Environment Configuration

```bash
# Copy the example environment file
cp .env.litellm.example .env

# Edit with your credentials:
# - GOOGLE_OAUTH_CLIENT_ID
# - GOOGLE_OAUTH_CLIENT_SECRET
# - USER_GOOGLE_EMAIL
# - OPENAI_API_KEY (or other LLM provider keys)
```

### Step 2: Start Services

```bash
# Using the provided docker-compose file
docker-compose -f docker-compose.litellm.yml up -d

# Check health
curl http://localhost:4000/health
curl http://localhost:8000/health
```

### Step 3: Initial Google Authentication

```bash
# Watch logs for OAuth authorization URL
docker-compose -f docker-compose.litellm.yml logs -f gws_mcp

# You'll see something like:
# "Please visit this URL to authorize: https://accounts.google.com/o/oauth2/auth?..."

# Open that URL in your browser, authorize access to your Google account
# The token will be saved and reused for all future requests
```

### Step 4: Test the Integration

```bash
# Test LiteLLM proxy
curl http://localhost:4000/v1/models

# Test with MCP tools
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {"role": "user", "content": "List my 5 most recent emails"}
    ]
  }'
```

## Docker Compose Setup

The `docker-compose.litellm.yml` file includes:

### Google Workspace MCP Service
- Single-user mode (your personal Google account)
- Configurable tool tier (core, extended, complete)
- Health checks and automatic restarts
- Persistent credential storage

### LiteLLM Proxy Service
- Multi-provider LLM support
- MCP tool integration
- API endpoint at `http://localhost:4000`
- Optional: Database for logging and caching

### Optional: PostgreSQL Database
- Stores request logs
- Enables caching
- Tracks costs and usage

## LiteLLM Configuration

### Config File Structure

LiteLLM uses a YAML configuration file (`litellm_config.yaml`):

```yaml
# Model configurations
model_list:
  # OpenAI models
  - model_name: gpt-4o-mini
    litellm_params:
      model: openai/gpt-4o-mini
      api_key: os.environ/OPENAI_API_KEY

  # Anthropic models
  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: os.environ/ANTHROPIC_API_KEY

  # Local Ollama models
  - model_name: llama3.2
    litellm_params:
      model: ollama/llama3.2
      api_base: http://host.docker.internal:11434

# MCP Server configuration
mcp_servers:
  - name: google_workspace
    url: http://gws_mcp:8000/mcp
    transport: http
    # No authentication needed - single-user mode handles it

# Optional: Enable caching
litellm_settings:
  cache: true
  cache_params:
    type: redis
    host: redis
    port: 6379
```

### Environment Variables

Configure in `.env`:

```bash
# Google Workspace MCP
GOOGLE_OAUTH_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=your-client-secret
USER_GOOGLE_EMAIL=your.email@gmail.com

# LLM Provider API Keys
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GEMINI_API_KEY=...

# LiteLLM Settings
LITELLM_MASTER_KEY=sk-1234  # For securing your proxy
LITELLM_PORT=4000

# Optional: Database
DATABASE_URL=postgresql://litellm:password@postgres:5432/litellm

# Optional: Redis for caching
REDIS_HOST=redis
REDIS_PORT=6379
```

## Usage Examples

### Python Client

```python
import openai

# Point to your LiteLLM proxy
client = openai.OpenAI(
    base_url="http://localhost:4000/v1",
    api_key="sk-1234"  # Your LITELLM_MASTER_KEY
)

# The model has access to Google Workspace MCP tools automatically
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "user", "content": "What meetings do I have tomorrow?"}
    ]
)

print(response.choices[0].message.content)
```

### cURL Examples

**Check available models:**
```bash
curl http://localhost:4000/v1/models \
  -H "Authorization: Bearer sk-1234"
```

**Search Gmail:**
```bash
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-1234" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {"role": "user", "content": "Find emails from john@example.com in the last week"}
    ]
  }'
```

**Create Calendar Event:**
```bash
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-1234" \
  -d '{
    "model": "claude-3-5-sonnet-20241022",
    "messages": [
      {"role": "user", "content": "Schedule a meeting titled Team Sync tomorrow at 2pm for 1 hour"}
    ]
  }'
```

**List Drive Files:**
```bash
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-1234" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {"role": "user", "content": "Show me my recent Google Drive files"}
    ]
  }'
```

### TypeScript/JavaScript Client

```typescript
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "http://localhost:4000/v1",
  apiKey: "sk-1234", // Your LITELLM_MASTER_KEY
});

async function getEmails() {
  const response = await client.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [
      { role: "user", content: "List my 10 most recent emails" }
    ],
  });

  console.log(response.choices[0].message.content);
}

async function createDoc() {
  const response = await client.chat.completions.create({
    model: "claude-3-5-sonnet-20241022",
    messages: [
      {
        role: "user",
        content: "Create a Google Doc titled 'Project Notes' with a header saying 'Weekly Updates'"
      }
    ],
  });

  console.log(response.choices[0].message.content);
}
```

## Advanced Configuration

### Multiple LLM Providers with Fallbacks

```yaml
model_list:
  # Primary: OpenAI GPT-4
  - model_name: gpt-4
    litellm_params:
      model: openai/gpt-4
      api_key: os.environ/OPENAI_API_KEY

  # Fallback: Anthropic Claude
  - model_name: gpt-4
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: os.environ/ANTHROPIC_API_KEY

litellm_settings:
  # Enable fallbacks
  fallbacks:
    - gpt-4: [claude-3-5-sonnet-20241022]
```

### Rate Limiting

```yaml
litellm_settings:
  rpm: 100  # Requests per minute
  max_parallel_requests: 10
```

### Cost Tracking

```yaml
litellm_settings:
  success_callback: ["langfuse"]  # Track to Langfuse

  # Or use built-in logging
  service_callback: ["logger"]
```

### Redis Caching

```yaml
litellm_settings:
  cache: true
  cache_params:
    type: redis
    host: redis
    port: 6379
    ttl: 3600  # Cache for 1 hour
```

Add Redis to docker-compose:

```yaml
services:
  redis:
    image: redis:alpine
    container_name: litellm_redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped
```

### Tool Selection

Choose which Google Workspace tools to expose:

```yaml
# In docker-compose.litellm.yml, gws_mcp service:
environment:
  # Option 1: Use tool tiers
  - TOOL_TIER=core  # Essential tools only

  # Option 2: Specific tools
  # - TOOLS=gmail,calendar,drive

  # Option 3: All tools
  # - TOOL_TIER=complete
```

## Monitoring and Debugging

### View Logs

```bash
# All services
docker-compose -f docker-compose.litellm.yml logs -f

# Specific service
docker-compose -f docker-compose.litellm.yml logs -f litellm
docker-compose -f docker-compose.litellm.yml logs -f gws_mcp
```

### Health Checks

```bash
# LiteLLM proxy health
curl http://localhost:4000/health

# Google Workspace MCP health
curl http://localhost:8000/health

# Check available models
curl http://localhost:4000/v1/models \
  -H "Authorization: Bearer sk-1234"
```

### LiteLLM UI (Optional)

LiteLLM includes a built-in UI for monitoring:

```yaml
# Add to docker-compose.litellm.yml litellm service:
environment:
  - UI_USERNAME=admin
  - UI_PASSWORD=your-secure-password
```

Access at: `http://localhost:4000/ui`

### Debugging MCP Tools

```bash
# List available MCP tools
curl http://localhost:8000/mcp/tools/list

# Test a specific tool directly
curl -X POST http://localhost:8000/mcp/tools/call \
  -H "Content-Type: application/json" \
  -d '{
    "name": "search_gmail_messages",
    "arguments": {
      "query": "from:example@gmail.com",
      "max_results": 5
    }
  }'
```

## Troubleshooting

### Issue: LiteLLM can't connect to MCP server

**Check network connectivity:**
```bash
docker-compose -f docker-compose.litellm.yml exec litellm curl http://gws_mcp:8000/health
```

**Verify services are on same network:**
```bash
docker network inspect litellm_network
```

### Issue: Google authentication failing

**Re-authenticate:**
```bash
# Clear stored credentials
docker-compose -f docker-compose.litellm.yml exec gws_mcp rm -rf /app/store_creds/*

# Restart and follow OAuth flow
docker-compose -f docker-compose.litellm.yml restart gws_mcp
docker-compose -f docker-compose.litellm.yml logs -f gws_mcp
```

### Issue: MCP tools not available to LLM

**Verify MCP configuration in litellm_config.yaml:**
```yaml
mcp_servers:
  - name: google_workspace
    url: http://gws_mcp:8000/mcp  # Must match service name
    transport: http
```

**Check MCP server is registered:**
```bash
# Look for "Registered MCP server" in logs
docker-compose -f docker-compose.litellm.yml logs litellm | grep MCP
```

### Issue: High latency or rate limits

**Enable caching:**
```yaml
litellm_settings:
  cache: true
  cache_params:
    type: redis
    ttl: 3600
```

**Use core tier for fewer tools:**
```yaml
# In gws_mcp environment:
- TOOL_TIER=core  # Instead of complete
```

## Security Best Practices

### 1. Secure the LiteLLM Proxy

```yaml
environment:
  # Use a strong master key
  - LITELLM_MASTER_KEY=sk-your-very-secure-random-key

  # Enable authentication
  - LITELLM_AUTH=true
```

### 2. Use HTTPS in Production

```yaml
services:
  caddy:
    image: caddy:latest
    ports:
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
    restart: unless-stopped
```

**Caddyfile:**
```
litellm.yourdomain.com {
    reverse_proxy litellm:4000
}
```

### 3. Network Isolation

```yaml
services:
  gws_mcp:
    # Don't expose port externally, only via Docker network
    expose:
      - "8000"
    # Remove:
    # ports:
    #   - "8000:8000"
```

### 4. Resource Limits

```yaml
services:
  litellm:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
```

### 5. Regular Updates

```bash
# Pull latest images
docker-compose -f docker-compose.litellm.yml pull

# Restart with new images
docker-compose -f docker-compose.litellm.yml up -d
```

## Production Deployment

### Complete Production Stack

```yaml
version: '3.8'

services:
  # Reverse proxy with HTTPS
  caddy:
    image: caddy:latest
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    restart: unless-stopped

  # LiteLLM proxy
  litellm:
    image: ghcr.io/berriai/litellm:latest
    expose:
      - "4000"
    environment:
      - LITELLM_MASTER_KEY=${LITELLM_MASTER_KEY}
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_HOST=redis
    volumes:
      - ./litellm_config.yaml:/app/config.yaml:ro
    restart: unless-stopped

  # Google Workspace MCP
  gws_mcp:
    image: ghcr.io/jonhearsch/google_workspace_mcp:latest
    expose:
      - "8000"
    environment:
      - GOOGLE_OAUTH_CLIENT_ID=${GOOGLE_OAUTH_CLIENT_ID}
      - GOOGLE_OAUTH_CLIENT_SECRET=${GOOGLE_OAUTH_CLIENT_SECRET}
      - USER_GOOGLE_EMAIL=${USER_GOOGLE_EMAIL}
      - MCP_SINGLE_USER_MODE=1
      - TOOL_TIER=core
    volumes:
      - gws_creds:/app/store_creds
    restart: unless-stopped

  # PostgreSQL for LiteLLM
  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=litellm
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=litellm
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  # Redis for caching
  redis:
    image: redis:alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  caddy_data:
  caddy_config:
  gws_creds:
  postgres_data:
  redis_data:
```

## Use Cases

### Personal AI Assistant

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:4000/v1",
    api_key="sk-1234"
)

# Daily briefing
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{
        "role": "user",
        "content": """
        Give me my daily briefing:
        1. Today's calendar events
        2. Unread emails from the last 24 hours
        3. Recent updates in my Google Drive
        """
    }]
)

print(response.choices[0].message.content)
```

### Email Management

```python
# Summarize unread emails
response = client.chat.completions.create(
    model="claude-3-5-sonnet-20241022",
    messages=[{
        "role": "user",
        "content": "Summarize all unread emails and highlight anything urgent"
    }]
)
```

### Document Creation

```python
# Generate and save a report
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{
        "role": "user",
        "content": """
        Create a Google Doc titled 'Weekly Report' with:
        - Summary of this week's calendar events
        - List of emails received from the team
        - Action items for next week
        """
    }]
)
```

### Calendar Management

```python
# Smart scheduling
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{
        "role": "user",
        "content": """
        Find a free 1-hour slot in my calendar this week
        and schedule a meeting titled 'Project Review'
        """
    }]
)
```

## Advantages of LiteLLM for Personal MCP

1. **✅ Unified Interface:** One API for all LLM providers
2. **✅ Cost Optimization:** Automatic caching and cost tracking
3. **✅ Fallback Support:** If one provider fails, use another
4. **✅ Simple Authentication:** Single-user mode, no complex token management
5. **✅ Built-in Monitoring:** Track usage, costs, and performance
6. **✅ Easy Integration:** Works with any OpenAI-compatible client
7. **✅ Local & Cloud Models:** Mix Ollama, OpenAI, Anthropic, etc.

## Comparison: LiteLLM vs Open WebUI

| Feature | LiteLLM | Open WebUI |
|---------|---------|------------|
| **Primary Purpose** | LLM proxy/router | Chat UI |
| **MCP Support** | ✅ Native | ✅ Via extension |
| **API Access** | ✅ Yes | ❌ UI only |
| **Multi-Provider** | ✅ Built-in | Limited |
| **Caching** | ✅ Yes | Limited |
| **Cost Tracking** | ✅ Yes | No |
| **Best For** | API integration, automation | Interactive chat |
| **Authentication** | Simple master key | User accounts |

**Recommendation:** Use **LiteLLM** for personal MCP API access, **Open WebUI** for interactive chat interface. Or use both together!

## Combining LiteLLM + Open WebUI

You can run both simultaneously:

```yaml
services:
  # LiteLLM as the backend
  litellm:
    # ... configuration

  # Google Workspace MCP
  gws_mcp:
    # ... configuration

  # Open WebUI pointing to LiteLLM
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    environment:
      - OPENAI_API_BASE_URL=http://litellm:4000/v1
      - OPENAI_API_KEY=sk-1234  # Your LITELLM_MASTER_KEY
```

This gives you:
- API access via LiteLLM (port 4000)
- UI access via Open WebUI (port 3000)
- Google Workspace tools in both

## Further Resources

- [LiteLLM Documentation](https://docs.litellm.ai/)
- [Google Workspace MCP Documentation](../README.md)
- [MCP Specification](https://modelcontextprotocol.io/)
- [LiteLLM GitHub](https://github.com/BerriAI/litellm)
- [Docker Setup Guide](./DOCKER_SETUP.md)

## Getting Help

If you encounter issues:

1. **Check logs:** `docker-compose -f docker-compose.litellm.yml logs -f`
2. **Verify health:** `curl http://localhost:4000/health`
3. **Test MCP directly:** `curl http://localhost:8000/mcp/tools/list`
4. **Review configuration:** Check `litellm_config.yaml`
5. **Open an issue:** Include logs and configuration (redact credentials)
