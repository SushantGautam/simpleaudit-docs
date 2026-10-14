## simpleaudit.targets.decision

DecisionTarget — audit a decision model through a System One endpoint.

### Classes

#### `DecisionTarget`

Send a scenario's decision question to a System One endpoint.

**Methods:**

- `ollama(model: str, base_url: str = 'http://localhost:11434', **kwargs: Any) -> DecisionTarget` — A decision model served by Ollama, e.g. `DecisionTarget.ollama("clef")`.
- `openrouter(model: str, api_key: str | None = None, **kwargs: Any) -> DecisionTarget` — A decision model on OpenRouter's Decisions API, e.g. `"typesafe/jev-1.13"`.
- `build_request(decision: Mapping[str, Any], user: str, documents: list[str | dict[str, Any]] | None = None) -> dict[str, Any]` — The request body for one decision question. Raises `ValueError` on a broken limit.
- `async send(system: str | None = None, user: str, history: list[dict[str, Any]] | None = None, file_uri: str | list[str] | None = None, documents: list[str | dict[str, Any]] | None = None, response_format: dict[str, Any] | None = None, params: dict[str, Any] | None = None, context: TargetContext | None = None) -> TargetResponse`

**Attributes:**

- `max_turns`

### Constants

- `OLLAMA_MAX_BODY_BYTES`

- `OPENROUTER_SYSTEMONE_URL`

