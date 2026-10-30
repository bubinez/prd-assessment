## PRD Assessment Custom Subagent Runbook

Use this runbook for explicit `/prd-assessment` or `/create-prd` runs when the workflow reaches a heavy analytical step. Do not search the internet for Codex custom-agent invocation details; this repository is the source of truth.

### Step sequencing (hard guard)

For `/prd-assessment`, execute workflow steps strictly in order.

- Only one step may be active at a time.
- Do not run multiple steps in parallel.
- Do not skip steps except explicit conditional transitions defined by the workflow (for example, skip `Step 13` when `--linear` is not set).
- In resume mode, start from the requested step and continue strictly step-by-step.

### Agent map

| Workflow step | Custom agent config | Expected responsibility |
| --- | --- | --- |
| `Step 4` | `agents/prd_explorer.toml` | Codebase analysis and execution-surface mapping |
| `Step 11` | `agents/prd_self_reviewer.toml` | PRD self-review guard rails |
| `Step 12` | `agents/prd_devils_advocate.toml` | Role-based adversarial PRD review |
| `Step 14` | `agents/prd_decomposer.toml` | Change-plan decomposition |

### Preferred path: Codex Desktop subagent tool

When the current Codex environment exposes a `spawn_agent` tool with the matching `agent_type`, use it directly:

1. Spawn the mapped agent type, for example `prd_self_reviewer` for `Step 11`.
2. Pass only the source-of-truth paths and the step-specific instruction file. For `Step 11`, pass:
   - current `prd.md`;
   - current `codebase-analysis.md`, if it exists;
   - `instructions/prd-self-review.md`;
   - the relevant `agents/<agent>.toml` path.
3. Tell the subagent not to ask the user directly and to return the workflow output contract.
4. Wait for the subagent result before marking the step complete.
5. If the subagent returns `decision_required`, surface only the decision-shaped question to the user.

### Delegation boundary (hard guard)

Only the parent workflow thread may spawn or invoke required heavy-step subagents.

- A heavy-step subagent (`prd_explorer`, `prd_self_reviewer`, `prd_devils_advocate`, `prd_decomposer`) must not spawn another agent.
- A heavy-step subagent must not call `codex exec` to invoke another custom agent.
- If runtime/tools are unavailable inside a subagent, return `blocked_by_runtime` to the parent flow with a short reason.
- The parent flow decides retry/escalation/stop-condition handling.

### CLI fallback: non-interactive `codex exec`

Use CLI fallback only when the Desktop subagent tool is unavailable but `codex exec` is available locally. The CLI help is local enough; do not web-search it.

1. Read the mapped `agents/<agent>.toml`.
2. Build a prompt that includes:
   - the TOML `developer_instructions`;
   - the workflow step instruction file, for example `instructions/prd-self-review.md`;
   - concrete input artifact paths;
   - the expected JSON or Markdown output contract.
3. Run `codex exec` with the applicable sandbox and the available default model.
4. Save the final response to a deterministic temp artifact under `.codex/`.

Example PowerShell template for `Step 11`:

```powershell
$prompt = @"
You are the skill-provided subagent `prd_self_reviewer`.

Load and follow:
- ./agents/prd_self_reviewer.toml
- ./instructions/prd-self-review.md

Inputs:
- PRD: <absolute path to prd.md>
- Codebase analysis: <absolute path to codebase-analysis.md, or NONE>

Return the Step 11 subagent output contract only:
- verdict: pass without changes | pass with fixes | rewind_required | decision_required
- fixed_violations
- remaining_issues
- questions_for_user, only for decision_required
- summary

Do not ask the user questions directly.
Do not replace missing evidence with assumptions.
"@

codex exec `
  --cd . `
  --sandbox read-only `
  --ask-for-approval never `
  --output-last-message .codex/tmp-step11-self-review.json `
  $prompt
```

### Stop condition

If neither the Desktop subagent tool nor `codex exec` can run the mapped custom agent, stop the workflow and report the missing agent or missing runtime. Do not complete the heavy step through a single-agent shortcut.
This check is owned by the parent workflow thread only. Subagents must never recurse into additional agent-launch paths.
