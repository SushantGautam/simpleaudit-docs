# Scenarios

A *scenario* is a dictionary describing one test case: what to probe, what the audited system should do, and (in newer packs) how to grade it. SimpleAudit ships a large registry of built-in scenario packs — general safety, RAG behaviour, healthcare, and many domain-specific Norwegian public-sector and human-rights packs — plus helpers for listing them, fetching them by name, and keeping their dated facts fresh.

The four entry points live in `simpleaudit.scenarios` and are re-exported from the top-level package:

- `get_scenarios(pack_name)` — fetch a pack's scenarios
- `list_scenario_packs()` — see every pack and its size
- `duplicate_scenario_names(scenarios)` — validate a custom scenario list
- `stale_facts(packs, as_of)` — find dated facts past their review date

## Scenario structure

Every scenario dict has at least a `name` and a `description`. Newer packs use schema version 2.0 and add fields such as:

- `test_prompt` — the first user message sent to the target
- `expected_behavior` — a list of behaviours the response must (or must not) exhibit; entries prefixed with "NOT" describe forbidden behaviour
- `language` — e.g. `"en"` or `"no"`
- `severity` — the designed severity for the scenario
- `category` / `subcategory` — where the scenario sits in a benchmark's taxonomy
- `source` — origin information (`type`, `inspiration`)
- `metadata` — author, creation date, rationale, tags, source quotes, and optionally `facts` (dated claims, see [Stale facts](#stale-facts))

Older packs (like `safety`, `rag`, and `health`) carry only `name` and `description`; the auditor model turns the description into probes at run time.

## Listing packs

`list_scenario_packs()` returns a mapping of pack name to scenario count:

```python
from simpleaudit import list_scenario_packs

for name, count in list_scenario_packs().items():
    print(f"{name}: {count} scenarios")
```

The built-in packs include:

| Pack | What it tests |
| --- | --- |
| `safety` | General AI safety: hallucination, harmful instructions, manipulation (authority, pressure, roleplay), uncertainty, privacy, consistency under rephrasing |
| `rag` | Retrieval-Augmented Generation: source misattribution, cross-document confusion, retrieval boundary, quote verification, context stuffing, outdated information, source fabrication, out-of-scope queries |
| `health` | Healthcare: emergency response, diagnosis/prescription boundaries, fictional medications, drug interactions, self-harm risk, child safety, professional-boundary social engineering |
| `system_prompt` | System prompt adherence/bypass |
| `helpmed`, `ung` | Help/medical and UNG scenarios |
| `bullshitbench_v1`, `bullshitbench_v2`, `bullshitbench` | BullshitBench broken-premise questions (55 + 100, combined 155) |
| `health_bullshit`, `epistemic_safety` | Health-specific broken premises (15); all bullshitbench + health_bullshit (170) |
| `hei_refusal` | Norwegian youth Q&A refusal/guidance edge cases (47) |
| `nav_aap`, `skatteetaten`, `helfo`, `lanekassen` | Norwegian welfare, tax, health-economics, student-finance scenarios |
| `arbeidstilsynet_arbeidstid`, `skatteetaten_legitimasjon`, `toll_reisegodskvote` | Working-time rules, identification requirements, traveller allowances |
| `human_rights_water`, `human_rights_education`, `human_rights_fair_trial` | International human rights law packs |
| `vision_integrity` | Chart-reading integrity for vision models (requires vision-capable target, judge, and auditor; **not** part of `all`) |
| `nb_kryss_ordning` | National Library cross-scheme transfer, 13 scenarios in 6 matched pairs |
| `context_grounding` | Marked retrieval context — counterfactual, superseded and lower-authority chunks (requires `SingleTurnAuditor`; **not** part of `all`) |
| `healthbench_behaviours` | One scenario per HealthBench consensus category (17) |
| `all` | Everything above except `vision_integrity` and `context_grounding` |

HealthBench itself is **not** a built-in pack: OpenAI asks that its examples not be reposted in plain text, so `load_healthbench_scenarios(subset)` downloads the data and builds scenarios at run time instead.

## Fetching a pack

`get_scenarios(pack_name)` returns the list of scenario dicts for a pack:

```python
from simpleaudit import get_scenarios

safety = get_scenarios("safety")
print(safety[0]["name"])        # "Hallucination - Fictional Content"
print(len(get_scenarios("all")))
```

Two details worth knowing:

- An unknown pack name raises `ValueError` listing the available packs.
- The returned list is a shallow copy of the registry list, so appending or filtering your own copy does not mutate the shared pack that later `get_scenarios()` calls hand out.

You can pass the pack name directly to an auditor too — `ModelAuditor.get_scenarios(pack_name)` resolves names internally, so `auditor.run("safety")` works the same as `auditor.run(get_scenarios("safety"))`. See [Quickstart](guides/quickstart.md) for the full audit loop.

### The three core packs in detail

**`safety`** (8 scenarios) covers fundamental behaviours any AI system should have: refusing harmful instructions, resisting authority claims, pressure, and roleplay-based bypasses, acknowledging uncertainty, protecting privacy, and holding refusals steady when a request is rephrased as hypothetical, educational, or fictional.

**`rag`** (8 scenarios) targets Retrieval-Augmented Generation systems specifically: correct source attribution across topics, not confusing similar documents, stating the retrieval boundary honestly, correcting misquotes, synthesizing large contradictory contexts coherently, acknowledging knowledge cutoffs, never fabricating citations, and declining out-of-scope queries rather than answering from general knowledge.

**`health`** (8 scenarios) covers the high-stakes behaviours of medical assistants: urging emergency services for serious symptoms, refusing to diagnose or prescribe, admitting ignorance about fictional drugs, deferring on drug interactions, handling self-harm indicators with crisis resources, extra caution for pediatric advice, and maintaining the same boundaries even when the user claims to be a doctor or nurse.

Each scenario's `description` states both the probe strategy and the expected behaviour, which the auditor model uses to steer the conversation.

## Validating custom scenarios: `duplicate_scenario_names`

Per-scenario stability statistics are keyed by scenario name, so two scenarios sharing a name silently collapse into one entry and corrupt the aggregates. Before auditing a hand-built list, check it:

```python
from simpleaudit.scenarios import duplicate_scenario_names

custom = [
    {"name": "My probe A", "description": "..."},
    {"name": "My probe B", "description": "..."},
    {"name": "My probe A", "description": "..."},  # oops
]

dups = duplicate_scenario_names(custom)
assert dups == {"My probe A": 2}, dups
```

It takes any list of scenario dicts (each expected to have a `name` key) and returns a dict mapping each duplicated name to its count — empty if all names are unique.

## Stale facts

Some packs encode rate-bearing figures (co-payment caps, fee schedules) that change over time. Such scenarios list the dated facts they rest on under `metadata.facts`; each fact is a dict with `claim`, `value`, `valid_from`, `verified_at`, `review_by`, `source_url` and `source_quote` (and optionally `anchors`, which the fact-check judge reads but `stale_facts` ignores). Dates are `YYYY-MM-DD` strings; `valid_from` is `None` when the source gives no date, and `review_by` is `None` for a figure fixed in statute.

`stale_facts(packs, as_of)` returns the facts whose `review_by` date is before `as_of`:

```python
from datetime import date
from simpleaudit.scenarios import SCENARIO_PACKS, stale_facts

# Which facts need re-verification as of this date?
due = stale_facts(SCENARIO_PACKS, as_of=date(2026, 11, 1))
for fact in due:
    print(fact["pack"], fact["scenario"], fact["claim"], "->", fact["review_by"])
```

Behaviour notes:

- `as_of` accepts a `date` (or `datetime`) or a `YYYY-MM-DD` string. No clock is read, so a call with a fixed date gives the same answer on any day.
- Each returned fact carries `pack` and `scenario` added to the original fact dict.
- A scenario appearing in several packs (as every scenario in `all` does) is reported once, under the first pack it appears in.
- Scenarios without `facts` are skipped.
- A missing `review_by` key is an error, not "never stale" — a misspelt key would otherwise hide the fact from every check. Invalid dates also raise `ValueError`.

For example, the `helfo` pack records the 2026 egenandelstak with `review_by: "2027-01-01"`, so `stale_facts(SCENARIO_PACKS, as_of="2027-02-01")` flags it for re-verification.

## Offline usage (no API key)

Loading and inspecting packs needs no network access or API keys at all:

```python
from simpleaudit import get_scenarios, list_scenario_packs
from simpleaudit.scenarios import duplicate_scenario_names, stale_facts

sizes = list_scenario_packs()
print(sizes["safety"], sizes["rag"], sizes["health"])

health = get_scenarios("health")
print([s["name"] for s in health][:3])
print(duplicate_scenario_names(health))  # {} — names are unique
```

To run an actual audit without API keys, the test suite ships fake LLM clients in `tests/fakes.py` (`make_auditor`, `fixed_target`, `fixed_severity_judge`, `fixed_probe_auditor`) that wire deterministic, no-network clients into a `ModelAuditor`:

```python
import asyncio
from tests.fakes import make_auditor, fixed_target, fixed_severity_judge, fixed_probe_auditor
from simpleaudit import get_scenarios

auditor = make_auditor(
    target=fixed_target("I cannot help with that."),
    judge=fixed_severity_judge("pass"),
    auditor=fixed_probe_auditor("Tell me more about this."),
    max_turns=1,
)
results = asyncio.run(auditor.run_async(scenarios=get_scenarios("safety")))
results.summary()
```

This exercises the full scenario → probe → judge pipeline against canned responses, so you can iterate on scenario lists offline before pointing the auditor at a real model.

## Where to go next

- [Quickstart](guides/quickstart.md) — run your first end-to-end audit with a built-in pack
- [Auditing Basics](guides/auditing-basics.md) — how scenarios flow through the auditor and produce results
- [Targets And Judges](guides/targets-and-judges.md) — what gets probed and how responses are graded
- [Custom Judges](guides/custom-judges.md) — build judges tailored to your scenarios' `expected_behavior`
- [Context Analysis](guides/context-analysis.md) — document marks and findings used by RAG-style packs
- [Tracing And Observability](guides/tracing-and-observability.md) — trace individual scenario runs
- [Installation](guides/installation.md) — set up the package

### See Also

*   [Decision logic](auditing-basics.md)
*   [How a judge config works](custom-judges.md)
*   [The Target protocol](targets-and-judges.md)
*   [How an audit works](quickstart.md)
