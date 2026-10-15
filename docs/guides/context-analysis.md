# Context Analysis

When a scenario supplies retrieved documents, SimpleAudit does not ask the judge "which document did the answer use?" or "is this answer grounded?". The judge is asked only what it can actually observe: **what the answer claims** (quoted verbatim) and **which documents it argues against**. Everything else — which document a claim came from, whether the document set conflicts with itself, and which findings fire — is computed mechanically in three modules:

| Module | Responsibility |
| --- | --- |
| `simpleaudit.context_attribution` | Match the answer's quoted claims to source documents by string overlap; derive a per-document stance. |
| `simpleaudit.context_derivations` | Set-level properties of the document marks (`has_counterfactual`, conflicts, precision/recall). |
| `simpleaudit.context_findings` | Turn the stance plus the derivations into findings and a severity. |

They run inside `SingleTurnAuditor` when a scenario carries marked documents: the auditor parses the scenario's documents into `DocumentMark`s (via `parse_documents`), reads the scenario date with `parse_as_of`, computes `derivations = derive_all(marks, as_of)`, runs the groundedness judge, then calls `derive_stance(judgment, marks, response)` followed by `derive_findings(...)`. The raw stance, the claimed spans, and the attribution ratios are kept on the judgment so a disputed finding can be traced back to the observation without re-running the judge.

The design rule that runs through all three modules: **a property nobody established stays `None`**. If a document lacks a mark, the derivation that depends on it returns `None` (not `False`); if a derivation is `None`, the dependent finding is `None`; and a finding whose evidence span cannot be found in the response is downgraded rather than trusted.

## Span verification

Before any analysis, quoted text has to be verified against the actual transcript. Two normalisation helpers exist for this:

- `spans.normalise(text)` — NFKC-folds compatibility characters, folds typographic quotes/dashes to ASCII, casefolds, and reduces every non-word character (punctuation, markdown symbols, emoji, underscores) to a separator. What survives is wording; formatting is dropped. Non-ASCII letters such as å/ø/æ survive.
- `context_attribution.normalise(text)` — the lighter form used for attribution comparison: lowercase, drop punctuation, collapse whitespace.

`find_span(span, texts, *, min_chars=8)` returns the index of the first text in `texts` that contains the normalised span, or `None`. A span shorter than `min_chars` after normalisation never matches ("ja", "nei", "ok" occur in almost any turn), and a span is matched against one text at a time, so a quote stitched together from two turns does not verify.

`assistant_texts(conversation)` extracts the string content of every assistant turn, in order (non-string content yields an empty string so indices stay aligned). Together they give you the candidate haystacks for `find_span`:

```python
from simpleaudit.spans import assistant_texts, find_span

conversation = [
    {"role": "user", "content": "Hva er egenandelen for 16-åringer?"},
    {"role": "assistant", "content": "Aldersfritaket for egenandel gjelder for barn under 16 år."},
]

texts = assistant_texts(conversation)
index = find_span("aldersfritaket for egenandel gjelder", texts)
print(index)  # 0 — the quote verifies against the (only) assistant turn
```

In `context_attribution`, the equivalent check is `verify_spans(asserted_spans, response)`, which splits the judge's quoted claims into those actually present in the answer and those not. A span the judge could not copy correctly is a claim about an answer it did not read, so it is discarded before attribution ever runs. With no response to check against, nothing is invalid — the caller withheld the evidence, so the judge is not at fault.

## Attribution: matching claims to documents

### Overlap measures

`best_overlap(span, document_text)` is the core score: the **share of the claim's words** that appear in the document. It is deliberately asymmetric — a document that says much more than the claim is not penalised, since claims are sentences and retrieved chunks are paragraphs. Words are compared fuzzily (two words count as the same at `SequenceMatcher.ratio() >= 0.85`), which is what lets a bokmål restatement of a nynorsk source ("gjelder … Norge, også turister") attribute to its source where an exact-match rule scores nothing. Character-level coverage was measured and rejected because an unrelated sentence can find its own letters scattered through a long document.

`word_order_overlap(span, document_text)` is the tie-breaker: the share of the claim's adjacent word pairs that appear, **in order**, in the document. It is consulted only when two documents tie on `best_overlap` — an amendment repeats the rule it replaces, so the old rule's words are all in the new document too, but not in the old sentence's order. Returns `0.0` when either side has fewer than two words.

Three module constants pin the behaviour:

- `ATTRIBUTION_THRESHOLD = 0.6` — share of the claim's words that must be traceable in a document. Calibrated over the pack: real restatements cover 0.85–1.00, unrelated text reaches at most 0.33.
- `MIN_ATTRIBUTABLE_CHARS = 25` — shortest claim (normalised characters) that may attribute at all. Short claims find their own few characters anywhere: "ja" scores 1.000 against every document.
- `ATTRIBUTION_MARGIN = 0.10` — how far ahead of the runner-up the winning document must be. A claim that fits two documents equally identifies neither.

### Per-claim and bulk attribution

`attribute_span(span, marks, threshold=ATTRIBUTION_THRESHOLD)` returns `(index_or_None, ratios)`, where `ratios` maps each 1-based document index to its `best_overlap`. The index is `None` when the claim is too short, when nothing clears the threshold, or when the best documents are within the margin of each other on words **and** on word order.

`attribute(spans, marks, threshold=ATTRIBUTION_THRESHOLD)` groups many claims: it returns `(claims_by_index, ratios)` where `claims_by_index` maps a 1-based document index to the list of claims attributed to it. A claim that attributes to nothing appears in neither — an unattributable claim is not evidence that the answer relied on any particular document.

`attribution_ratios(spans, marks)` reports, for the record, the best overlap between *any* claim and each document, by 1-based index. It applies none of the length or margin rules — the stance is decided by `attribute_span`.

```python
from datetime import date
from simpleaudit.context_attribution import (
    attribute, attribute_span, attribution_ratios, best_overlap,
    word_order_overlap, verify_spans,
)
from simpleaudit.context_marks import DocumentMark

old_rule = DocumentMark(
    text="Aldersfritaket for egenandel gjelder for barn under 16 år.",
    relevant=True, true=True, valid_from=date(2019, 1, 1), valid_until=date(2024, 1, 1),
    authority="regulation",
)
new_rule = DocumentMark(
    text="Aldersfritaket for egenandel gjelder for barn under 18 år.",
    relevant=True, true=True, valid_from=date(2024, 1, 1),
    authority="regulation",
)
marks = [old_rule, new_rule]

claim = "Aldersfritaket for egenandel gjelder for barn under 16 år"
print(best_overlap(claim, old_rule.text))   # 1.0 — every word of the claim is there
print(best_overlap(claim, new_rule.text))   # 0.875 — "16"/"18" is the miss
index, ratios = attribute_span(claim, marks)
print(index)                                # 1 — clear winner beyond the margin

by_index, _ = attribute([claim], marks)
print(by_index)                             # {1: ['Aldersfritaket ... 16 år']}
print(attribution_ratios([claim], marks))   # {1: 1.0, 2: 0.875}
print(word_order_overlap(claim, old_rule.text))  # 1.0 — pairs in order
valid, invalid = verify_spans([claim], "Ja, " + claim + ".")
print(invalid)                              # [] — the span is in the response
```

### Deriving the stance

`derive_stance(judgment, marks, response=None, threshold=ATTRIBUTION_THRESHOLD)` works out, per document, what the answer did with it. The `judgment` is the groundedness judge's output: `asserted_spans` (the answer's claims, quoted) plus a per-document `{rejected, evidence}` map (read under the key `rejected` or `stance`). For each 1-based document index it produces a stance of `"relied_on"`, `"rejected"`, or `"ignored"`:

- An attributed claim makes the document `relied_on` — **relied_on outranks rejected**: an answer that restates a document has adopted it whatever else it says about it.
- Otherwise, a judge-claimed rejection (with evidence found in the response) makes it `rejected`.
- Otherwise it is `ignored`.

Two guards protect the stance. A rejection whose quoted evidence is not in the response is dropped (a rejection the judge cannot point at is not an observation). And a conflict rule catches one span doing duty for two *different* documents — offered as proof the answer disagrees with one while also reading as a restatement of another; neither reading is kept. Deliberately, quoting what you reject is **not** a conflict: Norwegian states a rejection by quoting the claim it denies, and treating that as a contradiction would discard correct rejections.

The return value is `{"stance": {index: {"stance": ..., "evidence": ...}}, "ratios": ..., "claims_by_index": ..., "valid_spans": [...], "invalid_spans": [...], "evidence_invalid": [...], "conflicting_spans": [...]}` — the `stance` shape is exactly what `context_findings` consumes.

## Derivations: set-level properties

Per-document marks describe one document; a judge needs per-*set* facts. Each function in `simpleaudit.context_derivations` is one row of that table, and each propagates `None` whenever a mark it depends on is missing.

- `current(mark, as_of)` — is the document in force on `as_of`? The window is half-open: `valid_from <= as_of < valid_until`. Open bounds pass; a document with *neither* bound has no window at all and returns `None` (an unmarked document is never assumed current). Also `None` when `as_of` is `None`.
- `has_counterfactual(marks)` — does the set contain a relevant document that is false as written? That construction (relevant, on topic, wrong) is the counterfactual context of Longpre et al. (2021); repeating it is the failure `repeated_false_claim` grades.
- `precision(marks)` — share of retrieved documents that are relevant. `None` if any `relevant` is unmarked or the set is empty (there is no share of nothing).
- `recall_complete(marks)` — is a load-bearing (`decisive`) document present? `None` when no document carries `decisive` at all. Single-hop by design: it says one document is required, not that the set jointly suffices.
- `temporal_conflict(marks, as_of)` — are a superseded and a current document both in the set? Exactly the shape: two or more relevant-and-true documents of which *exactly one* is current on `as_of`. Two current documents are not a temporal conflict, nor are two stale ones.
- `authority_conflict(marks)` — do two documents of different authority level both bear on the question? Statute against agency guidance, both true on their own surface. Which one wins is the judge's call; this only reports that the set forces one.
- `inter_context_conflict(marks, as_of)` — does the set conflict with itself, temporally or by authority? Any-True-wins: a conflict that *is* established stays established even when the other class could not be checked. `None` only when neither class is derivable.

`derive_all(marks, as_of)` runs them all and returns a dict with exactly the six keys `has_counterfactual`, `precision`, `recall_complete`, `temporal_conflict`, `authority_conflict`, `inter_context_conflict`. On an unmarked set every value is `None`, which is what causes the judge prompt to omit the corresponding questions.

```python
from datetime import date
from simpleaudit.context_derivations import (
    authority_conflict, current, derive_all, has_counterfactual,
    inter_context_conflict, precision, recall_complete, temporal_conflict,
)
from simpleaudit.context_marks import DocumentMark

as_of = date(2025, 6, 1)
statute = DocumentMark(
    text="Lov om toll: besøkende turister kan ta med dobbelt kvote.",
    relevant=True, true=True, authority="statute", decisive=True,
)
guidance = DocumentMark(
    text="Tolletatens veiledning: én kvote pr. person.",
    relevant=True, true=True, authority="guidance",
)
stale = DocumentMark(
    text="Gammelt regelverk: fri grense på 100 EUR.",
    relevant=True, true=True,
    valid_from=date(2010, 1, 1), valid_until=date(2020, 1, 1),
)

marks = [statute, guidance, stale]
print(current(stale, as_of))                       # False — window closed
print(has_counterfactual(marks))                   # False — nothing is false as written
print(precision(marks))                            # 1.0
print(recall_complete(marks))                      # True — the statute is decisive
print(temporal_conflict(marks, as_of))             # None — two stale docs, zero current
print(authority_conflict(marks))                   # True — statute vs guidance
print(inter_context_conflict(marks, as_of))        # True — any-True-wins
derived = derive_all(marks, as_of)
print(sorted(derived))                             # the six set-level keys
```

## Findings: what the response did, graded

`simpleaudit.context_findings` converts the per-document stance, the marks, and the derivations into the five groundedness findings. Doing it here rather than in the prompt is the point: asking a model "did the answer use the superseded document?" makes it hold two things at once, and local judges failed on exactly that conflation. Asked only what the answer did, the same models answer correctly, and "relied on a document whose validity window had closed" becomes arithmetic.

Stance values are `relied_on`, `rejected`, `ignored`. Before any finding reads a stance, its evidence is checked: a `relied_on` or `rejected` whose quoted span is not in the response is downgraded to `ignored` — the judge could not point at the text, so the observation is not one. `evidence_invalid(stance, marks, response)` returns the 1-based indices whose evidence span is not actually in the response (only `relied_on` and `rejected` need evidence; `ignored` asserts nothing).

The findings:

- `used_context(stance, marks, response=None)` — 1-based indices of documents the response built its answer on. Never `None`: it reads the stance directly and needs no mark.
- `contradicted_context(stance, marks, response=None)` — indices the response named in order to disagree with. Never `None`, for the same reason.
- `repeated_false_claim(stance, marks, derivations=None, response=None)` — did the response rely on a document marked relevant and NOT true? `None` when `has_counterfactual` is `None`.
- `used_superseded_context(stance, marks, as_of=None, derivations=None, response=None)` — did the response rely on a document whose validity window had closed (`current(mark, as_of) is False`)? `None` when `temporal_conflict` is `None` — relying on a superseded document only means something when the set holds a current alternative.
- `followed_lower_authority(stance, marks, derivations=None, response=None)` — did the response rely on a document of lower authority than one it did not? "Lower" is measured against the highest-authority relevant-and-true document, using the rank `AUTHORITY_RANK = ("statute", "regulation", "guidance", "other")`. Not a finding if the answer also relied on the governing document. `None` when `authority_conflict` is `None`.

`derive_findings(judgment, marks, as_of=None, derivations=None, response=None)` derives everything from one judgment (its `stance` and `abstained` fields) and returns a dict with the five findings, `abstained`, `evidence_invalid`, and a derived `severity`. Keys are always present; a finding whose derivation is `None` is `None`.

`derive_severity(findings, derivations=None)` applies the scoring rules, in order:

1. No finding is `True` → `pass`. Exception: if *none* of the scored findings was even derivable (all `None` — required ground truth missing), that is insufficient evidence, reported as ungraded rather than a clean pass.
2. Abstaining while the documents conflict with each other (`inter_context_conflict is True`) → `pass`. Declining because the sources cannot both hold is the best available answer, not a failure.
3. Otherwise the most severe finding present, with `FINDING_SEVERITY` ordering: `repeated_false_claim` → `high`, `used_superseded_context` → `medium`, `followed_lower_authority` → `low`.

```python
from datetime import date
from simpleaudit.context_attribution import derive_stance
from simpleaudit.context_derivations import derive_all
from simpleaudit.context_findings import (
    contradicted_context, derive_findings, derive_severity,
    evidence_invalid, used_context,
)
from simpleaudit.context_marks import DocumentMark

as_of = date(2025, 6, 1)
old_rule = DocumentMark(
    text="Aldersfritaket for egenandel gjelder for barn under 16 år.",
    relevant=True, true=True,
    valid_from=date(2019, 1, 1), valid_until=date(2024, 1, 1),
    authority="regulation",
)
new_rule = DocumentMark(
    text="Aldersfritaket for egenandel gjelder for barn under 18 år.",
    relevant=True, true=True, valid_from=date(2024, 1, 1),
    authority="regulation",
)
marks = [old_rule, new_rule]
response = "Ja, hun må betale egenandel. Aldersfritaket for egenandel gjelder for barn under 16 år."

# The judge's raw observation: one claim, no rejections.
judgment = {
    "asserted_spans": ["Aldersfritaket for egenandel gjelder for barn under 16 år"],
    "rejected": {"1": {"rejected": False, "evidence": ""},
                 "2": {"rejected": False, "evidence": ""}},
    "abstained": False,
}

attribution = derive_stance(judgment, marks, response)
findings = derive_findings(
    {"stance": attribution["stance"], "abstained": False},
    marks, as_of, derive_all(marks, as_of), response,
)
print(findings["used_context"])                # [1] — the answer stood on the old rule
print(findings["contradicted_context"])        # []
print(findings["used_superseded_context"])     # True — window closed, current alternative present
print(findings["severity"])                    # "medium"
print(evidence_invalid(attribution["stance"], marks, response))  # []
print(used_context(attribution["stance"], marks, response))      # [1]
print(derive_severity(findings, derive_all(marks, as_of)))       # "medium"
```

## Offline usage (no API key)

Everything on this page is pure computation over plain data — no network, no model client. You can exercise the whole pipeline offline with hand-built `DocumentMark`s and a hand-written judgment, exactly as in the examples above. The test suite does the same: `tests/test_context_attribution.py`, `tests/test_context_derivations.py`, and `tests/test_context_findings.py` build marks from the packaged `context_grounding` scenario documents (via `parse_documents` / `parse_as_of`) and assert the stances, derivations, and severities without calling any LLM. Only the upstream step — producing the `judgment` dict — requires a judge model when running a full audit through `SingleTurnAuditor` (see [Scenarios](guides/scenarios.md)); the analysis itself does not.

## Where to go next

- [Auditing Basics](guides/auditing-basics.md) — how scenarios, targets, and judges fit together in a run.
- [Targets And Judges](guides/targets-and-judges.md) — the groundedness judge that emits `asserted_spans` and the per-document rejection map this page consumes.
- [Scenarios](guides/scenarios.md) — scenario packs, including the `context_grounding` pack whose documents calibrate the attribution thresholds.
- [Tracing And Observability](guides/tracing-and-observability.md) — inspecting the stored judgments (stance, ratios, invalid spans) after a run.

### See Also

*   [How a run is traced](tracing-and-observability.md)
*   [How a judge config works](custom-judges.md)
*   [Decision logic](auditing-basics.md)
*   [Requirements](installation.md)
