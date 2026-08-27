## simpleaudit.utils

Shared helpers: LLM client wrappers, JSON parsing, and misc utilities.

Utility functions for SimpleAudit.

This module contains shared utilities used across the framework.

### Functions

#### `normalize_severity(severity: Any) -> str`

Map a judge-emitted severity onto the framework's canonical ladder.

Case and surrounding whitespace are normalized, and known aliases are
folded in ("none" -> "pass", "error" -> "ERROR"). Values outside the
ladder are returned stripped-but-unchanged so custom judge vocabularies
survive; non-string values are coerced to str so downstream .upper()
calls cannot crash. A literal None means the judge omitted severity
entirely — that maps to "medium", NOT to the "none" alias, which is
reserved for judges that answer the string "none" meaning "no harm".

#### `severity_direction(severity_a: Any, severity_b: Any) -> Any`

Position of *severity_b* relative to *severity_a* on ``SEVERITY_ORDER``.

Returns ``idx_b - idx_a``; positive means *severity_b* is the stricter
verdict. Returns None when either value is off the canonical ladder
(e.g. "ERROR"), so callers can tell "no direction" apart from "no shift".

#### `severity_from_score(score: Any, max_score: float = 10.0) -> Any`

Map a numeric judge score onto the canonical severity ladder.

Score-based judges (helpfulness, factuality, abstention) emit a 1-10
score and no severity; without a mapping every such scenario shows up
as the "medium" default in summaries and plots. Bands (for a 1-10
scale):  9-10 -> pass, 7-8 -> low, 5-6 -> medium, 3-4 -> high,
1-2 -> critical. Returns None when score is not numeric so the caller
keeps its default behaviour.

#### `parse_json_response(response: str, default_severity: str = 'ERROR') -> Dict[str, Any]`

Parse JSON from LLM response with robust fallback handling.

This function handles common issues with LLM-generated JSON:
- Markdown code blocks (```json ... ```)
- Leading/trailing text around JSON
- Malformed JSON with best-effort extraction
- Complete parsing failures with sensible defaults

Args:
    response: Raw LLM response text that should contain JSON
    default_severity: Fallback severity level if parsing fails (default: "ERROR")

Returns:
    Parsed dictionary with guaranteed keys: severity, issues_found,
    positive_behaviors, summary, recommendations

#### `image_media_type(file_uri: str) -> str`

Resolve a URI to an image media type, raising if it is not an image.

Anything mimetypes cannot place, or that resolves to a non-image, is
rejected rather than being sent to a provider as a mislabelled image.

#### `image_data_uri(file_uri: str) -> str`

Read an image and inline it as a base64 data URI.

fsspec resolves local paths, http(s), s3, gs and friends through one call,
so scenario authors can point at wherever the image actually lives.

Cached because the first user message is re-sent on every turn of a
multi-turn audit — without this the same file is re-read and re-encoded
once per turn per scenario. Entries are keyed on the URI alone, so
run_async clears the cache to pick up files changed between runs.

#### `image_content_block(file_uri: str) -> Dict[str, Any]`

Build an OpenAI-style image content block from a URI.

### Constants

#### `VALID_SEVERITIES`
#### `SEVERITY_ORDER`
