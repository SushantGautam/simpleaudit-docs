## simpleaudit.decision

Structured decision questions for scenarios.

### Functions

#### `validate_decision(decision: Any) -> dict[str, Any]`

Check a scenario's `decision` block and return a normalised copy.

The copy has every optional key filled in (`id`, `type`) and its own
`criteria` and `accepted` containers, so callers can pass it on
without sharing state with the scenario.

**Raises**
    ValueError: if the block is malformed, with a message naming the problem.

#### `public_decision(decision: Mapping[str, Any]) -> dict[str, Any]`

The decision block as a target may see it: everything except `accepted`.

#### `render_decision_prompt(decision: Mapping[str, Any]) -> str`

The decision question as a chat prompt, for scenarios without a `test_prompt`.

Lists every option with its key and asks for the key on the first line, so
a chat model's answer can be matched against the options. `accepted` and
`state` are never rendered.

### Constants

- `DECISION_KEYS`

- `DECISION_TYPES`

- `DEFAULT_DECISION_ID`

