# Claude Certified Architect – Foundations (CCAR-F)

D1: stop_reason loop control, subagent isolation, hooks vs prompts for enforcement, Attention Dilution, Stale Context Problem

D2: tool description fix vs. tool-overload fix (different symptoms), access failure vs. valid empty result

D3: the -p flag (single most-tested fact), CLAUDE.md hierarchy trap, plan mode = ambiguity not difficulty

D4: explicit criteria > vague instructions, few-shot's 3 triggers, retry effectiveness boundary

D5: persistent case-facts block (most important D5 pattern), 3 valid vs 2 unreliable escalation triggers, aggregate metrics trap
# Form :
https://docs.google.com/forms/d/e/1FAIpQLSctZ8Xf__pu1a7FC-2nOuk40kLMm-8tgM7jdplx2P5bYufQ9A/viewform

I have made this form try and test your knowledge this based on important questions from Opensource question bank

# Extra Notes

1. Claude Code hooks are deterministic scripts or commands that run automatically during specific moments in an AI coding assistant's lifecycle, such as before or after a tool runs. They act like programmable middleware to format code, block dangerous commands, or enforce team rules without relying on the AI to choose to run them
2. How Claude Code Hooks WorkDeterministic Triggers: They fire based on system events (PreToolUse, PostToolUse, SessionStart) rather than LLM decisions.Veto Power: A pre-tool-use hook can return a specific exit code (like code 2) to block Claude from executing a bad command or modifying a protected file.Automation: Common uses include running linters, auto-formatting code edits, or triggering desktop alerts
3. Skills: Custom instructions or executable actions that the AI chooses to call based on its reasoning during a task.
4. MCP (Model Context Protocol): An open standard that connects an AI assistant to external data sources, APIs, and databases.
5. Subagents: Isolated context windows where a separate secondary AI instance handles a sub-task independently.
6. Guardrails / Middleware: General software patterns used across agent frameworks (like LangChain or LlamaIndex) to sanitize inputs and outputs before they hit an LLM.

# Claude Components

Models

Opus – Anthropic's most capable model tier, for the hardest reasoning and highest-stakes tasks.
Sonnet – the balanced mid-tier model, strong capability at lower cost/latency than Opus, used as the default for most work.
Haiku – the fastest, cheapest model tier, built for high-volume or latency-sensitive tasks.

Messages API – the core Anthropic API endpoint (/v1/messages) used to send conversation turns to Claude and get back text/tool-use responses.

Prompt engineering – the practice of structuring instructions (clarity, examples, step-by-step reasoning, XML tags, etc.) to reliably get better outputs from Claude.

Tool use – Claude's ability to call developer-defined functions (APIs, code, lookups) mid-conversation and use their results to continue reasoning.

MCP (Model Context Protocol) – an open, "USB-C for AI" standard connecting models to external tools and data:

Client – the app/agent side that connects to and calls an MCP server.
Server – exposes a set of tools/resources/prompts to any MCP client.
Resources – readable data/content an MCP server exposes to the model.
Tools – callable actions/functions an MCP server exposes.
Prompts – reusable prompt templates an MCP server can offer to clients.

Claude Code – Anthropic's agentic coding tool that lets Claude read, write, run, and debug code directly in your terminal/IDE/repo.

CLAUDE.md – a project-memory file Claude Code automatically reads for persistent context (conventions, architecture, commands) about your codebase.

Skills – reusable, on-demand instruction packages (SKILL.md + assets) that Claude loads into context only when relevant, instead of repeating instructions every time.

Subagents – isolated, spun-off Claude sessions that do focused sub-tasks and return just a summary, keeping the main context clean.

Hooks – deterministic, code-level triggers on Claude Code lifecycle events (e.g., before a tool runs) that enforce behavior regardless of what the model decides.

Plugins – installable bundles that package skills, subagents, hooks, and MCP servers together for easy sharing across a team.

Connectors – integrations (in Claude.ai/Cowork) that link Claude to external services like Slack, Google Drive, or Jira via MCP.

Agentic loop – the cycle of Claude taking an action (tool call), observing the result, and deciding the next step, repeated until the task is done.

Context management – techniques (compaction, subagents, memory files, retrieval) for keeping the most relevant information in the model's limited context window as work grows.

Safety and guardrails – the policies, classifiers, and design constraints (refusals, permission prompts, content filters) that keep Claude's behavior within intended, safe bounds.
