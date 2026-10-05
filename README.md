<p align="center">
  <img src="assets/samurai-logo-256.png" alt="samurAI logo" width="180">
</p>

<h1 align="center">samurAI 1.0</h1>

<p align="center"><b>A Java framework for multi-provider AI agents, orchestration and cost tracking —
from plain Java, from a single XML file, or from a desktop studio.</b></p>

samurAI lets you build AI-powered applications in Java without locking yourself to one AI vendor. You chain **agents** —
some that call a language model, some that run commands, wait, call HTTP endpoints or run Java code with **no model at
all** — into workflows. An **orchestrator** runs them synchronously, asynchronously or on a schedule. On every call it
picks the cheapest model that is good enough for the task, across all the providers you configured. Every token and
every cent is attributed to a client, user, session, API key and model.

```java
try (Orchestrator orch = Orchestrator.fromConfig(SamuraiConfig.load())
        .commandPolicy(CommandPolicy.builder().allow("df").build())                        // OS commands: allow-list
        .build()) {
    Agent collect  = CommandAgent.builder("collect").command("df", "-h").build();          // no model, cost 0
    Agent classify = LlmAgent.builder("classify").purpose(Purpose.CLASSIFICATION)
            .prompt("Is this disk report OK, WARNING or CRITICAL? {{input}}").build();
    Agent notify   = HttpAgent.builder("notify").post("https://hooks.example.com/ops", "{\"text\":\"{{input}}\"}")
            .allowHosts("hooks.example.com").build();

    orch.register("disk-check", collect.then(classify).then(notify));
    RunResult r = orch.run(RunRequest.builder("disk-check").principal(Principal.of("acme", "ana", "ops")).build());
    System.out.println(r.output() + " — billed " + r.cost().billedCost() + " " + r.currency());
}
```

The same workflow, with no Java at all:

```xml
<samurai version="1.0" name="ops">
  <config><security><commands allow="df"/></security></config>
  <agents>
    <command name="collect" command="df -h"/>
    <llm name="classify" purpose="CLASSIFICATION"><prompt>Is this disk report OK, WARNING or CRITICAL? {{input}}</prompt></llm>
  </agents>
  <workflows>
    <workflow name="disk-check" description="Checks disk usage"><ref agent="collect"/><ref agent="classify"/></workflow>
  </workflows>
  <schedules><schedule workflow="disk-check" every="1h"/></schedules>
</samurai>
```

```bash
java -jar samurai-studio-1.0-all.jar run app.xml disk-check --json
```

## Contents

- [Highlights](#highlights)
- [Downloads](#downloads)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick start](#quick-start)
- [AI providers](#ai-providers)
- [Choosing the model](#choosing-the-model)
- [Costs](#costs)
- [samurAI Studio](#samurai-studio)
- [Command line](#command-line)
- [Using samurAI from other LLMs and agents](#using-samurai-from-other-llms-and-agents)
- [Security](#security)
- [Documentation](#documentation)
- [Third-party notices](#third-party-notices)

## Highlights

- **Multi-provider, or single-provider.** 31 provider types are built in:
  - **API providers:** OpenAI, Anthropic, Google Gemini and Vertex AI, Azure OpenAI, AWS Bedrock, Mistral, Cohere,
    Groq, DeepSeek, xAI, OpenRouter, Together, Fireworks, Perplexity, Cerebras, SambaNova, Hugging Face,
    GitHub Models, Cloudflare Workers AI, NVIDIA NIM;
  - **local servers:** Ollama, LM Studio, vLLM, llama.cpp, LocalAI;
  - **paid subscriptions, through their official CLIs:** Claude Code, Gemini CLI, any other CLI.

  Any other OpenAI-compatible or custom HTTP API is configured without code (`openai-compatible`, `generic`), or added
  as a plugin.
- **Free tiers first.** Free-tier and subscription models cost 0. samurAI tracks their quotas and, when a model returns
  `429`, falls back to a paid or local model.
- **Automatic model selection.** For each task, the `OPTIMAL` strategy picks the cheapest model that reaches the
  quality target, across all providers. Set the purpose to `AUTO` and samurAI analyses the task itself.
- **Escalation.** With `acceptIf(...)`, an answer that fails validation is retried on a better model.
- **Blind agents.** Agents that use no model, at cost 0:
  - `CommandAgent`, a safe OS command: argument list, no shell, allow-list;
  - `WaitAgent`, which waits for a time, a file, a port or a variable;
  - `HttpAgent`, an HTTP call with SSRF protection;
  - `FunctionAgent`, which runs Java code;
  - `ApprovalAgent`, which waits for a human decision.
- **Composition:** sequence, parallel, router, loop and fallback; function calling (`ToolUseAgent`); RAG with
  per-client isolation (`RagAgent`, `IngestAgent`).
- **Orchestrator:**
  - **runs:** sync, async, run-all, periodic (`every`, `withDelay`, `dailyAt`), with cancellation and timeouts;
  - **limits:** concurrency limits per client, rate limiting, budgets (per client and global), maximum cost per run;
  - **records:** audit log, JSONL traces.
- **Sub-orchestrators.** An orchestrator can own other orchestrators, each with its own threads, configuration and
  schedules — for example a collector that gathers data from the internet every hour. The parent stays responsible for
  the whole operation:
  - it starts and stops them;
  - it totals their costs and enforces its budgets on them;
  - it receives their audit;
  - it restarts them after failures.

  Data flows through shared stores (JSONL, CSV, text).
- **Cost tracking.** Per client, user, session, API key, provider, model, agent, workflow, orchestrator and time
  interval, with a markup factor (default `1`). Monetization is built in: plans, prepaid credits, fair use, outcome
  fees, margins and ROI.
- **Declarative applications.** Configuration, agents, workflows, schedules and sub-orchestrators fit in one XML file,
  validated by an XSD. Task prompts can live in Markdown files and whole directories.
- **samurAI Studio.** A desktop app to open, edit, validate, run and watch XML applications. An attached web server
  (Spring Boot) shows a **real-time dashboard** in the browser.
- **Interfaces:**
  - a REST API with API keys and HMAC-signed webhooks;
  - an **MCP server** (stdio): your workflows become tools for Claude Code, Antigravity, Cursor…;
  - a scriptable CLI with JSON output.
- **Zero runtime dependencies** in the core, the XML module and the studio: only the JDK. Only the optional web server
  (`samurai-studio-web`) uses Spring Boot.

## Downloads

All jars are in the root of this repository.

| Jar | Size | What it is | Run with |
|---|---|---|---|
| `samurai-core-1.0.jar` | 0.5 MB | The framework (library): agents, providers, orchestrator, costs, security, REST, MCP | add to your classpath |
| `samurai-xml-1.0.jar` | 60 KB | XML applications (library); needs `samurai-core` | add to your classpath |
| `samurai-xml-1.0-all.jar` | 0.6 MB | Command line for XML applications, no GUI | `java -jar samurai-xml-1.0-all.jar …` |
| `samurai-studio-1.0-all.jar` | 0.7 MB | **samurAI Studio**: desktop app + the same command line | `java -jar samurai-studio-1.0-all.jar [app.xml]` |
| `samurai-studio-web-1.0.jar` | 20 MB | Studio + attached web server (Spring Boot 4): real-time dashboard | `java -jar samurai-studio-web-1.0.jar app.xml` |
| `samurai-examples-1.0-all.jar` | 0.6 MB | Runnable examples (console, REST server, desktop) | `java -jar samurai-examples-1.0-all.jar` |
| `samurai.xsd` | | XML schema for editor autocompletion | put it next to your `samurai.xml` |

> **Note:** the user-facing messages of version 1.0 (CLI output, studio UI, log lines, error messages) are in Romanian.
> The APIs, configuration keys, XML elements and JSON outputs are language-neutral.

## Requirements

| | |
|---|---|
| Java | **17 or newer**. On Java 21+, samurAI uses virtual threads automatically. |
| Runtime dependencies | None (core, XML, studio). `samurai-studio-web` bundles Spring Boot 4.1. |
| AI access | At least one of these: an API key (e.g. a free Gemini key), a local model server (Ollama…), or a logged-in Claude Code CLI. With no Anthropic key configured, Claude models automatically use the installed `claude` command and your subscription. |

## Installation

### As a library (Maven)

```bash
mvn install:install-file -Dfile=samurai-core-1.0.jar \
    -DgroupId=ro.softway.samurai -DartifactId=samurai-core -Dversion=1.0 -Dpackaging=jar
```

```bash
# optional, for XML applications from Java code
mvn install:install-file -Dfile=samurai-xml-1.0.jar \
    -DgroupId=ro.softway.samurai -DartifactId=samurai-xml -Dversion=1.0 -Dpackaging=jar
```

```xml
<dependency>
    <groupId>ro.softway.samurai</groupId>
    <artifactId>samurai-core</artifactId>
    <version>1.0</version>
</dependency>
```

### As tools

Download `samurai-studio-1.0-all.jar` (and, for the web dashboard, `samurai-studio-web-1.0.jar` into the **same
directory**). Nothing else is needed.

## Quick start

### 1. Configure a provider

Create a `.env` file next to your application (keep it out of version control):

```properties
# a free key from https://aistudio.google.com/apikey
GEMINI_API_KEY=...
```

…and reference it from the configuration (`samurai.properties` or the `<config>` section of `samurai.xml`). Keys are
**never** written in configuration files, only `${VARIABLE}` references:

```properties
samurai.providers.gemini.type=gemini
samurai.providers.gemini.apiKey=${GEMINI_API_KEY}
samurai.models.gemini-flash.provider=gemini
samurai.models.gemini-flash.model=gemini-flash-latest
samurai.models.gemini-flash.freeTier=true
samurai.models.gemini-flash.quality=8
samurai.models.gemini-flash.purposes=CLASSIFICATION,EXTRACTION,SUMMARIZATION,CHAT
```

Configuration is layered, from lowest to highest priority: classpath → `samurai.properties` / `.json` / `.xml` →
`.env` → Docker secrets (`/run/secrets`) → `SAMURAI_*` environment variables → `-Dsamurai.*`.

### 2a. From Java

```java
try (Orchestrator orch = Orchestrator.fromConfig(SamuraiConfig.load()).build()) {
    orch.register("summary", LlmAgent.builder("summarize").prompt("Summarize in 3 bullet points: {{input}}").build());
    RunResult r = orch.run(RunRequest.builder("summary").principal(Principal.of("local", "me", "user"))
            .input("…long text…").build());
    System.out.println(r.output());
}
```

### 2b. From XML, with no code

```bash
java -jar samurai-studio-1.0-all.jar validate app.xml
```

```bash
java -jar samurai-studio-1.0-all.jar list app.xml
```

```bash
java -jar samurai-studio-1.0-all.jar run app.xml summary "…long text…"
```

### 2c. In samurAI Studio

```bash
java -jar samurai-studio-1.0-all.jar app.xml
```

Two complete XML applications are in [`examples/`](examples/):

- [`support`](examples/support/): support-ticket triage, with human approval;
- [`exchange-rates`](examples/exchange-rates/): a sub-orchestrator collects exchange rates from the internet every
  hour, and the main orchestrator analyses them.

## AI providers

| Kind | Types |
|---|---|
| API (paid, many with free tiers) | `openai`, `anthropic`, `gemini`, `vertex`, `azure-openai`, `bedrock`, `mistral`, `cohere`, `groq`, `deepseek`, `xai`, `openrouter`, `together`, `fireworks`, `perplexity`, `cerebras`, `sambanova`, `huggingface`, `github-models`, `cloudflare`, `nvidia` |
| Local | `ollama`, `lmstudio`, `vllm`, `llamacpp`, `localai` (probed at startup; unavailable ones are reported, not fatal) |
| Subscriptions, through official CLIs (personal use only) | `claude-code` (Claude Pro/Max), `gemini-cli`, `cli` (any CLI) |
| Anything else | `openai-compatible`, `generic` (configured with JSON paths, no code), or an `LlmProviderPlugin` |

At startup, one log line summarizes the state of every provider: active, unavailable (with the reason), missing key, and which models are free-tier or subscription.

## Choosing the model

Each LLM agent declares a **purpose**: `CLASSIFICATION`, `EXTRACTION`, `SUMMARIZATION`, `TRANSLATION`, `CHAT`, `CODE`,
`REASONING`, `CREATIVE`, `GENERAL`, or `AUTO`. With `AUTO`, the default, samurAI analyses the task itself. Each model
declares its price and its quality, optionally per purpose.

| Strategy | Picks |
|---|---|
| `OPTIMAL` (default) | the cheapest model whose quality reaches the target for that purpose (within a tolerance) |
| `CHEAPEST` / `BEST_QUALITY` / `BALANCED` | as named |
| `LOCAL_FIRST` / `LOCAL_ONLY` | local models first / only (sensitive data) |

Free-tier, subscription and local models cost 0, so they win whenever they are good enough. If a model fails, the next one is tried; rate limits trigger cool-downs, and a circuit breaker isolates failing providers.

## Costs

Every model call and every blind-agent execution is recorded with its full attribution: client, user, session, API
key, run, workflow, agent, orchestrator, provider and model. Each record holds the tokens, the base cost and the billed
cost. You can then:

- report it, grouped by any of these dimensions or by hour, day or month;
- export it as CSV;
- cap it with budgets (`DAY`, `MONTH`, `TOTAL`, per client or global) and a maximum cost per run;
- bill it with a markup factor per client (default `1`).

The ledger can be in memory or a JSONL file.

```java
System.out.println(orch.costs().report("Costs by model", CostQuery.ALL, GroupBy.MODEL));
```

## samurAI Studio

`java -jar samurai-studio-1.0-all.jar [app.xml]` opens the desktop app.

| Area | What you do |
|---|---|
| **Toolbar** | New (from a template), Open (or drag & drop), Recent, Save, Validate (F7), Load/Reload (F5), schedules on/off, `.env` file |
| **XML editor** | Line numbers; validation errors highlight the offending line |
| **Run** | Pick a workflow (`sub/flow` for sub-orchestrators), input and identity; live progress; result, cost and tokens; cancel |
| **History** | Every run (manual, scheduled, from other orchestrators), with its agent tree, durations, costs and outputs |
| **Monitoring** | Orchestrators, active runs, sub-orchestrators (start/stop, restarts, last error), schedules (run now, cancel) |
| **Events & logs** | Real-time event stream, audit log, framework messages |
| **Costs / Approvals / Data** | Grouped costs and CSV export; approve, edit or reject pending approvals; browse data stores |

### Real-time web dashboard

`samurai-studio-web-1.0.jar` adds a Spring Boot web server that shows, live in the browser:

- runs in progress, step by step;
- the event stream;
- recent runs, with their agent trees;
- sub-orchestrators, schedules, costs, pending approvals and data.

```bash
# server: REST API (/api/v1) + schedules + dashboard (/) on ONE port
java -jar samurai-studio-1.0-all.jar serve app.xml --port 8099
```

```bash
# desktop studio + dashboard
java -jar samurai-studio-web-1.0.jar app.xml --web-port 8099
```

The studio jar contains no web server: `serve` hands the command over to `samurai-studio-web-1.0.jar`. It looks for it in
`SAMURAI_STUDIO_WEB_JAR`, then next to the studio jar. The dashboard is read-only and protected by a per-start token;
it also exposes a JSON API (`/api/state`, `/api/runs/{id}`, `/api/costs`, `/api/data`) and a Server-Sent Events stream
(`/api/events`). Use `--web-port N` to give it a port separate from the REST API.

## Command line

The same commands are available in `samurai-xml-1.0-all.jar`, `samurai-studio-1.0-all.jar` and `samurai-studio-web-1.0.jar`.

| Command | Does |
|---|---|
| `validate <app.xml>` | Checks the schema and builds everything without running anything |
| `list <app.xml>` | Agents, workflows (with descriptions), schedules, stores, sub-orchestrators |
| `run <app.xml> [sub/]<workflow> [text \| @file \| -]` | Runs a workflow once (`-` reads the input from stdin) |
| `serve <app.xml> [--port 8080]` | REST API, schedules, sub-orchestrators (+ dashboard in the studio jars) |
| `mcp <app.xml>` | MCP server on stdio: every workflow becomes a tool |
| `version`, `help` | |

Common options:

| Option | Does |
|---|---|
| `--env <file>` | Selects the `.env` file |
| `--json` | Machine-readable output on stdout; logs go to stderr |
| `--quiet` | Prints only warnings |
| `--client`, `--user`, `--roles`, `--session` | Sets the identity used by `run` |

Exit codes:

| Code | Means |
|---|---|
| `0` | success |
| `1` | invalid XML or failed run |
| `2` | usage error |
| `3` | rejected (access, budget, limits) |

## Using samurAI from other LLMs and agents

samurAI is designed to be driven by other AI assistants:

- **Generating Java code:** [`docs/AI-GUIDE.md`](docs/AI-GUIDE.md) gives an assistant the exact API, decision tables,
  recipes and pitfalls. [`llms.txt`](llms.txt) is the entry point.
- **Running applications, with no Java:** an agent writes `samurai.xml` and validates it with
  `validate --json`. On failure, the output gives the line and the error, so the agent fixes it and validates again.
  It then discovers what can run with `list --json` and runs it with `run --json`. Read the `status`, `output` and
  `cost` fields, and the exit code.
- **As tools:** `java -jar samurai-studio-1.0-all.jar mcp app.xml` exposes every workflow as an MCP tool.
  The `<workflow description>` becomes the tool description.

  ```bash
  claude mcp add samurai -- java -jar /path/samurai-studio-1.0-all.jar mcp /path/app.xml --env /path/.env
  ```
- **Watching:** the dashboard's `GET /api/state` and `GET /api/events` (SSE) stream every run event.

## Security

- **Keys:** only as `${VAR}` references in configuration. The values live in `.env`, environment variables or Docker
  secrets. A key written literally in XML triggers a warning, and secrets are masked in logs and audit records.
- **OS commands:**
  - run as argument lists, without a shell, in an empty working directory, with a restricted environment;
  - limited by an allow-list and timeouts;
  - a sub-orchestrator can never allow more commands than its parent.
- **Network:** `HttpAgent` blocks private networks (SSRF) unless explicitly allowed, and supports host allow-lists.
- **XML:** the parser rejects DOCTYPE and external entities (XXE). File paths cannot leave the application directory,
  and Java classes are instantiated only from allowed packages.
- **Access control:**
  - role-based access per workflow, rate limiting per client, concurrency limits;
  - guardrails: prompt injection, PII redaction, secrets, length, block-lists;
  - human approval for critical actions.
- **Servers:**
  - REST, the dashboard and MCP bind to `127.0.0.1` by default;
  - REST requires API keys, the dashboard a per-start token;
  - a port already used by another application is refused, even when that application listens on all interfaces.
- **Claude subscriptions:** `claude-code` and the automatic Claude CLI fallback are for the subscriber's personal use.
  For applications that serve third parties, configure an API key and set `samurai.llm.claudeCliFallback=false`.

See [SECURITY.md](SECURITY.md) to report a vulnerability.

## Documentation

| File | Content |
|---|---|
| [README.md](README.md) | This overview |
| [docs/CONFIGURATION.md](docs/CONFIGURATION.md) | Configuration: sources, providers, models, routing, costs, security, web, MCP, Docker, all keys |
| [docs/USAGE.md](docs/USAGE.md) | Using the Java API: agents, chaining, tools, RAG, approvals, runs, schedules, sub-orchestrators, costs, REST, MCP, tests |
| [docs/XML.md](docs/XML.md) | XML applications, the command line, samurAI Studio and the web dashboard |
| [docs/AI-GUIDE.md](docs/AI-GUIDE.md) | Reference for AI assistants: exact API, rules, decision tables, recipes, pitfalls |
| [docs/BUSINESS.md](docs/BUSINESS.md) | Integrating AI into business processes, monetization, ROI |
| [llms.txt](llms.txt) | Entry point for LLMs |
| [CHANGELOG.md](CHANGELOG.md) | Release notes |
| [examples/](examples/) | Complete XML applications |

## Third-party notices

The core, XML and studio jars contain only samurAI code; they have no third-party dependencies.

`samurai-studio-web-1.0.jar` redistributes the following open-source components, under their own licenses:

| Component | Version | License |
|---|---|---|
| Spring Boot | 4.1.1 | Apache License 2.0 |
| Spring Framework | 7.0.9 | Apache License 2.0 |
| Apache Tomcat (embedded) | 11.0.24 | Apache License 2.0 |
| Jackson (core, databind / annotations) | 3.1.5 / 2.21 | Apache License 2.0 |
| Micrometer (commons, observation) | 1.17.1 | Apache License 2.0 |
| SnakeYAML | 2.6 | Apache License 2.0 |
| Apache Log4j API, Log4j-to-SLF4J | 2.25.5 | Apache License 2.0 |
| Apache Commons Logging | 1.3.6 | Apache License 2.0 |
| JSpecify | 1.0.1 | Apache License 2.0 |
| Logback (classic, core) | 1.5.38 | EPL 1.0 / LGPL 2.1 |
| SLF4J (API, JUL bridge) | 2.0.18 | MIT |
| Jakarta Annotations API | 3.0.0 | EPL 2.0 / GPL 2.0 with Classpath Exception |

The full list is in [THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md).

---

<p align="center">samurAI is developed by <a href="https://www.softway.ro">softway.ro</a>.</p>
