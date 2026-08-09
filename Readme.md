Claude Certified Architect – Foundations (CCAR-F)

D1: stop_reason loop control, subagent isolation, hooks vs prompts for enforcement, Attention Dilution, Stale Context Problem

D2: tool description fix vs. tool-overload fix (different symptoms), access failure vs. valid empty result

D3: the -p flag (single most-tested fact), CLAUDE.md hierarchy trap, plan mode = ambiguity not difficulty

D4: explicit criteria > vague instructions, few-shot's 3 triggers, retry effectiveness boundary

D5: persistent case-facts block (most important D5 pattern), 3 valid vs 2 unreliable escalation triggers, aggregate metrics trap

# Extra Notes

1. Claude Code hooks are deterministic scripts or commands that run automatically during specific moments in an AI coding assistant's lifecycle, such as before or after a tool runs. They act like programmable middleware to format code, block dangerous commands, or enforce team rules without relying on the AI to choose to run them
2. How Claude Code Hooks WorkDeterministic Triggers: They fire based on system events (PreToolUse, PostToolUse, SessionStart) rather than LLM decisions.Veto Power: A pre-tool-use hook can return a specific exit code (like code 2) to block Claude from executing a bad command or modifying a protected file.Automation: Common uses include running linters, auto-formatting code edits, or triggering desktop alerts
3. Skills: Custom instructions or executable actions that the AI chooses to call based on its reasoning during a task.
4. MCP (Model Context Protocol): An open standard that connects an AI assistant to external data sources, APIs, and databases.
5. Subagents: Isolated context windows where a separate secondary AI instance handles a sub-task independently.
6. Guardrails / Middleware: General software patterns used across agent frameworks (like LangChain or LlamaIndex) to sanitize inputs and outputs before they hit an LLM.
