# Deployment Quick Reference

Quick reference for deploying Google Workspace MCP Server in different scenarios.

## 📦 Available Configurations

| Use Case | File | Best For |
|----------|------|----------|
| **Basic Docker** | `docker-compose.yml` | Simple standalone deployment |
| **Enhanced Docker** | `docker-compose.enhanced.yml` | Advanced features, multi-scenario |
| **Open WebUI** | `docker-compose.openwebui.yml` | Interactive chat UI |
| **LiteLLM** | `docker-compose.litellm.yml` | **Personal MCP proxy (Recommended)** |

## 🚀 Quick Start Commands

### 1. LiteLLM (Recommended for Personal Use)

**Best for:** API access, automation, personal assistant, multiple LLM providers

```bash
# Setup
cp .env.litellm.example .env
# Edit .env with your credentials

# Start
docker-compose -f docker-compose.litellm.yml up -d

# Test
curl http://localhost:4000/health

# Use with any OpenAI-compatible client
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-1234" \
  -d '{"model": "gpt-4o-mini", "messages": [{"role": "user", "content": "List my recent emails"}]}'
```

**Features:**
- ✅ OpenAI-compatible API
- ✅ Multiple LLM providers (OpenAI, Claude, Ollama)
- ✅ Automatic caching and cost tracking
- ✅ Fallback support
- ✅ Single-user authentication (simple)

📖 [Full Guide](./LITELLM_SETUP.md)

---

### 2. Open WebUI

**Best for:** Interactive chat interface, visual UI, team collaboration

```bash
# Setup
cp .env.docker.example .env
# Edit .env with your credentials

# Start
docker-compose -f docker-compose.openwebui.yml up -d

# Access
open http://localhost:3000
```

**Features:**
- ✅ Beautiful chat UI
- ✅ Multi-user support
- ✅ Document attachments
- ✅ Chat history

📖 [Full Guide](./OPENWEBUI_SETUP.md)

---

### 3. Basic Standalone

**Best for:** Direct MCP integration, Claude Desktop, VS Code

```bash
# Setup
cp .env.docker.example .env
# Edit .env

# Start
docker-compose up -d

# Configure in Claude Desktop
# Add MCP server: http://localhost:8000/mcp
```

**Features:**
- ✅ Minimal setup
- ✅ Direct MCP protocol
- ✅ Claude Desktop integration
- ✅ VS Code integration

📖 [Full Guide](./DOCKER_SETUP.md)

---

## 🔧 Configuration Comparison

### Authentication Mode

| Setup | Auth Mode | Multi-User | Best For |
|-------|-----------|------------|----------|
| **LiteLLM** | Single-user | No | Personal API access |
| **Open WebUI** | Single-user* | Limited | Personal/team UI |
| **Basic** | Single-user | No | Direct MCP clients |

*Open WebUI multi-user support depends on current feature availability

### Tool Selection

All setups support the same tool selection methods:

```bash
# Method 1: Tool Tiers (recommended)
TOOL_TIER=core      # Essential tools only
TOOL_TIER=extended  # Core + additional
TOOL_TIER=complete  # Everything

# Method 2: Specific Tools
TOOLS=gmail,drive,calendar  # Only these tools
```

### Port Mapping

| Service | Port | Protocol |
|---------|------|----------|
| Google Workspace MCP | 8000 | HTTP |
| LiteLLM Proxy | 4000 | HTTP (OpenAI-compatible) |
| Open WebUI | 3000 | HTTP |
| PostgreSQL | 5432 | TCP |
| Redis | 6379 | TCP |

## 📝 Environment Files

| File | Purpose |
|------|---------|
| `.env.docker.example` | General Docker deployments |
| `.env.litellm.example` | LiteLLM-specific settings |

**Setup:**
```bash
# For LiteLLM
cp .env.litellm.example .env

# For Open WebUI or basic Docker
cp .env.docker.example .env
```

## 🔐 Required Credentials

### Minimum Required (All Setups)

```bash
GOOGLE_OAUTH_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=your-client-secret
USER_GOOGLE_EMAIL=your.email@gmail.com
```

### Additional for LiteLLM

At least one LLM provider API key:

```bash
OPENAI_API_KEY=sk-...          # For GPT models
# OR
ANTHROPIC_API_KEY=sk-ant-...   # For Claude models
# OR use local Ollama (no key needed)
```

## 🎯 Decision Tree

```
Do you need an API for automation?
├─ YES → Use LiteLLM
│         ✓ OpenAI-compatible API
│         ✓ Multiple providers
│         ✓ Cost tracking
│
└─ NO → Do you want a chat UI?
         ├─ YES → Use Open WebUI
         │         ✓ Beautiful interface
         │         ✓ Chat history
         │
         └─ NO → Do you use Claude Desktop or VS Code?
                  └─ YES → Use Basic Docker
                            ✓ Direct MCP integration
```

## 📊 Feature Matrix

| Feature | Basic | Enhanced | Open WebUI | LiteLLM |
|---------|-------|----------|------------|---------|
| Google Workspace Tools | ✅ | ✅ | ✅ | ✅ |
| Single-user Auth | ✅ | ✅ | ✅ | ✅ |
| Multi-user Auth | ❌ | ✅ | Limited | ❌ |
| HTTP API | ✅ | ✅ | ❌ | ✅ |
| Chat UI | ❌ | ❌ | ✅ | ✅* |
| Multiple LLM Providers | ❌ | ❌ | Limited | ✅ |
| Caching | ❌ | ❌ | ❌ | ✅ |
| Cost Tracking | ❌ | ❌ | ❌ | ✅ |
| Fallback Providers | ❌ | ❌ | ❌ | ✅ |
| Health Checks | ✅ | ✅ | ✅ | ✅ |
| Resource Limits | ❌ | ✅ | ✅ | ✅ |
| Production Ready | ✅ | ✅ | ✅ | ✅ |

*LiteLLM has optional UI at http://localhost:4000/ui

## 🔍 Use Case Recommendations

### Personal AI Assistant
**→ Use LiteLLM**
- Automate email management
- Schedule calendar events
- Manage documents
- Cost-effective with caching

### Team Collaboration
**→ Use Open WebUI**
- Shared chat interface
- User accounts
- Conversation history

### Development & Testing
**→ Use Basic Docker**
- Direct MCP integration
- Test with Claude Desktop
- VS Code development

### Production API
**→ Use LiteLLM + Enhanced Setup**
- Add PostgreSQL for logging
- Add Redis for caching
- Enable monitoring
- Use reverse proxy (Caddy/nginx)

## 🚨 Troubleshooting

### Common Issues

**Service won't start:**
```bash
# Check logs
docker-compose -f <your-compose-file> logs -f

# Verify environment
cat .env

# Check health
curl http://localhost:8000/health  # MCP server
curl http://localhost:4000/health  # LiteLLM (if using)
```

**Authentication failing:**
```bash
# Clear credentials and re-authenticate
docker-compose exec gws_mcp rm -rf /app/store_creds/*
docker-compose restart gws_mcp
docker-compose logs -f gws_mcp
# Follow OAuth URL in logs
```

**Can't connect from client:**
```bash
# Verify network
docker network ls
docker network inspect <network_name>

# Test connectivity
docker exec <client-container> curl http://gws_mcp:8000/health
```

## 📚 Full Documentation

- **[LITELLM_SETUP.md](./LITELLM_SETUP.md)** - Complete LiteLLM integration guide
- **[OPENWEBUI_SETUP.md](./OPENWEBUI_SETUP.md)** - Open WebUI deployment guide
- **[DOCKER_SETUP.md](./DOCKER_SETUP.md)** - Comprehensive Docker guide (6 scenarios)
- **[REVIEW_SUMMARY.md](./REVIEW_SUMMARY.md)** - Architecture review and recommendations
- **[README.md](./README.md)** - Main project documentation

## 💡 Quick Tips

1. **Start with LiteLLM** if you want API access - it's the most flexible
2. **Use `TOOL_TIER=core`** to start - reduces API quota usage
3. **Enable caching** in production (requires Redis)
4. **Monitor Google API quotas** in Cloud Console
5. **Use HTTPS** in production (add Caddy or nginx)
6. **Regular updates**: `docker-compose pull && docker-compose up -d`

## 🎓 Example: Personal Setup (Recommended)

```bash
# 1. Clone repository
git clone https://github.com/jonhearsch/google_workspace_mcp.git
cd google_workspace_mcp

# 2. Configure
cp .env.litellm.example .env
# Edit .env with:
# - GOOGLE_OAUTH_CLIENT_ID
# - GOOGLE_OAUTH_CLIENT_SECRET
# - USER_GOOGLE_EMAIL
# - OPENAI_API_KEY (or use Ollama for free local models)

# 3. Start
docker-compose -f docker-compose.litellm.yml up -d

# 4. Authenticate (one-time)
docker-compose -f docker-compose.litellm.yml logs -f gws_mcp
# Open OAuth URL shown in logs

# 5. Test
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-1234" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {"role": "user", "content": "What meetings do I have today?"}
    ]
  }'

# 6. Use in your applications
# Point any OpenAI-compatible client to http://localhost:4000/v1
```

## 🌟 Next Steps

1. Choose your deployment method above
2. Follow the corresponding guide
3. Test with the examples provided
4. Integrate with your applications
5. Star the repo if it helps! ⭐

---

**Need help?** Check the full guides or open an issue with your logs.
