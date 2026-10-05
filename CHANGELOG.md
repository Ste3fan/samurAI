# Changelog

## 1.0 — first public release

### Framework (`samurai-core`)
- **Agents.**
  - **LLM agents:** `LlmAgent`, `ToolUseAgent` (function calling), `RagAgent` and `IngestAgent` (RAG with per-client
    isolation).
  - **Blind agents** (no model, cost 0): `CommandAgent`, `WaitAgent`, `HttpAgent`, `FunctionAgent`, `ApprovalAgent`
    (human in the loop).
  - **Composites:** sequential, parallel, router, loop, fallback.
- **Providers.** 31 provider types:
  - API providers;
  - local servers (probed at startup);
  - subscriptions through official CLIs (Claude Code, Gemini CLI, generic CLI);
  - `openai-compatible` and `generic` (configured without code), plus a plugin SPI.
- **Model choice.**
  - Model routing by purpose, with the `OPTIMAL` strategy (cheapest model reaching the quality target), plus
    `CHEAPEST`, `BALANCED`, `BEST_QUALITY`, `LOCAL_FIRST` and `LOCAL_ONLY`.
  - Automatic task analysis (`Purpose.AUTO`) and escalation (`acceptIf`).
  - Free tiers and subscriptions first, with quotas, 429 cool-downs and fallback to paid or local models.
  - Circuit breaker, response cache, semantic cache.
- **Claude without an API key.** A provider of type `anthropic` that has no API key automatically uses the installed
  `claude` command (Claude Code subscription). Disable it with `samurai.llm.claudeCliFallback=false`.
- **Orchestrator.**
  - Runs: sync, async and periodic, with timeouts and cancellation.
  - Security: access control, rate limiting, concurrency limits, guardrails, budgets, maximum cost per run.
  - Records: audit log and JSONL traces.
- **Sub-orchestrators** (`Orchestrator.attach`):
  - cascading lifecycle;
  - costs rolled up to the parent (`GroupBy.ORCHESTRATOR`);
  - inherited budgets and audit;
  - supervision with restart and back-off.

  Shared data stores (`DataStore`: JSONL, CSV, text) and `OrchestratorAgent` complete them.
- **Cost tracking.** Per client, user, session, API key, provider, model, agent, workflow, orchestrator and time
  interval, with a markup factor. Monetization: plans, credits, fair use, outcome fees, margins, ROI.
- **Configuration.** Layered: classpath, properties, JSON or XML files, `.env`, Docker secrets, environment variables,
  system properties. `${VAR:default}` interpolation.
- **Interfaces:**
  - REST server: API keys, approvals, credits, HMAC-signed webhooks;
  - MCP server (stdio);
  - tool setup (`ToolCatalog`, `ToolInstaller`, Docker).

### XML applications (`samurai-xml`)
- **One file per application:** configuration, agents, workflows, schedules and sub-orchestrators, validated by
  `samurai.xsd`.
- **Task prompts:** inline, from files, or from whole directories (`<llm-dir>`, with front matter).
- **Errors** report the file and the line.
- **Security:** XXE-safe parsing, paths restricted to the application directory, class allow-list.
- **Command line:**
  - commands: `validate`, `list`, `run`, `serve`, `mcp`;
  - `--json` output, stdin input, `--env`, `--opt=value` syntax;
  - strict option checking;
  - exit codes `0`, `1`, `2`, `3`.

### samurAI Studio (`samurai-studio`, `samurai-studio-web`)
- **Desktop app:**
  - XML editor with validation;
  - runs with live progress, history with agent trees;
  - monitoring of orchestrators, sub-orchestrators and schedules;
  - events, audit, costs (CSV export), approvals, data stores.
- **The same command line as `samurai-xml`.** `serve` hands over to `samurai-studio-web`.
- **Web server (Spring Boot 4):**
  - a real-time dashboard;
  - a JSON API and a Server-Sent Events stream;
  - the REST API and the dashboard served on one port.
- **Ports already used by another application are refused,** even when that application listens on all interfaces.
