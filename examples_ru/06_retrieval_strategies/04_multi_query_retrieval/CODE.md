# Мультизапросный ретривер — обзор кода

Пошаговое руководство по `example.js`: декомпозиция запроса через LLM (Qwen через node-llama-cpp), вспомогательные функции параллельного ретривера, консенсусный рейтинг (RRF) и взвешенная фузия, а также все восемь примеров.

---

## Содержание

1. [Что такое мультизапросный ретривер?](#what-is-multi-query-retrieval)
2. [Установка и конфигурация](#setup-and-configuration)
3. [Конфигурация, вспомогательные функции и логирование](#config-helpers-and-logging)
4. [Инициализация](#initialization)
5. [Декомпозиция запроса через LLM](#llm-query-decomposition)
6. [База знаний и векторное хранилище](#knowledge-base-and-vector-store)
7. [Пример 1: Декомпозиция запроса](#example-1-query-decomposition)
8. [Пример 2: Расширение запроса и параллельный ретривер](#example-2-query-expansion-and-parallel-retrieval)
9. [Пример 3: Консенсусный рейтинг (RRF)](#example-3-reciprocal-rank-fusion-rrf)
10. [Пример 4: Ретривер на основе перспектив](#example-4-perspective-based-retrieval)
11. [Пример 5: Многочастные вопросы](#example-5-multi-part-questions)
12. [Пример 6: Взвешенная фузия запросов](#example-6-weighted-query-fusion)
13. [Пример 7: Адаптивная стратегия](#example-7-adaptive-strategy)
14. [Пример 8: Стратегии дедупликации](#example-8-deduplication-strategies)
15. [Главный запускатель](#main-runner)
16. [Краткая справка](#quick-reference)

---

## Что такое мультизапросный ретривер?

Использование **нескольких поисковых запросов** вместо одного для улучшения покрытия и полноты выборки.

| Одиночный запрос | Мультизапросный ретривер |
|--------------|-------------|
| Один эмбеддинг, один поиск | Декомпозиция или расширение до нескольких запросов |
| Может переобучиться на одну формулировку | Разные формулировки извлекают разные документы |
| Может упустить аспекты сложного вопроса | Подзапросы охватывают каждый аспект; результаты объединяются (например, RRF) |

**Преимущества:** лучшее покрытие, обработка сложных вопросов, устойчивость к формулировкам, повышенная полнота выборки.

---

## Установка и конфигурация

| Компонент | Источник |
|-----------|--------|
| Модель эмбеддингов | `models/bge-small-en-v1.5.Q8_0.gguf` |
| LLM (декомпозиция) | `models/Qwen3-1.7B-Q8_0.gguf` |
| Векторное хранилище | `embedded-vector-db`, размерность 384, пространство имён `"multi_query"` |
| Конфигурация | `config.js` |
| Вспомогательные функции | `multi-query-retrieval.js`: `retrieveParallel`, `deduplicateById`, `deduplicateByMaxScore`, `rrfFuse` |
| Логгер | `logger.js`: `createLogger`, `timeAsync`, `metrics` |

**Импорты и деструктуризация из конфигурации:**

```javascript
const { dim: DIM, maxElements: MAX_ELEMENTS, namespace: NS, rrfK, topKPerQuery } = config;
const logger = createLogger({ logLevel: config.logLevel });
```

Пути: `EMBEDDING_MODEL_PATH`, `LLM_MODEL_PATH` указывают на файлы в каталоге `models/`.

---

## Конфигурация, вспомогательные функции и логирование

**`config.js`** — настраиваемые параметры (переопределение через переменные окружения в `ENV_MAP`): `dim`, `maxElements`, `namespace`, `rrfK`, `maxSubQueries`, `topKPerQuery`, `timeoutMs`, `retries`, `concurrency`, `logLevel`.

**`multi-query-retrieval.js`:**

| Функция | Назначение |
|----------|---------|
| `retrieveParallel(queries, options)` | Запускает каждый запрос параллельно. Параметры: `getEmbedding`, `search`, `namespace`, `topK`, `retries`, `timeoutMs`, `concurrency`. Возвращает `Array<Array<result>>`. |
| `deduplicateById(results)` | Один результат на ID документа (первое вхождение). |
| `deduplicateByMaxScore(results)` | Один результат на ID (наибольшая схожесть), отсортированный по оценке. |
| `rrfFuse(resultLists, k)` | RRF-фузия; возвращает один список с полем `rrfScore`, отсортированный по убыванию. |

**`logger.js`:** `logger.timeAsync(label, fn)`, `logger.metrics({ queryCount, totalResults, uniqueAfterDedup, durationMs })`.

---

## Инициализация

**Эмбеддинг:** `initializeEmbeddingModel()` — `getLlama` → `loadModel(EMBEDDING_MODEL_PATH)` → `createEmbeddingContext()`.

**LLM:** `initializeLLM()` — `getLlama` → `loadModel(LLM_MODEL_PATH)` → `createContext()` → `new LlamaChatSession({ contextSequence })`.

Оба компонента создаются один раз в главном запускателе; `chatSession` передаётся только в пример 1.

---

## Декомпозиция запроса через LLM

**Резервный вариант** (при сбое LLM или ошибке разбора):

```javascript
const FALLBACK_SUBQUERIES = [
    "What is web application architecture?",
    "How to make web applications scalable?",
    "What are testing best practices?",
    "How to implement automated testing?",
];
```

**`parseSubQueriesFromLLM(raw)`:** пытается разобрать JSON-массив или объект (ключи `sub_queries` / `subQueries` / `queries`); в противном случае — нумерованные или маркированные строки. Возвращает `null`, если ничего валидного не найдено.

**`decomposeQueryWithLLM(complexQuery, chatSession)`:** промпт запрашивает 3–5 подвопросов в виде JSON-массива. Вызывает `chatSession.prompt(prompt, { maxTokens: 400 })`, разбор через `parseSubQueriesFromLLM`. Если получено не менее 2 запросов: `parsed.slice(0, config.maxSubQueries)`; иначе — `FALLBACK_SUBQUERIES`.

---

## База знаний и векторное хранилище

**`createKnowledgeBase()`** — возвращает 25 экземпляров `Document` с полями `id`, `category`, `topic`, `subtopic` (frontend, backend, database, testing, devops).

**`addDocumentsToStore(vectorStore, embeddingContext, documents)`** — для каждого документа: `getEmbeddingFor(doc.pageContent)`, затем `vectorStore.insert(NS, doc.metadata.id, vector, metadata)` с `metadata = { content: doc.pageContent, ...doc.metadata }`.

---

## Пример 1: Декомпозиция запроса

**Цель:** один сложный запрос → подзапросы через LLM → базовый одиночный запрос vs мультизапросный ретривер + RRF; демонстрация того, что мультизапросный ретривер даёт более разнообразные, ранжированные по RRF результаты.

**Сложный запрос (в коде):**

```javascript
const complexQuery = "What are the best practices for designing scalable architectures and implementing effective testing strategies?";
```

**Сценарий:**

1. **Подзапросы** — `llmSubQueries = await decomposeQueryWithLLM(complexQuery, chatSession)`. Формируем `subQueries`, беря до `config.maxSubQueries - 1` из `llmSubQueries`. Логирование под заголовком «Decomposed Sub-Queries:».
2. **Базовый одиночный запрос** — эмбеддинг `complexQuery`, поиск с `topK: 3`. Вывод 3 результатов (сходство, тема, 60 символов содержимого). Обёрнуто в `logger.timeAsync("single-query", ...)`.
3. **Мультизапросный ретривер** — `resultLists = await retrieveParallel(subQueries, { getEmbedding, search, namespace: NS, topK: 4, retries, timeoutMs })`. Обёрнуто в `logger.timeAsync("multi-query", ...)`.
4. **Фузия** — `fusedResults = rrfFuse(resultLists, rrfK)`.
5. **Отображение** — «Results (N unique documents, RRF-ranked):», первые 8 элементов; оценка = `doc.rrfScore ?? doc.similarity`, затем тема и 60 символов содержимого.
6. **Ключевой вывод** — количество результатов одиночного запроса vs мультизапросного; документы, поддерживающие несколько аспектов, получают более высокий рейтинг.

---

## Пример 2: Расширение запроса и параллельный ретривер

**Цель:** сравнение последовательного и параллельного выполнения для расширенных формулировок; дедупликация по ID.

**Запрос и расширения:**

```javascript
const originalQuery = "How to optimize database queries?";
const expandedQueries = [
    originalQuery,
    "What are database performance optimization techniques?",
    "Ways to improve database query speed",
    "Database indexing and query optimization strategies"
];
```

**Сценарий:**

1. **Последовательный** — для каждого из `expandedQueries`: эмбеддинг, поиск `topK: 3`, добавление в `seqResults`. Измеряется `seqTimeMs`.
2. **Параллельный** — `parResultLists = await retrieveParallel(expandedQueries, { ..., topK: 3 })`. Измеряется `parTimeMs`. `parResults = parResultLists.flat()`.
3. **Дедупликация** — `uniqueResults = deduplicateById(parResults)`.
4. **Логирование** — `logger.metrics({ queryCount, totalResults, uniqueAfterDedup, durationMs: parTimeMs })`. Вывод «Unique Results (N documents):», первые 4 (тема, подтема, 70 символов содержимого). Ключевой вывод: покрытие и ускорение.

---

## Пример 3: Консенсусный рейтинг (RRF)

**Цель:** несколько вариантов запроса параллельно → результаты по каждому запросу → RRF-фузия → единый ранжированный список.

**Запросы:**

```javascript
const queries = [
    "React performance optimization",
    "How to make React apps faster?",
    "React rendering optimization techniques"
];
```

**Сценарий:**

1. **Параллельный** — `allResults = await retrieveParallel(queries, { ..., topK: topKPerQuery })`.
2. **По каждому запросу** — логирование «Individual Query Results:», для каждого списка «Query N:» и первые 3 (ранг, id, тема).
3. **RRF** — `fusedResults = rrfFuse(allResults, rrfK)`.
4. **Отображение** — «Fused Results (RRF):», первые 5 с `rrfScore`, id, тема, 60 символов содержимого. Ключевой вывод: формула RRF, документы из нескольких списков получают более высокий рейтинг, параметр `k` из `rrfK`.

---

## Пример 4: Ретривер на основе перспектив

**Цель:** одна тема под разными углами → результаты по каждой перспективе → объединённое количество уникальных документов.

**Тема и перспективы:**

```javascript
const topic = "microservices architecture";
const perspectives = [
    { query: "What are the benefits of microservices?", label: "Benefits" },
    { query: "What are the challenges of microservices?", label: "Challenges" },
    { query: "How to implement microservices?", label: "Implementation" },
    { query: "When to use microservices vs monolith?", label: "Comparison" }
];
```

**Сценарий:**

1. **Параллельный** — `perspectiveQueries = perspectives.map(p => p.query)`. `perspectiveResultLists = await retrieveParallel(perspectiveQueries, { ..., topK: 2 })`.
2. **Структурирование** — `perspectiveResults = perspectives.map((p, i) => ({ perspective: p.label, results: perspectiveResultLists[i] ?? [] }))`.
3. **Отображение** — «Results by Perspective:» для каждой метки и её результатов (тема, категория, 70 символов содержимого).
4. **Объединение** — `allDocs = perspectiveResults.flatMap(pr => pr.results)`, `uniqueDocs = deduplicateById(allDocs)`. Логирование «Combined Coverage: N unique documents».

---

## Пример 5: Многочастные вопросы

**Цель:** один многочастный вопрос → явные подвопросы → ретривер по каждой части → результаты по частям.

**Запрос и части:**

```javascript
const multiPartQuery = "What's the difference between unit testing and integration testing, and when should I use each?";
const parts = [
    { query: "What is unit testing?", label: "Unit Testing Definition" },
    { query: "What is integration testing?", label: "Integration Testing Definition" },
    { query: "When to use unit testing?", label: "Unit Testing Use Cases" },
    { query: "When to use integration testing?", label: "Integration Testing Use Cases" },
    { query: "Difference between unit and integration testing", label: "Comparison" }
];
```

**Сценарий:**

1. **Параллельный** — `partQueries = parts.map(p => p.query)`. `partResultLists = await retrieveParallel(partQueries, { ..., topK: 2 })`.
2. **Структурирование** — `partResults = parts.map((p, i) => ({ part: p.label, results: partResultLists[i] ?? [] }))`.
3. **Отображение** — «Results by Question Part:» для каждой части (сходство, тема, 80 символов содержимого). Ключевой вывод: каждый аспект обработан.

---

## Пример 6: Взвешенная фузия запросов

**Цель:** основной запрос + связанные запросы с весами → взвешенная фузия оценок → приоритет основного запроса.

**Основной и связанные запросы:**

```javascript
const mainQuery = "React state management";
const relatedQueries = [
    { query: "React hooks for state", weight: 0.8 },
    { query: "Redux state management", weight: 0.5 },
    { query: "Context API React", weight: 0.6 }
];
```

**Сценарий:**

1. **Основной** — эмбеддинг `mainQuery`, поиск `topK: 5` → `mainResults`.
2. **Связанные** — `relatedQueriesOnly = relatedQueries.map(rq => rq.query)`. `relatedResultLists = await retrieveParallel(relatedQueriesOnly, { ..., topK: 3 })`. Объединение с весами: `relatedResults = relatedQueries.map((rq, i) => ({ results: relatedResultLists[i] ?? [], weight: rq.weight }))`.
3. **Взвешенные оценки** — `weightedScores = new Map()`. Основной: для каждого документа `score = similarity * 1.0`. Связанные: для каждого документа добавляется `similarity * weight` к существующей или новой записи. Сортировка по убыванию оценки → `fusedResults`.
4. **Отображение** — «Weighted Fusion Results:», первые 5 (оценка, тема, 70 символов содержимого).

---

## Пример 7: Адаптивная стратегия

**Цель:** классификация каждого тестового запроса по сложности → выбор стратегии (одиночный, расширение или сложный с увеличенным topK) → запуск и отображение количества результатов и примера.

**`analyzeQueryComplexity(query)`:** использует количество слов, «and»/«or»/«vs»/«compare», завершающий «?», «difference»/«compare»/«versus»/«vs»:

- **Очень сложный** (многотемный и количество слов > 15) → «Multi-Part Decomposition»
- **Сложный** (сравнение) → «Perspective-Based Retrieval»
- **Умеренный** (количество слов > 10 и вопрос) → «Query Expansion»
- **Простой** → «Single Query (no decomposition needed)»

**Тестовые запросы:**

```javascript
const testQueries = [
    "What is React?",
    "How do I optimize React performance?",
    "What's the difference between REST and GraphQL?",
    "How do I build a scalable web app with microservices, implement CI/CD, and ensure good test coverage?"
];
```

**Выполнение для каждого запроса:**

- **Одиночный запрос** — `vectorStore.search(NS, vector, 3)` → `results`.
- **Расширение запроса** — `queries = [query, \`How to ${query.replace(/^How (do|to) I /i, "")}\`].slice(0, config.maxSubQueries)`. `retrieveParallel(queries, { ..., topK: 2 })` → `results = deduplicateById(resultLists.flat())`.
- **Иное (сложный)** — `vectorStore.search(NS, vector, 5)` → `results`.

Логирование «Results: N documents retrieved», первые 2 (тема, категория). Ключевой вывод: баланс между скоростью и качеством в зависимости от сложности.

---

## Пример 8: Стратегии дедупликации

**Цель:** одинаковые запросы параллельно → три стратегии дедупликации: по ID, по теме+категории, по максимальной оценке.

**Запросы:**

```javascript
const queries = [
    "React state management hooks",
    "useState and useEffect in React",
    "React hooks for managing state"
];
```

**Сценарий:**

1. **Ретривер** — `resultLists = await retrieveParallel(queries, { ..., topK: 4 })`. `allResults = resultLists.flat()`. Логирование «Total Results: N (with duplicates)».
2. **Стратегия 1 — По ID** — `byId = deduplicateById(allResults)`. «Unique documents: N», первые 3 (id, тема).
3. **Стратегия 2 — По теме + категории** — ключ `Map` `${topic}-${category}`, одно значение на ключ → `byTopicCategory`. «Unique topic-category pairs: N», первые 3 (тема, категория).
4. **Стратегия 3 — Максимальная оценка** — `byMaxScore = deduplicateByMaxScore(allResults)`. «Unique documents (best score): N», первые 3 (сходство, тема). Ключевой вывод: по ID — просто; по теме — группировка; по максимальной оценке — лучший вариант для ранжирования.

---

## Главный запускатель

**`runAllExamples()`:**

1. Очистка консоли; вывод заголовка и маркеров «What you'll learn».
2. Загрузка модели эмбеддингов (`OutputHelper.withSpinner`).
3. Создание `VectorDB`, `createKnowledgeBase()`, `addDocumentsToStore(...)`.
4. Загрузка LLM (`withSpinner`), получение `chatSession`.
5. Запуск примеров через `OutputHelper.runExample(title, fn)`:
   - Пример 1: Декомпозиция запроса (с `chatSession`)
   - Пример 2: Расширение запроса и параллельный ретривер
   - Пример 3: Консенсусный рейтинг (RRF)
   - Пример 4: Ретривер на основе перспектив
   - Пример 5: Многочастные вопросы
   - Пример 6: Взвешенная фузия запросов
   - Пример 7: Адаптивная стратегия
   - Пример 8: Дедупликация
6. Вывод «All examples completed successfully!», основные выводы, таблица «When to Use Multi-Query», лучшие практики, методы фузии результатов.
7. При ошибке: логирование сообщения и подсказки (зависимости, файлы моделей в `models/`).

**Предварительные требования:** файлы `bge-small-en-v1.5.Q8_0.gguf` и `Qwen3-1.7B-Q8_0.gguf` в каталоге `models/`.

---

## Краткая справка

| Пример | Стратегия | Фузия / дедупликация |
|---------|----------|----------------|
| 1. Декомпозиция | Подзапросы через LLM, topK 4, RRF | `rrfFuse(resultLists, rrfK)` |
| 2. Расширение | Фиксированный список расширений, topK 3 | `deduplicateById` |
| 3. RRF | Фиксированный список запросов, topK из конфигурации | `rrfFuse` |
| 4. Перспектива | Фиксированные перспективы, topK 2 | `deduplicateById` |
| 5. Многочастный | Фиксированные части, topK 2 | Вывод по частям |
| 6. Взвешенный | Основной + связанные с весами | Взвешенная сумма, сортировка |
| 7. Адаптивный | По сложности | Одиночный / расширение / больше результатов |
| 8. Дедупликация | Фиксированные запросы, topK 4 | По ID, по теме+категории, по максимальной оценке |

**Рекомендации:** предпочтительнее `retrieveParallel`; используйте RRF для нескольких ранжированных списков; используйте `deduplicateByMaxScore` для наилучшей схожести по документу; ограничивайте количество подзапросов (например, `maxSubQueries`); адаптируйте стратегию к сложности.
