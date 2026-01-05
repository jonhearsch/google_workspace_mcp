# Open WebUI Integration Guide

## Overview

This guide explains how to use the Google Workspace MCP server with Open WebUI, including authentication flow and deployment options.

## Understanding Authentication

### Is Authentication Proxied?

**Short answer:** No, authentication is **per-user**, not proxied by Open WebUI.

**How it works:**

```
┌─────────────────────────────────────────────────────────────┐
│ Open WebUI User 1                                            │
│   ↓                                                          │
│ Needs their own Google OAuth token (ya29.*)                 │
│   ↓                                                          │
│ Token sent in Authorization: Bearer header                  │
│   ↓                                                          │
│ Google Workspace MCP validates token                        │
│   ↓                                                          │
│ Accesses User 1's Gmail, Drive, Calendar                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Open WebUI User 2                                            │
│   ↓                                                          │
│ Needs their own separate Google OAuth token                 │
│   ↓                                                          │
│ Accesses User 2's Gmail, Drive, Calendar                    │
└─────────────────────────────────────────────────────────────┘
```

### Authentication Modes for Open WebUI

You have **three options**:

#### Option 1: Single-User Mode (Simplest)
- **Use case:** Personal instance, one Google account
- **Pro:** Simple setup, no per-request auth
- **Con:** All Open WebUI users access the same Google account
- **Configuration:** `MCP_SINGLE_USER_MODE=1`

#### Option 2: External OAuth 2.1 Provider Mode (Recommended)
- **Use case:** Multi-tenant, each user has their own Google account
- **Pro:** True multi-user, proper isolation
- **Con:** Requires OAuth token from each user
- **Configuration:** `MCP_ENABLE_OAUTH21=true` + `EXTERNAL_OAUTH21_PROVIDER=true`

#### Option 3: Per-Request Authentication
- **Use case:** Custom authentication flow
- **Pro:** Maximum flexibility
- **Con:** Complex setup, requires custom Open WebUI integration
- **Configuration:** Requires Open WebUI modifications

**Recommendation for Open WebUI:** Start with **Option 1** for personal use or **Option 2** for multi-user deployments.

## Deployment Options

### Option 1: Single-User Mode (Personal Instance)

This is the **simplest setup** where all Open WebUI users access your personal Google Workspace.

#### Docker Compose Setup

```yaml
version: '3.8'

services:
  # Google Workspace MCP Server
  gws_mcp:
    image: ghcr.io/jonhearsch/google_workspace_mcp:latest
    container_name: gws_mcp
    ports:
      - "8000:8000"
    environment:
      # OAuth credentials
      - GOOGLE_OAUTH_CLIENT_ID=${GOOGLE_OAUTH_CLIENT_ID}
      - GOOGLE_OAUTH_CLIENT_SECRET=${GOOGLE_OAUTH_CLIENT_SECRET}

      # Single-user mode - all requests use one Google account
      - MCP_SINGLE_USER_MODE=1
      - USER_GOOGLE_EMAIL=${USER_GOOGLE_EMAIL}

      # Development mode (use HTTPS in production)
      - OAUTHLIB_INSECURE_TRANSPORT=1

      # Server config
      - WORKSPACE_MCP_PORT=8000

      # Optional: Select specific tools
      - TOOL_TIER=core  # or: TOOLS=gmail,drive,calendar

    volumes:
      - gws_creds:/app/store_creds

    restart: unless-stopped

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Open WebUI
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - "3000:8080"
    environment:
      # Ollama connection (if using)
      - OLLAMA_BASE_URL=http://host.docker.internal:11434

      # Or OpenAI API
      # - OPENAI_API_KEY=${OPENAI_API_KEY}

      # Enable MCP support
      - ENABLE_MCP=true

    volumes:
      - open-webui:/app/backend/data

    extra_hosts:
      - "host.docker.internal:host-gateway"

    restart: unless-stopped

    depends_on:
      - gws_mcp

volumes:
  gws_creds:
  open-webui:
```

#### Environment File (.env)

```bash
# Google OAuth Credentials
GOOGLE_OAUTH_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=your-client-secret

# Your Google account email
USER_GOOGLE_EMAIL=your.email@gmail.com

# Optional: OpenAI API key if not using Ollama
# OPENAI_API_KEY=sk-...
```

#### Initial Setup Steps

1. **Start the services:**
   ```bash
   docker-compose up -d
   ```

2. **Authenticate with Google (one-time):**
   ```bash
   # The first API call will trigger OAuth flow
   # Check logs for the authorization URL:
   docker-compose logs -f gws_mcp

   # You'll see something like:
   # "Please visit this URL to authorize: https://accounts.google.com/o/oauth2/auth?..."

   # Open the URL in a browser, authorize, and the token will be saved
   ```

3. **Configure Open WebUI:**
   - Open http://localhost:3000
   - Go to Settings → Admin → Connections
   - Add MCP Server:
     - Name: `Google Workspace`
     - URL: `http://gws_mcp:8000/mcp`
     - Type: `HTTP`

4. **Test the connection:**
   - Start a new chat
   - Try: "List my recent emails"
   - Try: "What's on my calendar today?"

### Option 2: Multi-User with External OAuth (Advanced)

This setup allows each Open WebUI user to authenticate with their own Google account.

⚠️ **Note:** This requires Open WebUI to support passing user-specific bearer tokens to MCP servers. As of now, Open WebUI's MCP support may not fully support this. Check Open WebUI documentation for current capabilities.

#### Docker Compose Setup

```yaml
version: '3.8'

services:
  gws_mcp:
    image: ghcr.io/jonhearsch/google_workspace_mcp:latest
    container_name: gws_mcp
    ports:
      - "8000:8000"
    environment:
      # OAuth credentials
      - GOOGLE_OAUTH_CLIENT_ID=${GOOGLE_OAUTH_CLIENT_ID}
      - GOOGLE_OAUTH_CLIENT_SECRET=${GOOGLE_OAUTH_CLIENT_SECRET}

      # Multi-user mode with external OAuth
      - MCP_ENABLE_OAUTH21=true
      - EXTERNAL_OAUTH21_PROVIDER=true

      # Stateless mode (no file persistence)
      - WORKSPACE_MCP_STATELESS_MODE=true

      # Development mode
      - OAUTHLIB_INSECURE_TRANSPORT=1

      # Server config
      - WORKSPACE_MCP_PORT=8000

      # Tool selection
      - TOOL_TIER=extended

    # No credential volume needed in stateless mode
    restart: unless-stopped

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - "3000:8080"
    environment:
      - ENABLE_MCP=true
      - OLLAMA_BASE_URL=http://host.docker.internal:11434

    volumes:
      - open-webui:/app/backend/data

    extra_hosts:
      - "host.docker.internal:host-gateway"

    restart: unless-stopped

    depends_on:
      - gws_mcp

volumes:
  open-webui:
```

#### Authentication Flow

In this mode:

1. Each user needs to obtain their own Google OAuth token
2. The token must be passed in the `Authorization: Bearer <token>` header
3. Open WebUI would need to:
   - Redirect users to Google OAuth flow
   - Store the resulting access token per user
   - Include the token in MCP requests

**Current limitation:** Open WebUI may not support per-user MCP authentication yet. Monitor the [Open WebUI issues](https://github.com/open-webui/open-webui/issues) for updates.

### Option 3: Hybrid with OAuth Proxy

For production deployments, you can add an authentication proxy between Open WebUI and the MCP server.

```yaml
version: '3.8'

services:
  # Authentication proxy (custom - you need to build this)
  auth-proxy:
    build: ./auth-proxy
    container_name: auth_proxy
    ports:
      - "9000:9000"
    environment:
      - UPSTREAM_MCP_URL=http://gws_mcp:8000
      - OAUTH_CLIENT_ID=${GOOGLE_OAUTH_CLIENT_ID}
      - OAUTH_CLIENT_SECRET=${GOOGLE_OAUTH_CLIENT_SECRET}
    depends_on:
      - gws_mcp

  gws_mcp:
    image: ghcr.io/jonhearsch/google_workspace_mcp:latest
    environment:
      - MCP_ENABLE_OAUTH21=true
      - EXTERNAL_OAUTH21_PROVIDER=true
    # Expose only to auth-proxy, not externally
    expose:
      - "8000"

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "3000:8080"
    environment:
      - ENABLE_MCP=true
    depends_on:
      - auth-proxy
```

The auth-proxy would:
1. Handle OAuth flow for each user
2. Store tokens in Redis or database
3. Inject bearer tokens into MCP requests
4. Forward requests to the actual MCP server

## Configuration Reference

### Environment Variables for Open WebUI Integration

| Variable | Value | Purpose |
|----------|-------|---------|
| `MCP_SINGLE_USER_MODE` | `1` | All requests use one Google account |
| `MCP_ENABLE_OAUTH21` | `true` | Enable OAuth 2.1 multi-user support |
| `EXTERNAL_OAUTH21_PROVIDER` | `true` | Expect bearer tokens from external source |
| `WORKSPACE_MCP_STATELESS_MODE` | `true` | No file storage (cloud-friendly) |
| `USER_GOOGLE_EMAIL` | `email@gmail.com` | Default user in single-user mode |
| `TOOL_TIER` | `core`, `extended`, `complete` | Which tools to load |
| `TOOLS` | `gmail,drive,calendar` | Specific tools only |

### Open WebUI MCP Configuration

In Open WebUI's admin panel:

**Server URL:** `http://gws_mcp:8000/mcp`

**Headers (if using bearer token auth):**
```json
{
  "Authorization": "Bearer ya29.a0AfH6SMBx..."
}
```

## Testing the Integration

### 1. Health Check

```bash
# Check MCP server health
curl http://localhost:8000/health

# Expected response:
# {"status": "healthy"}
```

### 2. List Available Tools

```bash
# Using MCP Inspector (install with: npm install -g @modelcontextprotocol/inspector)
mcp-inspector http://localhost:8000/mcp
```

### 3. Test from Open WebUI

Try these prompts in Open WebUI:

- **Gmail:** "Show me my 5 most recent emails"
- **Calendar:** "What meetings do I have tomorrow?"
- **Drive:** "List my recent Google Drive files"
- **Docs:** "Create a new Google Doc titled 'Meeting Notes'"

## Troubleshooting

### Issue: MCP server not connecting from Open WebUI

**Solution:**
```bash
# Verify network connectivity
docker exec open-webui curl http://gws_mcp:8000/health

# Check if services are on the same network
docker network inspect <network_name>
```

### Issue: Authentication failing

**For Single-User Mode:**
```bash
# Re-authenticate
docker-compose exec gws_mcp rm -rf /app/store_creds/*
docker-compose restart gws_mcp

# Trigger OAuth flow again
docker-compose logs -f gws_mcp
# Follow the authorization URL shown in logs
```

**For External OAuth Mode:**
```bash
# Verify bearer token format
# Should be: ya29.* (Google access token) or valid JWT

# Test token manually
curl -H "Authorization: Bearer ya29...." \
     http://localhost:8000/mcp/tools/list
```

### Issue: Tools not showing in Open WebUI

**Check MCP configuration:**
```bash
# Verify tools are loaded
docker-compose logs gws_mcp | grep "Registered tool"

# Should see lines like:
# Registered tool: search_gmail_messages
# Registered tool: get_events
# Registered tool: search_drive_files
```

### Issue: Rate limiting or quota errors

**Google API has quotas.** To reduce usage:

```yaml
# Use TOOL_TIER=core instead of complete
environment:
  - TOOL_TIER=core

# Or select only needed tools
environment:
  - TOOLS=gmail,calendar
```

## Security Considerations

### For Single-User Mode

⚠️ **Warning:** All Open WebUI users will access YOUR personal Google account.

**Recommendations:**
1. Use only for personal instances
2. Don't share access with untrusted users
3. Monitor Google account activity
4. Consider creating a dedicated Google account for this purpose

### For Multi-User Mode

✅ **Better security:** Each user authenticates with their own Google account.

**Recommendations:**
1. Use HTTPS in production
2. Implement rate limiting
3. Enable audit logging
4. Use stateless mode for scalability

### For Production Deployments

1. **Use HTTPS with valid certificates:**
   ```yaml
   environment:
     - WORKSPACE_EXTERNAL_URL=https://mcp.yourdomain.com
     # Remove OAUTHLIB_INSECURE_TRANSPORT
   ```

2. **Add reverse proxy (Caddy, nginx):**
   ```yaml
   services:
     caddy:
       image: caddy:latest
       ports:
         - "443:443"
       volumes:
         - ./Caddyfile:/etc/caddy/Caddyfile
   ```

3. **Enable resource limits:**
   ```yaml
   deploy:
     resources:
       limits:
         cpus: '2'
         memory: 1G
   ```

4. **Use secrets management:**
   ```bash
   docker secret create google_client_id client_id.txt
   docker secret create google_client_secret client_secret.txt
   ```

## Advanced: Custom Authentication Middleware

If you need custom authentication logic, you can create a simple proxy:

```python
# auth-proxy.py
from fastapi import FastAPI, Header, HTTPException
import httpx
import os

app = FastAPI()

# Store user tokens (use Redis in production)
user_tokens = {}

@app.post("/api/authenticate")
async def authenticate(user_id: str, google_token: str):
    """Store user's Google OAuth token"""
    user_tokens[user_id] = google_token
    return {"status": "authenticated"}

@app.post("/mcp/{path:path}")
async def proxy_mcp(
    path: str,
    user_id: str = Header(...),
    request_body: dict = None
):
    """Proxy MCP requests with user-specific token"""
    if user_id not in user_tokens:
        raise HTTPException(401, "Not authenticated")

    token = user_tokens[user_id]

    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"http://gws_mcp:8000/mcp/{path}",
            headers={"Authorization": f"Bearer {token}"},
            json=request_body
        )
        return response.json()
```

Deploy this proxy between Open WebUI and the MCP server.

## Recommendations

### For Personal Use
- ✅ Use **Single-User Mode**
- ✅ Use `TOOL_TIER=core` to minimize API usage
- ✅ Monitor Google API quota in Cloud Console

### For Team/Multi-User
- ✅ Use **External OAuth 2.1 Provider Mode**
- ✅ Implement an authentication proxy
- ✅ Use HTTPS and proper security
- ✅ Consider using Valkey/Redis for token storage

### For Production
- ✅ Use stateless mode
- ✅ Add reverse proxy with HTTPS
- ✅ Enable monitoring and logging
- ✅ Implement rate limiting
- ✅ Use secrets management
- ✅ Regular security audits

## Limitations

1. **Open WebUI MCP Support:** Open WebUI's MCP integration is still evolving. Check the [Open WebUI documentation](https://docs.openwebui.com/) for the latest capabilities.

2. **Per-User Authentication:** Full per-user OAuth may require custom middleware or waiting for Open WebUI to add this feature.

3. **Google API Quotas:** Be mindful of Google API rate limits, especially in multi-user scenarios.

4. **Token Expiry:** Access tokens expire after 1 hour. The server handles refresh automatically, but in stateless mode, users may need to re-authenticate.

## Further Resources

- [Google Workspace MCP Documentation](../README.md)
- [Open WebUI Documentation](https://docs.openwebui.com/)
- [MCP Specification](https://modelcontextprotocol.io/)
- [Google Cloud Console](https://console.cloud.google.com/)
- [FastMCP Documentation](https://github.com/jlowin/fastmcp)

## Getting Help

If you encounter issues:

1. Check logs: `docker-compose logs -f gws_mcp`
2. Verify health: `curl http://localhost:8000/health`
3. Test authentication: Check Google Cloud Console audit logs
4. Review [DOCKER_SETUP.md](./DOCKER_SETUP.md) for general Docker guidance
5. Open an issue on GitHub with detailed logs
