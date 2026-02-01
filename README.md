# OpenClaw + Azure OpenAI

Connect [OpenClaw](https://github.com/openclaw/openclaw) to your Azure OpenAI deployments.

## Why Azure OpenAI?

- **Enterprise security**: Private endpoints, VNETs, managed identity
- **Regional compliance**: Data residency in your chosen Azure region
- **Consistent billing**: Through your existing Azure subscription
- **Content filtering**: Built-in content moderation

## Quick Start

OpenClaw uses the `azure-openai-responses` API type with environment variables for Azure configuration.

### 1. Get Your Azure OpenAI Details

From the Azure Portal:
1. Go to your **Azure OpenAI resource**
2. Navigate to **Keys and Endpoint**
3. Note the **Endpoint** (e.g., `https://your-resource.openai.azure.com`)
4. Copy one of the **Keys**
5. Note your **deployment name(s)** from Model deployments

### 2. Set Environment Variables

```bash
# Required
export AZURE_OPENAI_API_KEY="your-azure-api-key"
export AZURE_OPENAI_RESOURCE_NAME="your-resource-name"  # Just the name, not the full URL

# Optional (defaults to "v1")
export AZURE_OPENAI_API_VERSION="2024-02-01"

# Optional: Map model IDs to deployment names if they differ
# Format: model1=deployment1,model2=deployment2
export AZURE_OPENAI_DEPLOYMENT_NAME_MAP="gpt-4o=my-gpt4o-deployment,gpt-4o-mini=gpt4o-mini-prod"
```

Or in your `~/.openclaw/config.json5`:

```json5
{
  env: {
    vars: {
      AZURE_OPENAI_API_KEY: "your-azure-api-key",
      AZURE_OPENAI_RESOURCE_NAME: "your-resource-name",
      AZURE_OPENAI_API_VERSION: "2024-02-01",
      AZURE_OPENAI_DEPLOYMENT_NAME_MAP: "gpt-4o=my-gpt4o-deployment"
    }
  }
}
```

### 3. Configure OpenClaw

Add to your `~/.openclaw/config.json5`:

```json5
{
  models: {
    providers: {
      "azure-openai": {
        // No baseUrl needed - constructed from AZURE_OPENAI_RESOURCE_NAME
        api: "azure-openai-responses",
        models: [
          {
            id: "gpt-4o",  // Used to look up deployment name from AZURE_OPENAI_DEPLOYMENT_NAME_MAP
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

### 4. Restart OpenClaw

```bash
openclaw gateway restart
```

## Configuration Options

### URL Resolution Priority

The Azure client resolves the base URL in this order:
1. `azureBaseUrl` option (runtime)
2. `AZURE_OPENAI_BASE_URL` environment variable
3. Constructed from `AZURE_OPENAI_RESOURCE_NAME` → `https://{name}.openai.azure.com/openai/v1`
4. `model.baseUrl` from config

### Deployment Name Mapping

If your Azure deployment names differ from model IDs, use the mapping:

```bash
# Environment variable
export AZURE_OPENAI_DEPLOYMENT_NAME_MAP="gpt-4o=prod-gpt4o,gpt-4o-mini=prod-mini"
```

The format is `modelId=deploymentName`, comma-separated for multiple mappings.

If not mapped, the model ID is used as the deployment name.

### Multiple Deployments

Define models for each deployment:

```json5
{
  models: {
    providers: {
      "azure-openai": {
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
          },
          {
            id: "o1",
            name: "o1 (Azure)",
            contextWindow: 200000,
            maxTokens: 100000,
            reasoning: true,
            input: ["text", "image"]
          }
        ]
      }
    }
  }
}
```

Then set the deployment mapping:
```bash
export AZURE_OPENAI_DEPLOYMENT_NAME_MAP="gpt-4o=gpt4o-prod,gpt-4o-mini=mini-prod,o1=o1-preview"
```

### Custom Base URL

If you need to override the default URL construction:

```bash
export AZURE_OPENAI_BASE_URL="https://my-custom-endpoint.openai.azure.com/openai/v1"
```

Note: Don't include the `api-version` query parameter in the base URL — the client adds it automatically.

### Custom Headers

For additional headers (e.g., custom routing):

```json5
{
  models: {
    providers: {
      "azure-openai": {
        api: "azure-openai-responses",
        headers: {
          "X-Custom-Header": "value"
        },
        models: [/* ... */]
      }
    }
  }
}
```

## ⚠️ Important: Don't Use `openai-completions`

**Do not use `api: "openai-completions"` with Azure OpenAI.**

The standard OpenAI client appends `/chat/completions` directly to the base URL, which breaks Azure's URL structure that requires `?api-version=` as a query parameter.

Always use `api: "azure-openai-responses"` for Azure OpenAI — it uses the proper `AzureOpenAI` client from the OpenAI SDK that handles Azure's URL format correctly.

## Troubleshooting

### "Resource not found" (404)

- Verify your resource name is correct (just the name, not full URL)
- Check the deployment name mapping matches your Azure deployment
- Ensure the model is actually deployed in Azure

### "Invalid API Key" (401)

- Confirm `AZURE_OPENAI_API_KEY` is set correctly
- Check the key is from the correct Azure OpenAI resource

### "API version not supported"

- Update `AZURE_OPENAI_API_VERSION` to a supported version
- Check [Azure OpenAI API versions](https://learn.microsoft.com/en-us/azure/ai-services/openai/reference) for current options

### Model Mismatch

Use `AZURE_OPENAI_DEPLOYMENT_NAME_MAP` to map between OpenClaw model IDs and your Azure deployment names.

## Complete Example

```json5
// ~/.openclaw/config.json5
{
  env: {
    vars: {
      AZURE_OPENAI_API_KEY: "abc123...",
      AZURE_OPENAI_RESOURCE_NAME: "contoso-openai",
      AZURE_OPENAI_API_VERSION: "2024-02-01",
      AZURE_OPENAI_DEPLOYMENT_NAME_MAP: "gpt-4o=gpt-4o-prod,gpt-4o-mini=gpt-4o-mini-prod"
    }
  },
  
  models: {
    providers: {
      "azure": {
        api: "azure-openai-responses",
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
          },
          {
            id: "gpt-4o-mini",
            name: "GPT-4o Mini (Azure Prod)",
            contextWindow: 128000,
            maxTokens: 16384,
            input: ["text", "image"],
            cost: {
              input: 0.15,
              output: 0.60
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
        fallbacks: ["azure/gpt-4o-mini", "openai/gpt-4o"]
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
