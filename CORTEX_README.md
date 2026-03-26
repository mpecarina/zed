# Cortex Configuration

This repo carries compatibility support for enterprise OpenAI-compatible endpoints such as Cortex. Cortex is close enough to OpenAI-compatible APIs to reuse most of the existing provider pipeline, but not close enough to work without extra handling.

The main differences are:

1. Cortex may use Azure-style deployment URLs.
2. Cortex may reject some otherwise valid OpenAI request fields.
3. Cortex may require non-streaming requests.
4. Cortex may reject function parameter schemas that omit `properties` on object schemas.

## Canonical Zed Settings Shape

The canonical settings shape is a nested `compatibility` object under each `openai_compatible` provider entry.

Example:

```jsonc
{
  "language_models": {
    "openai_compatible": {
      "Cortex": {
        "api_url": "https://cortex.example.com/rest/v2/azure/openai/deployments/gpt-5.2-2025-12-11-us-data-zone?api-version=2024-10-21",
        "available_models": [
          {
            "name": "gpt-5.2-2025-12-11-us-data-zone",
            "max_tokens": 128000,
            "capabilities": {
              "tools": true,
              "images": true,
              "parallel_tool_calls": true,
              "prompt_cache_key": true
            }
          }
        ],
        "compatibility": {
          "headers": {
            "use-case": "Zed IDE"
          },
          "stream": false,
          "drop_fields": [
            "model",
            "stream",
            "prompt_cache_key",
            "temperature"
          ]
        }
      }
    }
  }
}
```

## What Each Field Does

- `api_url`
  - Points at the Cortex deployment endpoint.
  - May already include `?api-version=...`.
- `available_models`
  - Registers the model in Zed's UI and runtime limits.
  - Also describes tool/image/parallel-call capabilities.
- `compatibility.headers`
  - Adds custom HTTP headers such as `use-case`.
- `compatibility.stream = false`
  - Disables streaming requests and expects a full JSON response.
- `compatibility.drop_fields`
  - Removes request fields Cortex rejects.

## Deployment URL vs `model`

This is one of the most important differences from a normal OpenAI-compatible endpoint.

Many OpenAI-compatible servers route model selection from the request body:

```json
{ "model": "gpt-4.1" }
```

Cortex is usually deployment-routed instead:

```text
.../deployments/gpt-5.2-2025-12-11-us-data-zone?api-version=2024-10-21
```

In that model, the URL identifies the deployment. The body `model` field may be redundant or rejected, which is why Cortex configs often include `"model"` in `drop_fields`.

That is also why Zed configs are usually one provider entry per deployment. Your current local setup follows that pattern with separate entries such as `Cortex`, `Cortex-GPT5`, `Cortex-GPT5-Mini`, and `Cortex-GPT51`.

## What the Zed Compatibility Layer Does

The Cortex support added in this repo does the following:

- reads the nested `compatibility` object from settings
- preserves existing query strings like `?api-version=...` when composing `/chat/completions` and `/responses`
- injects custom headers
- removes configured unsupported request fields
- supports non-streaming chat completions by synthesizing stream events from the returned JSON body
- overrides the Responses API `request.stream` flag when configured
- normalizes tool parameter schemas so Azure/Cortex accept object schemas without an explicit `properties` field

## Backward Compatibility

Older flat keys still parse as a fallback:

- `headers`
- `stream`
- `drop_fields`

But those are legacy compatibility fields only. The canonical documented shape is now the nested `compatibility` object.

## Current Implementation Notes

Zed's Cortex support is split across:

- settings schema
- settings conversion
- provider runtime wiring
- chat-completions request helper
- responses request helper
- add-provider UI defaults

This broader footprint exists because Zed's architecture separates configuration parsing from transport logic.

## User Checklist

1. Create one `openai_compatible` provider entry per Cortex deployment URL.
2. Put the deployment URL and `api-version` in `api_url`.
3. Register the visible model metadata in `available_models`.
4. Put custom request behavior in `compatibility`.
5. Drop unsupported fields such as `model`, `stream`, `prompt_cache_key`, or `temperature` when Cortex rejects them.
6. Set `compatibility.stream = false` if the endpoint does not support SSE.
7. Keep tool capabilities aligned with what the deployment actually supports.

## Important Tradeoff

When `compatibility.stream = false`, Zed cannot receive true incremental chat-completion tokens from Cortex. It must wait for the full JSON completion and then synthesize stream events locally. That is expected behavior for non-streaming Cortex deployments.