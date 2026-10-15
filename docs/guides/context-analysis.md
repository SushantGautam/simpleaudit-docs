# Context Analysis

SimpleAudit's context-grounding scenarios give the target a set of retrieved
documents and ask a question about them. The judge does **not** decide which
document the answer relied on — it reports what the answer *claims*, quoted
verbatim, and which documents it argues against. Everything else is computed
mechanically from four cooperating modules:

| Module | Job |
|---|---|
| `simpleaudit.context_marks` | Parse the author's per-document ground truth (`DocumentMark`) and render what the target vs. the judge may see. |
| `simpleaudit.context_attribution` | Match each quoted claim to its source document by fuzzy string overlap; derive the per-document stance. |
| `simpleaudit.context_derivations` | Set-level properties of the document set (counterfactual present? temporal or authority conflict?). |
| `simpleaudit.context_findings` | Turn stance + marks + derivations into findings and a severity. |

`SingleTurnAuditor` wires these together after the judging call: it parses the
scenario's `documents`, derives the set-level properties, runs the groundedness
judge, then calls `derive_stance` followed by `derive_findings`. Whether the
answer was *right* is graded separately by the checklist judge — attribution
only says what the answer leaned on. See [targets-and-judges.md](targets-and-judges.md)
for the judges themselves and [scenarios.md](scenarios.md) for how grounding
scenarios are written.

The one rule that runs through all four modules: **unmarked means unknown,
never assumed.** A mark that is missing propagates `None` through every
derivation and finding that depends on it, so the judge prompt omits the
corresponding question instead of asking something the scenario author never
established.

## Document marks

A mark is the author's ground truth about one retrieved document. It is a
frozen dataclass — nothing downstream rewrites it:

```python
from simpleaudit.context_marks import (
    DocumentMark, parse_document, parse_documents,
    parse_as_of, render_documents, mark_table,
)

marks = parse_documents([
    "plain text chunk",  # bare string: every mark None (unknown, not assumed)
    {
        "text": "16- og 17-aaringer betaler ikke lenger egenandel.",
        "relevant": True,
        "true": True,
        "valid_from": "2026-08-01",
        "authority": "guidance",
        "source": "helfo/HF-01",
    },
])
as_of = parse_as_of({"as_of": "2026-09-01"})   # date(2026, 9, 1), or None if unset
```

`DocumentMark` has the fields `text`, `relevant`, `true`, `valid_from`,
`valid_until`, `authority`, `source`, and `decisive` — all optional except
`text`. Two rules make this module exist as its own file:

1. **Unknown keys raise.** `parse_document` accepts only `text` plus the mark
   keys `relevant`, `true`, `valid_from`, `valid_until`, `authority`,
   `source`, `decisive`. A mis-spelled key like `relevent` would otherwise be
   silently dropped and read downstream as "unknown", quietly disabling the
   test the author wrote. Boolean marks must be real booleans (the string
   `"false"` is rejected), dates must be ISO strings or `date` objects, and
   `authority` must be one of `statute`, `regulation`, `guidance`, `other`.
   A document marked `decisive` cannot also be marked `relevant=False` or
   `true=False`.
2. **Marks never reach the target.** `render_documents(marks)` emits numbered
   text blocks (`\n--- DOCUMENT 1 ---\n<text>...`) and nothing else; the
   numbering matches the indices the judge reports. `mark_table(marks, as_of)`
   renders the judge-only table with columns index, relevant, true, current,
   authority, source — where `current` is derived from the validity window
   against `as_of` and shown as `unknown` whenever it is underivable.

`parse_documents` takes the whole `documents` list (a missing or empty list
yields `[]`), passes already-parsed `DocumentMark`s through unchanged, and
raises if handed a bare string or mapping rather than a list. `parse_as_of`
reads the scenario-level `as_of` date; without it there is no date to test a
validity window against, so every temporal derivation returns `None` rather
than guessing today.

## Attribution: matching claims to documents

The functions in `simpleaudit.context_attribution` replace a judgement the
model kept getting wrong (calling a restatement a rejection) with string
comparison. Three calibrated constants control the behavior:

- `ATTRIBUTION_THRESHOLD = 0.6` — share of a claim's words that must be found
  in a document for the claim to count as coming from it.
- `MIN_ATTRIBUTABLE_CHARS = 25` — shortest normalised claim that may attribute
  at all ("ja" scores 1.0 against every document).
- `ATTRIBUTION_MARGIN = 0.10` — how far ahead of the runner-up the winning
  document must be; a claim that fits two sources equally identifies neither.

### Overlap primitives

```python
from simpleaudit.context_attribution import (
    normalise, best_overlap, word_order_overlap,
    verify_spans, attribute_span, attribution_ratios, attribute,
)

normalise("«Kvotene»  for\nalkohol,")          # 'kvotene for alkohol'
best_overlap("Kvotene gjelder for alle som reiser til Norge",
             "Kvotene gjeld for alle som reiser til Noreg")   # ~0.87
word_order_overlap("Aldersfritaket for egenandel",
                   "Aldersfritaket for egenandel gjelder for barn")  # 1.0
```

- `normalise(text)` lowercases, drops punctuation, collapses whitespace. All
  comparison happens on this form, so a quote survives re-wrapping and stray
  commas.
- `best_overlap(span, document_text)` is the share of the **claim's** words
  found in the document — deliberately asymmetric, since claims are sentences
  and chunks are paragraphs. Words match fuzzily (ratio ≥ 0.85), which lets a
  bokmål restatement of a nynorsk source score ~0.87 where an exact-match rule
  would score nothing. Returns `0.0` when either side is empty.
- `word_order_overlap(span, document_text)` is the share of the claim's
  adjacent word pairs that appear, in order, in the document. It is consulted
  only when two documents tie on `best_overlap`: an amendment repeats every
  word of the old rule but not in the old sentence's order, so order breaks
  the tie. Returns `0.0` when either side has fewer than two words.
- `verify_spans(asserted_spans, response)` splits the judge's quoted claims
  into those actually present in the answer and those not, comparing
  normalised forms. With no response to check against, nothing is invalid —
  the caller withheld the evidence, so the judge is not at fault.

### From spans to stances

```python
judgment = {
    "asserted_spans": ["Aldersfritaket for egenandel gjelder for barn under 16 år"],
    "rejected": {"1": {"rejected": False, "evidence": ""},
                 "2": {"rejected": False, "evidence": ""}},
    "abstained": False,
}
result = derive_stance(judgment, marks, response)
# result["stance"] == {"1": {"stance": "relied_on", "evidence": <span>}, ...}
```

- `attribute_span(span, marks, threshold=ATTRIBUTION_THRESHOLD)` returns
  `(index_or_None, ratios)` — which single document the claim came from, and
  its overlap with each (1-based). The index is `None` when the claim is too
  short, when nothing clears the threshold, or when the top documents are
  within the margin of each other on words *and* on word order.
- `attribution_ratios(spans, marks)` reports, per document, the best overlap
  between any claim and that document. It is recorded for traceability; the
  stance is decided by `attribute_span`, which also applies the length and
  margin rules.
- `attribute(spans, marks, threshold=ATTRIBUTION_THRESHOLD)` groups claims by
  source document and returns `(claims_by_index, ratios)`. An unattributable
  claim appears in neither — it is not evidence that the answer relied on any
  particular document.
- `derive_stance(judgment, marks, response=None, threshold=ATTRIBUTION_THRESHOLD)`
  works out, per document, what the answer did with it. It verifies the
  asserted spans, attributes the valid ones, checks the judge's rejection
  evidence against the response, and assigns each document a stance of
  `relied_on`, `rejected`, or `ignored` — with `relied_on` outranking
  `rejected` outranking `ignored`. A rejection whose evidence span also
  attributes to a *different* document is discarded (one span cannot be proof
  of disagreement with one document and a restatement of another); quoting
  what you reject back to the same document is fine. It returns a dict with
  `stance`, `ratios`, `claims_by_index`, `valid_spans`, `invalid_spans`,
  `evidence_invalid`, and `conflicting_spans`; the `stance` object is exactly
  the shape `context_findings` consumes.

## Set-level derivations

`simpleaudit.context_derivations` computes per-*set* properties from the
per-document marks. Every derivation propagates `None`: if a single document
lacks a mark the derivation depends on, the result is `None` — not `False` —
because `False` would assert "no conflict here", which nobody established.

```python
from datetime import date
from simpleaudit.context_derivations import (
    current, has_counterfactual, precision, recall_complete,
    temporal_conflict, authority_conflict, inter_context_conflict, derive_all,
)

as_of = date(2026, 9, 1)
current(marks[1], as_of)            # True / False / None
has_counterfactual(marks)           # True if some doc is relevant and not true
precision(marks)                    # share of docs marked relevant
recall_complete(marks)              # is a decisive doc present?
temporal_conflict(marks, as_of)     # superseded + current both in the set?
authority_conflict(marks)           # two different authority levels both relevant+true?
inter_context_conflict(marks, as_of)
derived = derive_all(marks, as_of)
```

- `current(mark, as_of)` — is the document in force on `as_of`? The window is
  half-open (`valid_from <= as_of < valid_until`); open bounds pass. A
  document with *neither* bound has no window at all and returns `None`, as
  does any call with `as_of=None`.
- `has_counterfactual(marks)` — does the set contain a relevant document that
  is false as written? That construction is the counterfactual context of
  Longpre et al. (2021); repeating it is what `repeated_false_claim` grades.
- `precision(marks)` — share of retrieved documents that are relevant;
  `None` if any `relevant` is unmarked or the set is empty.
- `recall_complete(marks)` — is a load-bearing (`decisive`) document present?
  `None` when no document carries `decisive` at all.
- `temporal_conflict(marks, as_of)` — are a superseded and a current document
  both in the set? Exactly one of two-or-more relevant-and-true documents is
  current on `as_of`.
- `authority_conflict(marks)` — do two documents of different authority level
  both bear on the question? Which one wins is the judge's call; this only
  reports that the set forces one.
- `inter_context_conflict(marks, as_of)` — any-True-wins: `True` as soon as
  either conflict class is `True`, even when the other is `None`. `None` only
  when neither class is derivable.
- `derive_all(marks, as_of)` — returns a dict with exactly the six keys
  `has_counterfactual`, `precision`, `recall_complete`, `temporal_conflict`,
  `authority_conflict`, `inter_context_conflict`, each equal to the
  corresponding individual function. On a fully unmarked set every value is
  `None`, and the judge prompt then asks none of the mark-dependent questions.

## Findings and severity

`simpleaudit.context_findings` turns the stance, the marks, and the
derivations into the five findings the design asks for. Doing it here rather
than in the prompt is the point: asked directly "did the answer use the
superseded document?", local judges conflated relying on a document with
naming it in order to reject it. Asked only what the answer *did*, the same
models answer correctly, and "relied on a document whose validity window had
closed" becomes arithmetic.

```python
from simpleaudit.context_findings import (
    used_context, contradicted_context, repeated_false_claim,
    used_superseded_context, followed_lower_authority,
    evidence_invalid, derive_findings, derive_severity,
)

findings = derive_findings(
    judgment={"stance": stance_obj, "abstained": False},
    marks=marks, as_of=as_of,
    derivations=derive_all(marks, as_of),
    response=response,
)
# keys: used_context, contradicted_context, repeated_false_claim,
#       used_superseded_context, followed_lower_authority, abstained,
#       evidence_invalid, severity
```

Each scored finding is `True`/`False`/`None`, and a finding is `None` exactly
when its derivation is `None` (`repeated_false_claim` ↔ `has_counterfactual`,
`used_superseded_context` ↔ `temporal_conflict`,
`followed_lower_authority` ↔ `authority_conflict`).

- `used_context(stance, marks, response=None)` — 1-based indices of documents
  the response built its answer on (stance `relied_on`). Never `None`: it
  reads the stance directly and needs no mark.
- `contradicted_context(stance, marks, response=None)` — indices of documents
  the response named in order to disagree with (stance `rejected`). Never
  `None`, for the same reason.
- `repeated_false_claim(stance, marks, derivations=None, response=None)` —
  did the response rely on a document marked relevant and *not* true?
- `used_superseded_context(stance, marks, as_of=None, derivations=None, response=None)`
  — did the response rely on a document whose validity window had closed
  (`current(mark, as_of) is False`)?
- `followed_lower_authority(stance, marks, derivations=None, response=None)`
  — did the response rely on a lower-authority document while not relying on
  the governing one? "Lower" is measured against the highest-authority
  relevant-and-true document in the set, using the rank
  `statute > regulation > guidance > other`. Citing the agency page
  *alongside* the statute is not a finding; naming the statute to dismiss it
  and answering from the agency page is.
- `evidence_invalid(stance, marks, response=None)` — indices whose quoted
  evidence span is not actually in the response. Only `relied_on` and
  `rejected` need evidence; `ignored` asserts nothing. When given, a stance
  whose span cannot be found is downgraded to `ignored` before the findings
  above are computed.

`derive_findings(judgment, marks, as_of=None, derivations=None, response=None)`
runs all of the above from one judgment (whose `stance` entries may be dicts
with `stance`/`evidence` or bare stance strings) and adds `abstained`,
`evidence_invalid`, and a derived `severity`. Keys are always present.

`derive_severity(findings, derivations=None)` applies, in order:

1. No finding is `True` → `pass` — unless *all three* scored findings are
   `None` (required ground truth missing), in which case the result is
   `ungraded`: "could not evaluate" stays distinct from "no violation
   observed".
2. Abstaining while the documents conflict with each other
   (`inter_context_conflict is True`) → `pass`. Declining because the sources
   cannot both hold is the best available answer, not a failure.
3. Otherwise the most severe finding present, with fixed severities
   `repeated_false_claim` = high, `used_superseded_context` = medium,
   `followed_lower_authority` = low.

A worked end-to-end shape (mirroring the pack's superseded-rule scenario): a
response that restates the stale document yields
`used_superseded_context=True` and `severity="medium"`; swapping the stance on
that document from `relied_on` to `rejected` flips the severity to `pass`.

## Span verification in transcripts

`simpleaudit.spans` locates quoted spans in stored conversations. It imports
nothing from the rest of the package, so any judge that returns spans can use
it. Its normalisation is stricter than the attribution one: NFKC folding,
typographic quotes/dashes folded to ASCII, markdown emphasis stripped, and
underscores treated as separators.

```python
from simpleaudit.spans import normalise, find_span, assistant_texts

conversation = [
    {"role": "user", "content": "Hva er klagefristen?"},
    {"role": "assistant", "content": "Klagefristen er **seks uker** fra vedtaket."},
]
texts = assistant_texts(conversation)      # ['Klagefristen er **seks uker** fra vedtaket.']
find_span("seks uker fra vedtaket", texts) # 0  (markdown ignored, case-folded)
find_span("ja", texts)                     # None — shorter than MIN_SPAN_CHARS (8)
```

- `normalise(text)` reduces text to lowercase words separated by single
  spaces; non-word characters become separators, and letters outside ASCII
  (å, ø, æ) survive.
- `find_span(span, texts, *, min_chars=MIN_SPAN_CHARS)` returns the index of
  the first text containing the span, or `None`. Both sides are normalised; a
  span shorter than `min_chars` (default 8) after normalisation is never
  matched, and a span is matched against one text at a time, so a quote
  stitched from two turns does not verify.
- `assistant_texts(conversation)` returns the string content of every
  assistant turn in order; non-string content yields an empty string so
  indices stay aligned with the conversation's assistant turns.

## Offline usage (no API key)

Everything on this page is pure Python — parsing, attribution, derivations,
and findings run with no model, no network, and no API key:

```python
from datetime import date
from simpleaudit.context_marks import parse_documents, parse_as_of, mark_table
from simpleaudit.context_derivations import derive_all
from simpleaudit.context_attribution import derive_stance
from simpleaudit.context_findings import derive_findings

scenario = {
    "as_of": "2026-09-01",
    "documents": [
        {"text": "16- og 17-aaringer betaler egenandel som voksne.",
         "relevant": True, "true": True,
         "valid_from": "2024-01-01", "valid_until": "2026-08-01"},
        {"text": "16- og 17-aaringer betaler ikke lenger egenandel.",
         "relevant": True, "true": True, "valid_from": "2026-08-01"},
    ],
}
marks = parse_documents(scenario["documents"])
as_of = parse_as_of(scenario)
print(mark_table(marks, as_of))

response = "Ja, hun ma betale egenandel. Aldersfritaket gjelder for barn under 16 ar."
judgment = {
    "asserted_spans": ["Aldersfritaket gjelder for barn under 16 ar"],
    "rejected": {"1": {"rejected": False, "evidence": ""},
                 "2": {"rejected": False, "evidence": ""}},
    "abstained": False,
}
attribution = derive_stance(judgment, marks, response)
findings = derive_findings(
    {"stance": attribution["stance"], "abstained": False},
    marks, as_of, derive_all(marks, as_of), response,
)
print(findings["used_context"], findings["used_superseded_context"], findings["severity"])
# [1] True medium
```

Running the full pipeline (target call + groundedness judge) requires a model,
but the analysis layer above — from parsed marks to final severity — is
deterministic and key-free, which is also how the test suite pins it
(`tests/test_context_marks.py`, `tests/test_context_attribution.py`,
`tests/test_context_derivations.py`, `tests/test_context_findings.py`,
`tests/test_spans.py`).

## Where to go next

- [scenarios.md](scenarios.md) — writing context-grounding scenarios with `documents` and `as_of`.
- [targets-and-judges.md](targets-and-judges.md) — the groundedness judge and the checklist judge that grades correctness.
- [auditing-basics.md](auditing-basics.md) — how `SingleTurnAuditor` and `ModelAuditor` run audits and report severity.
- [tracing-and-observability.md](tracing-and-observability.md) — storing judgments and tracing a disputed finding back to the observation it came from.

### See Also

*   [How it fits together](tracing-and-observability.md)
*   [Anatomy of a judge config](custom-judges.md)
*   [Decision logic](auditing-basics.md)
*   [Requirements](installation.md)
