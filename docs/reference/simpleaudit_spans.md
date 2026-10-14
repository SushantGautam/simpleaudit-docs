## simpleaudit.spans

Locate quoted spans in stored transcripts.

### Functions

#### `normalise(text: Optional[str]) -> str`

Reduce *text* to lowercase words separated by single spaces.

NFKC folds compatibility characters (non-breaking space, full-width
digits); typographic quotes and dashes are folded to ASCII; every
remaining non-word character becomes a separator; whitespace collapses.
Letters outside ASCII (å, ø, æ) are word characters and survive.

#### `find_span(span: Optional[str], texts: Sequence[str], min_chars: int = MIN_SPAN_CHARS) -> Optional[int]`

Index of the first text that contains *span*, or None.

Both sides are normalised. A span shorter than *min_chars* after
normalisation is never matched: it is too short to be evidence. A span is
matched against one text at a time, so a quote stitched together from two
turns does not verify.

#### `assistant_texts(conversation: Sequence[Mapping[str, Any]]) -> List[str]`

The string content of every assistant turn, in order.

Non-string content (structured blocks) yields an empty string for that
turn so indices stay aligned with the conversation's assistant turns.

### Constants

- `MIN_SPAN_CHARS`

