## simpleaudit.context_derivations

Set-level properties derived from document marks.

### Functions

#### `current(mark: DocumentMark, as_of: Optional[date]) -> Optional[bool]`

Is *mark* in force on *as_of*?

The window is half-open: `valid_from <= as_of < valid_until`. An open
bound passes — a document with no `valid_from` was not born on a date
that disqualifies it, and one with no `valid_until` has not been
superseded. But a document with *neither* bound has no window at all, and
an unmarked document is never assumed current, so that returns None.

**Returns**
bool or None
    None when *as_of* is None or the document carries no window at all.

#### `has_counterfactual(marks: Sequence[DocumentMark]) -> Optional[bool]`

Does the set contain a relevant document that is false as written?

That construction — relevant, on topic, and wrong — is the counterfactual
context of Longpre et al. (2021). Repeating it is the failure
`repeated_false_claim` grades.

**Returns**
bool or None
    None if any document's `relevant` or `true` is unmarked.

#### `precision(marks: Sequence[DocumentMark]) -> Optional[float]`

Share of the retrieved documents that are relevant.

**Returns**
float or None
    None if any `relevant` is unmarked, or if the set is empty — there is
    no share of nothing, and 0.0 would read as "retrieval returned only
    junk" rather than "retrieval returned nothing".

#### `recall_complete(marks: Sequence[DocumentMark]) -> Optional[bool]`

Is a load-bearing document present in the set?

`decisive` is single-hop by design: it says this one document is required
for a correct answer, not that the set jointly suffices. Sufficiency across
documents is out of scope.

**Returns**
bool or None
    None when no document carries `decisive` at all — nobody said which
    document the answer turns on, so nothing can be concluded about whether
    it made it into the set.

#### `temporal_conflict(marks: Sequence[DocumentMark], as_of: Optional[date]) -> Optional[bool]`

Are a superseded and a current document both in the set?

The shape is two or more relevant-and-true documents of which *exactly one*
is current on *as_of*. Both are true as written on their own date; only one
is the right thing to answer from today. Two current documents are not a
temporal conflict, and neither are two stale ones.

**Returns**
bool or None
    None if any `relevant` or `true` is unmarked, if *as_of* is None,
    or if any of the relevant-true documents has no derivable currency.

#### `authority_conflict(marks: Sequence[DocumentMark]) -> Optional[bool]`

Do two documents of different authority level both bear on the question?

Statute against agency guidance, both true on their own surface, is the
second conflict class the Norwegian register packs already contain. Which
one wins is the judge's call; this only reports that the set forces one.

**Returns**
bool or None
    None if any `relevant` or `true` is unmarked, or if any
    relevant-true document has no `authority`.

#### `inter_context_conflict(marks: Sequence[DocumentMark], as_of: Optional[date]) -> Optional[bool]`

Does the set conflict with itself, temporally or by authority?

Any-True-wins: a conflict that *is* established stays established even when
the other class could not be checked. None only when neither class is
derivable, which is the one case where the set says nothing at all.

#### `derive_all(marks: Sequence[DocumentMark], as_of: Optional[date]) -> Dict[str, Any]`

Compute every set-level derivation at once.

The keys match the judge fields that depend on them, so the prompt builder
can walk this dict and drop the question for any derivation that is None.

**Returns**
dict
    Keys `has_counterfactual`, `precision`, `recall_complete`,
    `temporal_conflict`, `authority_conflict`,
    `inter_context_conflict`.

