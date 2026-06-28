# Spec Kit — Agent Knowledge Reference

PURPOSE: Dense factual reference about how GitHub Spec Kit (`specify` framework) works internally. Intended as context for an AI coding agent. Not narrative; facts + file paths + rules. Source: code read of github/spec-kit @ 0.11.10.dev0.

## CORE MODEL
- Spec Kit = toolkit for Spec-Driven Development (SDD). Produces reviewable artifacts (constitution → spec → plan → tasks) before code.
- TWO LAYERS:
  - Layer 1 = the tool itself (repo github/spec-kit): the `specify` CLI + master templates/scripts. Edited only when forking.
  - Layer 2 = a generated project (created by `specify init`): contains `.specify/` + agent command files. Team customizations live here.
- Slash commands are NOT code. They are Markdown prompt files the AI agent reads and follows. The LLM is the execution engine. Spec Kit supplies disciplined prompts + deterministic shell scripts.
- State between steps lives on the FILESYSTEM, not in variables: per-feature folder `specs/NNN-name/` + pointer file `.specify/feature.json` + the git branch name.
- AGNOSTIC BY DESIGN (frameworks built on it inherit this):
  - Model-agnostic: commands are plain prompts; workflow steps can set per-step `model:`.
  - Platform/agent-agnostic: 37 integration adapters (src/specify_cli/integrations/); selectable at init and per workflow step (`integration:`).
  - Language-agnostic: templates carry no language assumptions; implement.md holds 44 language/tool profiles applied as needed.
  - Greenfield AND brownfield: specify→implement builds from scratch; analyze + converge adopt SDD onto an existing codebase. `/speckit.converge` assesses current code vs spec/plan/tasks and appends only unbuilt work (no rewrite).

## SLASH COMMANDS
- Defined in: `templates/commands/*.md` (Layer 1). One file per command.
- Installed per agent by `specify init`:
  - Claude: `.claude/skills/speckit-<name>/SKILL.md`; context file `CLAUDE.md`.
  - Copilot (markdown mode): `.github/prompts/speckit.<name>.prompt.md`; context file `.github/copilot-instructions.md`; also writes `.vscode/settings.json`. Copilot `--skills` mode: `.github/skills/speckit-<name>/SKILL.md`.
- Command list (file → purpose):
  - constitution.md → create/update project rules (writes `.specify/memory/constitution.md`)
  - specify.md → define WHAT/WHY (writes `specs/NNN/spec.md`)
  - clarify.md → ask questions, refine spec
  - plan.md → technical HOW (writes plan.md, research.md, data-model.md, contracts/, quickstart.md)
  - tasks.md → ordered task list (writes tasks.md with `[P]` parallel markers)
  - analyze.md → cross-check spec/plan/tasks consistency (read-only report)
  - checklist.md → domain quality checklist
  - implement.md → execute tasks, write source code, mark tasks `[X]`
  - taskstoissues.md → push tasks to GitHub issues
  - converge.md → reconcile drift between spec and code
- Command file structure: YAML frontmatter + Markdown instruction body.
  - frontmatter `scripts:` → deterministic helper run first (sh + ps variants). Returns JSON with file paths.
  - frontmatter `handoffs:` → next command(s); `send: true` = auto-fire next command in same chat.
  - frontmatter `description:` → shown in command picker.
  - body uses `$ARGUMENTS` placeholder = user text after the command.

## WHAT `specify init` DOES
- Fully offline; scaffolds from assets bundled in the CLI package (version-matched, no network).
- Steps: (1) check required tools; (2) select integration (default copilot non-interactive) + script type sh/ps; (3) install bundled templates, scripts, workflow, shared infra into `.specify/`; copy constitution-template → `.specify/memory/constitution.md` (preserves existing); (4) install the agent's command files + agent context file + optional `--preset`.
- No AI runs during init. It only lays down files. Command CONTENT is identical across agents; only packaging/location differs:
  - Claude: commands → `.claude/skills/speckit-<name>/SKILL.md`; context file `CLAUDE.md`.
  - Copilot: commands → `.github/prompts/speckit.<name>.prompt.md`; context file `.github/copilot-instructions.md`; plus `.vscode/settings.json`. (`--skills` mode → `.github/skills/speckit-<name>/SKILL.md`.)
- Flags: `--integration <agent>`, `--integration-options="--skills"`, `--here`/`.` (current dir), `--force`, `--preset <id>`, `--ignore-agent-tools`, `--integration generic --integration-options="--commands-dir <dir>"` (bring-your-own-agent).

## VERIFICATION / REGRESSION GATE (brownfield: ensure new feature doesn't break existing)
- No single `verify` command exists. Verification today = tasks-template per-story "Independent Test" + "Checkpoint" + final "Polish & Cross-Cutting Concerns" phase; implement.md completion validation; analyze (consistency); checklist (quality).
- To add a regression gate, layer it (upgrade-safe, auto per-feature):
  - Layer A (intent): add a "Backward Compatibility" principle to `.specify/memory/constitution.md` — new features MUST NOT break existing functionality; identify touched components; verify before completion.
  - Layer B (generation): `.specify/templates/overrides/tasks-template.md` — add "Phase R: Regression & Backward-Compatibility Verification" with tasks: identify touched modules, run full existing test suite (baseline vs post), smoke-test adjacent features, verify public contracts unchanged, halt on regression.
  - Layer C (enforcement): `.specify/extensions.yml` after_implement hook, optional:false (mandatory), command runs the real test suite (./gradlew test | pytest | npm test).
  - Layer D (optional): `.specify/templates/overrides/plan-template.md` — add "Impact Analysis" section listing affected existing components up front.
  - In auto-pipeline: add a `shell` step (run tests) + `gate` step after `implement` in the workflow YAML.
- Recommended: B + C as core; A for intent; D for large/opaque codebases. Prefer overrides+hooks over editing implement.md (automatic + survives upgrades).

## SCRIPTS (deterministic glue)
- Location: `scripts/bash/*.sh` and `scripts/powershell/*.ps1`. Agent picks one per OS.
- Key scripts:
  - create-new-feature.sh → compute branch/feature name, make `specs/NNN-name/`, copy spec template, write `.specify/feature.json`.
  - setup-plan.sh → locate current feature, copy plan template, return JSON paths.
  - setup-tasks.sh → prep tasks file.
  - check-prerequisites.sh → verify required artifacts exist.
  - common.sh → shared helpers: get_repo_root, resolve_template, feature-state read/write.
- get_repo_root finds project by walking up to `.specify/`.
- Scripts emit JSON; the agent parses it to learn read/write paths. Anything requiring judgment is left to the LLM.

## ARTIFACT TEMPLATES (output structure)
- Location: `templates/*-template.md` (Layer 1) → copied to `.specify/templates/` in a project.
- Files: spec-template.md, plan-template.md, tasks-template.md, constitution-template.md.
- TEMPLATE RESOLUTION PRIORITY (resolve_template in common.sh; first match wins):
  1. `.specify/templates/overrides/<name>.md`  (project override, highest)
  2. `.specify/presets/<id>/templates/`         (presets, by priority)
  3. `.specify/extensions/<id>/templates/`      (extension-provided)
  4. `.specify/templates/<name>.md`             (core, lowest)
- To change a step's OUTPUT STRUCTURE: edit the template or add an override (upgrade-safe).

## CONSTITUTION (project rules)
- File: `.specify/memory/constitution.md`. Created by `/speckit.constitution`.
- Read by `/speckit.plan`; plan performs "Constitution Check" gate, ERRORs on unjustified violations.
- Primary home for org-wide engineering rules.

## INTEGRATIONS (AI agents)
- Location: `src/specify_cli/integrations/<agent>/`. 37 supported (claude, copilot, gemini, cursor_agent, codex, etc.).
- Each defines: install folder, command format, args placeholder, context_file, dispatch_command() (non-interactive CLI invocation).
- Selected at init: `specify init <proj> --integration copilot` (optionally `--integration-options="--skills"`).

## EXTENSIONS & HOOKS
- Bundled extensions: `extensions/` (agent-context, git, bug, selftest, template).
- Hooks configured per-project in `.specify/extensions.yml`. Keys: before_<phase> / after_<phase> (e.g. before_plan, after_implement).
- Every core command checks extensions.yml for its phase hooks.
- Hook fields: id, enabled (default true), optional, extension, command, description, prompt, condition.
  - optional: false → mandatory, command auto-executes the hook and waits (blocking gate).
  - optional: true → surfaced as a suggestion.
- Use hooks to inject lint gates / ticket creation / compliance checks WITHOUT editing core command files.

## PRESETS
- Location: `presets/` (lean, scaffold, self-test). Bundles of template overrides + config. Priority 2 in template stack. Used to standardize across many projects.

## WORKFLOW ENGINE (auto-chaining = how "one command runs all steps")
- Engine: `src/specify_cli/workflows/engine.py`. Step types: `src/specify_cli/workflows/steps/`.
- Definitions: YAML, e.g. `workflows/speckit/workflow.yml` (id speckit, "Full SDD Cycle": specify → gate → plan → gate → tasks → implement).
- Commands:
  - `specify workflow add <id>`
  - `specify workflow run <id> --input key=value`
  - `specify workflow status`
  - `specify workflow resume <run_id>`  (continue after a gate pause)
- Execution: engine loads YAML, runs steps SEQUENTIALLY. For a `command` step it calls integration.dispatch_command() which spawns the agent CLI as a SUBPROCESS, non-interactively (one slash command at a time). State persisted after each step → resumable.
- STEP TYPES (11): command, prompt, shell, init, gate, if, switch, while, do-while, fan-out, fan-in.
- TWO CRITICAL FACTS:
  1. Each step may set its own `integration:` and `model:` → different steps can use different agents/models.
  2. Steps do NOT pass file CONTENTS via `{{ }}` expressions. A command step output captures only exit_code/stdout/stderr (stdout empty while streaming). Inter-step handoff = the feature folder on disk (`specs/NNN/` + `.specify/feature.json`). Design steps to read prior artifacts from disk.
- Expressions: `{{ inputs.x }}`, `{{ steps.id.output.exit_code }}`, comparisons, and/or/not, in, default/join/contains filters (expressions.py).

## HANDOFFS (auto-advance inside chat, no engine needed)
- Stock chain via frontmatter `handoffs: send: true`: specify→plan→tasks→implement.
- `send: true` = on completion, auto-fire next command in same chat session.
- Not all agents honor handoffs (e.g. Forge strips the key).
- DISTINGUISH auto-run mechanisms: terminal `specify workflow run ...` = engine/fork; one chat slash command that cascades = handoffs or a custom umbrella command.

## "SUBAGENTS" — REALITY
- No command declares a fixed roster/count of subagents. Commands are prompts.
- Only `plan.md` contains an agent-dispatch instruction (research phase). Count is dynamic ≈ (# NEEDS CLARIFICATION) + (# tech/dependency choices). Whether real parallel subagents spawn depends on the AI tool.
- `[P]` markers in tasks.md/implement.md = tasks that MAY run concurrently (metadata), not subagents.
- `handoffs`/`agent:` frontmatter = NEXT command, not a subagent.
- To get DEFINED, countable agents: write explicit prose in a command file, or use a `fan-out` workflow step (count = collection size).

## GENERATED PROJECT LAYOUT (Layer 2)
```
.specify/
  memory/constitution.md      # rules
  templates/                  # spec/plan/tasks/constitution templates
    overrides/                # project overrides (highest priority)
  scripts/                    # bash + powershell
  extensions/                 # installed extensions
  presets/                    # installed presets
  extensions.yml              # hook config
  feature.json                # current-feature pointer
.claude/skills/speckit-*/SKILL.md   # commands (Claude)   | OR
.github/prompts/speckit.*.prompt.md # commands (Copilot)
CLAUDE.md | .github/copilot-instructions.md   # always-loaded context
specs/NNN-name/             # per-feature: spec.md, plan.md, research.md,
                            #   data-model.md, contracts/, quickstart.md, tasks.md
```
- To clone a team's customization: copy `.specify/memory/constitution.md`, `.specify/templates/overrides/`, `.specify/presets/`, `.specify/extensions.yml`, and edited command files (`.claude/skills/...` or `.github/prompts/...`).

## CUSTOMIZATION MAP (where to change what)
- Step output structure → `.specify/templates/<name>-template.md` or `overrides/<name>.md`
- Step behavior/instructions → installed command file (`.claude/skills/speckit-*/SKILL.md` or `.github/prompts/*.prompt.md`); or fork `templates/commands/`
- Org-wide rules → `.specify/memory/constitution.md`
- Inject a step before/after a phase → `.specify/extensions.yml` hooks
- Attach reference docs to a step → referenced docs folder in the command, the constitution, or the context file; or inline via `$ARGUMENTS`
- Auto-run all steps → workflow YAML + `specify workflow run`; or `handoffs: send:true`
- Different agent per step → `integration:`/`model:` per workflow step
- Defined multi-agent → explicit prose in a command, or `fan-out` step
- Standardize across projects → a preset

## INTEGRATING A CUSTOM MULTI-STEP PIPELINE (generic template)
- Pattern: a custom N-step pipeline maps onto Spec Kit constructs:
  - data-gather step ≈ front half of `specify` → custom command writing `analysis.md`
  - detailed-spec step ≈ `plan` + `tasks` → custom command reading analysis.md + reference docs, writing strict `spec.md`
  - implement step ≈ reuse `speckit.implement`
- Where per-step agents plug in:
  - LLM-prompt agents → make each step a `command` (skill .md); dispatch internal subagents via prose or `fan-out`/`fan-in`.
  - Standalone programs/scripts → wrap as `shell` steps.
  - Mixed is supported.
- Artifact flow: wire all steps to the SAME feature folder; pass artifacts via disk, not `{{ }}`.
- Reference docs (e.g. architecture docs) injection: a docs folder read by the spec step, and/or the constitution, and/or a strict spec-template with explicit sections (Files to change / New packages / Public contracts).
- Add review `gate` steps between phases for human sign-off (resumable).

## AUTO-REFERENCE A FILE IN A STEP (no manual mention)
- Mirror how plan always reads constitution: path is BAKED INTO the command (plan.md: "Read /memory/constitution.md"), not typed each run.
- Patterns: (a) single fixed file → bake path into command/template override ("Read .specify/memory/regression-strategy.md"); (b) rules → constitution (auto-read at plan); (c) many/variable docs → folder convention + glob ("Read ALL files under docs/regression/"), optionally enumerated by a setup script returning JSON (like setup-plan.sh).
- Reliability: chat mention < baked path in command/template < constitution < hook script that executes against the file (deterministic). Combine baked-path + hook.
- Do NOT rely on typing "use XX.md" in $ARGUMENTS each run.

## UPGRADING (without losing customizations)
- CLI: `specify self check` (read-only), `specify self upgrade [--dry-run] [--tag vX.Y.Z]` (auto-detects uv/pipx).
- Project files: `specify init --here --force --integration <agent>` (re-scaffolds; --force overwrites/merges).
- Core templates at .specify/templates/<name>.md CAN be overwritten on re-init. Overrides (.specify/templates/overrides/), constitution (preserved if exists), presets, extensions are NOT clobbered the same way.
- RULE: never edit core files in place; use overrides/constitution/presets/extensions → upgrade-safe. Keep tooling upgrades in separate commits/PRs.

## DISTRIBUTING CUSTOMIZATIONS (team)
- Least→most heavyweight: constitution+overrides committed to repo → preset (`specify preset add`) → extension (`specify extension add`, adds commands+hooks) → private catalog (env SPECKIT_*_CATALOG_URL) → fork of spec-kit.
- A custom framework (e.g. ACME) is typically a fork OR internal preset/extension + custom workflow YAML.

## ADDING A NEW COMMAND
- Via extension: declare in extension.yml under provides.commands (name/file/description); ship command md; install drops it into agent command folder. (extensions/git/extension.yml example.)
- Manually: add templates/commands/<name>.md (fork) or drop installed file (.claude/skills/<name>/SKILL.md | .github/prompts/<name>.prompt.md) and re-init.
- Command file = frontmatter (description, optional scripts:, handoffs:) + body using $ARGUMENTS. Keep speckit./acme. prefix.

## CLI GROUPS & ENV VARS
- Groups: init; self (check/upgrade); workflow (run/resume/status/list/add/remove/search/info + catalog + step); extension (add/remove/list/search/info/enable/disable/update); preset (add/remove/list/set-priority); integration; bundle. Run `specify --help`.
- Env vars: SPECIFY_FEATURE / SPECIFY_FEATURE_DIRECTORY (force current feature; printed by create-new-feature); SPECIFY_INIT_DIR (root for non-interactive/CI); SPECKIT_COPILOT_ALLOW_ALL_TOOLS (default on; headless Copilot perms; old: SPECKIT_ALLOW_ALL_TOOLS); SPECKIT_INTEGRATION_<KEY>_EXECUTABLE/_EXTRA_ARGS; SPECKIT_*_CATALOG_URL (private catalogs); SPECKIT_WORKFLOW_RUN_ID.

## HEADLESS / CI
- Workflow engine dispatches each command as a non-interactive subprocess (dispatch_command, integrations/base.py).
- CI: set SPECIFY_INIT_DIR; install + authenticate the agent CLI; SPECKIT_COPILOT_ALLOW_ALL_TOOLS=1 (default) to avoid permission prompts.
- gate steps pause for humans → for unattended runs remove gates or pre-decide; state persists → `specify workflow resume <run_id>`. Inspect via `specify workflow status`.

## VERSION CONTROL
- Commit: .specify/ (constitution, templates, overrides, presets, extensions.yml, scripts), installed command files (.claude/skills or .github/prompts), context file (CLAUDE.md/copilot-instructions.md), specs/ artifacts, workflows/*.yml.
- Gitignore: agent folders may hold credentials/caches (init warns). Build outputs gitignored per language by implement.

## TROUBLESHOOTING / REVERSE-ENGINEER A CUSTOM SETUP
- Command not found → wrong agent init or different folder (claude .claude/skills vs copilot .github/prompts); re-init with correct --integration.
- Wrong feature → stale .specify/feature.json or unset SPECIFY_FEATURE.
- Edits lost after upgrade → edited core instead of override.
- Hooks not firing → invalid extensions.yml, enabled:false, or wrong phase key.
- Handoffs not advancing → agent doesn't support handoffs; use workflow engine.
- To find what a custom framework changed vs stock, diff against fresh `specify init`: constitution.md; templates/overrides/ + presets/; extensions.yml + extensions/; installed command files (bodies edited?); workflows/*/workflow.yml; SPECKIT_*_CATALOG_URL in env.

## KEY FILES INDEX
- Command logic: templates/commands/*.md
- Output structure: templates/*-template.md
- Deterministic glue: scripts/bash/*.sh, scripts/powershell/*.ps1
- Agent adapters: src/specify_cli/integrations/
- Workflow engine: src/specify_cli/workflows/engine.py
- Workflow step types: src/specify_cli/workflows/steps/
- Workflow definitions: workflows/*/workflow.yml
- Project rules: .specify/memory/constitution.md
- Hooks: .specify/extensions.yml
