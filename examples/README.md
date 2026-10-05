# Examples

Complete samurAI applications defined only in XML. Run them with any of the command-line jars, for example:

```bash
java -jar samurai-studio-1.0-all.jar validate examples/support/samurai.xml
```

```bash
java -jar samurai-studio-1.0-all.jar examples/support/samurai.xml      # open in samurAI Studio
```

Workflows that use a language model need a provider. Put your keys in a `.env` file in the directory you run from,
for example a free Gemini key: `GEMINI_API_KEY=...`. Workflows made only of blind agents cost nothing and need no key.

| Example | What it shows |
|---|---|
| [support](support/) | Classification with a cheap model; routing by category; a reply prompt kept in a Markdown file; human approval for complaints. Every file in `tasks/reports/` becomes its own workflow (front matter sets purpose and description). Also a system check made only of blind agents: two commands in parallel, then a wait. |
| [exchange-rates](exchange-rates/) | A main orchestrator with a **sub-orchestrator** (`collector/collector.xml`) that downloads the ECB exchange rates every hour from a public API and appends them to a shared JSONL store. The main orchestrator reads the store, analyses the trend with a model, or triggers a collection on demand (`<orchestrator-call>`). Includes restart-on-failure supervision and a monthly budget for the collector. |

`samurai.xsd` sits next to each file, so XML editors (IntelliJ IDEA, VS Code, Eclipse) offer autocompletion and
validation.
