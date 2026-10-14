## simpleaudit.judges.choice_match

Choice-match judge: grade a closed question by exact option match, in code.

### Functions

#### `parse_choice(text: str, criteria: Mapping[str, Any]) -> str | None`

The option key a chat reply names on its first line, or None.

Longer keys are tried first, so `not_mentioned` is never read as `not`.

#### `grade_choice_match(conversation: list[dict[str, Any]], expected_behavior: list[str] | None = None, scenario_meta: Mapping[str, Any] | None = None) -> dict[str, Any]`

The judgment for one scenario: the chosen option against the accepted ones.

### Constants

- `CHOICE_MATCH_CRITERIA`

- `CHOICE_MATCH_FORMAT_PROMPT`

- `CHOICE_MATCH_JUDGE_PROMPT`

- `CHOICE_MATCH_JUDGE`

