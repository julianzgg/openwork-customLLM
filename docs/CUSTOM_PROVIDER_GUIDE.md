# Custom Provider Integration Guide

This guide explains how to use the Custom Provider feature in Openwork to connect to any OpenAI-compatible API endpoint.

## What is Custom Provider?

The Custom Provider allows you to connect Openwork to any third-party AI service that implements the OpenAI-compatible API standard. This includes:

- Self-hosted models (vLLM, Text Generation Inference, LocalAI, etc.)
- Custom API gateways
- Enterprise AI platforms
- Any service with OpenAI-compatible endpoints

## How to Configure Custom Provider

### 1. Start Your API Server

First, ensure your OpenAI-compatible API server is running. For example:

#### Using vLLM
```bash
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-2-7b-chat-hf \
  --port 8000
```

#### Using Text Generation Inference (TGI)
```bash
docker run --gpus all --shm-size 1g -p 8080:80 \
  ghcr.io/huggingface/text-generation-inference:latest \
  --model-id meta-llama/Llama-2-7b-chat-hf
```

#### Using LocalAI
```bash
docker run -p 8080:8080 \
  -v $PWD/models:/models \
  localai/localai:latest
```

### 2. Configure in Openwork

1. Open Openwork application
2. Go to **Settings**
3. Find **Custom Provider** in the provider list
4. Fill in the configuration:

   - **Provider Name**: A friendly name for your provider (e.g., "My vLLM Server")
   - **Server URL**: The base URL of your API server
     - Examples:
       - `http://localhost:8000` (vLLM)
       - `http://localhost:8080/v1` (LocalAI)
       - `https://api.mycompany.com/v1` (Enterprise API)
   - **API Key**: (Optional) If your server requires authentication

5. Click **Connect**

### 3. Select a Model

After connecting, you'll need to manually enter the model ID:

- For vLLM: Use the model name you started the server with (e.g., `meta-llama/Llama-2-7b-chat-hf`)
- For LocalAI: Use the model name from your configuration
- For custom APIs: Check your API documentation for available model IDs

## Configuration Examples

### Example 1: Local vLLM Server

```
Provider Name: vLLM Local
Server URL: http://localhost:8000
API Key: (leave empty)
Model ID: meta-llama/Llama-2-7b-chat-hf
```

### Example 2: Text Generation Inference

```
Provider Name: TGI Server
Server URL: http://localhost:8080
API Key: (leave empty)
Model ID: tgi
```

### Example 3: Enterprise API with Authentication

```
Provider Name: Company AI Gateway
Server URL: https://ai-api.company.com/v1
API Key: sk-company-key-xxxxx
Model ID: gpt-4-company
```

### Example 4: LocalAI

```
Provider Name: LocalAI
Server URL: http://localhost:8080/v1
API Key: (leave empty)
Model ID: gpt-3.5-turbo (or your configured model name)
```

## Technical Details

### How It Works

When you configure a Custom Provider, Openwork:

1. Saves your configuration securely (API keys are stored in OS keychain)
2. Generates an OpenCode configuration that uses `@ai-sdk/openai-compatible` adapter
3. Routes all AI requests through your custom endpoint

### API Compatibility Requirements

Your API server must implement these OpenAI-compatible endpoints:

- `POST /v1/chat/completions` - For chat interactions
- `GET /v1/models` - (Optional) For model discovery

The API should support:
- Streaming responses
- Tool/function calling (for full Openwork functionality)
- Standard OpenAI request/response format

### Configuration File Location

Custom Provider settings are stored in:
- **Credentials**: OS keychain (secure)
- **Configuration**: `~/Library/Application Support/Openwork/opencode/opencode.json` (macOS)
- **Configuration**: `%APPDATA%/Openwork/opencode/opencode.json` (Windows)

## Troubleshooting

### Connection Issues

**Problem**: "Connection failed" error

**Solutions**:
1. Verify your server is running: `curl http://localhost:8000/v1/models`
2. Check the server URL format (should include protocol: `http://` or `https://`)
3. Ensure firewall allows connections to the port
4. Check server logs for errors

### Model Not Working

**Problem**: Model selected but tasks fail

**Solutions**:
1. Verify the model ID matches exactly what your server expects
2. Check if your server supports tool/function calling
3. Review server logs for API errors
4. Test the endpoint directly:
   ```bash
   curl http://localhost:8000/v1/chat/completions \
     -H "Content-Type: application/json" \
     -d '{
       "model": "your-model-id",
       "messages": [{"role": "user", "content": "Hello"}]
     }'
   ```

### Authentication Issues

**Problem**: 401 Unauthorized errors

**Solutions**:
1. Verify API key is correct
2. Check if your server requires specific header format
3. Some servers use `Bearer` token, others use custom headers

## Advanced Configuration

### Using with Reverse Proxies

If you're using a reverse proxy (nginx, Caddy, etc.), ensure:

1. WebSocket support is enabled (for streaming)
2. Request timeout is sufficient (at least 60 seconds)
3. Headers are properly forwarded

Example nginx configuration:
```nginx
location /v1/ {
    proxy_pass http://localhost:8000/v1/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 300s;
}
```

### Multiple Custom Providers

Currently, Openwork supports one Custom Provider at a time. To use multiple custom endpoints:

1. Use LiteLLM as a proxy to aggregate multiple backends
2. Or switch between providers by disconnecting and reconnecting with different settings

## Supported Frameworks

The Custom Provider has been tested with:

- ✅ vLLM
- ✅ Text Generation Inference (TGI)
- ✅ LocalAI
- ✅ Ollama (though native Ollama provider is recommended)
- ✅ FastChat
- ✅ LM Studio
- ✅ Jan
- ✅ Anything LLM

## Security Considerations

1. **API Keys**: Stored securely in OS keychain, never in plain text
2. **Local Servers**: Use `http://localhost` for local-only access
3. **Remote Servers**: Always use HTTPS for remote endpoints
4. **Network**: Consider using VPN for accessing remote custom providers

## Need Help?

- Check server logs for detailed error messages
- Verify API compatibility with OpenAI format
- Test endpoints with curl before configuring in Openwork
- Report issues at: https://github.com/accomplish-ai/openwork/issues
