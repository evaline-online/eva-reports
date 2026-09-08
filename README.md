# eva-reports
**Архив отчётов системы — строгая хронология. Один файл = один отчёт. Ноль дублей.**

## Хронологический индекс
| Дата | Отчёт | Файл / Источник |
|---|---|---|
| 2026-09-06 | System Optimization | `reports/2026-09-06-system-optimization.md` |
| 2026-09-06 | EvaBot System Analysis | `reports/2026-09-06-evabot-system-analysis.md` |
| 2026-09-06 | OmniRoute Free Coding Models | `reports/2026-09-06-omniroute-free-coding-models.md` |
| 2026-09-07 | OmniRoute Free Coding Models | `reports/2026-09-07-omniroute-free-coding-models.md` |
| 2026-09-07 | OmniRoute Free Coding Rating | `reports/2026-09-07-omniroute-free-coding-rating.md` |
| 2026-09-07 | Coding Top Models | `reports/2026-09-07-coding-top-models.md` |
| 2026-09-07 | Full Session Report (EN/RU/UK) | `reports/2026-09-07-full-session-report{,.en,.ru,.uk}.md` |
| 2026-09-07 | Session Final Report | `reports/2026-09-07-session-final-report.md` |
| 2026-09-07 | GCloud Servers Audit | → `evaline-server/reports/` (модуль edge-ноды) |
| 2026-09-07 | SSH Bridge Report ×2 | → `evaline-server/bridge/` (модуль моста) |
| 2026-09-08 | Coverage & Routing | `reports/2026-09-08-coverage-and-routing.md` |
| живой | Model Monitor | → `evabot-online/data/model-monitor/REPORT.md` (генерируется таймером) |

## Правила
1. **Именование:** `YYYY-MM-DD-<краткое-имя>.md`. Если дата неизвестна — берётся дата создания.
2. **Каноничность:** отчёты, привязанные к модулю (edge, bridge, monitor), хранятся в своём модуле; здесь — только ссылка в индексе.
3. **Не дублировать:** перед добавлением сверить md5 с существующими.
4. Белая книга (40K) = `eva-docs/architecture/CAPABILITIES_MANIFESTO.md` (ранее дублировалась как WHITEPAPER).
