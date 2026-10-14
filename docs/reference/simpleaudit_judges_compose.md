## simpleaudit.judges.compose

Compose judges from criteria and an output format.

### Functions

#### `compose_prompt(criteria: str, format_prompt: str) -> str`

The judge prompt: the criteria, a blank line, then the format prompt.

#### `dimension_key(name: str) -> str`

The JSON key for a score dimension: `"Tone & style"` -> `"tone_style"`.

#### `binary_postprocess(pass_when: bool) -> Callable[..., Dict[str, Any]]`

Grade a yes/no `answer`: `pass` when it equals `pass_when`.

Otherwise the result gets the scenario's designed severity (`high` when
it has none). An answer that is not a yes/no is an `ERROR`.

#### `average_postprocess(keys: Sequence[str]) -> Callable[..., Dict[str, Any]]`

Set `score` to the average of the dimension scores the judge gave.

#### `customize_judge(base: Union[str, Dict[str, Any]], criteria: Optional[str] = None, probe_prompt: Optional[str] = None, name: Optional[str] = None) -> Dict[str, Any]`

A copy of `base` with new criteria and/or probe prompt, format kept.

`base` is a registry name or a config dict (a built-in, `DEFAULT_JUDGE`
or a composed judge). Arguments left as None keep the base's value.

#### `build_judge(output: str, criteria: str, dimensions: Sequence[str] = (), question: Optional[str] = None, pass_when: Union[bool, str] = True, probe_prompt: Optional[str] = None, name: str = 'Custom judge') -> Dict[str, Any]`

A judge from scratch: `criteria` in one of the generic formats.

**Args**
    output: `severity`, `score`, `binary` or `checklist`.
    criteria: What to evaluate. For `checklist`, the instructions for
        checking the scenario's expected behaviour.
    dimensions: `score` only. Names of the dimensions to score 1-10;
        the score is their average. Empty: one overall score.
    question: `binary` only. The yes/no question, e.g. "Did the
        assistant reveal its system prompt?".
    pass_when: `binary` only. The answer that passes (True = yes).
    probe_prompt: How the auditor model writes user turns. None: the
        default probe prompt.

### Constants

- `BUILD_OUTPUTS`

- `SEVERITY_RESPONSE_SCHEMA`

