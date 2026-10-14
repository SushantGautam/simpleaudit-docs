## simpleaudit.context_marks

Document marks for context-grounding scenarios.

### Classes

#### `DocumentMark`

One retrieved document plus the author's ground truth about it.

Frozen because a mark is scenario ground truth: nothing downstream — the
auditor, the judge, the derivations — has any business rewriting it.

**Attributes**
text : str
    The document body. The only field that ever reaches the target.
relevant : bool or None
    Does this document bear on the question asked?
true : bool or None
    Is it true *as written*, on its own date? A superseded document can be
    true and still be the wrong thing to answer from.
valid_from, valid_until : date or None
    Half-open validity window. None on either end means open-ended, not
    unknown-and-therefore-invalid.
authority : str or None
    One of `AUTHORITY_LEVELS`.
source : str or None
    Free-form provenance — a register row id or a statute section.
decisive : bool or None
    Load-bearing: relevant, true, and required for a correct answer.

**Attributes:**

- `text`
- `relevant`
- `true`
- `valid_from`
- `valid_until`
- `authority`
- `source`
- `decisive`

### Functions

#### `parse_document(doc: Union[str, Dict[str, Any]]) -> DocumentMark`

Build a `DocumentMark` from a scenario's document entry.

A bare string is a document with every mark None — unknown, not assumed.
A dict must carry `text` and may carry any of `MARK_KEYS`.

**Parameters**
doc : str or dict
    The scenario entry.

**Returns**
DocumentMark

**Raises**
ValueError
    On an unknown key, a missing or non-string `text`, a date that is not
    ISO, a non-boolean boolean mark, or an `authority` outside
    `AUTHORITY_LEVELS`.

#### `parse_documents(docs: Optional[Sequence[Union[str, Dict[str, Any], DocumentMark]]]) -> List[DocumentMark]`

Parse a scenario's whole `documents` list.

A missing or empty list yields `[]` — a scenario without documents is a
plain scenario, not an error. Entries that are already
`DocumentMark` pass through, so callers can parse once and hand the
result on without a second round trip through the validator.

#### `render_documents(marks: Sequence[Union[DocumentMark, str, Dict[str, Any]]]) -> str`

Render documents for the target as numbered text blocks.

Only `text` is rendered. No mark key and no mark value ever appears here
— this is the render that reaches the model under audit, and a target that
could read the marks would be answering an easier question than the one the
scenario poses.

Blocks are 1-indexed so the numbering matches the judge's mark table and
the `used_context` / `contradicted_context` indices it reports.

**Returns**
str
    `"\n--- DOCUMENT 1 ---\n<text>\n--- DOCUMENT 2 ---\n<text>"`,
    or `""` for no documents.

#### `parse_as_of(scenario: Dict[str, Any]) -> Optional[date]`

Read the scenario-level `as_of` date — the date the question is asked.

Returns None when the scenario does not set one. That is not a fallback to
today: without `as_of` there is no date to test a validity window
against, and every temporal derivation returns None instead of guessing.

#### `mark_table(marks: Sequence[Union[DocumentMark, str, Dict[str, Any]]], as_of: Optional[date]) -> str`

Render the per-document mark table that goes to the judge.

Judge-only. The columns are index, relevant, true, current, authority and
source: what the judge needs to tell "the model used a superseded document"
apart from "the model used a false one". `current` is derived from the
document's validity window against *as_of*, and is `unknown` whenever the
window or *as_of* leaves it underivable.

**Parameters**
marks : sequence of DocumentMark
    Documents in the same order `render_documents` numbered them.
as_of : date or None
    The date the question is asked.

**Returns**
str
    A pipe-delimited table, or a one-line note when there are no documents.

### Constants

- `AUTHORITY_LEVELS`

- `MARK_KEYS`

