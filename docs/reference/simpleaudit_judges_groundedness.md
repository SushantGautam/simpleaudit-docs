## simpleaudit.judges.groundedness

Groundedness judge for marked context.

### Functions

#### `document_indices(context: Optional[Dict[str, Any]] = None) -> List[str]`

1-based document indices, as the string keys the stance object uses.

Strings rather than ints because JSON object keys are strings, and a
schema whose `properties` are typed ints would not match what any provider
returns.

#### `build_groundedness_prompt(context: Optional[Dict[str, Any]] = None) -> Tuple[str, List[str]]`

Build the judge prompt for a document set.

The prompt does not vary with the derivations — every document gets the
same three-way question regardless of what the author marked. That is the
change from the first version: an unmarked property can no longer produce
a bad finding, because the judge is not asked about properties at all.

**Args**
    context: Builder context carrying `marks` (parsed DocumentMarks) and,
        optionally, `criteria` — a customized judge's own criteria, which
        replace GROUNDEDNESS_CRITERIA ahead of the per-document format.
        None or empty yields the general form, with the output example
        written for two documents.

**Returns**
    `(prompt_text, document_index_keys)`.

#### `build_groundedness_schema(context: Optional[Dict[str, Any]] = None) -> Dict[str, Any]`

Build the `response_schema` matching `build_groundedness_prompt`.

Every document index is a required property, so a judge that skips one
fails validation instead of leaving the derivation to guess. The entry is
typed rather than enumerated — `rejected` is a boolean now, and which
document a claim came from is not the judge's to say.

**Args**
    context: Builder context carrying `marks`. None or empty yields a
        permissive stance object, since the document count is unknown.

**Returns**
    A JSON Schema object in the shape the framework threads into
    `response_format`.

#### `postprocess_groundedness(judgment: Dict[str, Any], conversation: Optional[List[Dict[str, Any]]] = None, expected_behavior: Optional[List[str]] = None, scenario_meta: Optional[Mapping[str, Any]] = None) -> Dict[str, Any]`

Report the judgment as ungraded where no verdict can be derived.

`derive_stance` and `derive_findings` both take the parsed marks; a
postprocess hook is given the conversation, the expectations and the
scenario metadata, never the marks. `SingleTurnAuditor` therefore derives
the findings itself after the judging call and passes no postprocess for
this judge — so this hook running means the derivation did not happen and
will not.

Left alone the judgment carries neither `severity` nor `score`, and
`ModelAuditor._severity_from_judgment` falls through to `"medium"`: the
same number for a grounded answer and an ungrounded one. `"ERROR"` is off
the severity ladder, so the run drops out of the severity statistics
rather than counting as a middling pass.

**Args**
    judgment: The parsed judge output. An existing `ERROR` judgment (a
        parse failure) is returned unchanged, as the other hooks do.
    conversation, expected_behavior, scenario_meta: Part of the hook
        contract, unused — none of them carries the marks.

**Returns**
    The judgment with `severity` set to `"ERROR"` and the reason named.

### Constants

- `STANCE_VALUES`

- `FIELD_ORDER`

- `GROUNDEDNESS_CRITERIA`

- `UNGRADED_REASON`

- `GROUNDEDNESS_JUDGE`

