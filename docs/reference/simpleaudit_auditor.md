## simpleaudit.auditor

Auditor — the primary, target-agnostic entry point.

### Classes

#### `Auditor`

Target-agnostic auditor.

**Parameters**
target:
    Any `Target`. Required.
judge_model / judge_provider / judge_api_key / judge_base_url:
    Judge LLM configuration (defaults to OpenAI).
`**kwargs`:
    Forwarded to `ModelAuditor` for advanced options (scenarios,
    system_prompt, max_retries, on_turn, etc.).

**Notes**
Because the engine's scenario/judge/report machinery lives in
`ModelAuditor`, `Auditor` constructs one and overrides its target.
The `model` / `provider` / `api_key` / `base_url` arguments are
accepted for compatibility but are ignored when an explicit `target` is
provided (the target already knows how to reach the system under test).

**Methods:**

- `target() -> Target`
- `engine() -> ModelAuditor` — The underlying `ModelAuditor` engine (advanced access).
- `async run_async(*args: Any, **kwargs: Any) -> Any`
- `run(*args: Any, **kwargs: Any) -> Any`

