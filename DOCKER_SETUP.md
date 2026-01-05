# Docker Setup Guide for Google Workspace MCP

This guide explains how to use the Docker images built by this repository's GitHub Actions.

## Quick Start

### 1. Prepare Environment

```bash
# Copy the example environment file
cp .env.docker.example .env

# Edit .env and add your Google OAuth credentials
# Minimum required:
# - GOOGLE_OAUTH_CLIENT_ID
# - GOOGLE_OAUTH_CLIENT_SECRET
# - OAUTHLIB_INSECURE_TRANSPORT=1 (for development)
```

### 2. Choose Your Method

#### Method A: Use Pre-built Image from GitHub Container Registry

```bash
# Pull the latest image
docker pull ghcr.io/jonhearsch/google_workspace_mcp:latest

# Run with docker-compose (update docker-compose.yml to use the image)
docker-compose up -d
```

#### Method B: Build Locally

```bash
# Build and run with docker-compose
docker-compose up -d

# Or build manually
docker build -t workspace-mcp .
docker run -p 8000:8000 --env-file .env -v $(pwd)/client_secret.json:/app/client_secret.json workspace-mcp
```

## Using the Enhanced Docker Compose

The repository includes `docker-compose.enhanced.yml` with more configuration options:

```bash
# Use the enhanced configuration
docker-compose -f docker-compose.enhanced.yml up -d

# View logs
docker-compose -f docker-compose.enhanced.yml logs -f

# Stop the service
docker-compose -f docker-compose.enhanced.yml down
```

## Configuration Scenarios

### Scenario 1: Basic Setup (Core Tools Only)

**Use case**: Minimal installation with essential Gmail, Drive, and Calendar tools.

**Configuration** (.env):
```bash
GOOGLE_OAUTH_CLIENT_ID=your-id
GOOGLE_OAUTH_CLIENT_SECRET=your-secret
OAUTHLIB_INSECURE_TRANSPORT=1
TOOL_TIER=core
```

**Start**:
```bash
docker-compose up -d
```

### Scenario 2: Specific Tools Only

**Use case**: Only need Gmail and Drive functionality.

**Configuration** (.env):
```bash
GOOGLE_OAUTH_CLIENT_ID=your-id
GOOGLE_OAUTH_CLIENT_SECRET=your-secret
OAUTHLIB_INSECURE_TRANSPORT=1
TOOLS=gmail,drive
```

### Scenario 3: Multi-User with OAuth 2.1

**Use case**: Multiple users accessing the MCP server with bearer token authentication.

**Configuration** (.env):
```bash
GOOGLE_OAUTH_CLIENT_ID=your-id
GOOGLE_OAUTH_CLIENT_SECRET=your-secret
OAUTHLIB_INSECURE_TRANSPORT=1
MCP_ENABLE_OAUTH21=true
TOOL_TIER=extended
```

**docker-compose.yml additions**:
```yaml
services:
  gws_mcp:
    # ... existing config
    environment:
      - MCP_ENABLE_OAUTH21=true
```

### Scenario 4: Production with Reverse Proxy

**Use case**: Running behind nginx/Caddy with HTTPS.

**Configuration** (.env):
```bash
GOOGLE_OAUTH_CLIENT_ID=your-id
GOOGLE_OAUTH_CLIENT_SECRET=your-secret
WORKSPACE_EXTERNAL_URL=https://your-domain.com
MCP_ENABLE_OAUTH21=true
TOOL_TIER=complete
```

**Nginx config example**:
```nginx
server {
    listen 443 ssl;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Scenario 5: Stateless Container (Kubernetes/Cloud)

**Use case**: Running in ephemeral environments with no persistent storage.

**Configuration** (.env):
```bash
GOOGLE_OAUTH_CLIENT_ID=your-id
GOOGLE_OAUTH_CLIENT_SECRET=your-secret
MCP_ENABLE_OAUTH21=true
WORKSPACE_MCP_STATELESS_MODE=true
EXTERNAL_OAUTH21_PROVIDER=true
```

**docker-compose.yml**:
```yaml
services:
  gws_mcp:
    # ... existing config
    volumes:
      # Remove credential storage volume
      - ./client_secret.json:/app/client_secret.json:ro
    environment:
      - MCP_ENABLE_OAUTH21=true
      - WORKSPACE_MCP_STATELESS_MODE=true
      - EXTERNAL_OAUTH21_PROVIDER=true
```

### Scenario 6: With Valkey/Redis for Distributed Setup

**Use case**: Multiple server instances sharing OAuth state.

**docker-compose.yml**:
```yaml
services:
  redis:
    image: valkey/valkey:latest
    container_name: valkey
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

  gws_mcp:
    # ... existing config
    depends_on:
      - redis
    environment:
      - MCP_ENABLE_OAUTH21=true
      - WORKSPACE_MCP_OAUTH_PROXY_STORAGE_BACKEND=valkey
      - WORKSPACE_MCP_OAUTH_PROXY_VALKEY_HOST=redis
      - WORKSPACE_MCP_OAUTH_PROXY_VALKEY_PORT=6379

volumes:
  redis_data:
  store_creds:
```

## Using Pre-built Images

The GitHub Actions workflow builds and publishes multi-platform Docker images on every release.

### Available Tags

- `latest` - Latest stable release
- `v1.7.1` - Specific version
- `1.7` - Major.minor version
- `1` - Major version

### Pull and Run

```bash
# Pull latest
docker pull ghcr.io/jonhearsch/google_workspace_mcp:latest

# Pull specific version
docker pull ghcr.io/jonhearsch/google_workspace_mcp:v1.7.1

# Run with environment variables
docker run -d \
  --name gws_mcp \
  -p 8000:8000 \
  -e GOOGLE_OAUTH_CLIENT_ID="your-id" \
  -e GOOGLE_OAUTH_CLIENT_SECRET="your-secret" \
  -e OAUTHLIB_INSECURE_TRANSPORT=1 \
  -e TOOL_TIER=core \
  -v $(pwd)/client_secret.json:/app/client_secret.json:ro \
  -v gws_creds:/app/store_creds \
  ghcr.io/jonhearsch/google_workspace_mcp:latest

# View logs
docker logs -f gws_mcp

# Stop
docker stop gws_mcp
docker rm gws_mcp
```

## Accessing the Server

### Health Check

```bash
curl http://localhost:8000/health
```

### MCP Endpoint

```bash
# For HTTP transport mode
curl http://localhost:8000/mcp/
```

### Configure MCP Clients

**Claude Code**:
```bash
claude mcp add --transport http workspace-mcp http://localhost:8000/mcp
```

**VS Code** (settings.json):
```json
{
    "servers": {
        "google-workspace": {
            "url": "http://localhost:8000/mcp/",
            "type": "http"
        }
    }
}
```

## Troubleshooting

### Container won't start

```bash
# Check logs
docker-compose logs gws_mcp

# Common issues:
# 1. Missing environment variables - check .env file
# 2. Port 8000 already in use - change PORT in .env
# 3. Invalid OAuth credentials - verify in Google Cloud Console
```

### Authentication failures

```bash
# Check if credentials directory has proper permissions
docker-compose exec gws_mcp ls -la /app/store_creds

# Restart container to clear credential cache
docker-compose restart gws_mcp
```

### Health check failing

```bash
# Test manually
docker-compose exec gws_mcp curl -f http://localhost:8000/health

# Check if port is exposed correctly
docker-compose ps
```

## Advanced Usage

### Custom Port

```bash
# In .env
PORT=9000

# Restart
docker-compose down
docker-compose up -d
```

### View Real-time Logs

```bash
docker-compose logs -f gws_mcp
```

### Execute Commands Inside Container

```bash
# Get a shell
docker-compose exec gws_mcp /bin/bash

# Check Python version
docker-compose exec gws_mcp python --version

# List installed packages
docker-compose exec gws_mcp uv pip list
```

### Resource Limits

Edit `docker-compose.enhanced.yml`:

```yaml
deploy:
  resources:
    limits:
      cpus: '4'        # 4 CPU cores max
      memory: 2G       # 2GB RAM max
    reservations:
      cpus: '1'        # Guaranteed 1 core
      memory: 512M     # Guaranteed 512MB
```

## Security Best Practices

1. **Never commit credentials**:
   - Add `.env` to `.gitignore`
   - Never commit `client_secret.json`

2. **Use secrets management**:
   ```bash
   # Docker secrets (Swarm mode)
   docker secret create google_client_id client_id.txt
   docker secret create google_client_secret client_secret.txt
   ```

3. **Run behind reverse proxy with HTTPS** in production

4. **Limit container resources** to prevent DoS

5. **Regular updates**:
   ```bash
   docker-compose pull
   docker-compose up -d
   ```

## GitHub Actions Workflow

The repository includes a GitHub Actions workflow that:

1. **Builds** multi-platform Docker images (amd64, arm64)
2. **Publishes** to GitHub Container Registry
3. **Tags** with semver patterns
4. **Caches** layers for faster builds

**Trigger a build**:
- Create a release on GitHub
- Or manually trigger: Actions → Build and Publish Docker Image → Run workflow

**Automated upstream sync**:
- Runs every 6 hours
- Checks for new releases from upstream
- Automatically creates matching releases
- Triggers Docker build via release event
