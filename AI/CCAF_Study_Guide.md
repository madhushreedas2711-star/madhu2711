# Claude Certified Architect – Foundations: Complete Study Reference

> A topic-by-topic study guide covering every domain, task statement, knowledge point, and concept from the official exam guide. Use this as your master reference and self-quiz sheet.

## Exam at a glance

- **Format:** Multiple choice, one correct answer + three distractors. No penalty for guessing.
- **Scoring:** Scaled 100–1,000. **Passing score: 720.** Pass/fail.
- **Structure:** Scenario-based. 4 of 6 scenarios appear, picked at random.
- **Target candidate:** Solution architect with 6+ months building on Claude API, Agent SDK, Claude Code, and MCP.

### Domain weightings

| Domain | Topic | Weight |
|--------|-------|--------|
| 1 | Agentic Architecture & Orchestration | 27% |
| 2 | Tool Design & MCP Integration | 18% |
| 3 | Claude Code Configuration & Workflows | 20% |
| 4 | Prompt Engineering & Structured Output | 20% |
| 5 | Context Management & Reliability | 15% |

### The 6 scenarios

1. **Customer Support Resolution Agent** — Agent SDK, MCP tools (`get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`), 80%+ first-contact resolution target. *(Domains 1, 2, 5)*
2. **Code Generation with Claude Code** — slash commands, CLAUDE.md, plan mode vs direct execution. *(Domains 3, 5)*
3. **Multi-Agent Research System** — coordinator + specialized subagents (search, analyze, synthesize, report). *(Domains 1, 2, 5)*
4. **Developer Productivity with Claude** — explore codebases, built-in tools (Read, Write, Bash, Grep, Glob), MCP servers. *(Domains 2, 3, 1)*
5. **Claude Code for CI/CD** — automated reviews, test generation, PR feedback, minimize false positives. *(Domains 3, 4)*
6. **Structured Data Extraction** — extract from unstructured docs, validate with JSON schemas, handle edge cases. *(Domains 4, 5)*

---

# Domain 1: Agentic Architecture & Orchestration (27%)

## 1.1 — Agentic loops for autonomous task execution

**Core concept:** The agentic loop sends a request to Claude, inspects `stop_reason`, executes any requested tools, appends results to conversation history, and iterates.

- Continue the loop when `stop_reason` is `"tool_use"`.
- Terminate when `stop_reason` is `"end_turn"`.
- Tool results are appended to conversation history so the model can reason about the next action.
- **Model-driven decision-making** (Claude reasons about which tool to call next from context) is distinct from **pre-configured decision trees** or fixed tool sequences.

**Anti-patterns to avoid:**
- Parsing natural-language signals to decide when to stop the loop.
- Using an arbitrary iteration cap as the *primary* stopping mechanism.
- Checking for assistant text content as a completion indicator.

> **Quiz yourself:** What is the *correct* signal that the loop should terminate? → `stop_reason == "end_turn"`.

## 1.2 — Multi-agent orchestration (coordinator-subagent)

- **Hub-and-spoke architecture:** the coordinator manages ALL inter-subagent communication, error handling, and information routing.
- Subagents have **isolated context** — they do NOT inherit the coordinator's conversation history automatically.
- Coordinator responsibilities: task decomposition, delegation, result aggregation, and deciding which subagents to invoke based on query complexity.
- **Risk:** overly narrow task decomposition leads to incomplete coverage of broad topics. *(This is the root cause in Sample Q7.)*

**Skills:**
- Coordinators should *dynamically* select which subagents to invoke, not always route through the full pipeline.
- Partition research scope across subagents to minimize duplication (distinct subtopics or source types per agent).
- Implement **iterative refinement loops**: coordinator evaluates synthesis output for gaps, re-delegates targeted queries, re-invokes synthesis until coverage is sufficient.
- Route all subagent communication through the coordinator for observability and controlled information flow.

## 1.3 — Subagent invocation, context passing, spawning

- The **`Task` tool** spawns subagents. The coordinator's `allowedTools` **must include `"Task"`**.
- Subagent context must be **explicitly provided in the prompt** — no automatic inheritance, no shared memory between invocations.
- **`AgentDefinition`** configures each subagent: descriptions, system prompts, tool restrictions.
- **Fork-based session management** explores divergent approaches from a shared analysis baseline.

**Skills:**
- Pass complete findings from prior agents directly in the subagent's prompt (e.g., feed search results + document analysis to the synthesis subagent).
- Use structured data formats to separate content from metadata (source URLs, doc names, page numbers) to preserve attribution.
- Spawn **parallel** subagents by emitting multiple `Task` calls in a *single* coordinator response (not across separate turns).
- Write coordinator prompts that specify research **goals and quality criteria**, not step-by-step procedures — this enables subagent adaptability.

## 1.4 — Multi-step workflows: enforcement and handoff

- **Programmatic enforcement** (hooks, prerequisite gates) vs **prompt-based guidance** for workflow ordering.
- When deterministic compliance is required (e.g., identity verification before financial operations), **prompt instructions alone have a non-zero failure rate.**
- Structured handoff protocols for mid-process escalation include customer details, root-cause analysis, and recommended actions.

**Skills:**
- Implement programmatic prerequisites that block downstream calls until prerequisites complete (e.g., block `process_refund` until `get_customer` returns a verified ID). *(This is Sample Q1's answer.)*
- Decompose multi-concern requests into distinct items, investigate each in parallel with shared context, then synthesize a unified resolution.
- Compile structured handoff summaries (customer ID, root cause, refund amount, recommended action) for human agents who lack the conversation transcript.

## 1.5 — Agent SDK hooks for interception and normalization

- **`PostToolUse` hooks** intercept tool *results* for transformation before the model processes them.
- Other hooks intercept outgoing tool *calls* to enforce compliance rules (e.g., blocking refunds above a threshold).
- Hooks give **deterministic guarantees**; prompts give **probabilistic compliance**.

**Skills:**
- Use `PostToolUse` to normalize heterogeneous formats (Unix timestamps, ISO 8601, numeric status codes) from different MCP tools.
- Use tool-call interception hooks to block policy-violating actions (e.g., refunds > $500) and redirect to human escalation.
- Choose hooks over prompts whenever business rules require guaranteed compliance.

## 1.6 — Task decomposition strategies

- **Fixed sequential pipelines (prompt chaining):** predictable, multi-aspect reviews. E.g., analyze each file individually, then a cross-file integration pass.
- **Dynamic adaptive decomposition:** open-ended investigation; generates subtasks based on what's discovered at each step.

**Skills:**
- Match the pattern to the workflow: chaining for predictable reviews, dynamic for open-ended investigation.
- Split large code reviews into per-file local passes + a separate cross-file integration pass (avoids **attention dilution**).
- For open-ended tasks ("add comprehensive tests to a legacy codebase"): map structure first, identify high-impact areas, create a prioritized plan that adapts as dependencies surface.

## 1.7 — Session state, resumption, forking

- **`--resume <session-name>`** continues a specific prior conversation.
- **`fork_session`** creates independent branches from a shared baseline for divergent approaches.
- Inform the agent about changes to previously analyzed files when resuming after code modifications.
- Starting a **new session with a structured summary** is more reliable than resuming with **stale tool results**.

**Skills:**
- Use `--resume` with names to continue investigations across work sessions.
- Use `fork_session` to compare two strategies (e.g., two testing or refactoring approaches) from a shared analysis.
- Choose resumption when prior context is mostly valid; choose fresh-start-with-summary when prior tool results are stale.
- Inform a resumed session about specific file changes for targeted re-analysis instead of full re-exploration.

---

# Domain 2: Tool Design & MCP Integration (18%)

## 2.1 — Effective tool interfaces

- **Tool descriptions are the primary mechanism LLMs use for tool selection.** Minimal descriptions cause unreliable selection among similar tools. *(This is Sample Q2's answer — the highest-leverage first fix.)*
- Good descriptions include: input formats, example queries, edge cases, and boundary explanations (when to use this vs. similar tools).
- Ambiguous/overlapping descriptions cause misrouting (e.g., `analyze_content` vs `analyze_document`).
- System prompt wording affects selection — keyword-sensitive instructions can create unintended tool associations.

**Skills:**
- Write descriptions that differentiate purpose, inputs, outputs, and "when to use vs. alternatives."
- Rename tools to eliminate overlap (e.g., `analyze_content` → `extract_web_results`).
- Split generic tools into purpose-specific ones (e.g., `analyze_document` → `extract_data_points`, `summarize_content`, `verify_claim_against_source`).
- Review system prompts for keyword-sensitive instructions that override good descriptions.

## 2.2 — Structured error responses for MCP tools

- The MCP **`isError`** flag communicates tool failures back to the agent.
- Error types: **transient** (timeouts, unavailability), **validation** (invalid input), **business** (policy violations), **permission**.
- **Uniform/generic errors** ("Operation failed") prevent appropriate recovery decisions.
- Returning structured metadata prevents wasted retries (retryable vs non-retryable).

**Skills:**
- Return `errorCategory` (transient/validation/permission), `isRetryable` boolean, human-readable descriptions.
- Include `retriable: false` + customer-friendly explanations for business-rule violations.
- Implement local recovery within subagents for transient failures; propagate to coordinator only what can't be resolved locally, with partial results and what was attempted.
- Distinguish **access failures** (need retry decisions) from **valid empty results** (successful query, no matches).

## 2.3 — Tool distribution and tool choice

- Too many tools (e.g., 18 instead of 4-5) **degrades selection reliability** by increasing decision complexity.
- Agents with out-of-specialization tools misuse them (e.g., a synthesis agent attempting web searches).
- **Scoped tool access:** give agents only role-relevant tools, with limited cross-role tools for high-frequency needs.
- **`tool_choice`** options: `"auto"`, `"any"`, and forced (`{"type": "tool", "name": "..."}`).

**Skills:**
- Restrict each subagent's tool set to its role.
- Replace generic tools with constrained alternatives (e.g., `fetch_url` → `load_document` that validates URLs).
- Provide scoped cross-role tools for high-frequency needs (e.g., `verify_fact` for the synthesis agent). *(Sample Q9's answer — principle of least privilege.)*
- Use forced `tool_choice` to ensure a specific tool runs first (e.g., `extract_metadata` before enrichment).
- Set `tool_choice: "any"` to guarantee a tool call rather than conversational text.

## 2.4 — Integrating MCP servers

- **Scoping:** project-level **`.mcp.json`** (shared team tooling) vs user-level **`~/.claude.json`** (personal/experimental).
- **Environment variable expansion** in `.mcp.json` (e.g., `${GITHUB_TOKEN}`) keeps secrets out of version control.
- All configured MCP servers' tools are discovered at connection time and available simultaneously.
- **MCP resources** expose content catalogs (issue summaries, doc hierarchies, DB schemas) to reduce exploratory tool calls.

**Skills:**
- Configure shared servers in `.mcp.json` with env-var expansion.
- Configure personal/experimental servers in `~/.claude.json`.
- Enhance MCP tool descriptions so the agent doesn't prefer built-in tools (like Grep) over more capable MCP tools.
- Prefer existing community MCP servers (e.g., Jira) over custom builds; reserve custom for team-specific workflows.
- Expose content catalogs as resources to give agents data visibility without exploratory calls.

## 2.5 — Built-in tools (Read, Write, Edit, Bash, Grep, Glob)

- **Grep:** content search (function names, error messages, imports).
- **Glob:** file *path* pattern matching (by name/extension, e.g., `**/*.test.tsx`).
- **Read/Write:** full file operations. **Edit:** targeted modifications via unique text matching.
- When **Edit fails** (non-unique text), fall back to **Read + Write**.

**Skills:**
- Grep for content across a codebase (find all callers, locate error messages).
- Glob for naming-pattern matches.
- Read → Write fallback when Edit can't find unique anchor text.
- Build codebase understanding incrementally: Grep for entry points → Read to follow imports and trace flows (not reading all files upfront).
- Trace function usage across wrappers: identify all exported names, then search each across the codebase.

---

# Domain 3: Claude Code Configuration & Workflows (20%)

## 3.1 — CLAUDE.md hierarchy, scoping, modular organization

**Hierarchy (precedence/scope):**
- **User-level:** `~/.claude/CLAUDE.md` — applies only to that user; **NOT shared via version control.**
- **Project-level:** `.claude/CLAUDE.md` or root `CLAUDE.md` — shared with the team.
- **Directory-level:** subdirectory `CLAUDE.md` files.

- **`@import`** syntax references external files to keep CLAUDE.md modular.
- **`.claude/rules/`** organizes topic-specific rule files as an alternative to a monolithic CLAUDE.md.

**Skills:**
- Diagnose hierarchy issues (e.g., a teammate not getting instructions because they're user-level, not project-level).
- Use `@import` to selectively include relevant standards per package.
- Split large CLAUDE.md into focused files in `.claude/rules/` (testing.md, api-conventions.md, deployment.md).
- Use the **`/memory`** command to verify which memory files are loaded.

## 3.2 — Custom slash commands and skills

- **Slash commands:** project-scoped **`.claude/commands/`** (shared via version control) vs user-scoped **`~/.claude/commands/`** (personal). *(Sample Q4: team-wide `/review` → `.claude/commands/`.)*
- **Skills:** **`.claude/skills/`** with **`SKILL.md`** files supporting frontmatter:
  - **`context: fork`** — run the skill in an isolated sub-agent context so its output doesn't pollute the main conversation.
  - **`allowed-tools`** — restrict tool access during skill execution.
  - **`argument-hint`** — prompt for required parameters when invoked without args.
- Personal skill customization: variants in `~/.claude/skills/` with different names.

**Skills:**
- Create project-scoped commands in `.claude/commands/` for team-wide availability.
- Use `context: fork` to isolate verbose output (codebase analysis) or exploratory context (brainstorming).
- Configure `allowed-tools` to restrict access (e.g., limit to file writes to prevent destructive actions).
- Use `argument-hint` to prompt for parameters.
- **Skills vs CLAUDE.md:** skills = on-demand, task-specific invocation; CLAUDE.md = always-loaded universal standards.

## 3.3 — Path-specific rules for conditional convention loading

- **`.claude/rules/`** files with YAML frontmatter **`paths`** fields containing glob patterns for conditional activation.
- Path-scoped rules load **only when editing matching files**, reducing irrelevant context and token usage.
- Advantage over directory-level CLAUDE.md: glob patterns apply to files **spread across directories** (e.g., test files throughout a codebase). *(Sample Q6's answer.)*

**Skills:**
- Create rules with frontmatter path scoping (e.g., `paths: ["terraform/**/*"]`).
- Use globs to apply conventions by file type regardless of location (e.g., `**/*.test.tsx`).
- Choose path-specific rules over subdirectory CLAUDE.md when conventions span the codebase.

## 3.4 — Plan mode vs direct execution

- **Plan mode:** complex tasks — large-scale changes, multiple valid approaches, architectural decisions, multi-file modifications. Enables safe exploration/design before committing, preventing costly rework. *(Sample Q5: monolith → microservices = plan mode.)*
- **Direct execution:** simple, well-scoped changes (single validation check, single-file bug fix with clear stack trace).
- **Explore subagent:** isolates verbose discovery output, returns summaries to preserve main conversation context.

**Skills:**
- Plan mode for architectural implications (microservice restructuring, library migrations affecting 45+ files).
- Direct execution for clear-scope changes.
- Explore subagent for verbose discovery phases (prevents context exhaustion).
- Combine: plan mode for investigation + direct execution for implementation.

## 3.5 — Iterative refinement techniques

- **Concrete input/output examples** are the most effective way to communicate transformations when prose is interpreted inconsistently.
- **Test-driven iteration:** write tests first, iterate by sharing test failures.
- **Interview pattern:** have Claude ask questions to surface unanticipated considerations before implementing.
- **Single message** for interacting problems; **sequential** for independent problems.

**Skills:**
- Provide 2-3 input/output examples to clarify requirements.
- Write test suites (behavior, edge cases, performance) before implementation, iterate on failures.
- Use the interview pattern for unfamiliar domains (cache invalidation strategies, failure modes).
- Provide specific test cases for edge cases (e.g., null values in migration scripts).
- Bundle interacting fixes; sequence independent ones.

## 3.6 — Claude Code in CI/CD pipelines

- **`-p` / `--print`** flag: non-interactive mode for automated pipelines. *(Sample Q10: fixes the hang.)*
- **`--output-format json`** and **`--json-schema`** for structured CI output.
- CLAUDE.md provides project context (testing standards, fixtures, review criteria) to CI-invoked Claude Code.
- **Session isolation:** the same session that generated code is less effective at reviewing its own changes than an independent review instance.

**Skills:**
- Run with `-p` to prevent interactive hangs.
- Use `--output-format json` + `--json-schema` for machine-parseable findings posted as inline PR comments.
- Include prior review findings when re-running after new commits; instruct Claude to report only new/unaddressed issues (avoid duplicate comments).
- Provide existing test files so generation avoids duplicate scenarios.
- Document testing standards/criteria/fixtures in CLAUDE.md to improve test quality.

---

# Domain 4: Prompt Engineering & Structured Output (20%)

## 4.1 — Explicit criteria to improve precision / reduce false positives

- **Explicit criteria beat vague instructions.** "Flag comments only when claimed behavior contradicts actual code behavior" >> "check that comments are accurate."
- Vague instructions like "be conservative" or "only report high-confidence findings" **fail** to improve precision vs. specific categorical criteria.
- High false-positive categories undermine developer trust in accurate categories.

**Skills:**
- Write specific review criteria defining what to report (bugs, security) vs skip (minor style, local patterns) — not confidence-based filtering.
- Temporarily disable high-false-positive categories to restore trust while improving prompts.
- Define explicit severity criteria with concrete code examples per level.

## 4.2 — Few-shot prompting for consistency and quality

- Few-shot examples are the **most effective** technique for consistent, actionable output when instructions alone are inconsistent.
- They demonstrate ambiguous-case handling and enable **generalization to novel patterns** (not just matching pre-specified cases).
- Effective for reducing hallucination in extraction (informal measurements, varied structures).

**Skills:**
- Create 2-4 targeted examples for ambiguous scenarios showing *reasoning* for why one action beat alternatives.
- Demonstrate the desired output format (location, issue, severity, suggested fix).
- Distinguish acceptable patterns from genuine issues to cut false positives while enabling generalization.
- Show varied document structures (inline citations vs bibliographies, methodology vs embedded details).
- Add examples for varied formats to fix empty/null extraction of required fields.

## 4.3 — Structured output via tool use and JSON schemas

- **Tool use (`tool_use`) with JSON schemas** is the most reliable approach for schema-compliant output; **eliminates JSON syntax errors.**
- **`tool_choice` distinctions:**
  - `"auto"` — model *may* return text instead of calling a tool.
  - `"any"` — model *must* call a tool, but can choose which.
  - forced — model *must* call a *specific named* tool.
- Strict schemas eliminate **syntax** errors but **not semantic** errors (line items not summing, values in wrong fields).
- Schema design: required vs optional fields; enum + `"other"` + detail string for extensible categories.

**Skills:**
- Define extraction tools with JSON-schema inputs; extract from the `tool_use` response.
- `tool_choice: "any"` to guarantee output when multiple schemas exist and doc type is unknown.
- Force a tool (`{"type": "tool", "name": "extract_metadata"}`) to run a particular extraction first.
- Make fields **optional/nullable** when source may lack info — prevents fabrication to satisfy required fields.
- Add `"unclear"` enums for ambiguous cases; `"other"` + detail for extensible categorization.
- Include format-normalization rules in prompts alongside strict schemas.

## 4.4 — Validation, retry, and feedback loops

- **Retry-with-error-feedback:** append specific validation errors to the prompt on retry.
- **Limit of retry:** ineffective when the required info is simply **absent** from the source (vs. format/structural errors which retry can fix).
- **Feedback loops:** track which constructs trigger findings (`detected_pattern` field) to analyze dismissal patterns.
- **Semantic** validation errors (don't sum, wrong field) vs **syntax** errors (eliminated by tool use).

**Skills:**
- Send follow-ups with the original doc + failed extraction + specific errors for self-correction.
- Identify when retries won't help (info exists only in an unprovided external doc) vs will (format mismatches).
- Add `detected_pattern` fields to analyze false-positive patterns.
- Design self-correction flows: extract `calculated_total` alongside `stated_total` to flag discrepancies; add `conflict_detected` booleans.

## 4.5 — Batch processing strategies

- **Message Batches API:** **50% cost savings**, up to **24-hour** processing window, **no guaranteed latency SLA.**
- Appropriate for **non-blocking, latency-tolerant** workloads (overnight reports, weekly audits, nightly tests). **Inappropriate for blocking workflows** (pre-merge checks). *(Sample Q11: batch the overnight report, keep real-time for pre-merge.)*
- Batch API **does NOT support multi-turn tool calling** within a single request.
- **`custom_id`** correlates request/response pairs.

**Skills:**
- Match API to latency: synchronous for blocking pre-merge; batch for overnight/weekly.
- Calculate submission frequency from SLA (e.g., 4-hour windows to guarantee a 30-hour SLA with 24-hour processing).
- Handle failures: resubmit only failed docs (by `custom_id`) with fixes (chunk oversized docs).
- Refine prompts on a sample set before batching large volumes.

## 4.6 — Multi-instance and multi-pass review architectures

- **Self-review limitation:** a model retains reasoning context from generation, so it's less likely to question its own decisions in the same session.
- **Independent review instances** (no prior reasoning context) catch subtle issues better than self-review instructions or extended thinking.
- **Multi-pass review:** per-file local passes + cross-file integration passes avoid attention dilution and contradictory findings. *(Sample Q12's answer.)*

**Skills:**
- Use a second independent Claude instance to review generated code.
- Split large reviews into per-file passes + integration passes.
- Run verification passes where the model self-reports confidence per finding for calibrated routing.

---

# Domain 5: Context Management & Reliability (15%)

## 5.1 — Preserving critical information across long interactions

- **Progressive summarization risk:** condensing numbers, percentages, dates, and customer-stated expectations into vague summaries.
- **"Lost in the middle":** models reliably process the **beginning and end** of long inputs but may omit **middle** sections.
- Tool results accumulate and consume tokens disproportionately (40+ fields per order lookup when 5 matter).
- Pass complete conversation history in subsequent requests to maintain coherence.

**Skills:**
- Extract transactional facts (amounts, dates, order numbers, statuses) into a persistent **"case facts"** block in each prompt, outside summarized history.
- Persist structured issue data into a separate context layer for multi-issue sessions.
- Trim verbose tool outputs to relevant fields before they accumulate.
- Place key findings at the **beginning** of aggregated inputs; use explicit section headers (mitigates position effects).
- Require subagents to include metadata (dates, source locations, methodology) for accurate synthesis.
- Have upstream agents return structured data (key facts, citations, relevance scores) instead of verbose reasoning chains when downstream budgets are limited.

## 5.2 — Escalation and ambiguity resolution

**Appropriate escalation triggers:** customer requests a human, policy exceptions/gaps (not just complex cases), inability to make meaningful progress.

- Escalate **immediately** when a customer explicitly demands it; otherwise *offer* to resolve when the issue is straightforward.
- **Sentiment-based escalation and self-reported confidence scores are UNRELIABLE proxies** for case complexity. *(Sample Q3: explicit criteria + few-shot beats both confidence scoring and sentiment analysis.)*
- Multiple customer matches → request additional identifiers, don't use heuristics.

**Skills:**
- Add explicit escalation criteria with few-shot examples to the system prompt.
- Honor explicit human-agent requests immediately, without first investigating.
- Acknowledge frustration while offering resolution if within capability; escalate only if the customer reiterates.
- Escalate when policy is ambiguous/silent (e.g., competitor price matching when policy only covers own-site).
- Ask for more identifiers on multiple matches.

## 5.3 — Error propagation across multi-agent systems

- **Structured error context** (failure type, attempted query, partial results, alternatives) enables intelligent coordinator recovery. *(Sample Q8's answer.)*
- Distinguish **access failures** (timeouts → retry decisions) from **valid empty results** (no matches).
- **Generic statuses** ("search unavailable") hide valuable context.
- **Anti-patterns:** silently suppressing errors (empty results as success) AND terminating the whole workflow on a single failure.

**Skills:**
- Return structured context (type, attempted action, partial results, alternatives).
- Distinguish access failures from valid empty results.
- Subagents do local recovery for transient failures; propagate only unresolvable errors with what was attempted + partial results.
- Structure synthesis output with **coverage annotations** (which findings are well-supported vs which areas have gaps from unavailable sources).

## 5.4 — Context management in large codebase exploration

- **Context degradation in extended sessions:** models give inconsistent answers and reference "typical patterns" instead of specific classes discovered earlier.
- **Scratchpad files** persist key findings across context boundaries.
- **Subagent delegation** isolates verbose exploration while the main agent coordinates high-level understanding.
- **Structured state persistence** for crash recovery: each agent exports state to a known location; coordinator loads a manifest on resume.

**Skills:**
- Spawn subagents for specific questions ("find all test files", "trace refund flow dependencies").
- Maintain scratchpad files; reference them to counteract degradation.
- Summarize findings from one phase before spawning next-phase subagents; inject summaries into initial context.
- Design crash recovery with structured state exports (manifests).
- Use **`/compact`** to reduce context usage during extended exploration.

## 5.5 — Human review workflows and confidence calibration

- **Aggregate accuracy (97% overall) can mask poor performance** on specific document types or fields.
- **Stratified random sampling** measures error rates in high-confidence extractions and detects novel patterns.
- **Field-level confidence scores** calibrated with labeled validation sets route review attention.
- Validate accuracy **by document type and field** before automating high-confidence extractions.

**Skills:**
- Stratified random sampling of high-confidence extractions for ongoing error measurement.
- Analyze accuracy by doc type and field before reducing human review.
- Output field-level confidence; calibrate thresholds with labeled validation sets.
- Route low-confidence or ambiguous/contradictory source docs to human review.

## 5.6 — Information provenance and uncertainty in multi-source synthesis

- Source attribution is lost during summarization when claim-source mappings aren't preserved.
- **Structured claim-source mappings** must be preserved and merged through synthesis.
- **Conflicting statistics:** annotate conflicts with source attribution, don't arbitrarily pick one value.
- **Temporal data:** require publication/collection dates so temporal differences aren't misread as contradictions.

**Skills:**
- Require subagents to output claim-source mappings (URLs, doc names, excerpts) preserved through synthesis.
- Structure reports to distinguish well-established from contested findings; preserve source characterizations + methodology.
- Complete doc analysis with conflicting values explicitly annotated; let the coordinator reconcile.
- Require publication/collection dates for correct temporal interpretation.
- Render content types appropriately (financial → tables, news → prose, technical → structured lists), not a uniform format.

---

# The 12 Sample Questions — answer key & reasoning

| # | Scenario | Answer | Core principle |
|---|----------|--------|----------------|
| 1 | Support agent skips `get_customer` | **A** | Programmatic prerequisite gate > prompt/few-shot for critical business logic with financial consequences. D fixes availability, not ordering. |
| 2 | Wrong tool chosen (minimal descriptions) | **B** | Tool descriptions are the primary selection mechanism — expand them first (low-effort, high-leverage). |
| 3 | 55% resolution, bad escalation calibration | **A** | Explicit criteria + few-shot fixes unclear boundaries. Confidence (B) is poorly calibrated; sentiment (D) ≠ complexity; classifier (C) over-engineered. |
| 4 | Team-wide `/review` command location | **A** | `.claude/commands/` is version-controlled and team-shared. `~/.claude/commands/` is personal. |
| 5 | Monolith → microservices approach | **A** | Plan mode: large-scale, architectural, multi-file. Complexity is known upfront. |
| 6 | Conventions for files spread across dirs | **A** | `.claude/rules/` with glob frontmatter auto-applies by path; CLAUDE.md is directory-bound. |
| 7 | Reports miss music/writing/film | **B** | Coordinator's task decomposition was too narrow. Subagents worked correctly within scope. |
| 8 | Web search subagent timeout | **A** | Structured error context enables intelligent recovery. Generic/suppressed/terminating = anti-patterns. |
| 9 | Synthesis agent verification overhead | **A** | Scoped `verify_fact` for the 85% simple case; coordinator for complex 15% (least privilege). |
| 10 | Pipeline hangs on interactive input | **A** | `-p` / `--print` is the documented non-interactive flag. Others are non-existent or workarounds. |
| 11 | Reduce cost: pre-merge + overnight | **A** | Batch the latency-tolerant overnight report; keep real-time for the blocking pre-merge check. |
| 12 | 14-file review, inconsistent results | **A** | Split into per-file + integration passes (fixes attention dilution). Bigger context ≠ better attention. |

**Distractor pattern to internalize:** The wrong answers consistently propose *over-engineered* solutions (custom classifiers, routing layers), *probabilistic* fixes where *deterministic* is needed (prompts instead of hooks/gates), or solutions to a *different problem* than the one stated.

---

# Appendix: technology cheat-sheet (memorize these)

### Claude Agent SDK
Agent definitions · agentic loops · `stop_reason` handling · hooks (`PostToolUse`, tool-call interception) · subagent spawning via `Task` tool · `allowedTools` config.

### Model Context Protocol (MCP)
MCP servers · tools · resources · `isError` flag · tool descriptions · tool distribution · `.mcp.json` config · environment variable expansion.

### Claude Code
CLAUDE.md hierarchy (user/project/directory) · `.claude/rules/` with YAML frontmatter path-scoping · `.claude/commands/` for slash commands · `.claude/skills/` with SKILL.md frontmatter (`context: fork`, `allowed-tools`, `argument-hint`) · plan mode · direct execution · `/memory` · `/compact` · `--resume` · `fork_session` · Explore subagent.

### Claude Code CLI
`-p` / `--print` (non-interactive) · `--output-format json` · `--json-schema` (structured CI output).

### Claude API
`tool_use` with JSON schemas · `tool_choice` (`"auto"`, `"any"`, forced) · `stop_reason` (`"tool_use"`, `"end_turn"`) · `max_tokens` · system prompts.

### Message Batches API
50% cost savings · up to 24h window · `custom_id` for correlation · polling for completion · **no multi-turn tool calling.**

### JSON Schema / Pydantic
Required vs optional · enum types · nullable fields · `"other"` + detail · strict mode (syntax-error elimination) · Pydantic validation · semantic validation errors · validation-retry loops.

### Other
Built-in tools (Read, Write, Edit, Bash, Grep, Glob) · few-shot prompting · prompt chaining · context management (token budgets, progressive summarization, lost-in-the-middle, scratchpad files) · session management · confidence scoring (field-level, calibration, stratified sampling).

---

# Out-of-scope (do NOT study these)

Fine-tuning/training · API auth, billing, account management · deep language/framework implementation · deploying/hosting MCP servers (infra/networking) · Claude's internal architecture/training · Constitutional AI, RLHF, safety training · embeddings/vector DBs · computer use (browser/desktop) · vision/image analysis · streaming API/SSE · rate limits/quotas/pricing math · OAuth/key rotation · cloud provider configs (AWS/GCP/Azure) · benchmarking/model comparison · prompt caching internals · tokenization specifics.

---

# Final self-test checklist

Before the exam, confirm you can answer each from memory:

- [ ] What `stop_reason` values control the agentic loop, and what are the three loop anti-patterns?
- [ ] Why do subagents need explicit context passing, and what tool spawns them?
- [ ] When do you use a programmatic hook/gate instead of a prompt instruction?
- [ ] What is the single highest-leverage fix for poor tool selection?
- [ ] What goes in `errorCategory`, and how do access failures differ from empty results?
- [ ] What are the three `tool_choice` modes and when to use each?
- [ ] Where do project vs personal slash commands and MCP servers live?
- [ ] When is plan mode required vs direct execution?
- [ ] Why does `.claude/rules/` with globs beat directory CLAUDE.md for scattered files?
- [ ] What flag runs Claude Code non-interactively in CI?
- [ ] When is the Batch API appropriate vs inappropriate? What can't it do?
- [ ] Why does an independent review instance beat self-review?
- [ ] What is "lost in the middle" and how do you mitigate it?
- [ ] Why are sentiment and self-confidence unreliable for escalation?
- [ ] How do you preserve provenance and handle conflicting sources in synthesis?

*Version of this guide based on exam guide v0.1 (Feb 10 2025). Cross-reference with current Anthropic docs at docs.anthropic.com as products evolve.*
