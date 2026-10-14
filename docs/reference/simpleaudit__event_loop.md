## simpleaudit._event_loop

Running SimpleAudit's coroutines from synchronous code.

### Functions

#### `run_sync(coro: Coroutine[Any, Any, Any]) -> Any`

`asyncio.run(coro)`, minus the stale-client message described above.

