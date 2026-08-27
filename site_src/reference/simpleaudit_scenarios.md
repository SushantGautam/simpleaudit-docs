## simpleaudit.scenarios

Scenario packs: curated test suites (health, safety, government, benchmarks).

Built-in scenario packs for SimpleAudit.

Available packs:
- safety: General AI safety scenarios
- rag: RAG-specific scenarios
- health: Healthcare domain scenarios
- system_prompt: System prompt adherence/bypass testing
- helpmed: Help and medical scenarios
- ung: UNG scenarios
- bullshitbench_v1: BullshitBench v1 (55 scenarios, business/management)
- bullshitbench_v2: BullshitBench v2 (100 scenarios, software/finance/legal/medical/physics)
- bullshitbench: BullshitBench v1+v2 combined (155 scenarios)
- health_bullshit: Health-specific broken premise scenarios (15 scenarios)
- epistemic_safety: All bullshitbench + health_bullshit combined (170 scenarios)
- hei_refusal: Norwegian youth Q&A refusal/guidance edge cases (47 scenarios)
- nav_aap: NAV Arbeidsavklaringspenger / Norwegian welfare scenarios (15 scenarios)
- skatteetaten: Norwegian Tax Administration scenarios (in development)
- helfo: Helfo health-economics scenarios (8 scenarios)
- lanekassen: Lånekassen student-finance scenarios (8 scenarios)
- vision_integrity: Chart-reading integrity for vision models (8 scenarios,
  requires vision-capable target, judge and auditor; not part of 'all')
- judge_the_judge: Judge qualification scenarios (8 scenarios, requires
  WiggleRunner with three model roles; not part of 'all')
- all: All scenarios combined

### Functions

#### `get_scenarios(pack_name: str) -> List[Dict]`

Get scenarios from a built-in pack.

Args:
    pack_name: Name of the scenario pack

Returns:
    List of scenario dictionaries

Raises:
    ValueError: If pack name is not recognized

#### `list_scenario_packs() -> Dict[str, int]`

List available scenario packs and their sizes.

Returns:
    Dict mapping pack names to number of scenarios

#### `duplicate_scenario_names(scenarios: List[Dict]) -> Dict[str, int]`

Return scenario names that occur more than once, mapped to their count.

Per-scenario stability statistics are keyed by scenario name (see
``RepeatedExperimentResults.stability``), so duplicate names within a pack
silently collapse into a single entry and corrupt the aggregates. Use this
to validate a custom scenario list before auditing.

Args:
    scenarios: List of scenario dicts (each expected to have a ``name`` key)

Returns:
    Dict mapping each duplicated name to the number of times it appears
    (empty if all names are unique)

### Constants

#### `SCENARIO_PACKS`

_Dict with 19 keys._

### Scenario Packs

| Pack | Scenarios |
| --- | --- |
| `safety` | 8 |
| `rag` | 8 |
| `health` | 8 |
| `system_prompt` | 8 |
| `helpmed` | 10 |
| `ung` | 1000 |
| `bullshitbench_v1` | 55 |
| `bullshitbench_v2` | 100 |
| `bullshitbench` |  |
| `health_bullshit` | 15 |
| `epistemic_safety` |  |
| `hei_refusal` | 47 |
| `nav_aap` | 15 |
| `skatteetaten` | 8 |
| `helfo` | 8 |
| `lanekassen` | 8 |
| `vision_integrity` | 8 |
| `judge_the_judge` | 8 |
| `all` |  |
