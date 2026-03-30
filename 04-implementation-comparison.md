---
layout: default
title: "Honcho vs Self-Hosted: полный разбор замены компонентов"
---

# Honcho vs Self-Hosted: полный разбор замены компонентов

Дата исследования: 2026-03-26

---

## 1. Executive Summary

### Контекст

Honcho (Plastic Labs) -- open-source фреймворк интерпретивной памяти для AI-ассистентов: экстракция фактов, офлайн-анализ, семантический поиск, карточка юзера. Документ описывает как построить аналогичную систему на enterprise-стеке для миллионов пользователей.

### Ключевые выводы

- Каждая механика Honcho имеет прямую замену на enterprise-стеке (Managed PostgreSQL + pgvector, LLM, Managed Kafka, Managed AI Platform Assistant API)
- pgvector 0.8.0 на Managed PostgreSQL -- лучший выбор для старта; Qdrant или OpenSearch -- при масштабе >100M векторов
- YDB Vector Search пока не готов для production (нет INSERT/UPDATE на таблицах с индексами)
- Mem0 (Apache 2.0) -- наиболее зрелый open-source фреймворк памяти, архитектура которого напрямую ложится на enterprise-стек с заменой провайдеров
- Batch import исторических данных (Streaming Platform, Music Service, Book Service) -- главное конкурентное преимущество Enterprise, недоступное стартапам

### Рекомендация

- Начинать с Managed PostgreSQL + pgvector + LLM Lite + asyncio.Queue (прототип за 1-4 недели), масштабировать через Kafka + Dagster + BGE-M3 self-hosted

### Для кого этот документ

Архитекторы и разработчики, проектирующие систему интерпретивной памяти для AI-ассистента. Покрывает: архитектуру, выбор технологий, лицензионную чистоту, этапность внедрения.

---

## 2. Сводная таблица

| Механика Honcho | Что делает | Open-source замена | Enterprise stack |
|---|---|---|---|
| **Deriver** | Фоновая экстракция фактов из сообщений | Kafka/Redis + любой LLM + pgvector (дедупликация) | Managed Kafka + LLM Lite + pgvector |
| **Dreaming (дедукция)** | Логические выводы, разрешение противоречий, обновление карточки | Cron/Dagster + LLM (Qwen3/Llama) + structured output | Dagster на Cloud Platform + LLM Pro (Chain-of-Reasoning) |
| **Dreaming (индукция)** | Поиск паттернов через множество фактов, уровни уверенности | Отдельный LLM-агент + min 2 sources constraint | LLM Pro + кастомный промпт |
| **Conclusions (хранение)** | Выводы с типом, посылками, уверенностью, embedding | PostgreSQL + pgvector (HNSW) | Managed PostgreSQL + pgvector 0.8 |
| **Peer Card** | Гарантированная визитка юзера в каждом промпте | PostgreSQL JSONB | Managed PostgreSQL JSONB |
| **Prefetch** | Сборка контекста: card + top-K conclusions + summary | pgvector cosine search + Redis cache | pgvector + Managed Memcached/Redis |
| **Semantic search** | Поиск релевантных conclusions по embedding сообщения | pgvector HNSW / Qdrant | pgvector / Managed OpenSearch k-NN |
| **Embeddings** | Векторизация текста для поиска | BGE-M3 (1024d, CPU) / GigaEmbeddings (2048d, GPU) | Managed AI Platform embeddings (256d, API) |
| **Суммаризация сессий** | Сжатие истории: short (каждые 20 msg) + long (каждые 60) | LLM + counter trigger | LLM Lite + counter trigger |
| **Dialectic (peer.chat)** | ReAct-агент с доступом к памяти, отвечает на вопросы | LangChain ReAct + tools (search, get_card) | LangChain + LLM Pro + tools |
| **Batch import** | Загрузка исторических данных (Streaming Platform, Music Service) | Dagster pipeline + cluster -> summarize | Dagster + Managed Kafka + LLM Lite |
| **Приоритизация фактов** | Какие conclusions важнее (Honcho НЕ умеет) | Теги + re-ranker + Peer Card как VIP-слой | То же + ARGUS embeddings как сигнал важности |
| **Observation control** | Какие сущности наблюдать, а какие нет | Флаг `observe=true/false` на юзере | То же |
| **Thread management** | Сессии, ротация, история | Своя реализация на PostgreSQL | Managed AI Platform Assistant API (threads, messages, runs) |

---

## 3. Детальный разбор каждой механики

### 3.1. Deriver -- экстракция фактов

**Продуктовый смысл:** без deriver бот -- рыбка с памятью 3 секунды. Юзер рассказал что любит Тарковского -- а бот через 5 минут забыл. Deriver превращает поток сообщений в структурированное знание о юзере. Это фундамент всей персонализации.

**Что в Honcho:** воркер поллит очередь, отправляет сообщение в LLM, получает JSON с фактами, дедуплицирует через cosine similarity, сохраняет.

**Open-source:**
- **Очередь:** Redis Streams (простой) или Kafka (масштабируемый)
- **LLM:** Qwen3-8B через vLLM (дешево, быстро) или Llama 3.1 8B
- **Дедупликация:** pgvector -- embed факт, поискать с threshold 0.9, если нашел похожий -- пропустить
- **Embedding:** BGE-M3 (1024d, работает на CPU, хорошо для русского -- 60.8 на ruMTEB)

**Enterprise stack (с учетом масштаба на миллионы):**
- **Очередь:** Managed Kafka (~$162/мес за 3-брокер кластер, версии 3.6-4.0). Партиционирование по user_id -- каждый partition обрабатывается своим воркером, юзеры не блокируют друг друга
- **LLM:** LLM Lite (0.20 руб/1K токенов, 32K контекст). При 10M сообщений/день x ~500 токенов = ~1000 руб/день на экстракцию
- **Дедупликация:** Managed PostgreSQL (PG 16-18) + pgvector 0.8.0 (HNSW, до 16000 измерений, cosine/L2/inner product). При миллионах юзеров -- sharding по user_id (каждый юзер в своей партиции, дедупликация только внутри юзера)
- **Embedding:** Managed AI Platform embeddings (256d, модели text-search-doc / text-search-query) для прототипа. BGE-M3 self-hosted для прода -- можно батчить, не зависеть от API rate limits
- **Горизонтальное масштабирование:** N воркеров x N Kafka partitions. Линейно масштабируется добавлением воркеров

**Совет:** для прототипа -- asyncio.Queue + LLM Lite + pgvector. Kafka + N воркеров на этапе масштабирования.

---

### 3.2. Dreaming -- офлайн-анализ

**Продуктовый смысл:** deriver записывает сырые факты, но факты сами по себе мало полезны. "Юзер упомянул Inception" и "юзер упомянул Arrival" -- это два отдельных факта. Dreaming превращает их в инсайт: "юзер ценит интеллектуальный sci-fi". Это то, что делает рекомендации точными, а разговор -- глубоким. Без dreaming бот знает *что* юзер говорил, но не понимает *кто* юзер.

**Что в Honcho:** два отдельных агента (дедукция + индукция), запускаются по триггерам (50 фактов / 60 мин тишины / 8ч интервал). Дедукция обновляет карточку и разрешает противоречия. Индукция ищет паттерны.

**Open-source:**
- **Оркестрация:** Dagster (asset-centric модель хорошо ложится на memory-артефакты) или Airflow
- **Дедукция:** LLM с structured output (JSON: premises -> conclusion). Qwen3-72B для качества или Qwen3-8B для скорости
- **Индукция:** отдельный LLM-вызов с constraint "минимум 2 источника на паттерн"
- **Триггеры:** Dagster sensors или простой cron + счетчик в Redis

**Enterprise stack (с учетом масштаба на миллионы):**
- **Оркестрация:** Dagster на Cloud Platform Compute или Managed Airflow
- **LLM:** LLM Pro с Chain-of-Reasoning (0.80 руб/1K токенов, 128K контекст). Альтернатива: Alternative LLM (0.50 руб/1K input, 1.20 руб/1K output, 32K контекст) -- #1 на SLAVA benchmark
- **Триггеры:** Managed Kafka events -> Dagster sensor
- **Масштабирование:** dreaming = 1 LLM-вызов на юзера за цикл. При 1M активных юзеров -- не вызывать для всех. Приоритизировать: dream только для юзеров, у которых накопилось 50+ новых фактов с последнего dream. Остальные ждут. Это снижает нагрузку с 1M до ~10-50K вызовов за цикл
- **Стоимость при масштабе:** 50K юзеров x ~2K токенов x 0.80 руб/1K = ~80K руб за dream-цикл. При ежедневном цикле -- ~2.4M руб/мес. Оптимизация: LLM Lite для дедукции (0.20 руб), Pro только для индукции

**Совет:** начинайте только с дедукции. Индукцию добавлять когда у юзера 100+ фактов. На 10 фактах индукция бесполезна.

---

### 3.3. Conclusions -- хранение выводов

**Продуктовый смысл:** conclusions -- это "мозг" системы. Не сырые факты ("упомянул Inception"), а осмысленные выводы ("ценит интеллектуальный sci-fi с философской глубиной"). Именно conclusions определяют качество рекомендаций, тон общения и способность бота удивлять юзера точным пониманием.

**Что в Honcho:** выводы с типом (deductive/inductive/abductive), посылками, уверенностью, embedding. Хранятся в PostgreSQL + pgvector.

**Open-source:**
```sql
CREATE TABLE conclusions (
    id UUID PRIMARY KEY,
    user_id TEXT NOT NULL,
    type TEXT NOT NULL,            -- 'deductive', 'inductive', 'abductive'
    content TEXT NOT NULL,
    premises JSONB,                -- ["факт 1", "факт 2"]
    confidence TEXT,               -- 'low', 'medium', 'high'
    embedding VECTOR(1024),        -- BGE-M3
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX ON conclusions USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON conclusions (user_id);
```

**Enterprise stack (с учетом масштаба):**
- Managed PostgreSQL (PG 16-18) + pgvector 0.8.0
- pgvector 0.8.0 возможности: HNSW и IVFFlat индексы, half-precision vectors (halfvec), binary vectors (bit), sparse vectors (sparsevec), параллельная сборка индексов, полный CRUD
- При 1M юзеров x ~200 conclusions = 200M записей. pgvector справится с sharding по user_id (каждый shard ~20M записей)
- При 10M+ юзеров -- миграция на Qdrant (1B+ векторов, 8ms latency) или Managed OpenSearch k-NN (NMSLIB HNSW / FAISS, hybrid search BM25 + vector)

**Совет:** pgvector -- лучший выбор на старте. Простой, проверенный, полный CRUD. Qdrant или OpenSearch -- при >100M векторов.

---

### 3.4. Peer Card -- визитка юзера

**Продуктовый смысл:** Peer Card -- это гарантия что бот никогда не забудет базовое. Имя юзера, его ключевые предпочтения, инструкции ("не рекомендуй хорроры"). Без Peer Card бот может в одном сообщении назвать по имени, а в следующем спросить "как тебя зовут?". Peer Card -- это zero-latency персонализация, которая работает даже без vector search.

**Что в Honcho:** массив строк `["Name: ...", "PREFERENCE: ...", "TRAIT: ..."]`. Перезаписывается целиком. Всегда в промпте.

**Реализация (open-source и Enterprise stack одинаково):**
```sql
CREATE TABLE peer_cards (
    user_id TEXT PRIMARY KEY,
    facts JSONB NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

Просто JSONB в PostgreSQL. При миллионах юзеров -- кешировать в Redis/Memcached (одна карточка ~1KB, 1M юзеров = ~1GB в кеше -- легко). Read latency <1ms.

---

### 3.5. Prefetch -- сборка контекста

**Продуктовый смысл:** prefetch -- это момент, когда память превращается в действие. Юзер написал про музыку -- и бот уже знает что он слушает synthwave, не любит попсу, и на прошлой неделе обсуждал Stranger Things. Без prefetch память лежит мертвым грузом в базе. Prefetch делает бота "вспоминающим".

**Что в Honcho:** один вызов `session.context()` возвращает card + top-K conclusions + summary. Семантический поиск по текущему сообщению.

**Open-source:**
```python
async def prefetch(user_id, message):
    card = await redis.get(f"card:{user_id}") or await db.get_card(user_id)
    embedding = await embed(message)  # BGE-M3
    conclusions = await db.vector_search(embedding, user_id, top_k=12)
    summary = await db.get_summary(user_id)
    return format_context(card, conclusions, summary)
```

**Enterprise stack (с учетом масштаба):** то же, но `embed()` через Managed AI Platform API (для прототипа) или self-hosted BGE-M3 (для прода -- нет rate limits). При миллионах юзеров prefetch -- это hot path: каждое сообщение = 1 embedding + 1 vector search + 1 cache read. Latency budget: <50ms total. Решение: card из Redis (<1ms) + pgvector search с индексом по user_id (<15ms) + embed через локальный BGE-M3 (<20ms).

---

### 3.6. Semantic Search -- поиск по выводам

**Продуктовый смысл:** юзер спросил про кино -- бот должен вытащить conclusions про кино, а не про музыку. Семантический поиск делает память *контекстной* -- бот "вспоминает" то, что релевантно именно сейчас. Без этого пришлось бы загружать все 200 conclusions в промпт -- дорого и шумно.

**Что в Honcho:** cosine similarity, top_k, max_distance, include_most_frequent.

**Сравнение вариантов:**

| Вариант | Latency (p50) | QPS | CRUD | Масштаб | Managed в Enterpriseе? |
|---------|--------------|-----|------|---------|-------------------|
| **pgvector HNSW** | ~15ms | ~500 | Да | До ~10M | Да (Managed PG) |
| **Qdrant** | ~8ms | ~1500 | Да | До ~1B | Marketplace (VM) |
| **OpenSearch k-NN** | ~20ms | ~1000 | Да | До ~1B | Да (Managed) |
| **YDB vector** | -- | -- | Нет (!) | ~1B+ | Да, но blocker |

**Детали по YDB Vector Search (v25.2, июль 2025):**
- Поддержка типов: Float, Int8, Uint8, Bit
- Три метода: exact (brute force, полный CRUD), approximate без индекса (полный CRUD, scalar quantization 32x), approximate с индексом (быстрый, но INSERT/UPDATE/DELETE не поддерживаются)
- Global vector index в roadmap (GitHub issue #8967)
- **Вердикт: не готов для production memory system**

**Рекомендация:** pgvector на старте, Qdrant или OpenSearch при >10M векторов.

---

### 3.7. Embeddings -- векторизация

**Продуктовый смысл:** embeddings -- это "язык", на котором работает семантический поиск. Чем качественнее embeddings для русского текста -- тем точнее бот находит релевантные facts. Плохие embeddings = "юзер спросил про Тарковского, а бот вспомнил про тарелки".

**Сравнение моделей:**

| Модель | Размерность | Max Tokens | Params | ruMTEB | Русский | Где работает |
|--------|------------|------------|--------|--------|---------|-------------|
| **GigaEmbeddings** | 2048 | 4096 | 3B | **69.1** (SOTA) | Лучший | GPU (self-hosted) |
| **BGE-M3** | 1024 | 8192 | ~560M | 60.8 | Хорошо (74.8 retrieval) | CPU (self-hosted) |
| **E5-mistral-7b-instruct** | 4096 | 32768 | 7B | 64.9 | Хорошо | GPU (тяжелый) |
| **mE5-large-instruct** | 1024 | 512 | 560M | 64.7 | Хорошо | CPU |
| **ru-en-RoSBERTa** | ~768 | 512 | 404M | 60.4 | 66.5 (retrieval) | CPU |
| **Managed AI Platform** | 256 | N/A | N/A | N/A | Да | Managed API |
| **OpenAI ada-002** | 1536 | -- | -- | -- | Да | API (недоступен) |

**Managed AI Platform Embeddings -- детали:**
- Модели: `text-search-doc` (для документов), `text-search-query` (для запросов)
- API: `https://llm.api.cloud.enterprise.net/foundationModels/v1/textEmbedding`
- Model URI: `emb://<folder_id>/text-search-doc/latest`
- Интеграции: LangChain (`CustomEmbeddings`), LlamaIndex (`LLMEmbedding`), Managed AI Platform SDK

**Self-hosted inference варианты:**
- **FastEmbed** (Qdrant) -- оптимизированный ONNX runtime, быстрый CPU inference
- **TEI** (HuggingFace) -- Docker container, batched inference
- **Ollama** -- простой деплой, BGE-M3 нативно (`ollama pull bge-m3`)

**Рекомендация:** Managed AI Platform (256d) для прототипа -- проще всего. BGE-M3 для прода -- лучший баланс качество/стоимость на русском. GigaEmbeddings если нужен максимум качества и есть GPU.

---

### 3.8. Суммаризация сессий

**Продуктовый смысл:** юзер общался с ботом 3 месяца, 500 сообщений. Загрузить все в промпт невозможно (дорого + не влезет). Суммаризация сжимает историю: бот "помнит" что было, но не дословно, а по сути. Юзер чувствует непрерывность общения.

**Что в Honcho:** short summary каждые 20 сообщений, long каждые 60. Автоматически в фоне.

**Реализация одинаковая для open-source и Enterprise:**
- Счетчик сообщений в сессии
- При достижении порога -- LLM-вызов для суммаризации
- Хранение в PostgreSQL

**LLM:** LLM Lite (дешево, для суммаризации хватит) или Qwen3-8B self-hosted.

---

### 3.9. Dialectic -- агентный reasoning по памяти

**Продуктовый смысл:** dialectic -- это то, что позволяет боту быть проактивным. Не просто отвечать на вопросы, а самому решать: "есть ли повод написать юзеру?", "что посоветовать на основе всего что знаю?". Это механизм для heartbeat (написать первым), nudge (продолжить тему), и глубоких рекомендаций.

**Что в Honcho:** ReAct-агент с tools (search_conclusions, get_card, search_sessions). 5 уровней глубины.

**Open-source:**
- **Framework:** LangChain ReAct agent (есть официальный `CustomEmbeddings` класс)
- **Tools:** search_conclusions (pgvector query), get_card (PostgreSQL read), search_sessions (text search)
- **LLM:** Qwen3-72B для max quality или LLM Pro

**Enterprise stack:**
- **Framework:** LangChain + LLM Pro. Managed AI Platform SDK имеет `.langchain()` метод для нативной интеграции
- **LLM:** LLM Pro с Chain-of-Reasoning для сложных запросов, Lite для простых. Альтернатива: Alternative LLM для диалоговых задач
- **Tools:** Search Index Tool, Generative Search Tool, Web Search Tool, Function Tool (все доступны через Managed AI Platform)

---

### 3.10. Batch Import -- загрузка исторических данных

**Продуктовый смысл:** это главное конкурентное преимущество Enterprise. Ни один стартап (Honcho, Mem0) не имеет доступа к 10-летней истории просмотров Streaming Platform, миллиону плейлистов Музыки, профилям Книг. Batch import превращает эти данные в персонализацию с первого сообщения -- юзер даже не успел ничего рассказать, а бот уже знает его вкус.

**Что в Honcho:** batch create conclusions (макс 100 за раз). Или import messages -> deriver обработает.

**Для enterprise-масштаба нужен принципиально другой подход** (100K+ записей на юзера):

```
Streaming Platform оценки -> группировка по жанру -> LLM суммаризация кластера -> conclusions
Music Service плейлисты -> группировка по стилю -> LLM суммаризация -> conclusions
Book Service -> группировка по автору/жанру -> LLM суммаризация -> conclusions
```

**Open-source:** Dagster pipeline + Kafka + LLM batch API

**Enterprise stack (с учетом масштаба):** Dagster на Compute + Managed Kafka + LLM Lite batch. 10K оценок -> ~50 кластеров -> 51 LLM-вызов на юзера. При 1M юзеров: 51M batch-вызовов. При 0.20 руб/1K токенов x ~500 токенов/вызов = ~5M руб. Разовая инвестиция, не recurring. Kafka партиционирует по user_id -- параллелизация на десятки воркеров.

**Контекст персонализации Enterprise:**
- ARGUS (AutoRegressive Generative User Sequential modeling) -- трансформер до 1B параметров, обрабатывает до 8192 взаимодействий юзера, dual-objective обучение. На Music Service: +2.26% время прослушивания, +6.37% лайки
- Yambda Dataset (май 2025) -- ~5 млрд анонимизированных взаимодействий из Music Service, крупнейший открытый recommender dataset
- Alternative "My Memory" (анонс окт 2025, запуск Q1 2026) -- пользователь диктует мысли, AI Assistant организует в списки/заметки/напоминания, кросс-девайсная синхронизация
- Alternative Pro -- запоминает детали прошлых разговоров, учитывает интересы и стиль общения, "сквозная персонализация" между устройствами

---

### 3.11. Приоритизация фактов

**Продуктовый смысл:** не все факты одинаково важны. "Юзера зовут Даниэль" важнее чем "юзер однажды упомянул погоду". Без приоритизации промпт забивается шумом -- 12 слотов для conclusions тратятся на мелочи вместо ключевых инсайтов.

**Honcho НЕ умеет.** Нет весов, тегов, приоритетов. Это можно сделать лучше.

**Как сделать лучше:**

| Механизм | Как работает |
|----------|-------------|
| **Peer Card = VIP** | Всегда в промпте, вне зависимости от поиска |
| **Теги на conclusions** | `tag: "core_taste"` vs `tag: "recent"` -> фильтрация при поиске |
| **Decay по времени** | Старые conclusions теряют вес, свежие -- приоритетнее |
| **Частота использования** | Логировать какие conclusions попали в промпт -> чаще используемые = важнее |
| **ARGUS embeddings** | Сигнал из рекомендательной системы как бустер релевантности |

---

### 3.12. Thread Management -- сессии

**Продуктовый смысл:** юзер может общаться с ботом месяцами. Нужно знать где одна тема закончилась и другая началась, уметь "перемотать" к прошлому разговору, и не путать контексты. Thread management -- это инфраструктура непрерывности.

**Что в Honcho:** sessions API, ротация, привязка к peer.

**Open-source:** своя реализация на PostgreSQL (sessions + messages таблицы).

**Enterprise stack:** Managed AI Platform Assistant API -- threads, messages, runs из коробки. OpenAI-compatible. При миллионах юзеров -- Managed AI Platform обрабатывает threads как managed service. Бесплатно на Preview stage (платите только за токены моделей).

**Важно:** LLM модели НЕ хранят контекст на сервере. Нужно либо отправлять полную историю с каждым запросом, либо использовать Managed AI Platform Assistant API с threads.

**AI Search (поиск + браузер):** помнит до 16 последних реплик в сессии, нет кросс-сессионной памяти -- контекст сбрасывается между сессиями.

---

## 4. Open-source альтернативы для масштаба Enterprise

Критерии отбора:
- Масштаб: миллионы пользователей
- Лицензия: Apache 2.0, MIT, BSD -- безопасные для коммерческого использования без ограничений. AGPL, SSPL, BSL -- потенциально проблемные
- Данные: self-hosted, данные не покидают инфраструктуру

### 4.1. Векторные базы данных

| Название | Лицензия | Статус лицензии | Масштаб | Latency (p50) | QPS | Русский | Примечания |
|----------|----------|----------------|---------|--------------|-----|---------|-----------|
| **Qdrant** | Apache 2.0 | Безопасная | До 1B векторов | 8ms | 1500 | N/A | Rust, ACORN filtered search, ACID, lightest footprint. Доступен на Cloud Marketplace как VM |
| **Milvus** | Apache 2.0 | Безопасная | До 10B+ векторов | 12ms | 1200 | N/A | Go+C++, disaggregated архитектура (compute/storage отдельно). Требует etcd, MinIO, Kafka. Overkill для большинства задач |
| **pgvector** | PostgreSQL License | Безопасная | До ~10M векторов | 15ms | 500 | N/A | Расширение PostgreSQL. HNSW, IVFFlat. Полный CRUD. Простейший вариант на старте |
| **Weaviate** | BSD-3 | Безопасная | До ~100M | 15ms | 800 | N/A | Go, встроенная векторизация. Тяжелый по ресурсам |
| **Neo4j** | GPL-3.0 | ОСТОРОЖНО | -- | -- | -- | -- | Не векторная БД, но часто используется вместе. GPL требует открытия производного кода |
| **Kuzu** | MIT | Безопасная | -- | -- | -- | -- | Embedded graph DB. Легковесная альтернатива Neo4j |
| **Memgraph** | BSL 1.1 | ОСТОРОЖНО | -- | -- | -- | -- | BSL конвертируется в Apache 2.0 через 4 года, но до конверсии ограничивает production use |

### 4.2. Embedding модели

| Название | Лицензия | Статус лицензии | Размерность | ruMTEB | Русский | Инфраструктура |
|----------|----------|----------------|-------------|--------|---------|---------------|
| **BGE-M3** | MIT | Безопасная | 1024 | 60.8 | Хорошо (74.8 retrieval) | CPU, 560M params, 8192 токенов. 100+ языков. Dense+sparse+ColBERT. **Рекомендованный выбор** |
| **GigaEmbeddings** | MIT | Безопасная | 2048 | **69.1** (SOTA) | Лучший | GPU (3B params), 4096 токенов. Flash Attention 2. Требует instruction prefix |
| **E5-mistral-7b-instruct** | MIT | Безопасная | 4096 | 64.9 | Хорошо | GPU (7B params), 32K токенов. Самая тяжелая |
| **mE5-large-instruct** | MIT | Безопасная | 1024 | 64.7 | Хорошо | CPU, 560M params, 512 токенов |
| **ru-en-RoSBERTa** | MIT | Безопасная | ~768 | 60.4 | 66.5 (retrieval) | CPU, 404M params. Обучена на ~4M русско-английских пар |

### 4.3. LLM фреймворки

| Название | Лицензия | Статус лицензии | Назначение | LLM интеграция |
|----------|----------|----------------|-----------|---------------------|
| **LangChain** | MIT | Безопасная | Оркестрация workflows, цепочки tools, conversational memory, LangGraph | Да: `CustomEmbeddings`, `llm-chain` (PyPI), Managed AI Platform SDK `.langchain()`, `@langchain/enterprise` (npm) |
| **LlamaIndex** | MIT | Безопасная | RAG, document retrieval, query routing, index fusion | Да: `LLMEmbedding` класс. 35% лучше retrieval accuracy vs LangChain |

**Рекомендация:** LangChain для системы памяти (multi-step agent work: extract -> compare -> ADD/UPDATE/DELETE). LlamaIndex для чистого document search/retrieval.

### 4.4. Memory фреймворки

| Название | Лицензия | Статус лицензии | Архитектура | Совместимость с Enterprise stackом |
|----------|----------|----------------|------------|------------------------------|
| **Mem0** | Apache 2.0 | Безопасная | LLM fact extraction -> embedding -> similarity search -> ADD/UPDATE/DELETE -> vector+graph stores. 24+ провайдера vector store, 15+ LLM, 11+ embedder | Прямая замена: OpenAI LLM -> LLM, OpenAI embeddings -> Managed AI Platform/BGE-M3, Qdrant -> pgvector/Qdrant. Docker deploy: Postgres + Neo4j + API server. **Нет аутентификации** -- CORS `allow_origins=["*"]`, нужен reverse proxy |
| **Graphiti** | Apache 2.0 | Безопасная | Temporal knowledge graph, episodic memory | Менее зрелый чем Mem0, но лучше для графовых данных |
| **Letta (ex-MemGPT)** | Apache 2.0 | Безопасная | Self-editing memory, tiered storage (core/archival/recall) | Оригинальный подход, но менее гибкий для кастомных пайплайнов |

### 4.5. Pipeline оркестрация

| Название | Лицензия | Статус лицензии | Парадигма | Local dev | Self-host сложность |
|----------|----------|----------------|----------|-----------|-------------------|
| **Dagster** | Apache 2.0 | Безопасная | Asset-centric, data lineage | Лучший (tight feedback loop, testable) | Средняя (dagit + daemon + DB) |
| **Apache Airflow** | Apache 2.0 | Безопасная | DAG-based, schedule-driven. v3.0 (апрель 2025) | Слабый (нужен полный scheduler) | Высокая (scheduler + webserver + DB + workers) |
| **Prefect** | Apache 2.0 | Безопасная | Dynamic flows, event-driven | Хороший (REPL-style debugging) | Низкая (server + agent) |

**Рекомендация для memory pipelines:** Dagster -- asset-centric модель естественно ложится на memory-артефакты (embeddings, summaries, user profiles). Встроенный data lineage отслеживает что обработано.

### 4.6. Message queues

| Название | Лицензия | Статус лицензии | Масштаб | Latency | Managed в Enterpriseе? |
|----------|----------|----------------|---------|---------|-------------------|
| **Apache Kafka** | Apache 2.0 | Безопасная | Миллиарды msg/день | Низкая (ms) | Да (Managed Kafka, версии 3.6-4.0, ~$162/мес за 3 брокера) |
| **Redis** (Streams) | BSD-3 | Безопасная | Миллионы msg/день | Суб-ms | Да (Managed Redis) |
| **NATS** | Apache 2.0 | Безопасная | Миллиарды msg/день | Суб-ms | Нет managed |

**Рекомендация:** Redis Streams для прототипа (простота), Kafka для продакшена (партиционирование по user_id, горизонтальное масштабирование, managed в Enterpriseе).

### 4.7. Graph databases

| Название | Лицензия | Статус лицензии | Масштаб | Примечания |
|----------|----------|----------------|---------|-----------|
| **Kuzu** | MIT | Безопасная | Embedded, до миллионов нод | Легковесная, embedded, отлично для прототипа и средних объемов |
| **Apache AGE** | Apache 2.0 | Безопасная | Расширение PostgreSQL | Графовый движок поверх PostgreSQL -- не нужна отдельная БД |
| **Neo4j Community** | GPL-3.0 | ОСТОРОЖНО | До миллиардов нод | Зрелый, богатый Cypher, но GPL требует открытия производного кода. Neo4j Enterprise -- коммерческая лицензия |
| **Memgraph** | BSL 1.1 | ОСТОРОЖНО | До миллиардов нод | BSL ограничивает production use. Конвертируется в Apache 2.0 через 4 года |

**Рекомендация:** Kuzu (MIT) для прототипа и средних объемов. Apache AGE (Apache 2.0) если хочется графовых запросов поверх существующего PostgreSQL. Neo4j -- только если лицензионные ограничения GPL приемлемы или куплена Enterprise лицензия.

### 4.8. Self-hosted LLM inference

| Название | Лицензия | Статус лицензии | Назначение |
|----------|----------|----------------|-----------|
| **vLLM** | Apache 2.0 | Безопасная | Serving LLM моделей. PagedAttention, continuous batching, tensor parallelism. OpenAI-compatible API |

**Доступные GPU на Cloud Platform:**

| GPU | Память | Примечания |
|-----|--------|-----------|
| NVIDIA Tesla V100 | 32 GB HBM2 | 5120 CUDA cores, 640 Tensor Cores |
| NVIDIA A100 | 80 GB HBM2e | Ampere, только в ru-central1-a |

Нет H100 или T4 в текущей документации Cloud Platform.

**Что можно запустить:** Qwen3-235B (multi-GPU A100), Llama 3.3 70B (1x A100 80GB с квантизацией), любая open-source модель с HuggingFace.

---

## 5. Что брать на каждом этапе

### Прототип (неделя 1-4)

| Компонент | Решение | Почему |
|-----------|---------|--------|
| DB | Managed PostgreSQL 16-18 + pgvector 0.8.0 | GA, полный CRUD, managed |
| Embeddings | Managed AI Platform (256d) | Простейший API, бесплатный на Preview |
| LLM (extraction) | LLM Lite (0.20 руб/1K) | Дешево, 32K контекст |
| LLM (reasoning) | LLM Pro (0.80 руб/1K) | 128K контекст, Chain-of-Reasoning |
| Queue | asyncio.Queue | Без внешних зависимостей |
| Cache | Без кеша | Пока не нужен |
| Threads | Managed AI Platform Assistant API | Бесплатно на Preview, OpenAI-compatible |
| Graph | Kuzu (embedded, MIT) | Нет отдельного сервиса, встраивается в приложение |

### Прод (месяц 2-4)

| Компонент | Решение | Почему |
|-----------|---------|--------|
| DB | Managed PostgreSQL + pgvector (sharded по user_id) | Проверенный масштаб |
| Embeddings | BGE-M3 self-hosted (1024d, CPU) | Нет rate limits, 60.8 ruMTEB |
| Vector search | pgvector -> Qdrant при >10M векторов | Qdrant: 8ms, 1500 QPS, Apache 2.0 |
| LLM | LLM Pro/Lite | Pro для reasoning, Lite для extraction/summarization |
| Queue | Managed Kafka 4.0 | Партиционирование по user_id |
| Orchestration | Dagster (Apache 2.0) | Asset-centric для memory artifacts |
| Cache | Redis / Memcached | Peer Card, hot data |
| Batch | Dagster + Kafka + LLM Lite | Batch import Streaming Platform/Music Service/Book Service |
| Threads | Managed AI Platform или своя реализация | Зависит от масштаба и контроля |
| Memory framework | Mem0 self-hosted (Apache 2.0) с cloud провайдерами | Проверенная архитектура |
| RAG framework | LangChain (MIT) + ai-platform-sdk | Workflow оркестрация + нативная LLM поддержка |
| Graph | Neo4j на VM или Apache AGE | Для entity relationships (Mem0-style) |

### Масштаб (месяц 4+)

Добавляется:
- ARGUS embeddings как input для dreaming -- сигнал из рекомендательной системы
- NRT updates через Kafka
- Горизонтальное масштабирование (Kafka partitions по user_id)
- Индукция (второй агент dreaming)
- Dialectic (ReAct-агент)
- GigaEmbeddings (2048d, GPU) если нужен максимум качества для русского
- Миграция на Qdrant/OpenSearch при >100M векторов
- Self-hosted LLM через vLLM на A100 для кастомных/fine-tuned моделей

---

## 6. Масштабирование на миллионы юзеров (Enterprise-scale)

> Перенесено из [02-memory-architecture-guide.md](./02-memory-architecture-guide.md) — раздел описывает архитектурные решения для масштабирования механик Honcho на десятки миллионов MAU.

Honcho спроектирован для тысяч юзеров. Для масштаба Enterprise (десятки миллионов MAU) нужны архитектурные решения на каждом уровне: хранение, очереди, LLM-вызовы, кэширование, и операционная модель.

---

### 6.1. Шардирование данных

#### Проблема

Один PostgreSQL с pgvector не выдержит 50M юзеров × 200 фактов = 10B записей + векторные индексы.

#### Решение: шардирование по `user_id`

**Почему по `user_id`:**
- Все данные одного юзера на одном шарде → нет cross-shard запросов
- Prefetch, deriver, dreaming — всё работает внутри одного шарда
- Шардирование по другим ключам (по дате, по типу) ломает локальность

> Псевдокод роутера и пула подключений: [02a-scaling-code-examples.md § 10.1](./02a-scaling-code-examples.md#101-шардирование-данных)

**Альтернатива для Enterprise: YDB**
- Нативное распределённое хранилище с автошардированием
- Partitioning key = `user_id` — YDB сам балансирует шарды
- Для векторного поиска — отдельный OpenSearch/Elasticsearch кластер с тем же партиционированием

**Ёмкость на шард:**

| Параметр | Значение |
|----------|----------|
| Юзеров на шард | 1-2M |
| Фактов на юзера (avg) | 200 |
| Размер шарда | ~50 GB данных + ~100 GB индексы |
| Шардов для 50M юзеров | 25-50 |

---

### 6.2. Очереди и воркеры

#### Проблема

asyncio.Queue умирает с процессом. При 100K RPS (сообщений в секунду в пике) нужна персистентная очередь с горизонтальным масштабированием.

#### Решение: Kafka + партиционирование по `user_id`

```
Producer (API) ──> Kafka topic: user-messages
                   Partition 0: user_id hash 0-99
                   Partition 1: user_id hash 100-199
                   ...
                   Partition N: user_id hash ...

Consumer Group: deriver-workers
  Worker 0 ← Partition 0
  Worker 1 ← Partition 1
  ...
  Worker N ← Partition N
```

**Зачем партиционирование по `user_id`:**
- Гарантия порядка сообщений одного юзера (важно для deriver — факты зависят от контекста)
- Один воркер обрабатывает одного юзера → нет race conditions на peer card
- Добавить воркер = перебалансировать партиции (Kafka делает автоматически)

> Конфигурация Kafka: [02a-scaling-code-examples.md § 10.2](./02a-scaling-code-examples.md#102-очереди-и-воркеры)

**Отдельные топики для разных процессов:**
- `user-messages` → deriver workers
- `dreaming-triggers` → dreaming workers (когда набралось 50+ фактов)
- `batch-import` → batch pipeline workers (импорт Streaming Platform/музыки)
- `card-updates` → cache invalidation (сброс кэша peer card)

---

### 6.3. LLM-вызовы: оптимизация стоимости и латентности

#### Проблема

50M юзеров × 10 сообщений/день = 500M сообщений/день. Если каждое → LLM-вызов для deriver = **500M вызовов/день**. Это невозможно ни по стоимости, ни по throughput.

#### Решение 1: батчинг сообщений

Вместо 1 сообщение → 1 LLM-вызов, собираем батч из 5-10 сообщений одного юзера (или по таймауту 5 сек) и делаем один вызов на весь батч. **Экономия:** 5-10x по количеству LLM-вызовов.

#### Решение 2: фильтрация перед LLM

Не все сообщения содержат факты. «ок», «лол», эмодзи — бесполезны. Дешёвая фильтрация ДО LLM-вызова по длине, наличию только эмодзи, и стоп-листу подтверждений. **Экономия:** ещё 30-50% вызовов отсекается.

> Псевдокод батчинга, фильтрации и rate limiting: [02a-scaling-code-examples.md § 10.3](./02a-scaling-code-examples.md#103-llm-вызовы-оптимизация-стоимости-и-латентности)

#### Решение 3: tiered модели

| Процесс | Модель | Стоимость | Почему |
|----------|--------|-----------|--------|
| Deriver | LLM Lite | $ | Простая экстракция, не нужен reasoning |
| Суммаризация | LLM Lite | $ | Сжатие текста — базовая задача |
| Dreaming (дедукция) | LLM Pro | $$ | Нужна логика, но задач мало (раз в 8 часов) |
| Dreaming (индукция) | LLM Pro | $$ | Поиск паттернов — сложная задача |
| Dialectic | LLM Pro | $$$ | On-demand, не масштабируется так сильно |
| Batch import | LLM Lite (batch API) | ¢ | Batch API с 50% скидкой, не в реалтайме |

#### Решение 4: rate limiting per user

Не более 1 dreaming в 8 часов на юзера. Не более 100 deriver-вызовов в день (после 100 — только батч в конце дня). Лимиты хранятся в Redis с TTL.

---

### 6.4. Кэширование

#### Проблема

Peer Card читается на **каждый** запрос. 100K RPS → 100K чтений peer_cards/сек из базы. Даже шардированный PostgreSQL этого не выдержит.

#### Решение: многоуровневый кэш

Три уровня: L1 (in-memory, TTL 60 сек) → L2 (Redis, TTL 5 мин) → L3 (PostgreSQL, source of truth). Запрос идёт по цепочке, первый hit возвращает результат.

> Реализация CachedPeerCard: [02a-scaling-code-examples.md § 10.4](./02a-scaling-code-examples.md#104-кэширование)

**Инвалидация:**
- При обновлении peer card → публикуем событие в Kafka топик `card-updates`
- Все инстансы API слушают топик и сбрасывают L1 кэш
- Redis TTL обеспечивает eventual consistency за 5 мин даже если событие потерялось

**Что ещё кэшировать:**
- Embeddings частых запросов (топ-100 фраз типа «привет», «как дела» → один и тот же вектор)
- Результаты prefetch для «горячих» юзеров (пишут каждую минуту)
- Conclusions не кэшируем — семантический поиск зависит от запроса

---

### 6.5. Горизонтальное масштабирование воркеров

#### Архитектура деплоя

```
                          ┌──────────────────────┐
                          │     Load Balancer     │
                          └──────────┬───────────┘
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
              API Pod ×50      API Pod ×50      API Pod ×50
              (prefetch)       (prefetch)       (prefetch)
                    │                │                │
                    └────────────────┼────────────────┘
                                    ▼
                              Kafka Cluster
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
            Deriver Pods ×32  Dreaming Pods ×8  Batch Pods ×16
            (consumer group)  (consumer group)  (consumer group)
                    │               │               │
                    └───────────────┼───────────────┘
                                    ▼
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              PG Shard ×25   PG Shard ×25    Redis Cluster
              (данные)       (pgvector)      (кэш + rate limits)
```

**Автоскейлинг:**

| Компонент | Метрика скейлинга | Мин | Макс |
|-----------|-------------------|-----|------|
| API Pods | RPS, CPU | 20 | 200 |
| Deriver Pods | Kafka lag (необработанные сообщения) | 8 | 64 |
| Dreaming Pods | Очередь триггеров | 2 | 16 |
| Batch Pods | Очередь импорта | 0 | 32 (on-demand) |

> Kubernetes HPA манифест: [02a-scaling-code-examples.md § 10.5](./02a-scaling-code-examples.md#105-горизонтальное-масштабирование-воркеров)

---

### 6.6. Отказоустойчивость и graceful degradation

#### Что падает и что делать

| Компонент упал | Влияние на юзера | Degradation |
|----------------|------------------|-------------|
| Deriver | Никакого (фоновый) | Сообщения копятся в Kafka, обработаются позже |
| Dreaming | Никакого (фоновый) | Карточка не обновляется, но старая работает |
| Redis (кэш) | Рост латентности | Fallback на PostgreSQL напрямую |
| PostgreSQL (шард) | Юзеры этого шарда без памяти | Бот работает без контекста + retry |
| Kafka | Deriver не получает сообщения | Write-ahead log в API, replay при восстановлении |
| LLM API | Deriver/Dreaming стоят | Exponential backoff + circuit breaker |

> Circuit breaker реализация: [02a-scaling-code-examples.md § 10.6](./02a-scaling-code-examples.md#106-отказоустойчивость-и-graceful-degradation)

**Принцип:** память — это enhancement, не критический путь. Если память недоступна — бот отвечает без контекста, как будто юзер новый. Лучше ответить без памяти, чем не ответить вовсе.

---

### 6.7. Мониторинг и observability

#### Ключевые метрики

Prometheus-метрики по компонентам: deriver (throughput, latency, дубликаты), prefetch (latency, cache hit ratio), dreaming (runs, card updates, противоречия), бизнес (facts per user, card fill rate, memory utilization).

> Полный список метрик: [02a-scaling-code-examples.md § 10.7](./02a-scaling-code-examples.md#107-мониторинг-и-observability)

#### Алерты

| Метрика | Порог | Действие |
|---------|-------|----------|
| Kafka consumer lag (deriver) | > 100K messages | Scale up deriver pods |
| Prefetch p99 latency | > 200ms | Проверить Redis, PG, сеть |
| Deriver error rate | > 5% | Проверить LLM API, circuit breaker |
| Facts per user (new) | < 0.5 за 24ч | Deriver сломан или промпт деградировал |
| Card fill rate | Падение > 5% за час | Dreaming удаляет больше чем добавляет |

---

### 6.8. Миграция и zero-downtime deploy

#### Проблема

Изменение схемы данных (новое поле в conclusions, изменение формата peer card) на 50 шардах с 10B записей — нельзя делать `ALTER TABLE` с локом.

#### Решение: online schema migration (expand-migrate-contract)

Три фазы:
1. **Expand** — добавить новое поле как nullable (без лока таблицы). Код читает старое, пишет и старое и новое.
2. **Migrate** — бэкфил в фоне batch job'ом по 1000 записей с throttling.
3. **Contract** — сделать NOT NULL, убрать поддержку старого формата.

> Псевдокод бэкфила: [02a-scaling-code-examples.md § 10.8](./02a-scaling-code-examples.md#108-миграция-и-zero-downtime-deploy)

**Для Enterprise:**
- YDB поддерживает online ALTER без даунтайма
- Бэкфил через MapReduce / YT для параллельной обработки шардов
- Blue-green deploy: новая версия API читает оба формата, старая — только старый

---

### 6.9. Data privacy и compliance

#### Проблема

Память хранит персональные данные. GDPR/152-ФЗ требуют: право на удаление, минимизация данных, ограничение хранения.

#### Решение

Полное удаление данных юзера — одна транзакция на шарде (conclusions, peer_cards, summaries, messages) + инвалидация кэша + аудит-лог. TTL на данные: conclusions старше 12 месяцев без обновления → stale, stale > 18 месяцев → удалить.

> Псевдокод удаления: [02a-scaling-code-examples.md § 10.9](./02a-scaling-code-examples.md#109-data-privacy-и-compliance)

**Изоляция данных:**
- Шардирование по `user_id` = данные юзера физически на одном шарде
- Удаление — одна транзакция, не размазано по кластеру
- LLM-вызовы: не отправлять `user_id` в LLM API (только контент). Mapping хранить локально

---

### 6.10. Сводка: чеклист масштабирования

| # | Компонент | MVP | 1M юзеров | 50M юзеров |
|---|-----------|-----|-----------|-------------|
| 1 | База данных | 1× PostgreSQL + pgvector | 5 шардов PG | 25-50 шардов YDB + OpenSearch |
| 2 | Очередь | asyncio.Queue | Redis Streams | Kafka (256 партиций) |
| 3 | Deriver | 1 воркер | 4 воркера | 32-64 пода (autoscale) |
| 4 | Dreaming | Cron каждые 2ч | Event-driven, 2 пода | 8-16 подов, rate limited |
| 5 | Кэш Peer Card | Нет | Redis | Redis Cluster + L1 in-memory |
| 6 | LLM-вызовы | По одному | Батчинг × 5 | Батчинг + фильтрация + tiered модели |
| 7 | Мониторинг | Логи | Prometheus + Grafana | + алерты + SLO + бизнес-метрики |
| 8 | Отказоустойчивость | Нет | Retry + fallback | Circuit breaker + graceful degradation |
| 9 | Миграции | ALTER TABLE | Blue-green | Expand-migrate-contract |
| 10 | Privacy | Manual delete | API endpoint | TTL + автоочистка + аудит |
