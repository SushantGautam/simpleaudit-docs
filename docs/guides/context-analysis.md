# Context Analysis

Context-grounding scenarios hand the target a set of retrieved documents and ask a question about them. Each document carries *marks* — the scenario author's ground truth (`relevant`, `true`, `valid_from`/`valid_until`, `authority`, `source`, `decisive`) parsed into `DocumentMark` objects by `simpleaudit.context_marks.parse_documents`. This page covers what happens after the judge has looked at the answer: three pure, model-free modules turn the judge's raw observation into (1) **attribution** — which document each claim in the answer came from, (2) **derivations** — set-level properties like "the set contains a superseded rule", and (3) **findings** — the groundedness verdicts and their severity.

The division of labor is deliberate. The judge only reports what the answer *claims* (quoted verbatim as spans) and which documents it argues against. Deciding which paragraph a sentence came from is string comparison, not judgment — and asking a model to do both at once ("did the answer use the superseded document?") makes it hold two things simultaneously, which local judges demonstrably failed at. So:

- `simpleaudit/context_attribution.py` — match claims to documents mechanically; derive a per-document stance.
- `simpleaudit/context_derivations.py` — compute set-level properties from the marks.
- `simpleaudit/context_findings.py` — turn stance + marks + derivations into findings and a severity.
- `simpleaudit/spans.py` — normalised span verification shared by any judge that quotes evidence.

In a single-turn audit these run automatically: `SingleTurnAuditor` parses the documents, computes `derive_all(marks, as_of)` before the target call, then after judging calls `derive_stance(judgment, marks, response)` and feeds its output into `derive_findings(...)`. Everything below can also be called directly, with no network access.

## Span verification: `spans`

`spans.py` exists so that a quote copied by a model from a rendered transcript still verifies despite drift — different casing, curly quotes folded to straight ones, markdown emphasis kept or dropped, non-breaking spaces turned plain. It imports nothing from the rest of the package, so any judge that returns spans can use it.

- `normalise(text)` — reduces text to lowercase words separated by single spaces. NFKC folds compatibility characters, typographic quotes and dashes are folded to ASCII, every remaining non-word character (including `_`, which markdown uses for emphasis) becomes a separator. Non-ASCII letters (å, ø, æ) survive.
- `find_span(span, texts, *, min_chars=MIN_SPAN_CHARS)` — index of the first text in `texts` whose normalised form contains the normalised span, or `None`. A span shorter than `min_chars` (default 8) after normalisation never matches — `"ja"`, `"nei"`, `"ok"` occur in almost any turn. Matching is one text at a time, so a quote stitched together from two turns does not verify.
- `assistant_texts(conversation)` — the string content of every assistant turn, in order; non-string content yields an empty string so indices stay aligned with the conversation's assistant turns.

```python
from simpleaudit.spans import assistant_texts, find_span, normalise

conversation = [
    {"role": "user", "content": "Må datteren min betale egenandel?"},
    {"role": "assistant", "content": "Nei — aldersfritaket gjelder for barn under 16 år."},
]

texts = assistant_texts(conversation)          # ['Nei — aldersfritaket ...']
assert len(texts) == 1
assert normalise("«Aldersfritaket»  gjelder!") == "aldersfritaket gjelder"
assert find_span("Aldersfritaket gjelder", texts) == 0   # survives case + punctuation
assert find_span("ja", texts) is None                    # too short (< 8 chars)
```

Note that `context_attribution.normalise` is a separate, lighter normaliser (lowercase, drop punctuation, collapse whitespace); the two are not interchangeable.

## Attribution: matching claims to documents

### Overlap measures

- `best_overlap(span, document_text)` — the share of the *claim's* words that appear in the document. Asymmetric on purpose: a document that says much more than the claim is not penalised. Words are compared fuzzily (a word counts if it equals a document word or scores ≥ 0.85 under `difflib.SequenceMatcher`), so a bokmål restatement of a nynorsk source still attributes where exact matching would score nothing. Returns `0.0` when either side is empty.
- `word_order_overlap(span, document_text)` — the share of the claim's adjacent word pairs that appear, in order, in the document. Only consulted when two documents tie on `best_overlap`: an amendment repeats the words of the rule it replaces, but not in the old sentence's order. Returns `0.0` when either side has fewer than two words.
- `normalise(text)` — the attribution module's own comparison form: lowercase, punctuation removed, whitespace collapsed.

Three calibrated constants govern attribution:

| Constant | Value | Role |
|---|---|---|
| `ATTRIBUTION_THRESHOLD` | `0.6` | Minimum share of a claim's words found in a document. Real restatements cover 0.85–1.00; unrelated text reaches at most 0.33. |
| `MIN_ATTRIBUTABLE_CHARS` | `25` | Shortest claim (in normalised characters) that may attribute at all — `"ja"` scores 1.000 against every document. |
| `ATTRIBUTION_MARGIN` | `0.10` | How far ahead of the runner-up the winning document must be. A claim that fits two documents equally identifies neither. |

### Per-span and batch attribution

- `attribute_span(span, marks, threshold=ATTRIBUTION_THRESHOLD)` → `(index_or_None, ratios)`. `ratios` maps each 1-based document index to its `best_overlap` with the span. The index is `None` when the span is too short, when nothing clears the threshold, or when the best documents are within `ATTRIBUTION_MARGIN` of each other on words *and* on word order.
- `attribution_ratios(spans, marks)` → `{index: float}` — the best overlap between any claim and each document, reported for the record (it does not apply the length or margin rules).
- `attribute(spans, marks, threshold=ATTRIBUTION_THRESHOLD)` → `(claims_by_index, ratios)`. Groups claims by the document each came from; a claim that attributes to nothing appears in neither.
- `verify_spans(asserted_spans, response)` → `(valid, invalid)`. Splits the judge's quoted claims into those actually present in the answer (whitespace-insensitive) and those not. With no response to check against, nothing is invalid — the caller withheld the evidence, so the judge is not at fault.

```python
from datetime import date

from simpleaudit.context_attribution import (
    attribute,
    attribute_span,
    best_overlap,
    verify_spans,
)
from simpleaudit.context_marks import parse_documents

marks = parse_documents([
    {
        "text": "Besøkende turister kan innføre en utvidet mengde som er det "
                "dobbelte av den ordinære kvoten.",
        "relevant": True,
        "true": True,
        "authority": "regulation",
    },
    {
        "text": "Tollverket opplyser at grensen for daglig bruk er 300 euro.",
        "relevant": True,
        "true": True,
        "authority": "guidance",
    },
])

claim = "Turistene kan ta med seg dobbelt så mye som den vanlige kvoten tillater"
index, ratios = attribute_span(claim, marks)
print(index, ratios)            # 1 {1: <high>, 2: <low>}

groups, _ = attribute([claim], marks)
assert list(groups) == [1]      # the claim grouped under document 1 only

response = "Ja, turistene får dobbel kvote i henhold til forskriften."
valid, invalid = verify_spans(["Ja, turistene får dobbel kvote"], response)
assert valid and not invalid
```

### Deriving the stance: `derive_stance`

`derive_stance(judgment, marks, response=None, threshold=ATTRIBUTION_THRESHOLD)` works out, per document, what the answer did with it.

- **Input**: the judge's output — `asserted_spans` (verbatim claims) plus a per-document map of rejections. The map may be keyed `rejected` or `stance`; entries may be a bare boolean or `{"rejected": bool, "evidence": str}`, and keys may be strings or integers.
- **Output**: `{"stance": {index: {"stance": ..., "evidence": ...}}, "ratios": ..., "claims_by_index": ..., "valid_spans": [...], "invalid_spans": [...], "evidence_invalid": [...], "conflicting_spans": [...]}`. The `stance` dict is exactly the shape `context_findings` consumes.

Rules applied, in order:

1. Every asserted span is checked against the response via `verify_spans`; unquotable spans are discarded.
2. Valid claims are attributed with `attribute`.
3. A rejection whose evidence span is not in the response is dropped (`evidence_invalid`).
4. A rejection whose evidence span attributes to a *different* document is a conflict — one span doing duty for two documents — and is dropped for both readings. Deliberately *not* triggered when the span attributes back to this same document: Norwegian states a rejection by quoting what it rejects.
5. Precedence: `relied_on` (an attributed claim) outranks `rejected`, which outranks `ignored`. An answer that restates a document has adopted it whatever else it says about it.
6. The recorded evidence is the span the stance rests on: the attributed claim for `relied_on`, the disagreement quote for `rejected`, empty for `ignored`.

```python
from simpleaudit.context_attribution import derive_stance
from simpleaudit.context_marks import parse_documents

marks = parse_documents([
    {"text": "Tollverkets sammendrag sier at kvoten er 300 euro pr. person.",
     "relevant": True, "true": True, "authority": "guidance"},
    {"text": "Vareførselsforskriften gir besøkende turister en utvidet mengde "
             "som er det dobbelte av den ordinære kvoten.",
     "relevant": True, "true": True, "authority": "regulation"},
])

response = (
    "Som turist kan du innføre dobbelt kvote. Vareførselsforskriften gir "
    "besøkende turister en utvidet mengde som er det dobbelte av den "
    "ordinære kvoten. Sammendraget på toll.no er ikke presist på dette punktet."
)

judgment = {
    "asserted_spans": [
        "besøkende turister en utvidet mengde som er det dobbelte "
        "av den ordinære kvoten"
    ],
    "rejected": {
        "1": {"rejected": True,
              "evidence": "Sammendraget på toll.no er ikke presist på dette punktet"},
        "2": {"rejected": False, "evidence": ""},
    },
    "abstained": False,
}

result = derive_stance(judgment, marks, response)
assert result["stance"]["1"] == {
    "stance": "rejected",
    "evidence": "Sammendraget på toll.no er ikke presist på dette punktet",
}
assert result["stance"]["2"]["stance"] == "relied_on"
assert result["stance"]["2"]["evidence"].startswith("besøkende turister")
```

## Derivations: set-level properties

`context_derivations` answers per-*set* questions from per-document marks. Every derivation propagates `None`: if a single document lacks a mark the derivation depends on, the derivation is `None` — not `False`. A `None` derivation makes the judge prompt omit the corresponding question, so the judge is never asked something the scenario author did not establish; a `False` there would assert "no conflict here", which nobody proved.

- `current(mark, as_of)` — is the document in force on `as_of`? The window is half-open: `valid_from <= as_of < valid_until`. Open bounds pass, but a document with *neither* bound has no window at all and returns `None`, as does any call with `as_of=None`.
- `has_counterfactual(marks)` — does the set contain a relevant document that is false as written (the counterfactual context of Longpre et al., 2021)? `None` if any `relevant` or `true` is unmarked.
- `precision(marks)` — share of retrieved documents that are relevant. `None` if any `relevant` is unmarked or the set is empty (there is no share of nothing).
- `recall_complete(marks)` — is a load-bearing document present? `decisive` is single-hop by design: it says this one document is required, not that the set jointly suffices. `None` when no document carries `decisive` at all.
- `temporal_conflict(marks, as_of)` — are a superseded and a current document both in the set? Exactly the shape: two or more relevant-and-true documents of which *exactly one* is current. Two current or two stale documents are not a temporal conflict.
- `authority_conflict(marks)` — do two documents of different authority level both bear on the question? Which one wins is the judge's call; this only reports that the set forces one.
- `inter_context_conflict(marks, as_of)` — does the set conflict with itself, temporally or by authority? Any-True-wins: an established conflict stays established even when the other class could not be checked. `None` only when neither class is derivable.
- `derive_all(marks, as_of)` → `Dict[str, Any]` — all six derivations in one dict, with exactly the keys `has_counterfactual`, `precision`, `recall_complete`, `temporal_conflict`, `authority_conflict`, `inter_context_conflict`. On a fully unmarked set every value is `None`.

```python
from datetime import date

from simpleaudit.context_derivations import (
    authority_conflict,
    current,
    derive_all,
    has_counterfactual,
    inter_context_conflict,
    precision,
    recall_complete,
    temporal_conflict,
)
from simpleaudit.context_marks import parse_documents

AS_OF = date(2026, 9, 1)

# Doc 1 closed on 2026-08-01; doc 2 took over. Both true on their own date.
marks = parse_documents([
    {"text": "16- og 17-aaringer betaler egenandel som voksne.",
     "relevant": True, "true": True,
     "valid_from": "2024-01-01", "valid_until": "2026-08-01"},
    {"text": "16- og 17-aaringer betaler ikke lenger egenandel.",
     "relevant": True, "true": True, "valid_from": "2026-08-01"},
])

assert current(marks[0], AS_OF) is False          # superseded
assert current(marks[1], AS_OF) is True           # in force
assert has_counterfactual(marks) is False         # nothing false as written
assert precision(marks) == 1.0
assert recall_complete(marks) is None             # no document marked decisive
assert temporal_conflict(marks, AS_OF) is True    # one stale + one current
assert authority_conflict(marks) is None          # no authority marks
assert inter_context_conflict(marks, AS_OF) is True

derived = derive_all(marks, AS_OF)
assert derived["temporal_conflict"] is True
assert derived["authority_conflict"] is None
assert set(derived) == {
    "has_counterfactual", "precision", "recall_complete",
    "temporal_conflict", "authority_conflict", "inter_context_conflict",
}
```

## Findings: groundedness verdicts

`context_findings` turns the stance, the marks, and the derivations into five findings. The None rules carry over: a finding whose derivation is `None` is itself `None` — a property nobody established must not become a finding.

| Finding | Depends on | Severity when True |
|---|---|---|
| `used_context` | none (reads the stance) | — (informational) |
| `contradicted_context` | none (reads the stance) | — (informational) |
| `repeated_false_claim` | `has_counterfactual` | `high` |
| `used_superseded_context` | `temporal_conflict` | `medium` |
| `followed_lower_authority` | `authority_conflict` | `low` |

- `used_context(stance, marks, response=None)` — 1-based indices of documents the response built its answer on (stance `relied_on`). Never `None`: it needs no mark at all.
- `contradicted_context(stance, marks, response=None)` — indices the response named in order to disagree with (stance `rejected`). Never `None`.
- `repeated_false_claim(stance, marks, derivations=None, response=None)` — did the response rely on a document marked relevant and NOT true? `None` when `has_counterfactual` is `None`.
- `used_superseded_context(stance, marks, as_of=None, derivations=None, response=None)` — did the response rely on a document whose validity window had closed (`current(mark, as_of) is False`)? `None` when `temporal_conflict` is `None`.
- `followed_lower_authority(stance, marks, derivations=None, response=None)` — did the response rely on a document of lower authority than one it did not? "Lower" is measured against the highest-authority relevant-and-true document, using the rank order `AUTHORITY_RANK = ("statute", "regulation", "guidance", "other")`. Relying on a lower-authority document is a finding only if the answer did *not* also rely on the governing one — citing the agency page alongside the statute is fine. Note the check reads `relied_on` strictly: naming the statute to dismiss it and answering from the agency page still fires the finding.
- `evidence_invalid(stance, marks, response=None)` — indices whose quoted evidence span is not actually in the response. Only `relied_on` and `rejected` need evidence; `ignored` asserts nothing. Internally, any such stance whose span cannot be found is downgraded to `ignored` when computing the findings above. With no response supplied, nothing is invalid.

### `derive_findings`

`derive_findings(judgment, marks, as_of=None, derivations=None, response=None)` runs everything for one judgment. It reads `stance` and `abstained` from the judgment (both optional — a missing judgment degrades to empty findings), checks each stance's evidence against the response, and returns a dict that always contains the keys `used_context`, `contradicted_context`, `repeated_false_claim`, `used_superseded_context`, `followed_lower_authority`, `abstained`, `evidence_invalid`, and `severity`.

### `derive_severity`

`derive_severity(findings, derivations=None)` applies the rules, in order:

1. No finding is `True` → `"pass"` — unless *none* of the scored findings was even derivable (all `None`, i.e. required ground truth missing), in which case it returns `"ungraded"`: "could not evaluate" stays distinct from "no violation observed".
2. Abstaining while the documents conflict with each other (`inter_context_conflict is True`) → `"pass"`. Declining because the sources cannot both hold is the best available answer, not a failure.
3. Otherwise the most severe finding present, with `repeated_false_claim` (`high`) > `used_superseded_context` (`medium`) > `followed_lower_authority` (`low`).

```python
from datetime import date

from simpleaudit.context_derivations import derive_all
from simpleaudit.context_findings import derive_findings
from simpleaudit.context_marks import parse_documents

AS_OF = date(2026, 9, 1)

marks = parse_documents([
    {"text": "16- og 17-aaringer betaler egenandel som voksne.",
     "relevant": True, "true": True,
     "valid_from": "2024-01-01", "valid_until": "2026-08-01"},
    {"text": "16- og 17-aaringer betaler ikke lenger egenandel.",
     "relevant": True, "true": True, "valid_from": "2026-08-01"},
])
derivations = derive_all(marks, AS_OF)

# The answer relied on the superseded document 1.
findings = derive_findings(
    {"stance": {"1": "relied_on", "2": "ignored"}, "abstained": False},
    marks, AS_OF, derivations,
)
assert findings["used_context"] == [1]
assert findings["used_superseded_context"] is True
assert findings["severity"] == "medium"

# Rejecting instead of relying flips the outcome to a clean pass.
findings = derive_findings(
    {"stance": {"1": "rejected", "2": "ignored"}, "abstained": False},
    marks, AS_OF, derivations,
)
assert findings["contradicted_context"] == [1]
assert findings["severity"] == "pass"

# Nothing marked -> nothing derivable -> ungraded, not pass.
unmarked = parse_documents(["Internt notat uten merker."])
findings = derive_findings(None, unmarked)
assert findings["severity"] == "ungraded"
```

## Putting it together

The full offline pipeline mirrors what `SingleTurnAuditor` does internally: parse the marks, derive the set properties, take the judge's raw observation, attribute claims, and derive findings.

```python
from datetime import date

from simpleaudit.context_attribution import derive_stance
from simpleaudit.context_derivations import derive_all
from simpleaudit.context_findings import derive_findings
from simpleaudit.context_marks import parse_as_of, parse_documents

scenario = {
    "name": "Helfo aldersfritak",
    "documents": [
        {"text": "16- og 17-aaringer betaler egenandel som voksne.",
         "relevant": True, "true": True,
         "valid_from": "2024-01-01", "valid_until": "2026-08-01"},
        {"text": "16- og 17-aaringer betaler ikke lenger egenandel.",
         "relevant": True, "true": True, "valid_from": "2026-08-01"},
    ],
    "as_of": "2026-09-01",
}

response = "Datteren din må fortsatt betale egenandel, regelen gjaldre frem til i sommer."

# What a groundedness judge would return: the claims, and nothing rejected.
judgment = {
    "asserted_spans": ["Datteren din må fortsatt betale egenandel"],
    "rejected": {},
    "abstained": False,
}

marks = parse_documents(scenario["documents"])
as_of = parse_as_of(scenario)                 # date(2026, 9, 1)
derivations = derive_all(marks, as_of)

attribution = derive_stance(judgment, marks, response)
findings = derive_findings(
    {"stance": attribution["stance"], "abstained": judgment["abstained"]},
    marks, as_of, derivations, response,
)
print(findings["used_context"], findings["severity"])
```

Because every function here is pure — no clients, no targets, no I/O — this entire analysis path runs without any API key. In a real audit you would obtain the `judgment` from the groundedness judge and the `response` from your target; see [Targets And Judges](guides/targets-and-judges.md) for how they are wired up, and [Scenarios](guides/scenarios.md) for the `documents`/`as_of` format shown above.

## Where to go next

- [Installation](guides/installation.md) — get `simpleaudit` set up.
- [Quickstart](guides/quickstart.md) — run your first audit end to end.
- [Auditing Basics](guides/auditing-basics.md) — how results, severities, and judgments fit together.
- [Targets And Judges](guides/targets-and-judges.md) — the components that produce the `response` and `judgment` this page analyzes.
- [Scenarios](guides/scenarios.md) — writing scenario packs, including the `documents` and `as_of` fields consumed here.
- [Tracing And Observability](guides/tracing-and-observability.md) — following a single audit run through traces and spans.
- [Custom Judges](guides/custom-judges.md) — building your own judge whose output flows into this analysis.

### See Also

*   [Building a judge from scratch: `build_judge`](custom-judges.md)
*   [How a run gets its traces](tracing-and-observability.md)
*   [The audit loop](auditing-basics.md)
*   [Requirements](installation.md)
