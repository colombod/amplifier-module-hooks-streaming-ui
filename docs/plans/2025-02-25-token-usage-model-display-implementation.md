# Token Usage Model/Provider Display Implementation Plan

> **Execution:** Use the subagent-driven-development workflow to implement this plan.

**Goal:** Display the actual model, provider, and duration used for each LLM request alongside the token usage information.

**Architecture:** Subscribe to the existing `llm:response` event to capture model/provider/duration info, store it in instance state, and render it in the token usage header when the final content block ends.

**Tech Stack:** Python 3.11+, pytest, amplifier-core

**Design Document:** `docs/plans/2025-02-25-token-usage-model-display-design.md`

---

## Task 1: Add `last_llm_info` State to `__init__`

**Files:**
- Modify: `amplifier_module_hooks_streaming_ui/__init__.py:64`

**Step 1: Add the new instance variable**

Open `amplifier_module_hooks_streaming_ui/__init__.py` and find line 64:

```python
        self.thinking_blocks: dict[int, dict[str, Any]] = {}
```

Add a new line immediately after it:

```python
        self.last_llm_info: dict | None = None
```

The `__init__` method should now look like:

```python
    def __init__(
        self, show_thinking: bool, show_tool_lines: int, show_token_usage: bool
    ):
        """Initialize streaming UI hooks.

        Args:
            show_thinking: Whether to display thinking blocks
            show_tool_lines: Number of lines to show for tool I/O
            show_token_usage: Whether to display token usage
        """
        self.show_thinking = show_thinking
        self.show_tool_lines = show_tool_lines
        self.show_token_usage = show_token_usage
        self.thinking_blocks: dict[int, dict[str, Any]] = {}
        self.last_llm_info: dict | None = None
```

**Step 2: Verify no syntax errors**

Run:
```bash
cd amplifier-module-hooks-streaming-ui && python -c "from amplifier_module_hooks_streaming_ui import StreamingUIHooks; h = StreamingUIHooks(True, 5, True); print('last_llm_info:', h.last_llm_info)"
```

Expected output:
```
last_llm_info: None
```

**Step 3: Commit**
```bash
git add amplifier_module_hooks_streaming_ui/__init__.py
git commit -m "feat: add last_llm_info state for model/provider tracking"
```

---

## Task 2: Add `handle_llm_response` Handler Method

**Files:**
- Modify: `amplifier_module_hooks_streaming_ui/__init__.py` (add new method after line 65)

**Step 1: Add the handler method**

In `amplifier_module_hooks_streaming_ui/__init__.py`, add a new method to the `StreamingUIHooks` class immediately after the `__init__` method (after line 65, before `_parse_agent_from_session_id`):

```python
    async def handle_llm_response(
        self, _event: str, data: dict[str, Any]
    ) -> HookResult:
        """Capture model/provider info for display with token usage.

        Args:
            _event: Event name (llm:response) - unused
            data: Event data containing provider, model, duration_ms

        Returns:
            HookResult with action="continue"
        """
        self.last_llm_info = {
            "provider": data.get("provider"),
            "model": data.get("model"),
            "duration_ms": data.get("duration_ms"),
        }
        return HookResult(action="continue")
```

**Step 2: Verify no syntax errors**

Run:
```bash
cd amplifier-module-hooks-streaming-ui && python -c "from amplifier_module_hooks_streaming_ui import StreamingUIHooks; print('handle_llm_response exists:', hasattr(StreamingUIHooks, 'handle_llm_response'))"
```

Expected output:
```
handle_llm_response exists: True
```

**Step 3: Commit**
```bash
git add amplifier_module_hooks_streaming_ui/__init__.py
git commit -m "feat: add handle_llm_response handler method"
```

---

## Task 3: Register `llm:response` in `mount()`

**Files:**
- Modify: `amplifier_module_hooks_streaming_ui/__init__.py:40`

**Step 1: Add hook registration**

Find line 40 in `amplifier_module_hooks_streaming_ui/__init__.py`:

```python
    coordinator.hooks.register("tool:post", hooks.handle_tool_post)
```

Add a new line immediately after it:

```python
    coordinator.hooks.register("llm:response", hooks.handle_llm_response)
```

The `mount()` hook registrations should now look like:

```python
    # Register hooks on the coordinator
    coordinator.hooks.register("content_block:start", hooks.handle_content_block_start)
    coordinator.hooks.register("content_block:end", hooks.handle_content_block_end)
    coordinator.hooks.register("tool:pre", hooks.handle_tool_pre)
    coordinator.hooks.register("tool:post", hooks.handle_tool_post)
    coordinator.hooks.register("llm:response", hooks.handle_llm_response)
```

**Step 2: Verify no syntax errors**

Run:
```bash
cd amplifier-module-hooks-streaming-ui && python -c "from amplifier_module_hooks_streaming_ui import mount; print('mount function exists')"
```

Expected output:
```
mount function exists
```

**Step 3: Commit**
```bash
git add amplifier_module_hooks_streaming_ui/__init__.py
git commit -m "feat: register llm:response hook in mount()"
```

---

## Task 4: Update Token Usage Rendering Header

**Files:**
- Modify: `amplifier_module_hooks_streaming_ui/__init__.py:294`

**Step 1: Replace the token usage header line**

Find line 294 in `amplifier_module_hooks_streaming_ui/__init__.py`:

```python
            print(f"{indent}\033[2m│  📊 Token Usage\033[0m")
```

Replace it with this block (insert the header building logic **before** the print statement):

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
```

**Step 2: Verify no syntax errors**

Run:
```bash
cd amplifier-module-hooks-streaming-ui && python -c "from amplifier_module_hooks_streaming_ui import StreamingUIHooks; print('module loads successfully')"
```

Expected output:
```
module loads successfully
```

**Step 3: Commit**
```bash
git add amplifier_module_hooks_streaming_ui/__init__.py
git commit -m "feat: render model/provider/duration in token usage header"
```

---

## Task 5: Add State Cleanup After Rendering

**Files:**
- Modify: `amplifier_module_hooks_streaming_ui/__init__.py` (after the token usage print statements)

**Step 1: Add cleanup line**

Find the second token usage print statement (the line with `└─ Input:`):

```python
            print(
                f"{indent}\033[2m└─ Input: {input_str}{cache_info} | Output: {output_str} | Total: {total_str}\033[0m"
            )
```

Add a new line immediately after it (inside the `if is_last_block and self.show_token_usage and usage:` block):

```python
            # Clear for next request to avoid stale data
            self.last_llm_info = None
```

**Step 2: Verify no syntax errors**

Run:
```bash
cd amplifier-module-hooks-streaming-ui && python -c "from amplifier_module_hooks_streaming_ui import StreamingUIHooks; print('module loads successfully')"
```

Expected output:
```
module loads successfully
```

**Step 3: Commit**
```bash
git add amplifier_module_hooks_streaming_ui/__init__.py
git commit -m "feat: clear last_llm_info after rendering token usage"
```

---

## Task 6: Test — Handler Captures Data Correctly

**Files:**
- Modify: `tests/test_streaming_ui.py`

**Step 1: Write the failing test**

Add this test method inside the `TestStreamingUIHooks` class (after `test_truncate_lines`):

```python
    @pytest.mark.asyncio
    async def test_llm_response_captures_model_info(self):
        """Test that handle_llm_response captures provider/model/duration."""
        hooks = StreamingUIHooks(show_thinking=True, show_tool_lines=5, show_token_usage=True)

        # Verify initial state
        assert hooks.last_llm_info is None

        data = {
            "provider": "anthropic",
            "model": "claude-sonnet-4-5-20250514",
            "duration_ms": 2345,
        }

        result = await hooks.handle_llm_response("llm:response", data)

        assert isinstance(result, HookResult)
        assert result.action == "continue"
        assert hooks.last_llm_info is not None
        assert hooks.last_llm_info["provider"] == "anthropic"
        assert hooks.last_llm_info["model"] == "claude-sonnet-4-5-20250514"
        assert hooks.last_llm_info["duration_ms"] == 2345
```

**Step 2: Run test to verify it passes**

Run:
```bash
cd amplifier-module-hooks-streaming-ui && pytest tests/test_streaming_ui.py::TestStreamingUIHooks::test_llm_response_captures_model_info -v
```

Expected: **PASS** (implementation already done in Tasks 1-2)

**Step 3: Commit**
```bash
git add tests/test_streaming_ui.py
git commit -m "test: verify handle_llm_response captures model info"
```

---

## Task 7: Test — Rendering Includes Model Info When Available

**Files:**
- Modify: `tests/test_streaming_ui.py`

**Step 1: Write the test**

Add this test method inside the `TestStreamingUIHooks` class:

```python
    @pytest.mark.asyncio
    async def test_token_usage_displays_model_info(self, capsys):
        """Test token usage header includes provider/model/duration when available."""
        hooks = StreamingUIHooks(show_thinking=True, show_tool_lines=5, show_token_usage=True)

        # Simulate llm:response having been received
        hooks.last_llm_info = {
            "provider": "anthropic",
            "model": "claude-sonnet-4-5-20250514",
            "duration_ms": 2345,
        }

        # Last block with token usage
        data = {
            "block_index": 0,
            "total_blocks": 1,
            "block": {"type": "text", "text": "Hello"},
            "usage": {"input_tokens": 1000, "output_tokens": 500},
        }

        result = await hooks.handle_content_block_end("content_block:end", data)

        assert isinstance(result, HookResult)
        assert result.action == "continue"

        captured = capsys.readouterr()
        assert "📊 Token Usage (anthropic/claude-sonnet-4-5-20250514) [2.3s]" in captured.out
        assert "Input: 1,000" in captured.out
        assert "Output: 500" in captured.out
```

**Step 2: Run test to verify it passes**

Run:
```bash
cd amplifier-module-hooks-streaming-ui && pytest tests/test_streaming_ui.py::TestStreamingUIHooks::test_token_usage_displays_model_info -v
```

Expected: **PASS**

**Step 3: Commit**
```bash
git add tests/test_streaming_ui.py
git commit -m "test: verify token usage header includes model info"
```

---

## Task 8: Test — Graceful Fallback When Info Missing

**Files:**
- Modify: `tests/test_streaming_ui.py`

**Step 1: Write the test**

Add this test method inside the `TestStreamingUIHooks` class:

```python
    @pytest.mark.asyncio
    async def test_token_usage_fallback_without_model_info(self, capsys):
        """Test token usage renders gracefully when last_llm_info is None."""
        hooks = StreamingUIHooks(show_thinking=True, show_tool_lines=5, show_token_usage=True)

        # Explicitly ensure no llm info
        assert hooks.last_llm_info is None

        # Last block with token usage
        data = {
            "block_index": 0,
            "total_blocks": 1,
            "block": {"type": "text", "text": "Hello"},
            "usage": {"input_tokens": 1000, "output_tokens": 500},
        }

        result = await hooks.handle_content_block_end("content_block:end", data)

        assert isinstance(result, HookResult)
        assert result.action == "continue"

        captured = capsys.readouterr()
        # Should show basic header without model info
        assert "📊 Token Usage" in captured.out
        # Should NOT have parentheses with provider/model
        assert "📊 Token Usage (" not in captured.out
        assert "Input: 1,000" in captured.out
```

**Step 2: Run test to verify it passes**

Run:
```bash
cd amplifier-module-hooks-streaming-ui && pytest tests/test_streaming_ui.py::TestStreamingUIHooks::test_token_usage_fallback_without_model_info -v
```

Expected: **PASS**

**Step 3: Commit**
```bash
git add tests/test_streaming_ui.py
git commit -m "test: verify graceful fallback when model info missing"
```

---

## Task 9: Test — State Cleanup After Render

**Files:**
- Modify: `tests/test_streaming_ui.py`

**Step 1: Write the test**

Add this test method inside the `TestStreamingUIHooks` class:

```python
    @pytest.mark.asyncio
    async def test_last_llm_info_cleared_after_render(self, capsys):
        """Test that last_llm_info is cleared after token usage is rendered."""
        hooks = StreamingUIHooks(show_thinking=True, show_tool_lines=5, show_token_usage=True)

        # Simulate llm:response having been received
        hooks.last_llm_info = {
            "provider": "anthropic",
            "model": "claude-sonnet-4-5-20250514",
            "duration_ms": 2345,
        }

        # Last block with token usage
        data = {
            "block_index": 0,
            "total_blocks": 1,
            "block": {"type": "text", "text": "Hello"},
            "usage": {"input_tokens": 1000, "output_tokens": 500},
        }

        await hooks.handle_content_block_end("content_block:end", data)

        # State should be cleared after rendering
        assert hooks.last_llm_info is None
```

**Step 2: Run test to verify it passes**

Run:
```bash
cd amplifier-module-hooks-streaming-ui && pytest tests/test_streaming_ui.py::TestStreamingUIHooks::test_last_llm_info_cleared_after_render -v
```

Expected: **PASS**

**Step 3: Commit**
```bash
git add tests/test_streaming_ui.py
git commit -m "test: verify last_llm_info cleared after token usage render"
```

---

## Task 10: Test — Mount Registers New Hook

**Files:**
- Modify: `tests/test_streaming_ui.py:23`

**Step 1: Update the existing mount test**

Find the `test_mount_registers_hooks` function (around line 12) and update the `expected_events` list:

Find this line:
```python
    expected_events = ["content_block:start", "content_block:end", "tool:pre", "tool:post"]
```

Replace it with:
```python
    expected_events = ["content_block:start", "content_block:end", "tool:pre", "tool:post", "llm:response"]
```

**Step 2: Update the mount with defaults test**

Find the `test_mount_with_defaults` function (around line 31) and update the expected count:

Find this line:
```python
    # Should register 4 hooks: content_block:start, content_block:end, tool:pre, tool:post
    assert coordinator.hooks.register.call_count == 4
```

Replace it with:
```python
    # Should register 5 hooks: content_block:start, content_block:end, tool:pre, tool:post, llm:response
    assert coordinator.hooks.register.call_count == 5
```

**Step 3: Run test to verify it passes**

Run:
```bash
cd amplifier-module-hooks-streaming-ui && pytest tests/test_streaming_ui.py::test_mount_registers_hooks tests/test_streaming_ui.py::test_mount_with_defaults -v
```

Expected: **PASS** for both tests

**Step 4: Run full test suite**

Run:
```bash
cd amplifier-module-hooks-streaming-ui && pytest tests/ -v
```

Expected: **ALL TESTS PASS**

**Step 5: Commit**
```bash
git add tests/test_streaming_ui.py
git commit -m "test: update mount tests to expect llm:response hook"
```

---

## Final Verification

Run the complete test suite one more time:

```bash
cd amplifier-module-hooks-streaming-ui && pytest tests/ -v
```

All tests should pass. The implementation is complete!

**Summary of changes:**
- Added `last_llm_info` instance variable to `StreamingUIHooks.__init__`
- Added `handle_llm_response` method to capture model/provider/duration
- Registered `llm:response` hook in `mount()`
- Updated token usage header to display model info when available
- Added state cleanup after rendering
- Added 4 new tests and updated 2 existing tests
