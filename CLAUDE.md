# CLAUDE.md — Token Dashboard

_Laatst bijgewerkt: 2026-04-21 16:41_

Projectspecifieke context voor Claude Code. Alles wat je moet weten over het Token Dashboard staat in deze projectmap.

## Wat is dit

Lokaal token/cost dashboard voor Claude Code sessies. Leest de JSONL-transcripten die Claude Code naar `~/.claude/projects/` schrijft en tovert ze om tot:

- per-prompt kostenanalyse en tool/file heatmaps
- subagent attributie + cache analytics
- project-vergelijking + skills overzicht
- rule-based tips engine voor token-waste

Alles draait lokaal — geen telemetry, geen remote calls, geen login. Pure Python stdlib + vanilla JS, dus geen `pip install`, geen Node.js, geen build-step.

## Metech setup

- **Fork**: `Metech-dev/Token_Dashboard` (upstream: `nateherkai/token-dashboard`)
- **Lokaal pad**: `~/.claude/projects/Token_Dashboard/`
- **Auto-sync**: zit in `/sync-all` (pullt bij session start)
- **SQLite cache**: `~/.claude/token-dashboard.db` (staat in `.gitignore`)

## Starten

Via slash command (aanbevolen):

```
/token-dashboard
```

Of handmatig:

```bash
cd ~/.claude/projects/Token_Dashboard
python cli.py dashboard
```

Wat er gebeurt:
1. Scant `~/.claude/projects/` (eerste run: 20–60 sec)
2. Start lokale server op **http://127.0.0.1:8080**
3. Opent browser automatisch
4. Refresht elke 30 sec via SSE

Stoppen: `Ctrl+C` in de terminal.

**Windows**: als `python` niet op PATH staat → `py -3` gebruiken. Als alleen `python3` werkt → dat overal invullen.

## Env vars

| Var | Default | Doel |
|---|---|---|
| `PORT` | `8080` | Poort voor de lokale webserver |
| `HOST` | `127.0.0.1` | Bind address. **Nooit `0.0.0.0` zetten** — dat legt je volledige prompthistorie open op het LAN |
| `CLAUDE_PROJECTS_DIR` | `~/.claude/projects` | Waar JSONL-sessies gescand worden |
| `TOKEN_DASHBOARD_DB` | `~/.claude/token-dashboard.db` | SQLite cache-locatie |

Pricing staat in [`pricing.json`](pricing.json) — direct editen als modelprijzen wijzigen of voor een nieuw plan.

## CLI reference

```bash
python cli.py scan          # alleen DB verversen, geen server
python cli.py today         # vandaag's totals in terminal
python cli.py stats         # all-time totals in terminal
python cli.py tips          # actieve suggesties in terminal
python cli.py dashboard     # scan + server op :8080

# flags
python cli.py dashboard --no-open   # geen browser openen
python cli.py dashboard --no-scan   # DB-only (geen rescan)
```

## De 7 tabs

1. **Overview** — all-time tokens, dagelijkse work/cache charts, per-project, per-model, top tools, recente sessies
2. **Prompts** — duurste prompts geranked, klik-through naar assistant response + tool calls
3. **Sessions** — turn-by-turn view per sessie
4. **Projects** — per-project tokens, session counts, meest aangeraakte files
5. **Skills** — welke skills je vaakst invoked (+ token cost waar meetbaar — zie [limitations](docs/KNOWN_LIMITATIONS.md#skills-token-counts-are-partial))
6. **Tips** — rule-based waste-suggesties (herhaalde file reads, oversized tool results, lage cache-hit)
7. **Settings** — API / Pro / Max / Max-20x pricing switch

## Architectuur

```
cli.py → token_dashboard/scanner.py → ~/.claude/token-dashboard.db (SQLite)
                                     ↓
                    token_dashboard/server.py exposes /api/* + SSE /api/stream
                                     ↓
                            web/ (vanilla JS + ECharts, no build)
```

- **Incrementeel scannen**: `files` tabel tracked mtime + byte offset per sessie, scanner leest alleen nieuwe bytes
- **Streaming dedup**: Claude Code schrijft elke assistant response 2–3× naar disk (snapshots tijdens streaming). Scanner dedupliceert op `(session_id, message_id)` — zie `scanner._evict_prior_snapshots` en migratie-note in `db._migrate_add_message_id`

## Conventies (bij code-wijzigingen)

- **Volledig lokaal**. Geen telemetry, geen remote calls voor user data. Tests draaien offline.
- **Alleen stdlib**. Geen `pip install`. Nieuwe third-party dep? Eerst bespreken — we betalen ergonomie-kosten om install friction op nul te houden.
- **SQLite parameter binding altijd**. F-strings in SQL mogen alleen interne, caller-controlled waarden interpoleren (kolomnamen, placeholder lists). User-reachable values via `?`.
- **Kleine files, heldere verantwoordelijkheden**. Boven ~400 regels of 3+ concerns → splitsen.
- **Streaming-snapshot dedup**: bij scanner-logic die `messages` joined, onthoud dat `(session_id, message_id)` de dedup-key is — niet `uuid`.

## Verifiëren

```bash
python -m unittest discover tests        # 68 unit tests
python cli.py dashboard --no-open        # start zonder browser
curl http://127.0.0.1:8080/api/overview  # endpoint sanity-check
```

## Upstream sync

Fork is gekoppeld via `upstream` remote naar `nateherkai/token-dashboard`:

```bash
cd ~/.claude/projects/Token_Dashboard
git fetch upstream
git merge upstream/main
git push origin main
```

## Troubleshooting

| Probleem | Fix |
|---|---|
| `python: command not found` | `winget install Python.Python.3.12` of python.org |
| Port 8080 in gebruik | `PORT=8090 python cli.py dashboard` |
| "No data" / lege charts | `python cli.py scan` om DB te populeren, dan reload |
| Numbers look wrong | DB weggooien (`~/.claude/token-dashboard.db`) + `python cli.py scan` |
| Twee dashboards tegelijk | Niet doen — SQLite DB-lock conflict. Kill bestaande instance eerst |

## Known limitations

Zie [`docs/KNOWN_LIMITATIONS.md`](docs/KNOWN_LIMITATIONS.md). Samenvatting: Skills `tokens_per_call` wordt alleen gevuld voor skills onder de drie gescande roots (`~/.claude/skills/`, `~/.claude/scheduled-tasks/`, `~/.claude/plugins/`). Project-lokale skills en subagent-dispatched skills tonen invocation counts maar lege token counts.

## Accuracy

Dashboard dedupliceert streaming snapshots op `message.id`, zodat totalen matchen met wat de API daadwerkelijk gefactureerd heeft. Vergelijk je met een tool dat elke JSONL-regel optelt → dit dashboard zit lager, maar dichter bij de realiteit.

## Verder lezen

- [`README.md`](README.md) — publieke README (upstream-georiënteerd)
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — dev workflow + tests
- [`docs/KNOWN_LIMITATIONS.md`](docs/KNOWN_LIMITATIONS.md) — rough edges
- [`docs/inspiration.md`](docs/inspiration.md) — prior art (phuryn/claude-usage) en verschillen
