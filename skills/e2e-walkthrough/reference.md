# E2E Walkthrough — Reference

Detailed execution mechanics and output procedures. Loaded on demand from SKILL.md.

---

## Phase 3 — Execution Details

### Browser State Check

Before opening a new browser session, check for stale sessions from previous skill invocations:

1. Check if agent-browser has an active session: `agent-browser get url 2>/dev/null`
   - If active and same `app` profile: navigate to `base_url` to reset page state
   - If active and different profile: close existing session first (`agent-browser close`)
   - If no active session: proceed normally with `open`
2. After opening/resetting, always verify auth before proceeding

### Startup

```bash
REPORT_DIR="$(pwd)/e2e-reports/$(date +%Y%m%d-%H%M%S)" && mkdir -p "$REPORT_DIR"
agent-browser --profile ~/.agent-browser/<app> --headed open <base_url>
agent-browser wait --load networkidle
```

**Verify auth** (skip if `auth.type: none`):
```bash
agent-browser get url
```
Check URL against `auth.verification` condition. If verification fails (auth expired):
1. Read `auth.manual_prompt` from mapping and present to user
2. Browser is already `--headed` — user logs in directly
3. After user confirms → `agent-browser get url` and re-check
4. Repeat until verified or user aborts

```bash
agent-browser trace start
```

### Multi-Site Startup (when `--sites` provided)

Open a session for each site:
```bash
# For each mapping in --sites:
agent-browser --session <app> --profile ~/.agent-browser/<app> --headed open <base_url>
agent-browser --session <app> wait --load networkidle
# Verify auth per site (same flow as single-site)
agent-browser --session <app> trace start
```

**Auth failure handling:** If a site's auth fails after 2 retry attempts, mark it SKIP. Report skipped sites before proceeding. Walkthrough continues on remaining sites.

### Site Switching During Walkthrough

When the plan transitions to a different site (or human requests it):
1. Current session stays alive (do NOT close)
2. Switch to target session: all subsequent `agent-browser` commands use `--session <target_app>`
3. Mapping context switches to the target site's mapping
4. Announce: "Switching to [portal] session..."

Human can request site switch anytime: "switch to admin", "go to portal", "check the other site".

### Per-Step Loop

1. `agent-browser snapshot` -> find element `@ref`
2. Execute action (click/fill via `@ref`)
3. `agent-browser wait --load networkidle`
4. `agent-browser screenshot "$REPORT_DIR/step-N.png"`
5. Health check (mode-dependent — see below)
6. Report to human (mode-dependent)

**Health check modes:**
- **Full walkthrough**: `agent-browser console --json` + `agent-browser errors --json` per step. Filter noise (defaults: HMR, favicon, React DevTools; app-specific noise may be defined in mapping under `health.known_noise`).
- **Smoke (`--smoke`)**: Skip per-step health checks — run `agent-browser errors --json` once at the end after all navigation steps. Smoke focus is selector verification, not runtime health.

**Selector verification strategy:**
- `text=` selectors: verify by comparing snapshot a11y tree text content against mapping values. Snapshot is the source of truth.
- `data-testid` / `aria-label` / `role=` selectors: **cannot** be verified via snapshot (a11y tree doesn't expose these attributes). Must use `agent-browser is visible "<selector>"` for DOM-level verification. In smoke mode, batch these into a post-walkthrough sweep (see `--smoke` rule 7).

### Interaction Modes

| | Guided (default) | Step | Auto |
|---|------------------|------|------|
| Before step | Show plan | Show + wait "go" | Silent |
| After step | Screenshot + summary | Screenshot + wait "go" | Continue |
| On anomaly | Pause, report | Pause, report | Log, continue |
| Human inserts | Anytime | Between steps | After done |

### Human Ad-Hoc Commands

- "click that button" -> agent uses latest snapshot
- "take a closer look at the table" -> snapshot + screenshot
- "go back" / "skip to step 6" / "stop here"

### Anomaly Handling

| Situation | Action | Track for Phase 4? |
|-----------|--------|---------------------|
| Unexpected dialog/toast | Screenshot, ask human | Yes — possible trigger mismatch |
| Element not found | Report mapping possibly stale | Yes — stale selector or missing element |
| Console error (real) | Guided/Step: pause; Auto: log | No |
| Step failure (required) | Present evidence (screenshot + console), offer: continue / debug / stop | Yes if mapping-related |
| Auth expired (per mapping verification) | Pause all modes. Present `auth.manual_prompt` from mapping. Browser is `--headed` — user re-logs in directly. `get url` to re-verify. Resume on success. | No |
| Page timeout | Retry once, then report | No |
| New element discovered | Note element name, selector, page | Yes — new element for mapping |

Maintain an in-memory list of mapping discrepancies throughout Phase 3. Each entry: `{type, page, element, details}`.

**Detection mechanisms:**
- **Stale selector**: `snapshot` a11y tree text differs from mapping's expected text for a `text=` selector, OR `is visible` returns `false` for a `data-testid`/`aria-label` selector
- **Missing element**: element listed in mapping is absent from both snapshot and `is visible` check
- **Trigger mismatch**: action on mapped element produces unexpected intermediate state (e.g., dropdown instead of direct dialog) — detected by post-action snapshot showing unexpected structure
- **New element**: element found in snapshot that has no entry in the current mapping page

**Debug pivot (on step failure):**
When human chooses "debug", keep browser open and switch to code investigation. After fix (hot reload), human says "re-run from step N" -> agent re-snapshots and continues from the failed step.

---

## Phase 4 — Output Details

### Stop Trace

```bash
agent-browser trace stop "$REPORT_DIR/trace.zip"
```

**Do NOT close the browser** — human may want to inspect the final state or continue exploring.

### Trace Analysis (Subagent)

Dispatch trace analysis to isolated context to keep verbose HAR data out of the walkthrough conversation.

| Field | Source | Required |
|-------|--------|----------|
| `trace_path` | `$REPORT_DIR/trace.zip` | YES |
| `report_dir` | `$REPORT_DIR` | YES |
| `noise_patterns` | mapping's `health.known_noise` list (if present) | NO |

```
Agent(subagent_type="e2e-trace-analyzer"):
  trace_path: "$REPORT_DIR/trace.zip"
  report_dir: "$REPORT_DIR"
  noise_patterns: <from mapping health.known_noise, or omit>
```

Agent returns: `analysis_path`, `api_failures`, `console_errors`, `clean` (true/false).

### Report

Write `$REPORT_DIR/report.md`: summary table, step results with screenshots, issues found, health log, artifacts. Include the trace-analysis.md content from the subagent.

### Flow Report (MANDATORY)

Write `$REPORT_DIR/flow-report.md`. This report visualizes the walkthrough as a mermaid flowchart with natural language descriptions, enabling developers and team members to understand and adjust the explored flow.

**File structure:**

````markdown
# Flow Report — <walkthrough context summary>

**日期**: YYYY-MM-DD HH:MM
**Mapping**: <mapping name>
**模式**: guided|step|auto
**結果**: 探索 N 個頁面、N 個對話框、N 個步驟 | N anomalies

---

## 流程總覽

> <2-3 sentence summary>

## 流程圖

```mermaid
flowchart TD
    ...
```

## 逐步敘述

### Step 1 — {source} → {target}
...

## 建議調整
<!-- omit section entirely when 0 anomalies -->
````

#### Mermaid Node Types

| UI concept | Syntax | Example |
|------------|--------|---------|
| Page | `["..."]` | `A["Dashboard"]` |
| Dialog/Modal | `{{"..."}}` | `C{{"新增成員 Dialog"}}` |
| Form submit | `(["..."])` | `F(["送出表單"])` |
| Conditional branch | `{"..."}` | `D{"選擇角色"}` |

#### Edge Rules

- Label format: `"N. 動作摘要"` (N = step number)
- Action summary ≤ 15 characters; truncate if longer
- Return to same page: dashed arrow `-.->` to distinguish "forward" from "back to origin"
- Same page appearing multiple times: reuse existing node (mermaid handles natively)

#### Node ID Rules

Use camelCase abbreviation of page name. Dialogs get `Dlg` suffix. Avoid mermaid reserved words.

#### Cross-Site Flowchart

Each site wrapped in `subgraph`, cross-site edges annotated with switch action:

```mermaid
flowchart TD
    subgraph admin["admin-panel"]
        A1["Users"] -->|"1. 建立使用者"| A2{{"Create User"}}
        A2 -.->|"2. 送出"| A1
    end
    subgraph portal["customer-portal"]
        P1["Users"] -->|"4. 確認使用者出現"| P2["User Detail"]
    end
    A1 -->|"3. 切換至 portal"| P1
```

#### Summary Generation

- 2-3 sentences: starting page, main path, conclusion
- Template: 「使用者從 `{start page}` 出發，{path summary}。{conclusion}。」
- Conclusion auto-select:
  - 0 anomalies → 「整體流程順暢，未發現異常。」
  - Has anomalies → 「發現 N 處異常，詳見建議調整區。」
  - Has health issues → 「發現 N 個 console error / API failure，詳見 trace analysis。」

#### Step Narrative

- Title: `### Step N — {source page} → {target page/element}`
- Body: one paragraph — what action, where, what result
- Result tag: `✅ PASS`, `⚠️ CONDITIONAL` (RBAC), `❌ FAIL`
- On FAIL: one-sentence reason summary (no screenshot paths — those belong in report.md)

#### Suggestions Section

| Source | Suggestion |
|--------|-----------|
| Stale selector | 「Step N 的 `{element}` selector 可能過期，建議 `/e2e-map --page {page}`」 |
| Missing element | 「Step N 預期的 `{element}` 未出現在 `{page}`，確認是否已移除或搬遷」 |
| Trigger mismatch | 「Step N 的 `{element}` 互動行為與 mapping 不一致」 |
| Console error | 「Step N 後出現 console error：`{message first 80 chars}`」 |
| API failure | 「Step N 觸發 API 失敗：`{method} {path}` → `{status}`」 |
| No anomalies | Omit the entire suggestions section |

#### report.md Integration

Add the following block at the top of `$REPORT_DIR/report.md` (before existing content):

```markdown
## Flow Report

> 探索 N 頁面 / N 對話框 / N 步驟 — N anomalies
> 詳見 [flow-report.md](./flow-report.md)

---
```

#### PR Posting (menu option 2)

When user selects "發佈 flow report 到 PR":

```bash
gh pr comment <PR> --body "$(cat $REPORT_DIR/flow-report.md)"
```

Mermaid renders natively in GitHub PR comments.

### Flow YAML Auto-Generation (MANDATORY)

**Always auto-generate a flow file after walkthrough completes.** Do NOT ask "Save as reusable flow?" — always write the file. This is mandatory because the proposal pattern gets skipped under context pressure, losing valuable walkthrough data.

**Auto-naming:**
```
walkthrough-<YYYYMMDD-HHMMSS>-<first-page>.yaml
```
- `<YYYYMMDD-HHMMSS>`: timestamp from `$REPORT_DIR` name (already in this format)
- `<first-page>`: the first page navigated to during the walkthrough, converted to kebab-case (replace `_` with `-`, strip characters outside `[a-z0-9-]`, truncate to 40 chars). E.g., `onboarding_get_started` → `onboarding-get-started`, `dashboard` → `dashboard`
- If the walkthrough starts with a verification (no navigation), use the page from the first action's `on <page>` reference
- Example: `walkthrough-20260308-143022-onboarding-get-started.yaml`

**Directory setup:** `mkdir -p .claude/e2e/flows/` before writing (directory may not exist on first walkthrough).

**Overlap detection (informational only):**
Before writing, scan `.claude/e2e/flows/*.yaml` for existing flows where 50%+ **of the new flow's** unique action target pages also appear in the existing flow. If overlap found:
```
Note: New flow overlaps with existing flow(s):
  - onboarding-full-flow.yaml (7/10 pages overlap)
You may want to consolidate or replace the older flow later.
```
This is informational — always write the new flow regardless.

**Serialization rules:**
- Use structured action references (`"Click <element> on <page>"`) not natural-language descriptions
- Set `mapping:` to the mapping filename without `.yaml` extension
- Format must match `/e2e-test` flow spec — valid keys: `name`, `description`, `tags`, `mapping` (single-site) or `sites` (cross-site), `variables`, `steps` (each with `id`, `site` (cross-site only), `action`, `expect`, `screenshot`, `optional`, `timeout`, `note`)
- Set `tags: [walkthrough, auto-generated]` plus any context-specific tags
- Set `description:` summarizing the walkthrough context

**Output path:** `.claude/e2e/flows/<auto-name>.yaml`

```
Flow saved: .claude/e2e/flows/walkthrough-20260308-143022-onboarding-get-started.yaml
  10 steps, tags: [walkthrough, auto-generated, onboarding]
  Replay: /e2e-test walkthrough-20260308-143022-onboarding-get-started
```

### Cross-Site Flow Generation

When a walkthrough used `--sites`, the auto-generated flow uses `sites:` instead of `mapping:`. The same mandatory auto-generation and auto-naming rules apply.

```yaml
# Auto-generated from cross-site walkthrough
name: <name>
description: "<description>"
tags: [cross-site]

sites:
  <alias1>: { mapping: <mapping1-filename-no-ext> }
  <alias2>: { mapping: <mapping2-filename-no-ext> }

steps:
  - id: <step-id>
    site: <alias>
    action: "<action string>"
    expect: [...]
```

Each step's `site:` is set based on which session was active during that walkthrough step.

### PR/Issue Posting

- `--pr`: `gh pr comment N --body-file pr-summary.md`
- `--issue`: Linear MCP `create_comment`

### Mapping Self-Repair

During the walkthrough, track every mapping discrepancy:
- **Stale selector**: element exists but selector doesn't match (text changed, role changed)
- **Missing element**: element removed or relocated to different page
- **Trigger mismatch**: interaction produces unexpected intermediate state
- **New element discovered**: element found during exploration that isn't in mapping

**Repair strategy by severity:**

| Discrepancies | Action |
|---------------|--------|
| 1-2 selector text changes | In-place patch: update selector/description in mapping. Human approves each change. |
| Structural change (trigger pattern, page reorganization) | In-place patch with `trigger_note` field documenting the new behavior. |
| 3+ stale selectors on same page | "Mapping for `<page>` appears significantly outdated. Recommend re-running `/e2e-map --page <page>` to refresh rather than patching individual selectors." |
| New elements discovered | Add to mapping under the correct page/dialog section. |

**`trigger_note` example:**
```yaml
project_settings:
  trigger_page: project
  trigger_element: project_settings_button
  trigger_note: "Two-step: button opens dropdown menu -> click menuitem 'Settings' to open dialog"
```

**Mapping file safety:** Before writing mapping changes, re-read the file to check for concurrent modifications (compare with the version loaded at skill start). If the file has changed since loading, present both versions and ask the user which to keep.

Always present the discrepancy list before making changes. Human approves -> agent updates mapping file.

### Browser Handoff (BLOCKING: flow YAML must be written first)

**Do NOT present this summary until Flow YAML Auto-Generation is complete.** Flow generation is lightweight (~20 lines YAML) and MUST finish even under context pressure.

Browser stays open after walkthrough. Present summary:

```
Walkthrough complete: 7/7 PASS
Browser still open at: <current URL>
-> Inspect manually, or say "close" when done
```

Only close after human confirms:
```bash
agent-browser close
```
