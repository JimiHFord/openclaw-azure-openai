# OpenClaw + Azure OpenAI

Connect [OpenClaw](https://github.com/openclaw/openclaw) to your Azure OpenAI deployments using custom base URLs.

## Why Azure OpenAI?

- **Enterprise security**: Private endpoints, VNETs, managed identity
- **Regional compliance**: Data residency in your chosen Azure region
- **Consistent billing**: Through your existing Azure subscription
- **Content filtering**: Built-in content moderation

## Quick Start

### 1. Get Your Azure OpenAI Endpoint

From the Azure Portal:
1. Go to your **Azure OpenAI resource**
2. Navigate to **Keys and Endpoint**
3. Copy the **Endpoint** (e.g., `https://your-resource.openai.azure.com`)
4. Copy one of the **Keys**

### 2. Configure OpenClaw

Add to your `~/.openclaw/config.json5`:

```json5
{
  models: {
    providers: {
      "azure-openai": {
        baseUrl: "https://YOUR-RESOURCE.openai.azure.com/openai/deployments/YOUR-DEPLOYMENT-NAME",
        apiKey: "${AZURE_OPENAI_API_KEY}",
        api: "openai-completions",
        headers: {
          "api-key": "${AZURE_OPENAI_API_KEY}"
        },
        authHeader: false,  // Azure uses api-key header, not Authorization
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
      model: {
        primary: "azure-openai/gpt-4o"
      }
    }
  }
}
```

### 3. Set Environment Variable

```bash
export AZURE_OPENAI_API_KEY="your-azure-api-key"
```

Or add to your `~/.openclaw/config.json5`:

```json5
{
  env: {
    vars: {
      AZURE_OPENAI_API_KEY: "your-azure-api-key"
    }
  }
}
```

### 4. Restart OpenClaw

```bash
openclaw gateway restart
```

## Configuration Options

### Base URL Format

Azure OpenAI uses a different URL structure than OpenAI:

```
https://{resource-name}.openai.azure.com/openai/deployments/{deployment-name}
```

The full chat completions endpoint becomes:
```
https://{resource-name}.openai.azure.com/openai/deployments/{deployment-name}/chat/completions?api-version=2024-02-01
```

OpenClaw appends `/chat/completions` automatically for the `openai-completions` API.

### API Version

Azure OpenAI requires an `api-version` query parameter. Add it to your config:

```json5
{
  models: {
    providers: {
      "azure-openai": {
        baseUrl: "https://YOUR-RESOURCE.openai.azure.com/openai/deployments/YOUR-DEPLOYMENT-NAME?api-version=2024-02-01",
        // ... rest of config
      }
    }
  }
}
```

### Multiple Deployments

If you have multiple Azure OpenAI deployments, create a provider for each:

```json5
{
  models: {
    providers: {
      "azure-gpt4o": {
        baseUrl: "https://my-resource.openai.azure.com/openai/deployments/gpt-4o-deployment?api-version=2024-02-01",
        apiKey: "${AZURE_OPENAI_API_KEY}",
        api: "openai-completions",
        headers: { "api-key": "${AZURE_OPENAI_API_KEY}" },
        authHeader: false,
        models: [
          { id: "gpt-4o", name: "GPT-4o (Azure)", contextWindow: 128000, maxTokens: 16384 }
        ]
      },
      "azure-gpt4o-mini": {
        baseUrl: "https://my-resource.openai.azure.com/openai/deployments/gpt-4o-mini-deployment?api-version=2024-02-01",
        apiKey: "${AZURE_OPENAI_API_KEY}",
        api: "openai-completions",
        headers: { "api-key": "${AZURE_OPENAI_API_KEY}" },
        authHeader: false,
        models: [
          { id: "gpt-4o-mini", name: "GPT-4o Mini (Azure)", contextWindow: 128000, maxTokens: 16384 }
        ]
      }
    }
  }
}
```

Then reference as `azure-gpt4o/gpt-4o` or `azure-gpt4o-mini/gpt-4o-mini`.

## Authentication Methods

### API Key (Most Common)

```json5
{
  headers: {
    "api-key": "${AZURE_OPENAI_API_KEY}"
  },
  authHeader: false  // Don't send Authorization header
}
```

### Managed Identity (Azure AD Token)

For managed identity, you'll need to fetch a token and inject it:

```json5
{
  headers: {
    "Authorization": "Bearer ${AZURE_AD_TOKEN}"
  },
  authHeader: false
}
```

> **Note**: Token refresh is not yet built into OpenClaw. For production use with managed identity, consider using a token proxy.

## Troubleshooting

### "Resource not found" (404)

- Verify your deployment name matches exactly
- Check the resource name in your endpoint URL
- Ensure the model is actually deployed in Azure

### "Invalid API Key" (401)

- Confirm you're using the correct key from Azure Portal
- Make sure you're using `api-key` header (not `Authorization: Bearer`)
- Check that `authHeader: false` is set

### "API version not supported"

- Update the `api-version` query parameter to a supported version
- Check [Azure OpenAI API versions](https://learn.microsoft.com/en-us/azure/ai-services/openai/reference) for current options

### Model Mismatch

Azure deployments have their own names. The model in your config should match what you want OpenClaw to call it, but the deployment name in the URL is what Azure actually uses.

## Complete Example

```json5
// ~/.openclaw/config.json5
{
  env: {
    vars: {
      AZURE_OPENAI_API_KEY: "abc123..."  // Or use environment variable
    }
  },
  models: {
    providers: {
      "azure": {
        baseUrl: "https://contoso-openai.openai.azure.com/openai/deployments/gpt-4o-prod?api-version=2024-02-01",
        apiKey: "${AZURE_OPENAI_API_KEY}",
        api: "openai-completions",
        headers: {
          "api-key": "${AZURE_OPENAI_API_KEY}"
        },
        authHeader: false,
        models: [
          {
            id: "gpt-4o",
            name: "GPT-4o (Azure Prod)",
            contextWindow: 128000,
            maxTokens: 16384,
            input: ["text", "image"],
            cost: {
              input: 2.50,   // per 1M tokens
              output: 10.00  // per 1M tokens
            }
          }
        ]
      }
    }
  },
  agents: {
    defaults: {
      model: {
        primary: "azure/gpt-4o",
        fallbacks: ["openai/gpt-4o"]  // Optional: fallback to direct OpenAI
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
