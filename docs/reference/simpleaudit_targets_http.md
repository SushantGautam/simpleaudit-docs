## simpleaudit.targets.http

HTTPAppTarget — audit an external application over HTTP (black-box).

### Classes

#### `HTTPAppTarget`

Send audit probes to an external HTTP application and parse the answer.

**Methods:**

- `async send(system: Optional[str] = None, user: str, history: Optional[List[Dict[str, Any]]] = None, file_uri: Optional[Union[str, List[str]]] = None, documents: Optional[List[Union[str, Dict[str, Any]]]] = None, response_format: Optional[Dict[str, Any]] = None, params: Optional[Dict[str, Any]] = None, context: Optional[TargetContext] = None) -> TargetResponse`

