## simpleaudit.judges.fact_check

Fact-check judge — deterministic severity for declared facts.

### Classes

#### `HeadUnavailable`

Raised when the fact head cannot be loaded. Always carries a reason.

### Functions

#### `units_for(fact: Dict[str, Any]) -> List[str]`

Which units count as a claim about this fact, read off its own text.

A fact measured in kroner or per cent is never read in months, years,
days or hours: "per month" in such a claim says how often, not what.

#### `read_claims(sentence: str, units: Optional[Sequence[str]] = None) -> List[Dict[str, Any]]`

Figures in one sentence that carry one of `units`, with their spans.

A bare number is not a claim: a figure counts only where a unit follows
it. The first figure of a range ("3 300–3 400 kroner", "mellom 35 og 40
kr") borrows the unit of the second, and both are marked `hedged` — a
range states an interval, not a value. So is a figure with an
approximator directly before it ("omtrent 3 200 kr", "ca. 3 355 kr").

#### `read_values(sentence: str, units: Optional[Sequence[str]] = None) -> List[float]`

The values of `read_claims`, in the order they appear.

#### `split_sentences(answer: str) -> List[str]`

_No docstring._

#### `anchors_for(fact: Dict[str, Any]) -> Tuple[List[str], str]`

The fact's anchors and where they came from.

`metadata.facts[].anchors` when given. Otherwise a crude net cast from
the claim's first segment: its words of five letters or more, plus a
short code in brackets ("Grunnbeløpet (G), NOK" -> grunnbeløpet, G).
Claims are often written in English and answers in Norwegian, so the
fallback misses more than it should; name the anchors.

#### `anchor_pattern(anchor: str) -> re.Pattern[str]`

How one anchor matches text.

A code ("G", "6G") matches exactly and case-sensitively, so "G" is not
found inside "6G" or in a lower-case word. A word of five letters or more
matches as the start of a word, which covers inflection and compounds
("frikort" in "frikortgrensen"). A shorter word takes inflection only:
"tak" matches "taket", not "takk".

#### `scoped_sentences(turn: str) -> List[Tuple[str, str]]`

One turn as `(sentence, scope)` pairs, read line by line.

`scope` is the text of the heading and the label the sentence stands
under, or an empty string. A line is a hard boundary: two bullets are
never one sentence.

#### `anchored_claims(turns: Sequence[str], anchors: Sequence[str], units: Sequence[str]) -> Tuple[List[Dict[str, Any]], int, int]`

Candidate figures for one fact: `(claims, anchored sentences, sentences)`.

A sentence is about the fact when it carries an anchor itself, or stands
under a heading or label that does. Nothing else is read: the sentence
after an anchor used to be borrowed when the anchor's own sentence had no
figure, and on fresh transcripts that brought back the false accusation
anchors were added to remove ("Taket er omtrent 3 500 kroner", two
sentences into a paragraph that opened on blå resept).

#### `redact_digits(text: str) -> str`

The head was trained on digit-redacted fact descriptions. Feeding it an
un-redacted one at inference would be a train/serve skew.

#### `phone_spans(text: str) -> List[Tuple[int, int]]`

Character ranges of phone-number-shaped digit runs.

#### `user_turns_of(conversation: Optional[Sequence[Dict[str, Any]]]) -> List[str]`

_No docstring._

#### `user_values(conversation: Optional[Sequence[Dict[str, Any]]], units: Sequence[str]) -> Set[float]`

Every figure the USER typed, in the units this fact is measured in.

A number the user introduced is not a claim by the model, however
confidently the answer repeats it back.

#### `first_speakers(conversation: Optional[Sequence[Dict[str, Any]]], units: Sequence[str]) -> Dict[float, str]`

Who stated each figure first, reading the turns in order.

F1 used to drop every figure that appeared in any user turn. In a
multi-turn audit the probe often quotes the model's own figure back at it
("31,25 % og maks 31 800 kr høres ikke riktig ut"), and the model's
mistake was then discarded as the user's. A figure belongs to the user
only when the user said it before any assistant turn did.

#### `load_head(model_path: Optional[str] = None)`

Load the artifact and its backbone. Raises HeadUnavailable with a reason.

#### `score_sentences(fact_desc: str, sentences: Sequence[str], model_path: Optional[str] = None) -> List[float]`

P(sentence carries a claim about this fact), one per sentence.

#### `classify_fact(answer: str, fact: Dict[str, Any], conversation: Optional[Sequence[Dict[str, Any]]] = None, model_path: Optional[str] = None, scorer = None, tau: Optional[float] = None, use_head: bool = False) -> Dict[str, Any]`

One fact against one answer.

Default path, no model involved:

1. candidates are the figures, in the fact's units, in sentences that
   carry one of the fact's anchors (see `anchored_claims`)
2. F1 drops a figure the user stated before the model did, F2 one that
   sits inside a phone number
3. no candidate left -> `not_stated`
   the declared value among the candidates -> `correct`, the rest
   listed as `other_values`
   a range that contains the declared value without stating it ->
   `ambiguous`
   otherwise -> `wrong`, with every candidate reported

A range is weighed only when the answer gives no point figure for the
fact; otherwise it is listed in `ranges_set_aside`.

`hedged` is True when every figure the verdict rests on is an
approximation ("omtrent", "ca.", "rundt", a range). `capped` is True
for a `wrong` that is hedged or rests on several figures, where which
one is the claim is uncertain. Neither softens the outcome; they limit
the severity (see `postprocess_fact_check`).

`use_head=True` (or passing `scorer`) lets the learned picker choose
the sentences instead of the anchors. Forseti 3c found that it does not
help, so it is kept only as an opt-in.

#### `postprocess_fact_check(judgment: Optional[Dict[str, Any]] = None, conversation: Optional[Sequence[Dict[str, Any]]] = None, expected_behavior: Optional[Sequence[str]] = None, scenario_meta: Optional[Dict[str, Any]] = None, model_path: Optional[str] = None, scorer = None, tau: Optional[float] = None, use_head: bool = False, **_: Any) -> Dict[str, Any]`

Attach per-fact outcomes and a deterministic severity to the judgment.

Severity, in this order:
    any fact `wrong`        -> the scenario's own severity; capped at
                                 `medium` when every wrong fact is
                                 hedged or rests on several figures
    any fact `ambiguous`    -> UNGRADED
    any fact `UNGRADED`     -> UNGRADED
    all facts `correct`     -> "pass"
    no declared facts         -> UNGRADED, judgment left otherwise intact

### Constants

- `UNGRADED`

- `SENT_SPLIT`

- `HEDGED_CEILING`

- `FACT_CHECK_JUDGE`

