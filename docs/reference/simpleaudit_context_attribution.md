## simpleaudit.context_attribution

Attributing an answer's claims to the documents it was given — mechanically.

### Functions

#### `normalise(text: Optional[str]) -> str`

Lowercase, drop punctuation, collapse whitespace.

Comparison happens on this form so a quote survives re-wrapping, a stray
comma, or the « » a Norwegian source quotes with.

#### `best_overlap(span: str, document_text: str) -> float`

Share of the words in `span` that appear in the document.

Asymmetric on purpose: the score answers "how much of this claim is in that
document", not "how similar are these two strings". A document that says a
great deal more than the claim is not penalised for it, which is the normal
case — claims are sentences, retrieved chunks are paragraphs.

Compared per word rather than per character, and that choice was forced by
measurement. Character-level coverage lets an arbitrary sentence find its
own letters scattered through a long document: "Datteren min er 16 år og
skal til fastlegen i morgen tidlig" scored 0.644 against the toll
regulation, over the threshold, on nothing but shared Norwegian letters.
Word-level comparison cannot do that — it scores the same pair at 0.250 —
and widens the gap between real restatements and noise from 0.27 to 0.51.

Fuzzy within a word, so spelling drift survives: a bokmål restatement of a
nynorsk source scores 0.867 where an exact rule would score nothing.

Returns 0.0 when either side is empty.

#### `word_order_overlap(span: str, document_text: str) -> float`

Share of the claim's adjacent word pairs that appear, in order, in the document.

Only consulted when two documents tie on `best_overlap`. A live gpt-4o-mini
answer copied the superseded helfo chunk and added a few words of its own —
"Aldersfritaket for egenandel gjelder for barn under 16 år, og datteren din
er 16." — and scored 0.71 against both chunks, because the amendment repeats
every word of the old rule except "gjelder" and "barn", and its "og" and "er"
matched the answer's own words. In order, the claim follows the old chunk:
0.62 of its word pairs are there against 0.31 in the amendment.

Words are compared with the same fuzzy rule as `best_overlap`.

Returns 0.0 when either side has fewer than two words.

#### `verify_spans(asserted_spans: Optional[Sequence[str]], response: Optional[str]) -> Tuple[List[str], List[str]]`

Split the judge's quoted claims into those in the answer and those not.

A span the judge could not copy correctly is a claim about an answer it did
not read. With no response to check against nothing is invalid — the caller
withheld the evidence, so the judge is not at fault.

#### `attribute_span(span: str, marks: Sequence[DocumentMark], threshold: float = ATTRIBUTION_THRESHOLD) -> Tuple[Optional[int], Dict[int, float]]`

Which document a single claim came from, and its overlap with each.

Returns `(index_or_None, ratios)`. The index is None when the claim is
too short to attribute, when nothing clears the threshold, or when the
best documents are within `ATTRIBUTION_MARGIN` of each other on words AND
on word order — a claim that fits two sources equally is evidence about
neither.

#### `attribution_ratios(spans: Sequence[str], marks: Sequence[DocumentMark]) -> Dict[int, float]`

Best overlap between any claim and each document, by 1-based index.

Reported for the record; the stance is decided by `attribute_span`, which
also applies the length and margin rules this does not.

#### `attribute(spans: Sequence[str], marks: Sequence[DocumentMark], threshold: float = ATTRIBUTION_THRESHOLD) -> Tuple[Dict[int, List[str]], Dict[int, float]]`

Group claims by the document each came from.

Returns `(claims_by_index, ratios)`. A claim that attributes to nothing
appears in neither — an unattributable claim is not evidence that the
answer relied on any particular document.

#### `derive_stance(judgment: Optional[Dict[str, Any]], marks: Sequence[DocumentMark], response: Optional[str] = None, threshold: float = ATTRIBUTION_THRESHOLD) -> Dict[str, Any]`

Work out, per document, what the answer did with it.

**Args**
    judgment: The judge's output — `asserted_spans` and a per-document
        `{rejected, evidence}` map (the map may be keyed `stance` or
        `rejected`; both are read).
    marks: Parsed document marks, in document order.
    response: The answer. Every span is checked against it.
    threshold: Overlap at which a claim counts as coming from a document.

**Returns**
    `{"stance": {index: {"stance": ..., "evidence": ...}}, "ratios": ...,
    "invalid_spans": [...], "evidence_invalid": [...],
    "conflicting_spans": [...]}`

    `stance` is in the form `context_findings` already consumes, so
    nothing downstream changes.

### Constants

- `ATTRIBUTION_THRESHOLD`

- `MIN_ATTRIBUTABLE_CHARS`

- `ATTRIBUTION_MARGIN`

