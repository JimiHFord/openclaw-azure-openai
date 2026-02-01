# OpenClaw + Azure OpenAI

Connect [OpenClaw](https://github.com/openclaw/openclaw) to Azure OpenAI deployments — including those behind custom domains or API gateways.

## Why Azure OpenAI?

- **Enterprise security**: Private endpoints, VNETs, managed identity
- **Regional compliance**: Data residency in your chosen Azure region
- **Custom domains**: Route through your own API gateway or proxy
- **Consistent billing**: Through your existing Azure subscription

## Quick Start

### 1. Set Environment Variables

```bash
# Required
export AZURE_OPENAI_API_KEY="your-azure-api-key"
export AZURE_OPENAI_API_VERSION="2024-02-01"

# For standard Azure endpoints:
export AZURE_OPENAI_RESOURCE_NAME="your-resource-name"

# OR for custom domains:
export AZURE_OPENAI_BASE_URL="https://your-custom-domain.company.com/openai/v1"

# Map model IDs to deployment names (if they differ)
export AZURE_OPENAI_DEPLOYMENT_NAME_MAP="gpt-4o=my-gpt4o-deployment"
```

### 2. Configure OpenClaw

Add to `~/.openclaw/config.json5`:

```json5
{
  models: {
    providers: {
      "azure": {
        api: "azure-openai-responses",
        models: [
          {
            id: "gpt-4o",
            name: "GPT-4o (Azure)",
            contextWindow: 128000,
            maxTokens: 16384,
            input: ["text", "image"]
          }
        ]
      }
    }
  },
  agents: {
    defaults: {
      model: { primary: "azure/gpt-4o" }
    }
  }
}
```

### 3. Restart OpenClaw

```bash
openclaw gateway restart
```

## Custom Domain / API Gateway Setup

If your Azure OpenAI is behind a custom domain (e.g., `https://ai.company.com`), use `AZURE_OPENAI_BASE_URL`:

```bash
export AZURE_OPENAI_BASE_URL="https://ai.company.com/openai/v1"
export AZURE_OPENAI_API_KEY="your-key"
export AZURE_OPENAI_API_VERSION="2024-02-01"
export AZURE_OPENAI_DEPLOYMENT_NAME_MAP="gpt-4o=prod-gpt4o"
```

The client constructs the full URL as:
```
{BASE_URL}/deployments/{DEPLOYMENT}/chat/completions?api-version={VERSION}
↓
https://ai.company.com/openai/v1/deployments/prod-gpt4o/chat/completions?api-version=2024-02-01
```

### Config-Only Approach

You can also set everything in config instead of environment variables:

```json5
{
  env: {
    vars: {
      AZURE_OPENAI_API_KEY: "your-key",
      AZURE_OPENAI_API_VERSION: "2024-02-01",
      AZURE_OPENAI_BASE_URL: "https://ai.company.com/openai/v1",
      AZURE_OPENAI_DEPLOYMENT_NAME_MAP: "gpt-4o=prod-gpt4o,gpt-4o-mini=prod-mini"
    }
  },
  
  models: {
    providers: {
      "azure": {
        api: "azure-openai-responses",
        models: [
          {
            id: "gpt-4o",
            name: "GPT-4o (Azure)",
            contextWindow: 128000,
            maxTokens: 16384,
            input: ["text", "image"]
          },
          {
            id: "gpt-4o-mini", 
            name: "GPT-4o Mini (Azure)",
            contextWindow: 128000,
            maxTokens: 16384,
            input: ["text", "image"]
          }
        ]
      }
    }
  },
  
  agents: {
    defaults: {
      model: {
        primary: "azure/gpt-4o",
        fallbacks: ["azure/gpt-4o-mini"]
      }
    }
  }
}
```

## URL Construction

The `AzureOpenAI` client constructs URLs like this:

| Setting | Value |
|---------|-------|
| `AZURE_OPENAI_BASE_URL` | `https://ai.company.com/openai/v1` |
| `AZURE_OPENAI_DEPLOYMENT_NAME_MAP` | `gpt-4o=prod-gpt4o` |
| `AZURE_OPENAI_API_VERSION` | `2024-02-01` |
| **Result** | `https://ai.company.com/openai/v1/deployments/prod-gpt4o/chat/completions?api-version=2024-02-01` |

### Standard vs Custom Domain

| Scenario | Base URL |
|----------|----------|
| Standard Azure | Set `AZURE_OPENAI_RESOURCE_NAME=myresource` → `https://myresource.openai.azure.com/openai/v1` |
| Custom domain | Set `AZURE_OPENAI_BASE_URL=https://ai.company.com/openai/v1` |

## Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `AZURE_OPENAI_API_KEY` | Yes | Your Azure OpenAI API key |
| `AZURE_OPENAI_API_VERSION` | Yes | API version (e.g., `2024-02-01`) |
| `AZURE_OPENAI_RESOURCE_NAME` | One of these | Standard Azure resource name |
| `AZURE_OPENAI_BASE_URL` | One of these | Custom base URL for API gateway setups |
| `AZURE_OPENAI_DEPLOYMENT_NAME_MAP` | No | Map model IDs to deployment names |

## Deployment Name Mapping

If your Azure deployment names differ from the model IDs in your config:

```bash
# Format: modelId=deploymentName,modelId2=deploymentName2
export AZURE_OPENAI_DEPLOYMENT_NAME_MAP="gpt-4o=production-gpt4o,gpt-4o-mini=production-mini,o1=o1-preview-v1"
```

If not set, the model ID is used as the deployment name.

## ⚠️ Important: Use the Right API Type

**Always use `api: "azure-openai-responses"`** for Azure OpenAI.

Do **not** use `api: "openai-completions"` — it doesn't understand Azure's URL format and will break.

## Custom Headers

If your API gateway requires additional headers:

```json5
{
  models: {
    providers: {
      "azure": {
        api: "azure-openai-responses",
        headers: {
          "X-Custom-Header": "value",
          "X-API-Gateway-Key": "${MY_GATEWAY_KEY}"
        },
        models: [/* ... */]
      }
    }
  }
}
```

## Troubleshooting

### "Resource not found" (404)

- Check `AZURE_OPENAI_BASE_URL` format — should end with `/openai/v1`
- Verify deployment name mapping is correct
- Test the full URL manually with curl

### "Invalid API Key" (401)

- Confirm `AZURE_OPENAI_API_KEY` is set
- If using a gateway, check if it expects a different auth header

### URL looks wrong

Debug the constructed URL by checking OpenClaw logs:
```bash
openclaw gateway logs | grep -i azure
```

### Test with curl

```bash
curl -X POST "https://ai.company.com/openai/v1/deployments/prod-gpt4o/chat/completions?api-version=2024-02-01" \
  -H "Content-Type: application/json" \
  -H "api-key: $AZURE_OPENAI_API_KEY" \
  -d '{"messages": [{"role": "user", "content": "Hello"}], "max_tokens": 10}'
```

## Complete Example (Custom Domain)

```json5
// ~/.openclaw/config.json5
{
  env: {
    vars: {
      AZURE_OPENAI_API_KEY: "abc123...",
      AZURE_OPENAI_API_VERSION: "2024-02-01",
      AZURE_OPENAI_BASE_URL: "https://ai.mycompany.com/openai/v1",
      AZURE_OPENAI_DEPLOYMENT_NAME_MAP: "gpt-4o=prod-gpt4o,gpt-4o-mini=prod-mini"
    }
  },
  
  models: {
    providers: {
      "azure": {
        api: "azure-openai-responses",
        models: [
          {
            id: "gpt-4o",
            name: "GPT-4o (Azure)",
            contextWindow: 128000,
            maxTokens: 16384,
            input: ["text", "image"],
            cost: { input: 2.50, output: 10.00 }
          },
          {
            id: "gpt-4o-mini",
            name: "GPT-4o Mini (Azure)",
            contextWindow: 128000,
            maxTokens: 16384,
            input: ["text", "image"],
            cost: { input: 0.15, output: 0.60 }
          }
        ]
      }
    }
  },
  
  agents: {
    defaults: {
      model: {
        primary: "azure/gpt-4o",
        fallbacks: ["azure/gpt-4o-mini"]
      }
    }
  }
}
```

## Resources

- [OpenClaw Documentation](https://docs.openclaw.ai)
- [Azure OpenAI Service Documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/)
- [Azure OpenAI API Reference](https://learn.microsoft.com/en-us/azure/ai-services/openai/reference)
- [OpenClaw Discord](https://discord.com/invite/clawd)

## License

MIT
