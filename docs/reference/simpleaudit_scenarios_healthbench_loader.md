## simpleaudit.scenarios.healthbench_loader

Load OpenAI's HealthBench as SimpleAudit scenarios at run time.

### Functions

#### `default_cache_dir() -> Path`

_No docstring._

#### `load_healthbench_scenarios(subset: str = 'main', themes: Optional[Iterable[str]] = None, min_criteria: Optional[int] = None, max_criteria: Optional[int] = None, limit: Optional[int] = None, seed: int = 0, path: Optional[str] = None, cache_dir: Optional[str] = None) -> List[Dict]`

Build v2 scenarios from a HealthBench subset.

**Args**
    subset: "main", "hard", "consensus" or "professional".
    themes: keep only these themes, e.g. ["emergency_referrals", "hedging"]. For
        "professional" the themes are the use cases "consult", "writing", "research".
    min_criteria, max_criteria: keep only examples whose rubric has this many
        criteria. The scenario guideline asks for 3-7 expected behaviours; HealthBench
        rubrics have 2-48 (Professional 1-5), so the default keeps them all.
    limit: return a random sample of at most this many scenarios (after filtering).
    seed: seed for that sample, so the same call returns the same scenarios.
    path: read a local copy (JSONL, or the Professional zip) instead of downloading.
    cache_dir: where downloads are kept (default ~/.cache/simpleaudit/healthbench).

**Returns**
    Scenario dicts in file order. Multi-turn examples are always skipped.

### Constants

- `CANARY`

- `SUBSETS`

- `THEMES`

- `PHYSICIAN_CATEGORIES`

