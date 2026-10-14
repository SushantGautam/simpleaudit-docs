## simpleaudit.targets.model

ModelTarget — an LLM endpoint target backed by AnyLLM.

### Classes

#### `ModelTarget`

A target that talks to an LLM endpoint via an AnyLLM client.

**Methods:**

- `client() -> Any`
- `model() -> Optional[str]`
- `async send(system: Optional[str] = None, user: str, history: Optional[List[Dict[str, Any]]] = None, file_uri: Optional[Union[str, List[str]]] = None, documents: Optional[List[Union[str, Dict[str, Any]]]] = None, response_format: Optional[Dict[str, Any]] = None, params: Optional[Dict[str, Any]] = None, context: Optional[TargetContext] = None) -> TargetResponse`

### Constants

- `Transport`

