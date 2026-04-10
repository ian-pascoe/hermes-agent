# Plugin Runtime Agent-Access Seam Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Add the smallest generic core seam that lets plugins access the active agent/runtime context during tool execution, so a plugin-owned `reasoning_effort` tool can mutate live session state without feature-specific logic in Hermes core.

**Architecture:** Extend the tool dispatch path to optionally pass a narrow runtime context into plugin tool handlers. The first practical version can include `agent=self`, `session_id`, and `tool_call_id`, but the implementation should be written as a generic plugin-runtime facility, not a reasoning-effort special case. Keep hooks unchanged for now; the plugin tool itself should own the behavior by mutating the live agent when called.

**Tech Stack:** `run_agent.py`, `model_tools.py`, `tools/registry.py`, `hermes_cli/plugins.py`, pytest, existing plugin system.

---

## Why this seam

Current findings in this checkout:

- plugin tool handlers currently receive generic registry dispatch kwargs like `task_id`
- they do **not** receive the live `AIAgent`
- `pre_api_request` hooks are currently read-only in practice because return values are collected but not applied

If the goal is to empower plugins to do more than patch request kwargs — including richer session-aware behaviors — then passing runtime agent context is the more flexible seam.

Compared with request-patching, this gives plugins the ability to:
- mutate live session state
- invalidate cached prompt state if necessary
- inspect current model/provider/runtime configuration
- support more advanced plugin behaviors beyond request shaping

The downside is more power and therefore more risk. To manage that, the design should be explicit and documented as a **plugin-runtime API**, not an accidental leak of internals.

---

## Design principles

1. **Generic, not feature-specific**
   - no `reasoning_effort` special case in core
   - core only exposes runtime context to plugin tools

2. **Minimal first exposure**
   - pass `agent=self` to plugin tools only from the agent loop path
   - do not retrofit every call site blindly without tests

3. **Safe fallback behavior**
   - if a plugin ignores runtime kwargs, nothing breaks
   - non-plugin tools should behave exactly as before

4. **Document the contract honestly**
   - plugins may access the active agent during tool execution
   - this is powerful and should be used deliberately

---

## Proposed runtime contract

When a plugin tool is dispatched from a live conversation, its handler may receive:

```python
handler(args, agent=self, task_id=..., session_id=..., tool_call_id=..., user_task=...)
```

Recommended v1 guaranteed fields:
- `agent`
- `task_id`
- `session_id`
- `tool_call_id`

Optional fields can continue to flow as they already do.

---

## Task 1: Add failing tests for plugin tool agent access

**Objective:** Define the new runtime contract before implementation.

**Files:**
- Modify: `tests/hermes_cli/test_plugins.py`
- Modify: `tests/run_agent/test_run_agent.py`

**Step 1: Add a failing plugin-runtime unit test proving plugin handlers can receive `agent`**

Add a test in `tests/hermes_cli/test_plugins.py` that registers a tiny plugin tool and verifies that a dispatch call can pass an `agent` kwarg through to the handler.

Suggested shape:

```python
def test_plugin_tool_handler_can_receive_agent_runtime_context(...):
    ...
    result = registry.dispatch("kwarg_probe", {}, task_id="t1", session_id="s1", agent=fake_agent)
    parsed = json.loads(result)
    assert parsed["has_agent"] is True
```

This is the narrowest proof that the registry path allows the seam.

**Step 2: Add a failing agent-loop integration test**

In `tests/run_agent/test_run_agent.py`, simulate a plugin tool call routed through the real agent execution path and assert the plugin handler can observe and mutate the active agent.

Suggested behavior:
- set `agent.reasoning_config = {"enabled": True, "effort": "medium"}`
- register a fake plugin tool handler that does:

```python
agent = kwargs["agent"]
agent.reasoning_config = {"enabled": True, "effort": "high"}
return json.dumps({"success": True})
```

- call the tool through the same path real tools use
- assert `agent.reasoning_config` became `high`

**Step 3: Add a regression test proving non-plugin behavior is unchanged**

Add a test that dispatching a normal built-in tool still works without any special plugin-only breakage.

**Verification command:**

```bash
uv run --with pyyaml --with pytest --with pytest-asyncio python -m pytest -o addopts='' tests/hermes_cli/test_plugins.py tests/run_agent/test_run_agent.py -q
```

Expected before implementation: new tests fail.

---

## Task 2: Add plugin-tool identification helper if needed

**Objective:** Ensure Hermes can distinguish plugin tools from built-in tools when deciding whether to pass live agent context.

**Files:**
- Inspect/modify: `hermes_cli/plugins.py`
- Inspect/modify: `model_tools.py`

**Step 1: Reuse existing plugin tool tracking if already available**

`PluginContext.register_tool(...)` already records plugin tool names in the manager.

If there is already a helper like `get_plugin_tool_names()`, use it.
If not, add a tiny helper returning the current plugin-provided tool-name set.

Suggested public helper:

```python
def is_plugin_tool(name: str) -> bool:
    return name in get_plugin_tool_names()
```

**Step 2: Keep the helper cheap and side-effect free**

No new caching layer is needed.
This should just consult plugin-manager state.

---

## Task 3: Pass `agent=self` through the agent execution path for plugin tools

**Objective:** Make live agent context available to plugin tool handlers without changing built-in tool semantics.

**Files:**
- Modify: `run_agent.py`
- Modify: `model_tools.py` if needed

**Step 1: Identify the agent-loop-to-tool boundary**

Current flow:
- `run_agent.py` invokes tool execution helpers
- non-intercepted tools eventually flow to `handle_function_call(...)` in `model_tools.py`
- `model_tools.py` calls `registry.dispatch(...)`

**Step 2: Thread the active agent into `handle_function_call(...)`**

Extend `handle_function_call(...)` with an optional parameter:

```python
def handle_function_call(..., agent=None) -> str:
```

Default must remain `None` for all existing callers.

**Step 3: Only pass `agent` to plugin tools**

Inside `handle_function_call(...)`, when dispatching through the registry, check whether the tool is plugin-provided.

Suggested pattern:

```python
extra_dispatch_kwargs = {}
if agent is not None and is_plugin_tool(function_name):
    extra_dispatch_kwargs["agent"] = agent
```

Then:

```python
result = registry.dispatch(..., **extra_dispatch_kwargs)
```

This avoids spraying `agent` into every tool handler unnecessarily.

**Step 4: Pass `self` from `run_agent.py` into the dispatcher**

Wherever `run_agent.py` calls `handle_function_call(...)`, add:

```python
agent=self
```

Only the live agent-loop path needs this.

**Step 5: Preserve existing agent-loop special cases**

Do not disturb:
- `clarify`
- `delegate_task`
- memory-manager routing
- any special-cased loop tools already intercepted in `run_agent.py`

This seam should simply make normal registry-routed plugin tools more capable.

---

## Task 4: Document the plugin runtime contract

**Objective:** Make the new capability explicit for plugin authors.

**Files:**
- Modify: `hermes_cli/plugins.py`
- Optionally add/update docs comments in `tools/registry.py` or plugin docs references

**Step 1: Update plugin docs/comments**

Add a note that plugin tool handlers may receive runtime kwargs like:

```python
args, agent=..., task_id=..., session_id=..., tool_call_id=...
```

Clarify that:
- runtime kwargs are only available during live agent execution
- plugins should treat `agent` as advanced API surface
- misuse can break session behavior, so plugins should mutate only what they own

**Step 2: Keep the contract modest**

Do not promise more than Hermes can stably support.
Recommend plugin authors prefer:
- session-scoped mutations they understand
- minimal touch points
- no random mutation of unrelated internal state

---

## Task 5: Add a reasoning-effort-enabling regression test

**Objective:** Prove that this seam is sufficient for the plugin-first feature you actually care about.

**Files:**
- Modify: `tests/run_agent/test_run_agent.py`

**Step 1: Simulate a plugin-owned reasoning tool**

Create a fake plugin tool handler inside the test that:

```python
def _handler(args, **kwargs):
    agent = kwargs["agent"]
    level = args["level"]
    agent.reasoning_config = {"enabled": True, "effort": level}
    return json.dumps({"success": True, "level": level})
```

**Step 2: Route it through the real agent tool path**

Call through `_execute_single_tool(...)`, `_invoke_single_tool(...)`, or the closest real path in this checkout.

**Step 3: Assert live state changed**

```python
assert agent.reasoning_config == {"enabled": True, "effort": "high"}
```

That proves the seam is enough for the plugin implementation to proceed.

---

## Task 6: Run focused tests

**Objective:** Verify the seam works and does not regress the plugin system.

**Run:**

```bash
uv run --with pyyaml --with pytest --with pytest-asyncio python -m pytest -o addopts='' tests/hermes_cli/test_plugins.py tests/run_agent/test_run_agent.py -q
```

If failures occur, fix in this order:
1. plugin tool identification helper
2. `handle_function_call(..., agent=None)` plumbing
3. run-agent call-site updates
4. docs/comments if tests rely on helper names

---

## Task 7: Save a follow-up note for the plugin implementation branch

**Objective:** Make the next plugin step obvious once the seam lands.

**Files:**
- Create: `docs/plans/2026-04-10-reasoning-effort-plugin-followup.md`

**Include:**
- plugin tool can now receive `agent`
- plugin should validate level with existing Hermes parser
- plugin should set `agent.reasoning_config`
- plugin v1 should reject `persist=true`
- plugin may optionally add compact current-level guidance via `pre_llm_call`
- plugin should avoid mutating unrelated agent internals

---

## Guardrails

### Do not overbuild a plugin runtime framework
This does **not** need:
- a new runtime context class
- a new plugin RPC layer
- a broad permission system
- a generic mutable request abstraction

For v1, just passing `agent=self` to plugin tools in the live path is enough.

### Keep the seam explicit
Prefer code that makes it obvious this is a plugin capability, not a silent global behavior change.

### Preserve backwards compatibility
- existing plugin tools that ignore `**kwargs` keep working
- built-in tools remain unchanged
- non-agent call sites can still use `handle_function_call(..., agent=None)`

---

## Acceptance criteria

This seam is complete when:

1. plugin tool handlers can receive the live `AIAgent` from the agent loop
2. a plugin-style handler can mutate `agent.reasoning_config`
3. existing built-in tool behavior is unchanged
4. tests prove the runtime contract works
5. the seam is generic and not reasoning-effort-specific

---

## Implementation summary

If done correctly, the plugin-first `reasoning_effort` flow becomes:

1. plugin registers `reasoning_effort`
2. plugin handler receives `agent`
3. plugin validates `level`
4. plugin sets:

```python
agent.reasoning_config = parsed
```

5. next request uses the updated config through existing core logic

That gives you a true plugin-owned implementation with a small generic host seam and more room for future powerful plugins.
