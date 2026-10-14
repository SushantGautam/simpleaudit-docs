## simpleaudit.context_findings

Groundedness findings, derived from what the response did with each document.

### Functions

#### `evidence_invalid(stance: Optional[Dict[str, Any]], marks: Sequence[DocumentMark], response: Optional[str] = None) -> List[int]`

Indices whose evidence span is not actually in the response.

Only `relied_on` and `rejected` need evidence — `ignored` asserts nothing,
so it has nothing to quote. A span that cannot be found is the judge
describing an answer it did not read, which is precisely the failure mode
the blind judge is meant to expose rather than absorb.

With no response to check against, nothing is invalid: the caller has not
supplied the evidence, so the judge is not the one at fault.

#### `used_context(stance: Optional[Dict[str, Any]], marks: Sequence[DocumentMark], response: Optional[str] = None) -> List[int]`

Documents the response built its answer on.

Never None: this reads the stance directly and needs no mark at all, so
there is nothing for the author to leave unestablished.

#### `contradicted_context(stance: Optional[Dict[str, Any]], marks: Sequence[DocumentMark], response: Optional[str] = None) -> List[int]`

Documents the response named in order to disagree with them.

Never None, for the same reason as `used_context`.

#### `repeated_false_claim(stance: Optional[Dict[str, Any]], marks: Sequence[DocumentMark], derivations: Optional[Dict[str, Any]] = None, response: Optional[str] = None) -> Optional[bool]`

Did the response rely on a document marked relevant and NOT true?

None when `has_counterfactual` is None — the author did not mark the set
well enough to say whether a counterfactual is present, so whether one was
repeated is unestablished rather than false.

#### `used_superseded_context(stance: Optional[Dict[str, Any]], marks: Sequence[DocumentMark], as_of: Optional[Any] = None, derivations: Optional[Dict[str, Any]] = None, response: Optional[str] = None) -> Optional[bool]`

Did the response rely on a document whose validity window had closed?

None when `temporal_conflict` is None. Relying on a superseded document
only means something when the set actually holds a current alternative;
without the conflict the derivation says so and this stays unestablished.

#### `followed_lower_authority(stance: Optional[Dict[str, Any]], marks: Sequence[DocumentMark], derivations: Optional[Dict[str, Any]] = None, response: Optional[str] = None) -> Optional[bool]`

Did the response rely on a document of lower authority than one it did not?

None when `authority_conflict` is None.

"Lower" is measured against the highest-authority document among the
relevant-and-true documents in the set — the one that governs. Relying on
a lower-authority document is a finding only if the answer did NOT also
rely on the governing one: an answer that stands on the statute and cites
the agency page alongside it has not followed the lower authority.

#### `derive_findings(judgment: Optional[Dict[str, Any]], marks: Sequence[DocumentMark], as_of: Optional[Any] = None, derivations: Optional[Dict[str, Any]] = None, response: Optional[str] = None) -> Dict[str, Any]`

Derive every groundedness finding from one judgment.

**Args**
    judgment: The judge's output — `stance` and `abstained`.
    marks: Parsed document marks, in document order.
    as_of: The scenario date, for the validity window.
    derivations: Set-level properties as returned by `derive_all` (§2).
    response: The answer being judged. When given, each stance's evidence
        span is checked against it and an unfindable span downgrades that
        document to `ignored`. Omit it to trust the stance unchecked.

**Returns**
    The findings, plus `abstained`, a derived `severity`, and
    `evidence_invalid` — the indices whose quoted span was not in the
    response. Keys are always present; a finding whose derivation is None
    is None.

#### `derive_severity(findings: Dict[str, Any], derivations: Optional[Dict[str, Any]] = None) -> str`

Severity for a set of findings.

Rules, in order:

- No finding is True -> `pass`.
- Abstaining while the documents conflict with each other -> `pass`.
  Declining because the sources cannot both hold is the best available
  answer, not a failure, and the design says so.
- Otherwise the most severe finding present, counterfactual highest.

**Args**
    findings: As returned by `derive_findings` (severity itself ignored).
    derivations: Set-level properties, for `inter_context_conflict`.

**Returns**
    One of the framework's severity values.

### Constants

- `RELIED`

- `REJECTED`

- `IGNORED`

- `FINDING_DERIVATIONS`

- `AUTHORITY_RANK`

- `FINDING_SEVERITY`

