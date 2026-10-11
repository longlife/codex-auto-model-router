# Routing profiles

The profile workflow is adapted from David Soff's PR #6; the optional prompt hook comes from his PR #4. Integration preserves the reviewed catalog, explicit model pins, fallback dispatch safety, and the historical GPT-5.6 programmatic policy API.

## Commands and scope

Run commands from the installed Skill directory, or use absolute script paths:

```bash
python3 scripts/router_lite.py profile-show --repository /path/to/project
python3 scripts/router_lite.py profile-set economy --scope project --repository /path/to/project
python3 scripts/router_lite.py profile-set balanced --scope global --repository /path/to/project
python3 scripts/router_lite.py decide --profile quality --repository /path/to/project
```

Global configuration lives in `${CODEX_HOME:-$HOME/.codex}/router.toml`; project configuration lives in `<repository>/.codex/router.toml`. Profile selection is command-line > project > global > balanced. Route overrides for the selected profile are built-in < global < project. Named subagent overrides resolve global < project by subagent name. Explicit model/effort arguments override profile routes. A model-only argument does not authorize high Astra effort.

`profile-set` changes only the selected scope, preserves existing comments and route overrides, and uses atomic replacement. Invalid configuration is rejected without rewriting it. Lite routing fails open to local execution; strict CLI reports configuration errors. Lite and strict routing check project opt-out before loading profiles. Profile commands do not re-enable a disabled project.

## Built-in profiles

| Profile | Difference from balanced |
|---|---|
| balanced | Existing default routes: Luna for bounded work, Sol for bounded complexity, Astra for uncertainty/high consequence. |
| economy | Uses Luna/high for complex bounded and complex uncertain tasks; preserves the high-consequence and failure lanes. |
| quality | Uses Sol for ordinary work and deterministic scans; does not automatically increase Astra effort. |

All built-ins use Astra/low, with medium reserved for reasoning/verification failure escalation. Astra high, xhigh, or max requires an explicit effort argument or an explicit configured lane override. Available-model checks and fallback routing still apply; a configured lane is not proof that the model executed.

## Lane overrides

```toml
schema_version = 1
profile = "balanced"

[profiles.balanced.routes.complex_uncertain]
model = "gpt-6-astra"
effort = "low"
```

Valid lanes: `mechanical_default`, `ordinary_default`, `bounded_scan`, `bounded_deep_deterministic`, `latency_priority`, `complex_bounded`, `complex_uncertain`, `high_consequence`, `complex_failed_escalation`. Each override requires both `model` and `effort`; only reviewed catalog/legacy model IDs and low-through-max efforts are accepted. Unknown keys, profiles, lanes, or unsupported efforts are errors. Overrides do not edit the model catalog or executor presets.

Strict CLI uses the same resolved profile. Programmatic `route_policy` APIs retain legacy defaults unless passed a `routing_config` from `resolve_routing_config`. Linear and parallel plans capture the selected routes once; later config changes cannot mutate an existing plan. Historical benchmark tables remain unchanged.

## Subagent model overrides

CC Switch aggregation can expose models the reviewed catalog does not contain, such as `ccs-<profile>/deepseek-*` or `ccs-<profile>/glm-*`. Name subagents to choose the model and effort each routed leaf actually runs:

```toml
schema_version = 1

[subagents.quick_worker]
model = "ccs-jiong-nas-opencode/glm-5.3-flash"
effort = "low"
lanes = ["mechanical_default", "ordinary_default"]

[subagents.deep_worker]
model = "ccs-jiong-nas-opencode/deepseek-v4.1-flash"
effort = "high"
lanes = ["complex_bounded", "complex_uncertain", "high_consequence"]
```

Names are free-form and need no `_effort` suffix: every subagent declares its own `model`, `effort`, and the routing `lanes` it serves. A lane may be claimed by only one subagent, and an unclaimed lane keeps the built-in catalog route. Project entries override global entries with the same name.

`model` is passed through verbatim, so aggregated IDs are allowed. When `${CODEX_HOME}/config.toml` points `model_catalog_json` at a readable catalog, or `cc-switch-model-catalog.json` exists, a listed model only accepts the efforts it advertises; unlisted models are unrestricted. Efforts outside low-through-max are always rejected.

Routing selects the lane, not the model. `recommended_route` stays the catalog recommendation, `execution_route` becomes the configured subagent route, and `spawn_contract` carries `subagent`, `model`, `model_reasoning_effort`, `model_source`, `agent_type`, and `spawn_without_agent_type`. Without a matching subagent the contract keeps the resolved catalog route. Subagent overrides never edit the model catalog or executor presets.

An agent template always runs the model baked into its TOML, and the runtime ignores a spawn-time model override whenever `agent_type` is set, so a template is reused only when its own model and effort already match the configured execution route. Aggregated models with no matching template return `agent_type=null` with `spawn_without_agent_type=true`; the coordinator must then spawn without `agent_type` and pass the model and effort explicitly, and must carry the bounded-execution rules in the task capsule. New agent TOML files are only picked up after a Codex restart, so the router does not generate presets at runtime.

## Optional prompt hook

Install with `./install.sh --install-hook` or `./install.ps1 -InstallHook`, review/trust with `/hooks`, and restart Codex. Ordinary installation does not enable it. The hook adds a short routing reminder only when the Skill is installed and the project has not opted out; it does not include the user's prompt in its output, change the active model, or spawn an executor itself. Unrelated hooks are preserved, reinstall is idempotent, and installer failure rolls back the hook configuration along with the Skill payload.
