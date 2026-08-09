# CCAR-F — Short Notes, All 5 Domains
Source: claudecertificationguide.com/learn (30 lessons)

---

## Domain 1 — Agentic Architecture & Orchestration (27%)

- **Loop control:** only `stop_reason` matters. `"tool_use"` → run tools, append results, continue. `"end_turn"` → done. Never use text-parsing, iteration caps as primary control, or content-type checks.
- **Hub-and-spoke is the only tested multi-agent pattern.** All comms flow through the coordinator; subagents never talk directly to each other (exam rule, even though nested subagents exist in real Claude Code).
- **Subagent isolation:** no inherited history, no shared memory between calls. Everything needed goes explicitly in the prompt. If output is bad, check what the coordinator passed in — not the subagent.
- **Narrow Decomposition Failure:** if a report misses whole categories (not just depth), the coordinator's decomposition was too narrow — never blame downstream agents.
- **Task tool** (now called `Agent` in current Claude Code; `Task` still works as alias) spawns subagents. Coordinator's `allowedTools` must include it — hard binary gate.
- **Context passing 3 rules:** pass complete findings explicitly; separate content from metadata (source URL, doc name, page #) or citations get lost; write goal-oriented prompts, not step-by-step scripts.
- **Parallel subagents:** emit multiple Task calls in one response, not across turns.
- **`fork_session`** = divergent branches from a shared baseline (compare approaches). **`--resume`** = continue same line of work. Don't confuse them.
- **Enforcement spectrum:** prompts = probabilistic (~90-95%); hooks/prerequisite gates = deterministic (100%). Money/security/compliance → always hooks. Formatting/style → prompts fine.
- **SubagentStart** = observational only, can't block. **SubagentStop** = can block/send back with `decision: "block"`. Output reshaping = PostToolUse hook on the `Agent` tool call, not SubagentStop.
- **Multi-concern requests:** decompose → investigate in parallel → synthesize one unified response.
- **Handoff to human:** must be self-contained (customer ID, summary, root cause, exact amount, recommended action) — humans have no transcript access.
- **PostToolUse hook** = normalize data after execution (timestamps, status codes). **PreToolUse hook** = block/modify/redirect before execution (the actual mechanism for policy gates). Using PostToolUse to "block" is a classic trap — too late by then.
- **Task decomposition:** fixed pipeline/prompt chaining = predictable structured work. Dynamic/adaptive decomposition = open-ended/unknown-scope work. Don't swap them.
- **Attention Dilution:** uneven quality across a batch (great on early items, weak/missed on later ones) = architectural problem, NOT fixed by bigger model or context window. Fix = multi-pass: per-item passes + a separate cross-item integration pass.
- **Stale Context Problem:** resuming after files changed = old tool results linger and cause contradictory advice. Fix = fresh session + injected summary + list of changed files (targeted re-analysis), not naive resume-and-reread. `fork_session` does NOT fix this (it inherits the same stale history).

---

## Domain 2 — Tool Design & MCP Integration (18%)

- **Tool descriptions are the primary selection mechanism.** Misrouting between 2 similar tools (small toolkit) → fix descriptions first (inputs, examples, boundaries). Few-shot examples, routing classifiers, consolidation are all wrong first answers here.
- **But** if the toolkit is large (~18+ tools) description quality can't save it — that's a tool-overload problem, not a description problem. Optimal: **4-5 tools per agent**, scoped to role.
- **Consolidate near-duplicate tools** (19 similar transforms → 1 `transform_data` with an enum param) vs **split** generic tools into purpose-specific ones (`extract_data_points`, `summarize_content`, `verify_claim`) — different fixes for different symptoms. Moving tools to a second MCP server does NOT reduce toolkit size (client sees one flat list).
- **Structured errors — 4 categories:** transient (retryable), validation (retryable after input fix), business (never retryable — policy violation), permission (never retryable — needs different access). Use `isError` flag; never a plain "Operation failed" string.
- **Access failure vs. valid empty result:** timeout/can't-reach ≠ successfully-queried-and-found-nothing. Conflating them causes wasted retries or missed retries. This is the single most-tested Domain 2 distinction.
- **Error propagation:** local recovery for transient failures first; only propagate what can't be resolved locally; always include partial results + what was attempted.
- **`tool_choice`:** `"auto"` = model may return text. `"any"` = must call a tool, picks which one (for unknown-doc-type extraction). Forced/named = must call that specific tool (enforces a mandatory first step).
- **Scoped cross-role tools:** if 85% of coordinator round-trips are simple lookups, give the requesting agent a constrained version directly (e.g. `verify_fact`) instead of always routing through the coordinator — cuts latency ~40%.
- **Least privilege:** prefer constrained tools (`load_document` validating doc URLs only) over generic ones (`fetch_url`).
- **MCP config scoping:** `.mcp.json` (project root) = version-controlled, team-shared, use `${ENV_VAR}` for secrets. `~/.claude.json` = personal, not shared.
- **MCP Resources** expose data catalogs (schemas, doc hierarchies) so agents don't burn tool calls just discovering what exists.
- **Build vs. use MCP servers:** always evaluate community servers first (Jira, GitHub, Slack, etc.); build custom only for team-specific workflows they can't cover.
- **Sparse MCP tool descriptions lose to built-in tools** (e.g. Grep) in selection — enhance MCP descriptions to compete.
- **Built-in tools:** Grep = searches file *contents*. Glob = matches file *paths* by pattern. Confusing them is a classic trap.
- **Edit vs Read+Write:** try Edit first; on non-unique match, widen the anchor text or use `replace_all: true`; only fall back to Read+Write as a last resort (burns way more context).
- **Incremental codebase exploration:** Grep for entry points → Read to trace → Grep again for usage/wrappers. Never read all files upfront.

---

## Domain 3 — Claude Code Configuration & Workflows (20%)

- **CLAUDE.md 3 levels:** user (`~/.claude/CLAUDE.md`, personal, not shared) < project (`.claude/CLAUDE.md` or root `CLAUDE.md`, shared via git) < directory-level (one subdirectory only). New-teammate-gets-inconsistent-behavior trap = conventions were in user-level, not project-level config.
- **CLAUDE.md files are concatenated, not strictly overriding** — conflicts may resolve arbitrarily ("Claude may pick one arbitrarily"). For anything that MUST hold every time, use `settings.json` (strict precedence, client-enforced) or a hook — not CLAUDE.md.
- **`@path` imports** inline a file's content at load time (no `@import` keyword) — good for modularity, but doesn't reduce what Claude actually sees in context (eager loading).
- **`.claude/rules/`** with `paths:` frontmatter = token-efficient, loads only for matching files — the answer whenever a convention applies to one file type spread across many directories (e.g. all `*.test.tsx` files). Directory-level CLAUDE.md would require duplicating into every directory — wrong answer for that scenario.
- **`/memory`** (exam-guide term) / **`/context`** (current CLI) shows which files loaded — diagnostic only, doesn't trigger loading.
- **Skills = unified system** for custom commands. `.claude/skills/<name>/SKILL.md` (canonical, directory + entrypoint) or `.claude/commands/<name>.md` (flat file alias) — both create `/name`. Project-scoped = shared via git; user-scoped (`~/.claude/skills/`) = personal.
- **Skill frontmatter:** `context: fork` = isolate verbose output in a sub-agent so it doesn't pollute main context. `allowed-tools` = pre-approve tools (not a hard restriction — use `disallowed-tools` for actual removal). `argument-hint` = prompt for missing params.
- **Skills vs CLAUDE.md:** skills = on-demand, task-specific workflows (invoked or auto-matched by description). CLAUDE.md = always-loaded universal standards. Don't put task workflows in CLAUDE.md or always-on conventions in skills.
- **Plan mode vs direct execution:** decision is about *ambiguity*, not difficulty. Plan mode = large-scale/multi-file/multiple-valid-approaches/architectural decisions. Direct execution = well-scoped, single approach already known. Hybrid "plan then execute" is common and tested (e.g., library migration across 30 files).
- **Explore subagent** keeps verbose discovery output out of the main conversation during multi-phase tasks.
- **Iterative refinement technique hierarchy:** concrete input/output examples (fixes inconsistent interpretation) > test-driven iteration via failing tests (fixes complex transformations) > interview pattern (fixes unfamiliar domains — have Claude ask questions first). Don't confuse "more precise prose" with examples — prose still needs interpreting.
- **Batch vs sequential feedback:** batch (one message) when fixes interact; sequential when issues are independent.
- **CI/CD:** `-p` (`--print`) flag is the single most-tested Domain 3 fact — without it, CI hangs waiting for interactive input. `--output-format json` + `--json-schema` for machine-parseable findings (schema-conformant data lands in `structured_output`).
- **Self-review is weaker than independent review** — same session retains generation reasoning and is less likely to challenge itself. Use a separate `claude -p` invocation with no shared context for review.
- **Incremental review:** feed prior findings back in, instruct "report only new/still-unresolved issues" — prevents duplicate PR comments that erode trust.
- **CLAUDE.md feeds CI context too** (testing standards, fixtures, review criteria) — without it, CI-generated tests are generic boilerplate.
- **Batch API vs sync for CI:** pre-merge/blocking checks → synchronous (no latency SLA on batch, up to 24h). Overnight/weekly non-blocking work → Batch API (50% cheaper).

---

## Domain 4 — Prompt Engineering & Structured Output (20%)

- **Explicit categorical criteria beat vague instructions.** "Be conservative"/"only high-confidence findings" give no actionable boundary. Define exactly what to flag vs. skip, with concrete code examples per severity level (not prose).
- **False-positive trust problem:** high FP rate in ONE category destroys trust in ALL categories (developers stop reading even accurate findings). Fix: temporarily disable the noisy category while refining it, rather than degrading trust system-wide.
- **Confidence-based filtering ("only report high-confidence") doesn't fix precision** — LLM self-confidence is poorly calibrated. Explicit criteria first; confidence-based routing is a later/different tool (Domain 5).
- **Few-shot examples are the first-line fix** for: (1) inconsistent formatting despite detailed instructions, (2) inconsistent judgment calls on ambiguous cases, (3) empty/null extraction fields for info that exists but is in an unexpected format (e.g. narrative text vs. table). NOT "add more instructions."
- **Effective examples:** 2-4 targeted at the actual failing scenarios, each showing *reasoning* (why this choice over alternatives) — not just input/output pairs. Reasoning is what lets the model generalize instead of pattern-match narrowly.
- **Structured output reliability hierarchy:** `tool_use` + JSON schema (eliminates syntax errors entirely) > prompt-based "output JSON" (no guarantees).
- **`tool_choice` 3 modes:** `"auto"` (may return text) / `"any"` (must call a tool, model picks which — for unknown doc type) / forced named tool (mandatory specific first step).
- **tool_use eliminates syntax errors, NOT semantic errors** — sum discrepancies, misplaced values, fabrication still happen and need separate validation logic.
- **Schema design against fabrication:** make fields nullable/optional when source may lack the info (required fields pressure fabrication); add "unclear" enum values; "other" + freeform detail string for uncategorized cases; put formatting rules (ISO dates etc.) in the prompt, not just the schema.
- **Retry-with-error-feedback:** send the model (1) original document, (2) failed extraction, (3) the specific validation error. Naive retries without the specific error usually reproduce the same mistake.
- **Retry effectiveness boundary:** retries fix format/structural/misplaced-value/math errors. Retries can NEVER produce information that's genuinely absent from the source — flag those for human review instead, don't keep retrying.
- **Self-correction schema fields:** `calculated_total` vs `stated_total` (auto-flags discrepancy), `conflict_detected` boolean (contradictory source info), `detected_pattern` (tracks which code construct triggered a finding, enables dismissal-rate analysis for prompt tuning).
- **Pydantic / schema validators** turn "validation failed" into specific per-field errors — the third ingredient for retry-with-feedback. `strict: true` tool defs and SDK-level parsing enforce schema server-side, but semantic rules (sums, ordering) still need validators.
- **Batch API facts:** 50% cheaper, up to 24h processing, NO latency SLA, no multi-turn tool calling within a batch request, `custom_id` correlates request/response. Blocking workflows (pre-merge checks) must stay synchronous — this exact temptation ("switch everything to batch for cost savings") is a named exam trap. On failure: identify by `custom_id`, resubmit only failures with targeted fixes (chunking, etc.), never resubmit the whole batch. Refine prompts on a 5-10 doc sample BEFORE full batch submission.
- **Self-review in the same session is weaker** (retains generation reasoning, less likely to challenge itself) — use an independent instance for review.
- **Attention dilution in reviews:** same failure/fix as Domain 1 — split into per-file local passes + a cross-file integration pass. A bigger context window does NOT fix it (it's an attention-quality problem, not a capacity problem).
- **Confidence-based routing:** raw self-reported confidence is uncalibrated — must be calibrated against labeled validation sets before using thresholds to route findings to human review.

---

## Domain 5 — Context Management & Reliability (15%)

- **Progressive summarisation trap:** default summarisation destroys transactional specifics (amounts, order #s, dates) — "refund of $247.83 for order #8891" becomes "a recent order." Fix: **persistent case-facts block**, included in every prompt, kept OUTSIDE the summarized history, never itself summarized. This is the single most important Domain 5 pattern.
- **"Lost in the middle":** models weight the start/end of long inputs more than the middle. Fix is structural — put a "Key Findings Summary" at the top of aggregated inputs, use explicit section headers — not a prompt reminder to "pay attention to everything."
- **Tool result trimming:** a 40-field tool response should be trimmed to the 5 relevant fields before it ever enters context (via PostToolUse hook or in the tool itself) — untrimmed results silently drown multi-turn context over time.
- **API is stateless** — full conversation history must be resent every request; you can't selectively truncate without losing coherence (case-facts block is the alternative to truncation).
- **Prompt caching:** put stable content (system instructions, tool defs, reference docs) first with a `cache_control` breakpoint at the end of that static block; put the volatile/per-request content after it. Wrong order = cache never hits. `ephemeral` breakpoints last ~5 min since last use — good for bursts, not hours-apart reuse.
- **3 valid escalation triggers:** (1) explicit human request — honor IMMEDIATELY, no "let me try first"; (2) policy gap (not the same as policy violation — a gap has no documented answer, a violation does); (3) genuine inability to progress after an actual attempt.
- **2 unreliable escalation triggers (named anti-patterns):** sentiment/frustration-based (frustration ≠ complexity), self-reported confidence scores (poorly calibrated, confidently wrong on hard cases).
- **Frustration nuance:** frustrated + resolvable issue → acknowledge + resolve, don't escalate. Customer reiterates wanting a human after being offered help → now escalate.
- **Ambiguous customer match** (multiple records found): must ask for a disambiguating identifier. NEVER pick by heuristic (most recent/most active) — privacy/wrong-account risk.
- **Structured error context (4 elements):** failure type (transient/validation/business/permission), what was attempted, partial results gathered, alternative approaches — gives the coordinator what it needs to decide next.
- **2 anti-patterns:** silent suppression (returning empty-as-success on a real failure — the worst one, invisible) and workflow termination (killing the whole pipeline over one subagent failure, discarding completed work from others).
- **Access failure vs. valid empty result** — same critical distinction as Domain 2, retested here for multi-agent error propagation specifically.
- **Coverage annotations:** synthesis output should explicitly note gaps ("limited due to unavailable source X") rather than silently omitting a topic — a silent gap looks like "not relevant" instead of "data unavailable."
- **Context degradation ≠ token limit problem.** Symptom: agent starts saying "typical pattern" instead of citing the specific class/file it found earlier. A bigger context window does NOT fix it — it's an attention/recency problem, not a capacity problem.
- **Mitigations:** scratchpad files (persist specific findings outside conversation, used from the start of a session, not just as a rescue); subagent delegation (primary benefit is context ISOLATION, not just parallelism); summary injection between exploration phases (prevents "cold start" duplication); `/compact` used proactively, not just at the limit; structured state manifests for crash recovery (explored paths, key findings, next steps) so a crash doesn't lose all progress.
- **Aggregate metrics trap:** 97% overall accuracy can hide 40-60% accuracy on specific document types/fields (e.g. handwritten receipts). Always validate by document type AND field segment before automating — never decide from the aggregate alone.
- **Stratified random sampling must include high-confidence (automated) extractions**, not just low-confidence ones — otherwise a novel error pattern in the "safe" automated bucket goes undetected.
- **Confidence calibration:** raw model confidence isn't absolute — 0.90 might mean 94% actual accuracy on one field type and 82% on another. Calibrate against labeled ground-truth sets before setting routing thresholds.
- **Reviewer capacity:** route by dynamically-ranked uncertainty (lowest calibrated confidence / most ambiguous source first), never spread evenly or by chronological order.
- **Claim-source mappings** (claim + source URL + doc name + excerpt + publication date) must be explicitly preserved and merged at every synthesis step — attribution dies during paraphrasing/compression by default, especially at the synthesis step.
- **Conflicting sources:** annotate BOTH values with attribution — never average, never auto-pick "most recent," never silently choose one. Always include publication/data-collection dates — different dates often explain "conflicting" numbers as a trend, not a contradiction.
- **Content-appropriate rendering:** financial data → tables; news/narrative → prose; technical findings → structured lists. Don't flatten everything into one format.

---

*All content sourced from claudecertificationguide.com (independent, unofficial study resource), accessed August 2026.*
