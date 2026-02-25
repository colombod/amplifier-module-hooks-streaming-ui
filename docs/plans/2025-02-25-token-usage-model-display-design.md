# Token Usage Model/Provider Display Design

## Goal

Display the actual model and provider used for each LLM request alongside the token usage information in the streaming UI output, including request duration.

## Background

Currently, the token usage display shows only token counts without indicating which model or provider actually handled the request. This information is important because tools like the delegate tool may cause use of a different model/provider than what was configured by the user. Users need visibility into what model actually processed their request.

## Approach

Subscribe to the existing `llm:response` event that providers already emit, rather than modifying ChatResponse contracts. This is a self-contained change to hooks-streaming-ui only.

**Why this approach:**
- The `llm:response` event already contains accurate model/provider/duration from the actual API response
- No changes needed to amplifier-core, providers, or orchestrators
- Single-module change
- Works today for Anthropic and OpenAI providers

## Architecture

The change adds a new event subscription and instance state to capture model information as it flows through the system. The existing token usage rendering is extended to include this captured information.

**Modules affected:**
| Module | Change needed? | Reason |
|--------|----------------|--------|
| hooks-streaming-ui | Yes | Subscribe to `llm:response`, display the info |
| provider-anthropic | No | Already emits `llm:response` with model/provider/duration |
| provider-openai | No | Already emits `llm:response` with model/provider/duration |
| amplifier-core | No | No contract changes |
| loop-streaming | No | No changes to what it emits |

## Components

### Hook Registration & State

Register a new handler for `llm:response` in `mount()` alongside existing handlers. Add instance state to store the captured information:

```python
self.last_llm_info: dict | None = None  # Stores {provider, model, duration_ms}
```

Using a single dict keeps it clean and makes it obvious these fields travel together.

### The `llm:response` Handler

New handler method to capture model/provider info:

```python
async def handle_llm_response(self, _event: str, data: dict[str, Any]) -> HookResult:
    """Capture model/provider info for display with token usage."""
    self.last_llm_info = {
        "provider": data.get("provider"),
        "model": data.get("model"),
        "duration_ms": data.get("duration_ms"),
    }
    return HookResult(action="continue")
```

Registered in `mount()`:

```python
coordinator.hooks.register("llm:response", hooks.handle_llm_response)
```

This is safe because `llm:response` fires before `content_block:end` — provider emits it, returns response, then orchestrator processes blocks.

### Rendering Changes

Modify token usage rendering in `handle_content_block_end`:

```python
# Build the header with model info if available
if self.last_llm_info:
    provider = self.last_llm_info.get("provider", "")
    model = self.last_llm_info.get("model", "")
    duration_ms = self.last_llm_info.get("duration_ms")
    
    # Format duration as seconds with 1 decimal
    duration_str = f" [{duration_ms / 1000:.1f}s]" if duration_ms else ""
    
    header = f"📊 Token Usage ({provider}/{model}){duration_str}"
else:
    header = "📊 Token Usage"

print(f"{indent}\033[2m│  {header}\033[0m")
print(f"{indent}\033[2m└─ Input: {input_str}{cache_info} | Output: {output_str} | Total: {total_str}\033[0m")
```

### State Cleanup

After displaying token usage, clear `last_llm_info` to avoid showing stale data if a subsequent request somehow fails to emit `llm:response`:

```python
# After printing the token usage lines...
self.last_llm_info = None  # Clear for next request
```

This is safe because:
- `content_block:end` with `is_last_block=True` fires once per LLM response
- The next `llm:response` will repopulate before the next `content_block:end`
- Clearing ensures we never show stale provider/model info

## Data Flow

```
Provider emits "llm:response" {provider, model, duration_ms, usage, ...}
    ↓
hooks.handle_llm_response captures to self.last_llm_info
    ↓
Orchestrator emits "content_block:end" {usage, block, ...}
    ↓
hooks.handle_content_block_end renders with self.last_llm_info
    ↓
self.last_llm_info cleared
```

## Error Handling

Graceful fallback when `last_llm_info` is missing or incomplete:
- If `last_llm_info` is None, display falls back to the original format without model info
- Individual fields use `.get()` with empty string defaults to handle partial data
- Duration display is conditional on `duration_ms` being present

**Output examples:**

With full info:
```
│  📊 Token Usage (anthropic/claude-sonnet-4-5-20250514) [2.3s]
└─ Input: 93,039 (69% cached) | Output: 1,642 | Total: 94,681
```

Graceful fallback:
```
│  📊 Token Usage
└─ Input: 93,039 (69% cached) | Output: 1,642 | Total: 94,681
```

## Testing Strategy

Follow existing test patterns in `tests/test_streaming_ui.py` using capsys to capture output:

1. **Handler captures data correctly**
   - Mock `llm:response` event with provider/model/duration
   - Verify `last_llm_info` is populated

2. **Rendering includes model info when available**
   - Set `last_llm_info` before calling `handle_content_block_end`
   - Verify output contains provider, model, and duration

3. **Graceful fallback when info missing**
   - Leave `last_llm_info` as None
   - Verify output still renders token usage without crashing

4. **State cleanup after rendering**
   - Verify `last_llm_info` is None after `content_block:end` with `is_last_block=True`

## Open Questions

None — design is complete and self-contained.
