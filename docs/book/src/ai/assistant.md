# AI Assistant

Hugin includes a built-in AI assistant powered by OpenRouter, Anthropic, Google Gemini, or local Ollama models. The assistant lives in the **Assistant** sidebar tab with four sub-tabs: Chat, Agent, Auto, and History.

## Provider Support

- **OpenRouter** — access to 200+ models including Llama, Mixtral, Claude, GPT. Default: `meta-llama/llama-3.3-70b-instruct` ($0.30/M tokens).
- **Anthropic** — Claude models with 90% cache savings via `cache_control` blocks.
- **Google Gemini** — via `generateContent` API with `cachedContents` support.
- **Ollama** — local models, zero cost, OpenAI-compatible wire format.

## Chat

Streaming chat with markdown rendering. When a flow is selected in HTTP History, the assistant automatically includes request/response context (method, URL, key headers, status, body size).

## Per-Tab AI Buttons

17 tabs have dedicated AI actions:

- **Repeater** — "Analyze Request" + right-click context menu
- **Findings** — "Triage" + "Draft Report with AI"
- **Intruder** — "Analyze Results" + "AI Payloads"
- **Scanner** — "Explain Finding"
- **Search** — Natural language search
- **Match & Replace** — "Generate with AI"
- **Decoder** — "What is this?"
- **Workflows** — "Create with AI"
- **Intercept** — "AI Modify" for NL request rewriting
- **Sitemap** — "Analyze Attack Surface"
- **Cookie Jar** — "Analyze Cookies"
- **Comparer** — "Explain Differences"
- **Sequencer** — "Analyze Randomness"
- **Scopes** — "Suggest with AI"
- **Plugins/Scripts** — "Generate with AI"

## Configuration

Edit `~/.config/hugin/assistant.json`:

```json
{
  "enabled": true,
  "provider": "open_a_i_compatible",
  "base_url": "https://openrouter.ai/api/v1",
  "api_key_encrypted": "...",
  "model": "meta-llama/llama-3.3-70b-instruct",
  "max_tokens": 4096
}
```
