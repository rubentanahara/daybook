# Glossary

| Term | Meaning |
|---|---|
| Generate | The user action that fetches activity, builds the prompt and calls the AI provider. |
| Test connection | The user action that checks a CLI provider: `claude auth status`, then one fixed prompt with no repository data. |
| `CliProvider` | The single adapter that spawns a user's installed CLI with a per-CLI argument builder. V1 has one argument builder, for Claude Code. |
| Configured | A provider whose stored path exists and is executable, or whose key is stored. Not proof it works. |
| Ready | A provider whose Test connection succeeded. A failed Generate demotes it to configured. |
| Gate | A test with a pass condition and a fallback decided in advance. A feature whose gate fails takes its fallback, or moves to V1.x. |
| Golden set | Fixture repos with expected citations, used to measure Trust and Coverage. |
| Trust | Share of generated bullet lines with at least one source ID that exists in the run's fetched set and falls inside the report window, counted on the model's raw output. |
| Coverage | Share of expected activity items on the golden set that appear in the standup. |
| Source chip | The control on each generated line that shows the commit or ticket it came from. |
| Exception (E1 to E5) | A named way data leaves the machine, listed in ADR-0003. |
