# CCAR-F Flashcards — Question & Answer Format

Cover the answer and test yourself. Organized by domain, matching the blueprint weights.

---

## Domain 1: Agentic Architecture & Orchestration (27%)

**Q: How does the agentic loop know when to keep calling tools vs. present a final answer?**
A: It checks `stop_reason`. `"tool_use"` → execute the requested tool(s) and send results back. `"end_turn"` → present the response. Never infer completion by scanning response text for phrases.

**Q: How does a tool's result actually get back to the model?**
A: Your code appends a tool result block (referencing the original tool call's ID) to the conversation history and sends the whole updated messages array back to Claude. There's no automatic delivery mechanism.

**Q: Do subagents inherit the parent's conversation history?**
A: No. Subagents are isolated by default — whatever context they need must be explicitly included in their prompt.

**Q: A coordinator's system prompt says to delegate, but it never spawns subagents. Most likely cause?**
A: Its `allowedTools` doesn't include `Task` — the tool required to spawn subagents. Capability configuration overrides instructions.

**Q: How do you run two independent subagents in parallel instead of sequentially?**
A: Emit both `Task` tool calls in a single response. Calls across separate turns run sequentially.

**Q: When should you use prompt chaining vs. dynamic/adaptive decomposition?**
A: Prompt chaining for predictable, fixed-sequence work (e.g., a 5-point checklist review). Dynamic decomposition for work where the next step depends on what was just discovered (e.g., incident investigation, open-ended legacy test coverage).

**Q: What's the difference between `--resume`, `fork_session`, and starting a fresh session with an injected summary?**
A: `--resume <name>` continues a specific session with its context intact — best when underlying facts haven't changed much. `fork_session` branches from a shared baseline into independent explorations. A fresh session with a summary is best when much of the prior session's tool results are now stale (e.g., major code changes since).

**Q: `/compact` — what does it do and when do you reach for it?**
A: Reduces context usage within an ongoing session so it can keep going without degrading, without ending or forking the session.

**Q: A multi-issue customer message arrives (refund + address change + missing credit). Correct handling?**
A: Decompose into distinct items, investigate each (in parallel where possible) using shared context, then synthesize one unified response covering all of them — not resolving one and dropping the rest.

**Q: What should a handoff to a human include when escalating mid-process?**
A: A structured summary: identifying info, root-cause analysis so far, relevant amounts/order numbers, and a recommended action — not raw transcripts or unprocessed tool output.

---

## Domain 2: Tool Design & MCP Integration (18%)

**Q: Two tools keep getting confused by the model. What's the most direct fix?**
A: Improve their descriptions — document expected input formats, add example queries, and add explicit boundary language differentiating when to use each. Descriptions are the primary selection signal, not tool names.

**Q: A tool returns the generic string `"Operation failed"` on any failure. What's wrong and what's the fix?**
A: The agent can't tell transient failures from permanent ones. Fix: return structured error metadata — `errorCategory` (transient/validation/business/permission), an `isRetryable` boolean, and a human-readable description.

**Q: A refund request is rejected for violating a returns-window policy. What should the error response look like?**
A: `errorCategory: "business"`, `retriable: false`, plus a customer-friendly explanation — so the agent explains the refusal instead of retrying it.

**Q: What is the MCP `isError` flag for?**
A: It marks a tool response as a failure at the protocol level, so the agent programmatically recognizes failure instead of treating error prose as if it were successful content.

**Q: An agent keeps burning tool calls just discovering what data/collections exist before doing real work. What MCP capability reduces this?**
A: Exposing content catalogs (document hierarchies, schema listings, etc.) as MCP **resources** — visibility without exploratory tool calls.

**Q: One tool handles extraction, summarization, and verification via a `mode` parameter, and the model keeps picking the wrong mode. Fix?**
A: Split into purpose-specific tools (e.g., `extract_data_points`, `summarize_content`, `verify_claim_against_source`), each with its own clear contract — turning a hidden parameter decision into a visible tool-selection decision.

**Q: Every subagent gets the full tool catalog and keeps misusing tools outside its role. Fix?**
A: Scope each subagent's tool set to only what its role needs. Large tool inventories degrade selection reliability, and available-but-irrelevant tools invite misuse.

**Q: When should you use Glob vs. Grep?**
A: Glob for file *path* pattern matching (find files by name/location). Grep for searching file *contents* for a string/pattern.

**Q: Edit fails because the target text appears in six places in a legacy file. Standard fallback?**
A: Read the full file, construct the complete modified content, then Write it back — the reliable path when Edit can't find a unique anchor.

---

## Domain 3: Claude Code Configuration & Workflows (20%)

**Q: You have a personal slash command you don't want teammates to see when they clone the repo. Where does it go?**
A: `~/.claude/commands/` (user-scoped, outside version control) — not `.claude/commands/` in the repo, which is shared.

**Q: A skill's exploration output floods and crowds out the main conversation. Which frontmatter option fixes it?**
A: `context: fork` — runs the skill in an isolated sub-agent context and returns only the result to the main session.

**Q: You want a skill to be able to create files but *never* run shell commands, guaranteed. How?**
A: Set `allowed-tools` in the skill's frontmatter to permit only the file-write operations it needs, excluding Bash. Prose instructions ("never run Bash") aren't a guarantee.

**Q: A new teammate's sessions ignore team coding standards, which live in your personal `~/.claude/CLAUDE.md`. Fix, and how do you verify it worked?**
A: Move the standards into the project-level `CLAUDE.md` (checked into the repo) so every clone gets them. Use `/memory` to verify which memory files a session actually loaded.

**Q: A 2,000-line CLAUDE.md covering testing, API conventions, deployment, and styling loads in full every session regardless of task. Recommended fix?**
A: Split into focused topic files under `.claude/rules/` (e.g., `testing.md`, `deployment.md`), with path-scoped frontmatter so rules load only when relevant.

**Q: How do you resume a specific named session from yesterday exactly where it left off?**
A: `claude --resume <session-name>`.

**Q: You've done a thorough analysis and want to try two competing strategies from that same baseline, without them contaminating each other. What do you use?**
A: `fork_session` — branches independent explorations from a shared baseline.

**Q: You want to resume last week's refactoring session, but the team has since merged major changes to the files it analyzed. Reliable approach?**
A: Start a new session seeded with a structured summary of prior conclusions — don't resume as-is, since stale tool results will mislead the model into reasoning over code that no longer exists.

**Q: Mid-session, context is filling with verbose output and quality is degrading, but you're not ready to end the session. Which command helps?**
A: `/compact`.

**Q: What are the two config-level fields that matter most for CI review invocation, and why?**
A: `-p` (non-interactive/print mode) plus `--output-format json` (and `--json-schema`) — produces machine-parseable structured findings without waiting on interactive input or scraping prose.

**Q: A single Claude Code session generates code and then reviews its own change — reviews are conspicuously lenient. Why, and the fix?**
A: The session retains the reasoning that justified its own generation choices, biasing it against catching its own issues. Fix: run review in an independent instance without the generator's context.

**Q: Which task gets plan mode, and which gets direct execution: a well-scoped single-function bug fix with a clear stack trace, vs. a 45-file library migration with multiple viable approaches?**
A: Direct execution for the bug fix; plan mode for the migration — scale and multiple valid approaches call for design before commitment.

---

## Domain 4: Prompt Engineering & Structured Output (20%)

**Q: A pipeline asks Claude to "respond with JSON" in plain text and parsing keeps breaking (fences, commentary, malformed syntax). What's the structural fix?**
A: Define the extraction as a tool call whose input schema is your JSON schema, and read the structured data from the `tool_use` block — this guarantees schema compliance instead of hoping prose stays clean.

**Q: What's the difference between `tool_choice: "auto"` and `tool_choice: "any"`?**
A: `"auto"` — model decides whether to call a tool at all. `"any"` — model must call *some* tool but chooses which one. Use `"any"` when structured output is mandatory but the right schema depends on the input.

**Q: Extractions are now always syntactically valid, but invoice line items sometimes don't sum to the stated total. What does this tell you, and what's the fix?**
A: Tool use guarantees structural validity, not semantic correctness. Add a semantic validation layer — e.g., have the model also emit a `calculated_total` alongside `stated_total` with a `conflict_detected` flag.

**Q: A schema marks `vendor_tax_id` as required, and the model fabricates values on invoices that genuinely lack one. Fix?**
A: Make the field optional/nullable so the model has a legitimate way to return null instead of inventing a value.

**Q: An extraction fails validation with specific field errors. What should the retry request include, and when should you skip retrying?**
A: Include the original document, the failed extraction, and the specific validation errors (enables targeted self-correction). Skip retrying when the required information is simply absent from the source — no retry manufactures data that isn't there.

**Q: A review prompt says "be conservative and only report high-confidence findings," but false positives stay high. What actually reduces false positives?**
A: Replace vague confidence language with explicit categorical criteria — which issue types to report (bugs, security) vs. skip (minor style), with concrete boundaries.

**Q: A "code style" finding category is 70% false positives, and trust is collapsing across all categories including accurate ones. Right move?**
A: Temporarily disable the noisy category to protect trust in the accurate ones, while improving its prompts offline before re-enabling.

**Q: A reviewer classifies near-identical issues as "critical" one day and "minor" the next. Fix for consistent severity?**
A: Define explicit severity criteria with concrete code examples for each level, so severity has an operational definition to match against.

**Q: A reviewer keeps flagging accepted idiomatic patterns as bugs despite detailed prose instructions. Most effective addition?**
A: Few-shot examples contrasting accepted patterns vs. genuine issues, with the reasoning shown — this generalizes judgment better than more prose.

**Q: Which CI workloads suit the Message Batches API, and which don't?**
A: Latency-tolerant, non-blocking work (nightly test generation, weekly audits) — ~50% cost savings. Blocking workflows (pre-merge checks) should stay synchronous, since batches can take up to 24 hours with no latency SLA.

**Q: What makes few-shot prompting effective?**
A: A small number (2–4) of targeted examples aimed at genuinely ambiguous cases, showing the reasoning for the chosen action over alternatives — plus examples that demonstrate exact desired output format for consistency. More examples ≠ better; targeting matters more than volume.

---

## Domain 5: Context Management & Reliability (15%)

**Q: In long sessions, refund amounts and order numbers get misquoted because they were condensed into vague summaries. Fix?**
A: Extract transactional facts (amounts, dates, order numbers) into a persistent case-facts block included in every prompt, kept outside the lossy summarized history. Also trim verbose tool results to relevant fields before they enter context.

**Q: During multi-hour exploration, the agent starts answering from "typical patterns" instead of what it actually found. Two fixes?**
A: (1) Maintain a scratchpad recording key findings as discovered, referenced for later questions. (2) Delegate verbose investigation to subagents that return summaries, keeping the main session's context for coordination.

**Q: Final research reports state findings but cite nothing — attribution got lost during summarization. Structural fix?**
A: Require subagents to output structured claim-source mappings (claim, evidence excerpt, source) that every downstream step preserves and merges through synthesis — not citations bolted on afterward.

**Q: Two credible sources report different market-size figures. Correct handling?**
A: Annotate the conflict explicitly with both values and their sources, and let the coordinator decide how to reconcile — don't average, don't silently pick one. Also carry dates in structured output, since apparent conflicts are sometimes just different points in time.

**Q: A web search subagent's searches sometimes fail and sometimes just return no results. What two behaviors are correct?**
A: Attempt local recovery for transient failures (e.g., retry a timeout), and propagate only unresolvable errors to the coordinator with context. Also distinguish access failures from valid empty results — they require different coordinator decisions.

**Q: An extraction system reports 97% aggregate accuracy and leadership wants to cut human review. What must happen first?**
A: Segment accuracy by document type and field (aggregate metrics can mask systematic failures in specific segments), and implement stratified random sampling of high-confidence extractions for ongoing error measurement — not just reviewing what the model flags as uncertain.

**Q: Reviewer capacity only covers a fraction of extractions. Sound routing design?**
A: Have the model output field-level confidence scores, calibrate thresholds against a labeled validation set, then route low-calibrated-confidence or ambiguous-source extractions to human review — not raw self-reported confidence, not fixed-order sampling.

**Q: Why is trusting a model's raw self-reported confidence score risky?**
A: It's often poorly calibrated on its own — it needs to be validated/calibrated against a labeled dataset before being used to make review-routing decisions.

---

## Rapid-fire cross-cutting reflexes

**Q: When a model "won't" follow a critical rule reliably via prompting alone, what's the general fix pattern?**
A: Move enforcement into configuration/code — hooks, `allowed-tools`, schema-enforced tool use, `isError` flags — instead of relying on instruction-following for guarantees that matter.

**Q: What's the default assumption about context sharing between sessions or subagents?**
A: Isolation. Nothing is inherited automatically — anything needed must be explicitly passed in.

**Q: Facts that must survive summarization (numbers, attributions, error metadata) — how should they be carried?**
A: As structured data (facts blocks, claim-source mappings, structured error fields), not as prose that a compression step might blur.

**Q: How do you decide between a fixed sequential process and an adaptive one?**
A: Match the mechanism to the uncertainty: fixed/predictable work → prompt chaining or direct execution; exploratory/uncertain work → dynamic decomposition or plan mode.

---

Want me to turn this into an interactive self-quiz where I ask one at a time and track your score by domain?
