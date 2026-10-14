## simpleaudit.targets.callable

CallableTarget — an in-process Python callable as an audit target.

### Classes

#### `CallableTarget`

Wrap an in-process callable as a `Target`.

**Methods:**

- `async send(system: Optional[str] = None, user: str, history: Optional[List[Dict[str, Any]]] = None, file_uri: Optional[Union[str, List[str]]] = None, documents: Optional[List[Union[str, Dict[str, Any]]]] = None, response_format: Optional[Dict[str, Any]] = None, params: Optional[Dict[str, Any]] = None, context: Optional[TargetContext] = None) -> TargetResponse`

### Constants

- `TargetCallable`

