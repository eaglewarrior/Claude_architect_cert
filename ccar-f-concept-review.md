# CCAR-F Concept Review

A domain-by-domain walkthrough of the underlying mental models the practice exam is testing. Read this *before* re-attempting questions you missed — the goal is to internalize the pattern, not memorize the answer letter.

---

## Domain 1: Agentic Architecture & Orchestration (27%)

**The agentic loop runs on `stop_reason`, not text-sniffing.**
Every response from Claude has a `stop_reason`. `"tool_use"` means Claude wants to act — your code executes the requested tool(s) and sends results back. `"end_turn"` means Claude is done reasoning and has given its final answer. Never try to detect completion by scanning the text for phrases like "I'm done" — that's fragile and the model isn't obligated to phrase things that way.

**Tool results only exist in the conversation history.**
There's no side channel where a tool's output "arrives" in the model's head. Your harness appends a `tool_result` block (tied to the original `tool_use` call's ID) to the messages array and sends the *whole updated conversation* back. The model only knows what's in that array.

**Subagents are isolated by default.**
A subagent does **not** inherit the parent's conversation history. If the parent wants the subagent to use specific findings, those findings must be explicitly written into the subagent's prompt. This isolation is a feature (clean context, no bleed-through) but it's also the most common bug source — "the subagent didn't know about X" almost always traces back to X never being passed in.

**Spawning subagents requires the `Task` tool.**
If a coordinator's `allowedTools` doesn't include `Task`, it *cannot* delegate — no matter how explicit the system prompt is about delegating. Capability configuration overrides instructions. This is a recurring exam theme: **when a model "won't" do something, check what it's actually permitted to do before assuming a prompting problem.**

**Parallel vs sequential delegation is a matter of response shape.**
Emitting multiple `Task` calls in a *single* response spawns subagents concurrently. Calling them across separate turns forces sequential execution. If latency is doubling for two independent subtasks, look at whether the calls are batched into one turn.

**Decomposition strategy should match task predictability.**
- **Prompt chaining** (fixed sequence of steps) — use when the workflow is the same every time (e.g., a 5-point code review checklist).
- **Dynamic/adaptive decomposition** — use when the next step depends on what the current step discovers (e.g., an incident investigation, open-ended "add tests to this legacy codebase").
Don't over-engineer flexibility into predictable work, and don't force predictable structure onto exploratory work.

**Session management tools solve different problems — don't conflate them:**
- `--resume <name>` — continue a specific past session with its context intact. Best when the underlying code/facts haven't materially changed.
- `fork_session` — branch from a shared baseline into independent explorations (e.g., comparing two refactor strategies) without cross-contamination.
- `/compact` — reduce context *within* an ongoing session so it can keep going.
- Starting fresh with an injected summary — the right move when resuming would mean reasoning over now-stale tool results (e.g., files changed significantly since the session ran).

---

## Domain 2: Tool Design & MCP Integration (18%)

**Tool descriptions are the primary routing signal — not tool names.**
When two tools get confused, the fix is almost always: (1) document expected input formats + example queries, and (2) add explicit boundary language ("use X when Y is present; use Z for..."). Shortening descriptions or relying on names alone doesn't resolve ambiguity — it just removes information the model needed.

**Structured errors, not error strings.**
A tool that returns `"Operation failed"` gives the agent nothing to reason with. Good error design returns metadata: an `errorCategory` (transient / validation / business / permission), an `isRetryable` boolean, and a human-readable description. This lets the agent retry transient failures, explain business-rule refusals to the user, and stop wasting attempts on permanent failures — all from one structured signal.

**MCP's `isError` flag is how tool failures get communicated as failures.**
If a parsing failure comes back as ordinary successful text ("Error: could not parse document"), the agent may treat that sentence as *content*. Setting `isError: true` lets the agent programmatically recognize failure instead of interpreting error prose as data.

**MCP Resources reduce exploratory tool-call overhead.**
If agents are burning calls just discovering what data exists (listing collections, probing schemas), exposing that inventory as an MCP **resource** gives visibility without round-trips. Resources are for *what's available*; tools are for *taking action*.

**Split overloaded tools into purpose-specific ones.**
A single tool with a `mode` parameter (extract / summarize / verify) invites wrong-mode selection because the decision is buried inside arguments instead of being a visible tool choice. Splitting into `extract_data_points`, `summarize_content`, `verify_claim_against_source` — each with its own contract — makes selection a first-class decision the model is good at.

**Scope tools per-agent role.**
Give every subagent the full 18-tool catalog and misuse follows — agents will attempt tools outside their role simply because they're available. Restrict each subagent to the handful of tools relevant to its job.

**Glob vs Grep — know the division of labor.**
- **Glob**: file *path* pattern matching (find files by name/extension/location).
- **Grep**: file *content* searching (find a string inside files).
Mixing these up (e.g., "Grep for a filename pattern," "Glob across file contents") is a documented anti-pattern.

**Read+Write is the fallback when Edit can't find a unique anchor.**
`Edit` requires unique matching text. When the same snippet appears multiple times, the reliable path is `Read` the full file, construct the complete modified version, and `Write` it back — not deleting occurrences to force uniqueness, not switching to shell text-editing.

---

## Domain 3: Claude Code Configuration & Workflows (20%)

**Know the memory/config hierarchy and what belongs where:**
| Mechanism | Scope | Use for |
|---|---|---|
| `CLAUDE.md` (project root) | Whole team, version-controlled | Universal, always-relevant standards (coding conventions, testing philosophy) |
| `~/.claude/CLAUDE.md` | You only, not shared | Personal preferences |
| `.claude/commands/` (repo) | Whole team | Shared slash commands |
| `~/.claude/commands/` | You only | Personal slash commands you don't want to impose on teammates |
| `.claude/rules/` | Team, topic-scoped, optionally path-scoped | Breaking up a bloated CLAUDE.md into focused files (testing.md, deployment.md), loaded only when relevant |
| `.claude/skills/` (SKILL.md) | On-demand, invoked when needed | Multi-step workflows used occasionally (not every session) |

The general principle: **always-relevant → CLAUDE.md; occasional/task-specific → skills; team-shared → repo; personal → home directory.** A monolithic 2,000-line CLAUDE.md loaded on every session regardless of task is the anti-pattern rules-scoping fixes.

**Diagnosing "why isn't my config applying":** run `/memory` to see exactly which memory files a session actually loaded. This is the go-to diagnostic when standards seem to be ignored — usually the answer is "the file's in the wrong scope."

**Skill frontmatter options solve specific problems — match the symptom to the field:**
- `context: fork` — skill produces heavy exploratory output; run it in an isolated sub-agent context so it doesn't flood the main conversation.
- `allowed-tools` — restrict what a skill can invoke (e.g., guarantee it can never run Bash), enforced at the configuration level, not just requested in prose.
- `argument-hint` — skill needs parameters; prompt the user for them when invoked bare instead of running underspecified.

**Plan mode vs direct execution is a function of scope and ambiguity, not task type generally.**
Well-scoped, single-approach changes (a bug fix with a clear stack trace) → direct execution. Large-scale, multiple-valid-approaches changes (a 45-file library migration) → plan mode, so design happens before commitment.

**Non-interactive CI invocation needs the full trio:**
`-p` (print/non-interactive) + `--output-format json` + `--json-schema` → machine-parseable structured findings, no waiting on interactive input, no regex-scraping prose.

**Review quality degrades when the reviewer shares context with the generator.**
A session that generated code and then reviews it retains the reasoning that justified its own decisions, biasing it toward leniency. Fix: run CI review in an independent instance without the generator's context — this is a context-isolation problem, not a "the model is being polite" problem.

**Re-review noise:** feed prior findings back into context and instruct the model to report only *new* or *still-unresolved* issues, rather than re-evaluating the diff in isolation each time.

---

## Domain 4: Prompt Engineering & Structured Output (20%)

**Tool use is the structural fix for JSON reliability — not prompt phrasing.**
Asking for "valid JSON only" in prose still risks markdown fences, commentary, and malformed syntax. Defining the extraction as a tool call with an input schema forces schema-conformant output at the protocol level — errors are eliminated by construction, not reduced by emphasis.

**`tool_choice` modes matter:**
- `"auto"` — model decides whether to call a tool at all.
- `"any"` — model *must* call a tool, but chooses which one (right for "always extract, but let the model pick the right schema for an unknown document type").
- forced to one named tool — every input goes through the same schema regardless of fit.

**Structural validity ≠ semantic correctness.**
Tool use guarantees well-formed output; it says nothing about whether the *content* is right. Line items that don't sum to the stated total, values in the wrong field — these need a semantic validation layer (e.g., have the model also emit a `calculated_total` and a `conflict_detected` flag) on top of schema enforcement.

**Required vs nullable fields prevent fabrication.**
If a field is marked required but the source document genuinely lacks that data, the model will often fabricate a plausible-looking value to satisfy the schema. Making the field optional/nullable gives it a legitimate way to say "not present."

**Effective retries include the failure, not just an instruction to try harder.**
A good retry request contains: the original input, the failed output, and the specific validation errors — enabling targeted self-correction. And know when *not* to retry: if the needed information simply isn't in the source, no amount of retrying manufactures it.

**Precision engineering means categorical criteria, not intensity of wording.**
"Be conservative" or "be very, very conservative" doesn't transfer a decision boundary. What works: explicit categories of what to report vs. skip, with concrete examples — an operational rule the model can pattern-match against, not a vibe to embody harder.

**Few-shot prompting: quality and targeting, not volume.**
2–4 examples aimed at genuinely ambiguous cases, showing the *reasoning* for why one action was chosen over a plausible alternative, beat twenty generic examples. Examples are also how you lock exact output *format* (fields, ordering) for consistency. Few-shot doesn't replace explicit criteria — it complements it, especially for judgment calls plain rules can't fully specify (e.g., "this idiom is fine, that one is a bug").

**Trust erosion is categorical, not just quantitative.**
If one finding category (e.g., "style") is mostly noise, users start discounting *all* categories, including accurate ones. The fix is often to disable the noisy category temporarily rather than let it poison trust in the whole system.

**Batch API tradeoffs:**
- Good fit: latency-tolerant, non-blocking work (nightly generation, weekly reports) — ~50% cost savings.
- No latency SLA — can take up to 24 hours, so blocking/pre-merge workflows should stay synchronous.
- No multi-turn tool calling within a single batched request.
- `custom_id` correlates responses back to requests — essential for handling partial failures (resubmit only the failed subset, not the whole batch).

---

## Domain 5: Context Management & Reliability (15%)

**Protect precise facts from lossy summarization.**
In long sessions, numeric/transactional details (amounts, order numbers, dates) get blurred when history is condensed into vague summaries. Fix: maintain a dedicated "facts block" (or scratchpad file) that persists exact values outside the summarized narrative, and trim verbose tool outputs to only the fields relevant to the current task *before* they enter context — don't let 40-field dumps accumulate.

**Delegate verbose exploration to subagents to protect the main session's context.**
Long discovery work ("find all test files," "trace the refund flow") belongs in a subagent that returns a summary — keeping the coordinating session's context reserved for high-level decisions rather than raw exploration noise.

**`/compact` is the in-session relief valve; a scratchpad is the persistent-facts valve.** They solve different problems — compaction reduces volume, a scratchpad preserves precision. Both may be needed together.

**Provenance must be structural, not reconstructed after the fact.**
If citations are missing from a synthesized report, the fix isn't "add citations at the end" (fabrication) or "reduce summarization steps" (delays the problem). Structured claim-source mappings — carried through every hop as data — are what preserve attribution through multiple compression steps.

**Conflicting data should be surfaced, not resolved by a default heuristic.**
Two credible sources, different numbers → annotate both with attribution and let a human/coordinator reconcile deliberately. Don't average, don't silently pick one, don't always trust "the more recent" source without checking whether they're actually describing the same point in time (carry dates in structured output).

**Error handling should distinguish failure from empty success.**
A search that fails (timeout, service error) is not the same as a search that succeeds and finds nothing. Conflating them either escalates non-problems or silently treats real failures as valid empty results. Attempt local recovery for transient issues; escalate what can't be resolved *with* what was already tried.

**Confidence-based routing needs calibration, not raw self-report.**
Model-reported confidence scores are often poorly calibrated on their own. The sound pattern: calibrate thresholds against a labeled validation set, then route review capacity to low-calibrated-confidence or contradictory-source cases — where errors actually concentrate, not to arbitrary buckets (first 1%, longest documents).

**Aggregate accuracy metrics can hide segment-level failures.**
97% overall doesn't mean 97% everywhere. Before reducing human review, segment accuracy by document type/field, and keep a stratified random sample under ongoing measurement (not just model-flagged "uncertain" cases — the model can be confidently wrong too).

---

## Cross-Cutting Themes Worth Internalizing

A few patterns repeat across almost every domain and are worth having as reflexes going into the exam:

1. **Configuration beats instruction for anything that must be guaranteed.** Hooks, `allowed-tools`, schema-enforced tool use, and `isError` flags all show up as "the deterministic fix" when a prompt-only approach has a nonzero failure rate that matters (money, security, data integrity).
2. **Isolation is the default; explicit passing is required.** Subagents don't inherit history. New sessions don't inherit old ones. If something "should have known X," check whether X was actually placed in its context.
3. **Structure survives compression; prose doesn't.** Facts blocks, claim-source mappings, structured error metadata — anything that needs to survive summarization should be carried as structured data, not left to hope that a summary preserves it.
4. **Match the mechanism to the shape of the uncertainty.** Fixed steps for predictable work, adaptive decomposition for exploratory work; plan mode for scale/ambiguity, direct execution for well-scoped work; category-based criteria for a stable judgment boundary, few-shot examples for judgment that resists being fully stated as rules.

---

Want to go deeper on any one domain, or switch to quizzing mode now that the concepts are fresh?
