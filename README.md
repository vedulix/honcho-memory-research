# AI Memory Platforms Research

Исследование платформ долгосрочной памяти для AI-компаньонов enterprise-масштаба.

## Начните здесь

**[Единый отчёт: 14 платформ памяти →](06-unified-research.md)** — TL;DR, сравнительная таблица, decision matrix, detailed profiles, roadmap.

## Все документы

| Документ | Что внутри |
|----------|-----------|
| [**06 — Единый отчёт**](06-unified-research.md) | Финальный документ. 14 платформ, decision matrix, Honcho coverage matrix, roadmap |
| [05 — Deep Research Report](05-deep-research-report.md) | Детальные профили 12 платформ (из structured JSON) |
| [04 — Обзор решений](04-memory-solutions-comparison.md) | Первая итерация: 20+ решений, community insights из Telegram |
| [03 — Mem0 Feasibility](03-mem0-feasibility.md) | Глубокий анализ mem0 |
| [02 — Архитектура Honcho](02-memory-architecture.md) | 6 механизмов, промпты, JSON, схемы |
| [04 — Honcho vs Self-Hosted](04-implementation-comparison.md) | Покомпонентная замена Honcho |

## Raw Data

- [`results/`](results/) — 14 structured JSON-файлов (по одному на платформу)
- [`fields.yaml`](fields.yaml) — определения полей исследования
- [`outline.yaml`](outline.yaml) — список объектов исследования

## TL;DR

Ни одна платформа не покрывает все 6 механик Honcho. Три кандидата на замену:

| | Hindsight | EverMemOS | mem0 |
|---|---|---|---|
| Механики | 9/12 | 10/12 | 3/12 |
| Лицензия | MIT | Apache 2.0 | Apache 2.0 |
| Инфра | PostgreSQL only | Mongo + ES + Milvus | 23+ vector backends |
| Риск | Молодой проект | Мало документации | 70% строить самим |
