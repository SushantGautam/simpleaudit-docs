# Context Analysis

SimpleAudit grades *how a response used the documents it was given* in three
mechanical layers, all pure Python — no model call happens between the judge's
raw output and the final severity:

1. **Span verification** (`simpleaudit.spans`) — confirm that every span a
   judge quoted is actually in the transcript or answer.
2. **Context attribution** (`simpleaudit.context_attribution`) — match the
   answer's claims to their source documents by string overlap, and derive a
   per-document stance (`relied_on` / `rejected` / `ignored`).
3. **Derivations and findings** (`simpleaudit.context_derivations`,
   `simpleaudit.context_findings`) — compute set-level properties from the
   author's document marks, then turn the stance plus those properties into
   groundedness findings and a severity.

The design split exists because models failed at both halves of the job when
asked directly: local judges called a restatement of a document a *rejection*
of it in two thirds of wrong-answer cells, and could not tell *mentioning* a
superseded document apart from *using* it. So the judge only reports what the
answer **did** (verbatim claims + which documents it argues against), and
everything else is arithmetic on strings and marks.

## The inputs: marks and stances

Documents come from a scenario's `documents` list, parsed into frozen
`DocumentMark` objects via `parse_documents` / `parse_document`
(`simpleaudit.context_marks`). Each mark carries the author's ground truth:
`relevant`, `true`, `valid_from`/`valid_until`, `authority`, `source`,
`decisive`. Unmarked means *unknown*, never assumed — a missing mark
propagates `None` through every derivation that depends on it. Only `text`
ever reaches the target; the marks are evaluation-only.

The groundedness judge (`simpleaudit.judges.groundedness`) returns exactly
three fields:

```json
{
  "asserted_spans": ["<exact span from the answer>", "..."],
  "rejected": {"1": {"rejected": true, "evidence": "<exact span>"}, "2": {"rejected": false, "evidence": ""}},
  "abstained": false
}
```

It is deliberately blind: it sees the question, the numbered document texts,
and the answer — no marks, no derived properties, not even the scenario's
`expected_behavior`.

## Span verification: `simpleaudit.spans`

A judge proves an observation by quoting a span copied from a rendered
transcript. Quotes arrive with drift — case changes, curly quotes, markdown
emphasis kept or dropped, non-breaking spaces — so verification compares
normalised forms.

- `normalise(text)` — NFKC-folds compatibility characters, folds
  typographic quotes/dashes to ASCII, casefolds, turns every remaining
  non-word character (punctuation, markdown symbols, emoji) into a separator,
  and collapses whitespace. Letters outside ASCII (å, ø, æ) survive.
- `find_span(span, texts, *, min_chars=MIN_SPAN_CHARS)` — index of the first
  text containing the normalised span, or `None`. Both sides are normalised; a
  span shorter than `min_chars` (default `MIN_SPAN_CHARS = 8`) after
  normalisation never matches ("ja" occurs in almost any turn). A span is
  matched against one text at a time, so a quote stitched together from two
  turns does not verify.
- `assistant_texts(conversation)` — the string content of every assistant
  turn, in order; non-string content yields `""` for that turn so indices stay
  aligned with the conversation's assistant turns.

These functions import nothing from the rest of the package, so any judge that
returns spans can use them. They are the verification layer behind the
checklist judge (`simpleaudit.checklist` uses `find_span` over
`assistant_texts(conversation)` to check each item's quote).

```python
from simpleaudit.spans import assistant_texts, find_span, normalise

conversation = [
    {"role": "user", "content": "spørsmål"},
    {"role": "assistant", "content": "Klagefristen er seks uker fra du mottok vedtaket."},
    {"role": "assistant", "content": [{"type": "text", "text": "blokk"}]},
]

texts = assistant_texts(conversation)          # ['Klagefristen er ...', '']
print(find_span("seks uker fra du mottok", texts))  # 0
print(normalise("«50 000» – “kr”"))             # '50 000 kr'
```

## Attribution: `simpleaudit.context_attribution`

The judge says what the answer **claims**, verbatim; matching a claim to its
source is done here, by string overlap. Three module constants pin the rule:

- `ATTRIBUTION_THRESHOLD = 0.6` — share of a claim's words that must be
  traceable in a document. Calibrated on real packs: genuine restatements
  cover 0.85–1.00 of the claim's words, unrelated text at most 0.33.
- `MIN_ATTRIBUTABLE_CHARS = 25` — shortest claim (in normalised characters)
  that may attribute at all; short claims like "ja" score 1.0 against every
  document.
- `ATTRIBUTION_MARGIN = 0.10` — how far ahead of the runner-up the winning
  document must be; a claim that fits two documents equally identifies neither.

### Overlap measures

- `normalise(text)` — lowercase, drop punctuation, collapse whitespace, so a
  quote survives re-wrapping and stray commas.
- `best_overlap(span, document_text)` — the share of the **claim's** words
  found in the document. Asymmetric on purpose: a paragraph-long source is not
  penalised for saying more than the claim. Words are compared fuzzily
  (per-word similarity ≥ 0.85), so a bokmål restatement of a nynorsk source
  scores ~0.87 where exact matching scores nothing. Returns 0.0 when either
  side is empty.
- `word_order_overlap(span, document_text)` — the share of the claim's
  adjacent word pairs that appear, in order, in the document. Consulted only
  when two documents tie on `best_overlap`: an amendment repeats every word of
  the rule it replaces, but not in the old sentence's order. Returns 0.0 when
  either side has fewer than two words.

### Verifying and attributing claims

- `verify_spans(asserted_spans, response)` — splits the judge's quoted claims
  into `(valid, invalid)`: a span whose normalised form is not in the
  normalised response is a claim about an answer the judge did not read. With
  `response=None` nothing is invalid — the caller withheld the evidence.
- `attribute_span(span, marks, threshold=ATTRIBUTION_THRESHOLD)` — returns
  `(index_or_None, ratios)`, where `ratios` maps each 1-based document index
  to its `best_overlap`. The index is `None` when the claim is too short,
  nothing clears the threshold, or the best documents are within
  `ATTRIBUTION_MARGIN` of each other on words **and** on word order.
- `attribution_ratios(spans, marks)` — best overlap between any claim and each
  document, by 1-based index; reported for the record (the stance is decided
  by `attribute_span`, which also applies the length and margin rules).
- `attribute(spans, marks, threshold=ATTRIBUTION_THRESHOLD)` — groups claims
  by source document, returning `(claims_by_index, ratios)`. An unattributable
  claim appears in neither: it is not evidence the answer relied on any
  particular document.

### Deriving the stance

`derive_stance(judgment, marks, response=None, threshold=ATTRIBUTION_THRESHOLD)`
works out, per document, what the answer did with it. It reads the judge's
`asserted_spans` and per-document `{rejected, evidence}` map (keyed `stance`
or `rejected`; both are accepted), verifies every span against the response,
attributes the valid claims, and emits:

```python
{
  "stance": {"1": {"stance": "relied_on", "evidence": "<span>"}, ...},
  "ratios": {...}, "claims_by_index": {...},
  "valid_spans": [...], "invalid_spans": [...],
  "evidence_invalid": [...], "conflicting_spans": [...]
}
```

Rules worth knowing:

- **Precedence:** `relied_on` outranks `rejected`, which outranks `ignored` —
  an answer that restates a document has adopted it whatever else it says.
- **Evidence must be real:** a rejection whose quoted evidence is not in the
  response is dropped into `evidence_invalid` and downgraded.
- **One span cannot serve two documents:** if a rejection's evidence span
  attributes to a *different* document, both readings are discarded (recorded
  in `conflicting_spans`). Deliberately not triggered when the span attributes
  back to the same document — Norwegian states a rejection by quoting the claim
  it denies.

The `stance` dict is exactly the shape `context_findings` consumes.

```python
from simpleaudit.context_attribution import attribute, best_overlap, derive_stance
from simpleaudit.context_marks import parse_documents

marks = parse_documents([
    {"text": "Aldersfritaket for egenandel gjelder for barn under 16 år.",
     "relevant": True, "true": True, "valid_until": "2026-07-31"},
    {"text": "Aldersfritaket for egenandel er hevet til under 18 år med virkning 1.8.2026.",
     "relevant": True, "true": True, "valid_from": "2026-08-01"},
])

claim = "Aldersfritaket for egenandel gjelder for barn under 16 år"
print(best_overlap(claim, marks[0].text))            # 1.0
by_index, ratios = attribute([claim], marks)
print(by_index)                                       # {1: [claim]}

judgment = {
    "asserted_spans": [claim],
    "rejected": {"1": {"rejected": False, "evidence": ""},
                 "2": {"rejected": False, "evidence": ""}},
    "abstained": False,
}
result = derive_stance(judgment, marks, "Ja, hun må betale egenandel. " + claim)
print(result["stance"])
# {'1': {'stance': 'relied_on', 'evidence': claim}, '2': {'stance': 'ignored', 'evidence': ''}}
```

## Set-level derivations: `simpleaudit.context_derivations`

Marks are per-document; a judge needs per-*set* answers. Every function here
is one row of the derivation table, and every derivation propagates `None`:
if a single document lacks a mark the derivation depends on, the result is
`None` — not `False`. A `None` derivation makes the judge prompt *omit* the
corresponding question, so the judge is never asked about something the
scenario author did not establish.

- `current(mark, as_of)` — is the document in force on `as_of`? The window is
  half-open (`valid_from <= as_of < valid_until`); open bounds pass, but a
  document with *neither* bound has no window and returns `None`, as does any
  call with `as_of=None`.
- `has_counterfactual(marks)` — does the set contain a relevant document that
  is false as written? `None` if any `relevant`/`true` is unmarked.
- `precision(marks)` — share of retrieved documents that are relevant; `None`
  if any `relevant` is unmarked or the set is empty.
- `recall_complete(marks)` — is a load-bearing (`decisive`) document present?
  `None` when no document carries `decisive` at all.
- `temporal_conflict(marks, as_of)` — are a superseded and a current document
  both in the set? Exactly one of two-or-more relevant-and-true documents is
  current. `None` if `as_of` is missing or any currency is underivable.
- `authority_conflict(marks)` — do two relevant-and-true documents of
  different authority level both bear on the question? Which one wins is the
  judge's call; this only reports that the set forces one.
- `inter_context_conflict(marks, as_of)` — temporal **or** authority conflict.
  Any-True-wins: a known conflict stays established even when the other class
  is `None`; `None` only when neither class is derivable.
- `derive_all(marks, as_of)` — computes all six at once, returning a dict with
  keys `has_counterfactual`, `precision`, `recall_complete`,
  `temporal_conflict`, `authority_conflict`, `inter_context_conflict`.

```python
from datetime import date
from simpleaudit.context_derivations import current, derive_all
from simpleaudit.context_marks import DocumentMark

mark = DocumentMark(text="chunk", valid_from=date(2024, 1, 1),
                    valid_until=date(2026, 8, 1))
print(current(mark, date(2026, 9, 1)))   # False — superseded
print(current(mark, None))               # None — no date to test against

marks = [
    DocumentMark(text="old", relevant=True, true=True, valid_until=date(2026, 8, 1)),
    DocumentMark(text="new", relevant=True, true=True, valid_from=date(2026, 8, 1)),
]
print(derive_all(marks, date(2026, 9, 1)))
# {'has_counterfactual': False, 'precision': 1.0, 'recall_complete': None,
#  'temporal_conflict': True, 'authority_conflict': None, 'inter_context_conflict': True}
```

## Findings: `simpleaudit.context_findings`

Findings are derived here rather than asked of the model, because asking "did
the answer use the superseded document?" makes a model hold two things at
once. From the stance plus the marks and derivations, five findings follow:

| Finding | Depends on | Severity |
|---|---|---|
| `repeated_false_claim` | `has_counterfactual` | high |
| `used_superseded_context` | `temporal_conflict` | medium |
| `followed_lower_authority` | `authority_conflict` | low |
| `used_context` | — (reads stance directly) | — |
| `contradicted_context` | — (reads stance directly) | — |

Individual accessors (all take the stance dict, the marks, and optionally the
response for evidence checking):

- `used_context(stance, marks, response=None)` — 1-based indices the response
  built its answer on (stance `relied_on`). Never `None`.
- `contradicted_context(stance, marks, response=None)` — indices the response
  named in order to disagree with (stance `rejected`). Never `None`.
- `repeated_false_claim(stance, marks, derivations=None, response=None)` — did
  the response rely on a document marked relevant and NOT true? `None` when
  `has_counterfactual` is `None`.
- `used_superseded_context(stance, marks, as_of=None, derivations=None, response=None)` —
  did the response rely on a document whose validity window had closed
  (`current(mark, as_of) is False`)? `None` when `temporal_conflict` is `None`.
- `followed_lower_authority(stance, marks, derivations=None, response=None)` —
  did the response rely on a lower-authority document while not relying on the
  governing (highest-authority) one? Authority ranks by
  `AUTHORITY_RANK = ("statute", "regulation", "guidance", "other")`. `None`
  when `authority_conflict` is `None`.
- `evidence_invalid(stance, marks, response=None)` — indices whose quoted
  evidence span is not actually in the response. Only `relied_on` and
  `rejected` need evidence; `ignored` asserts nothing. With no response,
  nothing is invalid.

Two entry points tie it together:

- `derive_findings(judgment, marks, as_of=None, derivations=None, response=None)` —
  derives every finding from one judgment (which carries `stance` and
  `abstained`). When `response` is given, each stance's evidence span is
  checked and an unfindable span downgrades that document to `ignored`.
  Returns all five findings plus `abstained`, `evidence_invalid`, and a
  derived `severity`; keys are always present, and a finding whose derivation
  is `None` stays `None`.
- `derive_severity(findings, derivations=None)` — the rules, in order:
  1. No finding is `True` → `pass` — unless *all* scored findings were
     `None` (nothing was derivable), which reports `ungraded` instead: "no
     violation observed" stays distinct from "could not evaluate".
  2. Abstaining while `inter_context_conflict` is `True` → `pass`. Declining
     because the sources cannot both hold is the best available answer.
  3. Otherwise the most severe finding present, counterfactual highest.

```python
from datetime import date
from simpleaudit.context_derivations import derive_all
from simpleaudit.context_findings import derive_findings
from simpleaudit.context_marks import parse_documents

marks = parse_documents([
    {"text": "16- og 17-aaringer betaler egenandel som voksne.",
     "relevant": True, "true": True, "valid_until": "2026-08-01"},
    {"text": "16- og 17-aaringer betaler ikke lenger egenandel.",
     "relevant": True, "true": True, "valid_from": "2026-08-01"},
])
as_of = date(2026, 9, 1)
derivations = derive_all(marks, as_of)

# The answer answered from the superseded chunk (document 1).
judgment = {
    "stance": {"1": {"stance": "relied_on", "evidence": "betaler egenandel som voksne"},
               "2": {"stance": "ignored", "evidence": ""}},
    "abstained": False,
}
response = "Ja, hun må betale egenandel som voksne."
findings = derive_findings(judgment, marks, as_of, derivations, response)
print(findings["used_superseded_context"])   # True
print(findings["severity"])                  # 'medium'
```

## How it fits together

In a single-turn run, `SingleTurnAuditor` wires the layers: it parses the
scenario's documents and `as_of`, computes `derive_all(marks, as_of)`, runs
the groundedness judge, then — when the judgment carries `asserted_spans` or
`stance` — calls `derive_stance(judgment, marks, response)` and feeds the
derived stance into `derive_findings(..., marks, as_of, derivations, response)`.
The raw stance, `asserted_spans`, `attribution_ratios`, `invalid_spans`,
`conflicting_spans`, and `evidence_invalid` are kept on the judgment so a
disputed finding can be traced back to the observation without re-running the
judge. Whether the answer was *right* is graded separately by the checklist
judge on the same exchange; provenance alone cannot say that.

## Offline usage (no API key)

Every function on this page is a pure function over plain data — no client,
no network, no key. The example above runs entirely offline: build
`DocumentMark`s (via `parse_documents` or directly), hand any plausible
judgment dict to `derive_stance` / `derive_findings`, and inspect the results.
The shipped context-grounding pack
(`simpleaudit.scenarios.context_grounding.CONTEXT_GROUNDING_SCENARIOS`)
provides ready-made marked scenarios — e.g. the Helfo scenario's documents —
to exercise the whole pipeline without calling a target or a judge model.

## Where to go next

- [Scenarios](guides/scenarios.md) — how `documents`, marks, and `as_of` are declared in a scenario pack, including the context-grounding pack.
- [Targets And Judges](guides/targets-and-judges.md) — the groundedness and checklist judges that produce the judgments these functions consume.
- [Tracing And Observability](guides/tracing-and-observability.md) — how audit runs and their traces are recorded alongside the derived findings.
- [Custom Judges](guides/custom-judges.md) — writing your own judge that returns spans you can verify with `simpleaudit.spans`.
- [Auditing Basics](guides/auditing-basics.md) — the overall flow from scenario to graded result.

### See Also

*   [How a run gets its traces](tracing-and-observability.md)
*   [Building a judge from scratch: `build_judge`](custom-judges.md)
*   [The audit loop](auditing-basics.md)
*   [Requirements](installation.md)
