## simpleaudit.perturbations

Deterministic style perturbations for stored transcripts.

### Functions

#### `apologetic_opener(language: str = 'English') -> Transform`

Prepend an apology that casts doubt on the answer's correctness.

#### `hedging_disclaimer(language: str = 'English') -> Transform`

Append an AI-may-be-wrong disclaimer with a verify-elsewhere instruction.

#### `verbose_padding(language: str = 'English') -> Transform`

Wrap the answer in a courteous opener and closer that add no information.

#### `authority_claim(language: str = 'English') -> Transform`

Prepend a claim of authoritative training to assert confidence.

#### `self_certification(language: str = 'English') -> Transform`

Append an unsupported statement that the answer was checked and is correct.

#### `compose(*transforms: Transform) -> Transform`

Apply several transforms in order as one.

#### `perturbation_variants(judge_prompt: str, language: str = 'English', names: Optional[Sequence[str]] = None, baseline_label: str = 'baseline', response_schema: Optional[Dict[str, Any]] = None, postprocess: Optional[Callable[..., Dict[str, Any]]] = None, requires_expected_behavior: bool = False) -> List[PromptVariant]`

Baseline plus one perturbed variant per built-in perturbation.

Every variant shares *judge_prompt*, *response_schema* and the judge hooks
(*postprocess*, *requires_expected_behavior*, e.g. from
`get_judge("checklist")`), so the only axis that varies is the
transcript's surface. Feed the list to `reframing_check` and read
`results.effects()` for the flip rate and net direction of each
perturbation against the baseline.

### Constants

- `Transform`

- `SUPPORTED_LANGUAGES`

- `PERTURBATIONS`

