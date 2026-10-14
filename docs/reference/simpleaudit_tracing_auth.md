## simpleaudit.tracing.auth

OTLP receiver authentication — the "WHO is allowed to submit" layer.

### Classes

#### `Authenticator`

_No docstring._

#### `AuthResult`

Outcome of an `OTLPAuth.verify` call.

`ok` is True when the request may proceed. `target_id` is set (and
non-empty) when a credential was matched; it is `None` in `none` mode
and on failure. `reason` is a short, log-safe explanation on failure.

**Methods:**

- `authenticated() -> bool`
- `identity() -> Optional[str]`

**Attributes:**

- `ok`
- `target_id`
- `reason`

#### `OTLPAuth`

Credential store + verifier for an OTLP receiver.

Construct with a mode, add credentials, then hand the instance to an
`OTLPTraceReceiver` or
`EphemeralOTLPReceiver`. The receiver
calls `verify` on each request's `Authorization` header.

**Example**

    auth = OTLPAuth(mode="basic")
    auth.add_basic("sa_owui_17", "p4ss", target_id="owui_17")
    result = auth.verify("Basic " + b64("sa_owui_17:p4ss"))
    assert result.ok and result.target_id == "owui_17"

**Methods:**

- `add_basic(username: str, password: str, target_id: str) -> None` — Register (or replace) a Basic-Auth credential for `target_id`.
- `add_bearer(token: str, target_id: str) -> None` — Register (or replace) a Bearer credential for `target_id`.
- `revoke(username_or_token: str) -> bool` — Revoke a credential by username (basic) or token (bearer).
- `rotate_basic(username: str, new_password: str) -> None` — Re-issue the password for an existing Basic credential (same target).
- `rotate_bearer(old_token: str, new_token: str) -> None` — Replace a Bearer token, keeping the same target_id.
- `verify(authorization_header: Optional[str]) -> AuthResult` — Verify an `Authorization` header and return the target id.

### Functions

#### `new_salt() -> bytes`

_No docstring._

#### `hash_secret(secret: str, salt: bytes) -> bytes`

_No docstring._

#### `verify_secret(secret: str, salt: bytes, expected_hash: bytes) -> bool`

_No docstring._

#### `generate_password() -> str`

_No docstring._

#### `generate_token() -> str`

_No docstring._

#### `token_lookup_prefix(token: str) -> str`

_No docstring._

#### `parse_basic_header(authorization: Optional[str]) -> Optional[Tuple[str, str]]`

_No docstring._

#### `parse_bearer_header(authorization: Optional[str]) -> Optional[str]`

_No docstring._

#### `make_basic_bearer_authenticator(verify: Callable[[Optional[str]], Optional[str]]) -> Authenticator`

_No docstring._

#### `tag_spans(spans, target_id: Optional[str]) -> None`

Stamp `simpleaudit.target_id` onto each span's attributes in place.

No-op when `target_id` is falsy (e.g. `none` mode). Mutates the raw
span dicts before they are normalized/stored, so the tag flows through
`simpleaudit.tracing.otlp.parse_otlp_json` and is queryable via
`SpanStore.by_attribute`.

### Constants

- `TARGET_ID_ATTR`

- `SALT_LEN`

- `BEARER_TOKEN_PREFIX`

- `TOKEN_LOOKUP_PREFIX_LEN`

