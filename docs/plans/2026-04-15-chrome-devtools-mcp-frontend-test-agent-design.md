# Frontend Test And Auto-Fix Agent

## 1. Product Vision

Build an agent product that verifies whether a frontend implementation is actually following the PRD, finds behavior or UI mismatches in a real browser, and proposes or applies safe frontend fixes with human approval.

This is not a generic QA bot. The product's core value is closing the loop between:

- what the PRD says should happen
- what the browser actually does
- what code change is needed to resolve the gap

The ideal outcome is:

1. A PM or engineer gives the agent a PRD, target URL, and repo.
2. The agent turns the PRD into executable expectations.
3. The agent drives Chrome through `chrome-devtools-mcp`.
4. The agent records evidence, explains failures, and maps them to likely frontend files.
5. The agent generates a patch or fix plan.
6. The agent reruns the scenario and shows whether the fix closed the gap.

## 2. Who The Product Is For

### Primary users

- Frontend engineers who want fast regression detection and auto-repair suggestions
- Founders or PMs who want to check whether a shipped UI matches the intended flow
- QA or product ops users who want reproducible browser evidence instead of vague bug reports

### Jobs to be done

- "Check whether this page or flow matches the PRD."
- "Tell me what is broken in the real UI."
- "Show me evidence, not just an LLM opinion."
- "Suggest the smallest safe code fix."
- "Re-run the scenario after the fix and confirm the outcome."

## 3. MVP Positioning

The first version should be a developer-facing copilot, not a full autonomous QA platform.

### Recommended MVP boundaries

- Single app per workspace
- Manual run, not CI-first
- One PRD document in Markdown
- One target environment at a time
- Focus on 1-3 critical user flows
- Frontend-only fixes at first
- Human approval before any file write

### Explicit non-goals for MVP

- Full natural-language understanding of any PRD format
- Backend bug fixing
- Pixel-perfect visual review across every breakpoint
- Autonomous merge to main branch
- Large-scale multi-tenant orchestration

## 4. Product Workflow

### User input

The user provides:

- PRD document or feature spec
- target URL
- local repo path
- optional credentials or seeded test data
- run mode: inspect only, suggest fix, or apply fix

### Agent run lifecycle

1. Parse the PRD into a machine-readable test contract.
2. Plan the browser journey needed to validate the contract.
3. Use `chrome-devtools-mcp` to execute the journey in Chrome.
4. Capture screenshots, DOM state, console logs, network failures, and user-visible outcomes.
5. Evaluate actual behavior against the contract.
6. If a mismatch exists, classify the failure:
   - missing UI
   - wrong text or state
   - broken interaction
   - navigation mismatch
   - data rendering issue
   - console or network issue surfaced in the UI
7. Map the failure to likely code areas.
8. Generate a minimal fix proposal.
9. If approved, edit code and rerun the flow.
10. Produce a final report with pass or fail status and evidence.

## 5. Core Product Modules

### A. PRD Compiler

Turns human PRD text into structured expectations.

Suggested output schema:

- feature name
- page or route
- preconditions
- user actions
- expected visible results
- forbidden states
- acceptance checks
- fix severity

This layer matters because raw PRD text is too ambiguous for stable browser validation.

### B. Browser Execution Engine

Wraps `chrome-devtools-mcp` with a deterministic runner.

Responsibilities:

- open page
- wait for stable UI state
- click, type, scroll, and navigate
- inspect DOM and accessibility tree
- capture screenshots
- collect console errors and failed requests
- standardize retries and timeouts

### C. Assertion And Evidence Engine

Compares expected states from the PRD contract with browser evidence.

Outputs:

- pass or fail per acceptance check
- confidence score
- evidence bundle
- natural-language explanation
- machine-readable failure type

### D. Code Fix Engine

Uses LLM tool calling against repo tools.

Responsibilities:

- search likely files
- explain root-cause hypothesis
- generate minimal patch
- keep fix scope bounded
- rerun validation after patch

### E. Safety Layer

Prevents reckless edits.

Rules:

- only edit allowlisted frontend paths
- require confidence threshold before patching
- block destructive refactors
- show diff before applying
- always keep before/after evidence

## 6. Agent Architecture

### High-level components

```text
User / CLI / Web UI
        |
        v
Run Orchestrator
        |
        +--> PRD Compiler
        |
        +--> Scenario Planner
        |
        +--> Browser Runner (chrome-devtools-mcp)
        |
        +--> Evaluator
        |
        +--> Repo Fixer
        |
        +--> Verifier
        |
        +--> Report Store
```

### Recommended runtime model

Use a stateful run graph instead of a single free-form agent loop.

Recommended states:

- `drafting_contract`
- `planning_flow`
- `executing_browser_steps`
- `collecting_evidence`
- `evaluating_result`
- `proposing_fix`
- `applying_fix`
- `retesting`
- `completed`
- `needs_human_input`

This makes the product more debuggable than an opaque "LLM decides everything" loop.

## 7. Tool Calling Design

The main agent should not talk directly to every raw tool in an unconstrained way. Put a thin product layer between the LLM and the tools.

### Recommended tool groups

#### Browser tools

- `open_page`
- `inspect_dom`
- `inspect_accessibility_tree`
- `click_element`
- `type_text`
- `capture_screenshot`
- `read_console_logs`
- `read_network_failures`
- `wait_for_selector`
- `evaluate_script`

#### Spec tools

- `compile_prd_to_contract`
- `list_acceptance_checks`
- `get_current_check`

#### Repo tools

- `search_code`
- `read_file`
- `propose_patch`
- `apply_patch`
- `run_frontend_tests`
- `start_dev_server`

#### Report tools

- `save_evidence`
- `write_run_summary`
- `compare_before_after`

### Why this matters

If you expose raw DevTools and raw filesystem tools directly, the agent will be flexible but unstable. A productized tool surface lets you enforce safe sequencing and consistent outputs.

## 8. PRD Contract Format

The most important product decision is to define a narrow "PRD contract" format early.

Suggested schema:

```json
{
  "feature": "Checkout CTA",
  "route": "/pricing",
  "persona": "new visitor",
  "preconditions": [],
  "steps": [
    {
      "action": "open",
      "target": "/pricing"
    },
    {
      "action": "click",
      "target": "Start free trial"
    }
  ],
  "assertions": [
    {
      "type": "visible_text",
      "target": "Create your account",
      "required": true
    },
    {
      "type": "url_contains",
      "target": "/signup",
      "required": true
    }
  ],
  "forbidden": [
    {
      "type": "console_error"
    }
  ]
}
```

You can let the LLM generate this schema first, then let a validator normalize it before execution.

## 9. User Experience

### MVP entry point

The cleanest first product is a CLI with a readable run report.

Example:

```bash
agent run \
  --prd docs/prd/signup.md \
  --url http://localhost:3000 \
  --mode suggest-fix
```

### Suggested output

- overall run status
- checks passed and failed
- screenshots
- console and network issues
- suspected root cause
- proposed patch diff
- rerun result after patch

### Later product expansion

After CLI proves useful, add:

- local desktop UI
- PR comment integration
- CI mode
- multi-run history
- flaky check suppression

## 10. MVP Technical Architecture

### Recommended stack

- Orchestrator: Node.js with TypeScript
- LLM integration: tool-calling capable model with structured output
- Browser control: `chrome-devtools-mcp`
- Repo operations: local filesystem plus patch tool
- Contract validation: `zod`
- Job state: local JSON or SQLite
- Report rendering: Markdown plus static HTML

### Why TypeScript first

- natural fit with frontend repos
- easier tool schema definitions
- good DX for MCP and JSON contracts
- strong validation around agent state transitions

## 11. Fix Scope For V1

Do not try to auto-fix every frontend issue.

### Good V1 fix categories

- wrong copy or label
- missing button or link rendering
- incorrect conditional rendering
- broken navigation wiring
- wrong empty state
- simple prop or state bug
- minor selector or event handler issue

### Avoid in V1

- deep CSS layout debugging
- animation timing issues
- complex async race conditions
- backend API contract mismatches
- auth and session edge cases
- large component refactors

## 12. Evidence Model

Every failed assertion should produce a structured record:

- run id
- contract id
- step id
- current URL
- screenshot path
- DOM snippet
- console errors
- network failures
- expected result
- actual result
- root-cause hypothesis
- fix proposal id

This evidence model becomes one of the product's biggest defensibility layers because it makes runs auditable and reproducible.

## 13. Key Risks

### Risk 1: PRD ambiguity

Mitigation:

- constrain PRD format
- add approval step for compiled contract
- support "ask for clarification" when confidence is low

### Risk 2: False positives

Mitigation:

- require multi-signal evidence
- wait for UI stability before asserting
- distinguish content mismatch from selector mismatch

### Risk 3: Dangerous fixes

Mitigation:

- patch only inside scoped frontend paths
- show diff before apply
- rerun only the impacted scenario first

### Risk 4: Flaky browser execution

Mitigation:

- standardize retries
- add deterministic wait policies
- save browser traces for replay

## 14. Success Metrics

### Product metrics

- time from run start to useful failure report
- precision of detected PRD mismatches
- percentage of failures with actionable evidence
- percentage of suggested fixes accepted by engineers
- percentage of applied fixes that pass revalidation

### MVP quality bars

- one critical flow can be checked end to end
- failures include screenshot plus reasoning plus suspected file area
- simple fixes can be applied and revalidated safely

## 15. Step-By-Step Build Plan

### Phase 0: Nail the contract

Build the PRD-to-contract pipeline first.

Deliverables:

- contract schema
- Markdown PRD parser
- contract validator
- sample PRD fixtures

Exit criterion:

- one PRD can be converted into stable executable assertions

### Phase 1: Browser runner

Build a reliable execution wrapper on top of `chrome-devtools-mcp`.

Deliverables:

- browser session manager
- typed browser tool wrappers
- screenshot and log capture
- stable wait and retry policy

Exit criterion:

- one user flow can be executed deterministically

### Phase 2: Evaluation engine

Build the expected-vs-actual comparison layer.

Deliverables:

- assertion runner
- evidence bundling
- pass or fail reporting
- confidence scoring

Exit criterion:

- failures become reproducible and explainable

### Phase 3: Repo fixer

Add safe code search and patch generation.

Deliverables:

- likely-file locator
- minimal patch generator
- diff preview
- selective patch apply

Exit criterion:

- the system can propose a credible fix for simple UI issues

### Phase 4: Verify-after-fix loop

Close the loop.

Deliverables:

- rerun impacted checks
- compare before and after evidence
- final report

Exit criterion:

- one failed scenario can be fixed and revalidated end to end

### Phase 5: Productization

Only after the loop works locally.

Deliverables:

- CLI polish
- HTML report viewer
- config file
- CI mode
- branch or PR integration

## 16. Recommended First Slice

If you want the fastest path to something impressive, build this exact slice first:

- input: one Markdown PRD for one landing page or signup flow
- target: one local dev server
- browser: Chrome via `chrome-devtools-mcp`
- checks: text, visibility, click navigation, console errors
- fix: only suggest patch, do not auto-apply yet
- output: Markdown report with screenshots and proposed diff

That slice is narrow enough to finish, but already demonstrates the full product thesis.

## 17. My Recommendation

Start with a CLI-first "PRD Guard" style product:

- compile PRD into a structured contract
- run the contract in a real browser
- show evidence-backed failures
- propose minimal frontend fixes
- revalidate after approval in the next iteration

Do not start with a general autonomous agent. Start with a constrained workflow engine powered by LLM tool calling. That gives you a product that is easier to debug, easier to trust, and much easier to demo.

## 18. Assumptions

This design assumes:

- the first users are engineers, not non-technical PMs
- the first environments are local dev or staging
- the first target stack is modern frontend apps such as React, Next.js, or Vite
- the first fixes are safe frontend-only diffs

## 19. One Decision To Make Next

The biggest fork in the road is this:

- local dev assistant first
- staging or CI validation first

My recommendation is local dev assistant first, because it gives you faster iteration, simpler auth handling, and a much shorter path to a compelling demo.
