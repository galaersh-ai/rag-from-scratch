# Алгоритмы поиска ближайших соседей — разбор кода

Подробное объяснение алгоритмов поиска ближайших соседей, сравнение точного и приближённого поиска, а также оптимизация производительности поиска в векторных базах данных.

## Обзор

Данный пример демонстрирует:
- Понимание поиска k ближайших соседей (kNN)
- Сравнение точного (полного перебора) и приближённого (HNSW) поиска
- Характеристики производительности и оптимизацию
- Метрики расстояний (косинусное сходство, евклидово расстояние)
- Пакетные операции поиска
- Компромисс между качеством поиска и скоростью
- Техники оптимизации производительности

**Векторная база данных:** `embedded-vector-db` (beta)
- Приближённый поиск ближайших соседей на основе HNSW
- Быстрые kNN-запросы с настраиваемыми параметрами
- Косинусное сходство в качестве метрики расстояний

---

## Настройка и конфигурация

### Импорты

```javascript
import { fileURLToPath } from "url";
import path from "path";
import { VectorDB } from "embedded-vector-db";
import { getLlama } from "node-llama-cpp";
import { Document } from "../../../src/index.js";
import { OutputHelper } from "../../../helpers/output-helper.js";
import chalk from "chalk";
```

**Те же импорты, что и в 01_in_memory_store**, с упором на операции поиска.

### Константы конфигурации

```javascript
const MODEL_PATH = path.join(__dirname, "..", "..", "..", "models", "bge-small-en-v1.5.Q8_0.gguf");
const DIM = 384;
const MAX_ELEMENTS = 10000;
const NS = "nn_search";
```

**Ключевое отличие:** Пространство имён изменено на `"nn_search"` для разделения с примерами оперативного хранилища.

---

## Основные функции

### Поиск с замером времени

```javascript
async function searchWithTiming(vectorStore, embeddingContext, query, k = 3) {
    const startEmbed = Date.now();
    const queryEmbedding = await embeddingContext.getEmbeddingFor(query);
    const embedTime = Date.now() - startEmbed;

    const startSearch = Date.now();
    const results = await vectorStore.search(NS, Array.from(queryEmbedding.vector), k);
    const searchTime = Date.now() - startSearch;

    return { results, embedTime, searchTime };
}
```

**Что делает:** Измеряет время эмбеддинга и поиска отдельно для анализа производительности.

**Почему замеры разделены?**
- Генерация эмбеддингов обычно является узким местом
- kNN-поиск, как правило, очень быстрый
- Помогает выявить возможности оптимизации

**Возвращаемое значение:**
- `results`: результаты поиска с оценками сходства
- `embedTime`: время генерации эмбеддинга запроса (мс)
- `searchTime`: время выполнения kNN-поиска (мс)

### Косинусное сходство

```javascript
function cosineSimilarity(vec1, vec2) {
    let dotProduct = 0;
    let norm1 = 0;
    let norm2 = 0;
    
    for (let i = 0; i < vec1.length; i++) {
        dotProduct += vec1[i] * vec2[i];
        norm1 += vec1[i] * vec1[i];
        norm2 += vec2[i] * vec2[i];
    }
    
    return dotProduct / (Math.sqrt(norm1) * Math.sqrt(norm2));
}
```

**Что делает:** Вычисляет косинусное сходство между двумя векторами.

**Формула:**
```
similarity = (A · B) / (||A|| × ||B||)

Где:
- A · B = скалярное произведение
- ||A|| = длина вектора A
- ||B|| = длина вектора B
```

**Диапазон:** от -1 до 1 (обычно от 0 до 1 для эмбеддингов)
- 1.0 = одинаковое направление (наибольшее сходство)
- 0.0 = ортогональные (не связаны)
- -1.0 = противоположное направление (наименьшее сходство)

**Почему косинусное сходство?**
- Для эмбеддингов направление важнее длины
- Нормировано по длине вектора
- Стандартная метрика для семантического поиска

### Евклидово расстояние

```javascript
function euclideanDistance(vec1, vec2) {
    let sum = 0;
    for (let i = 0; i < vec1.length; i++) {
        const diff = vec1[i] - vec2[i];
        sum += diff * diff;
    }
    return Math.sqrt(sum);
}
```

**Что делает:** Вычисляет прямолинейное расстояние между двумя векторами.

**Формула:**
```
distance = √(Σ(v1[i] - v2[i])²)
```

**Диапазон:** от 0 до бесконечности
- 0.0 = идентичные векторы
- Большие значения = большее различие

**Сравнение с косинусным:**
- Евклидово: учитывает и направление, и длину
- Косинусное: учитывает только направление
- Для нормированных эмбеддингов оба работают хорошо

### Полный перебор (Brute Force)

```javascript
async function bruteForceSearch(embeddingContext, documents, query, k) {
    const queryEmbedding = await embeddingContext.getEmbeddingFor(query);
    const queryVec = Array.from(queryEmbedding.vector);
    
    const distances = [];
    for (const doc of documents) {
        const docEmbedding = await embeddingContext.getEmbeddingFor(doc.pageContent);
        const docVec = Array.from(docEmbedding.vector);
        const similarity = cosineSimilarity(queryVec, docVec);
        distances.push({ doc, similarity });
    }
    
    // Sort by similarity (highest first)
    distances.sort((a, b) => b.similarity - a.similarity);
    
    return distances.slice(0, k);
}
```

**Что делает:** Реализует точный kNN-поиск путём сравнения запроса с каждым документом.

**Алгоритм:**
1. Вычислить эмбеддинг запроса
2. Вычислить эмбеддинги всех документов (если не кэшированы)
3. Вычислить сходство с каждым документом
4. Отсортировать по сходству (по убыванию)
5. Вернуть top-k результатов

**Сложность:** O(N × D)
- N = количество документов
- D = размерность векторов

**Когда использовать:**
- Небольшие наборы данных (< 1 000 документов)
- Необходимость гарантированно точных результатов
- Валидация и тестирование

---

## Пример 1: Понимание k ближайших соседей

```javascript
async function example1() {
    const vectorStore = new VectorDB({ dim: DIM, maxElements: MAX_ELEMENTS });
    const context = await initializeEmbeddingModel();
    const documents = createSampleDocuments();
    await addDocumentsToStore(vectorStore, context, documents);

    const query = "programming languages";
    const kValues = [1, 3, 5, 10];

    for (const k of kValues) {
        const results = await vectorStore.search(NS, 
            Array.from((await context.getEmbeddingFor(query)).vector), k);
        // Display results...
    }
}
```

**Что демонстрирует:** Как параметр k влияет на результаты поиска.

**Параметр k:**
- **k=1**: Только самый похожий документ
- **k=3**: Топ-3 самых похожих (сбалансированный вариант)
- **k=5**: Больше кандидатов для фильтрации
- **k=10**: Комплексные результаты, могут включать менее релевантные

**Пример вывода:**
```
Top 1 Results:
1. [0.8234] doc_1: Python is a high-level programming language...

Top 3 Results:
1. [0.8234] doc_1: Python is a high-level programming language...
2. [0.7891] doc_2: JavaScript is essential for web development...
3. [0.7543] doc_10: TypeScript adds static typing to JavaScript...

Top 5 Results:
(includes above + 2 more, possibly less relevant)
```

**Ключевой вывод:** Большее значение k предоставляет больше вариантов, но требует постобработки для фильтрации по качеству.

---

## Пример 2: Точный vs приближённый поиск

```javascript
async function example2() {
    // Setup...
    const query = "artificial intelligence";
    
    // Brute force (exact) search
    const startExact = Date.now();
    const exactResults = await bruteForceSearch(context, documents, query, 5);
    const exactTime = Date.now() - startExact;
    
    // HNSW (approximate) search
    const startApprox = Date.now();
    const queryEmbedding = await context.getEmbeddingFor(query);
    const approxResults = await vectorStore.search(NS, Array.from(queryEmbedding.vector), 5);
    const approxTime = Date.now() - startApprox;
}
```

**Что демонстрирует:** Разницу в производительности между точным и приближённым поиском.

**Точный поиск (полный перебор):**
- Сравнивает запрос с каждым документом
- Гарантирует нахождение истинных ближайших соседей
- Время: O(N × D) — линейная зависимость от размера набора данных
- Пример: 10 документов × 384 измерения ≈ 500 мс (включая повторное вычисление эмбеддинга)

**Приближённый поиск (HNSW):**
- Использует графовую структуру для пропуска нерелевантных документов
- Находит приближённых ближайших соседей (обычно >95% полноты)
- Время: O(log N × D) — логарифмическая зависимость
- Пример: log₂(10) × 384 измерения ≈ 50 мс

**Сравнение производительности:**
```
Dataset Size    Exact         Approximate    Speedup
10 docs         500ms         50ms          10x
100 docs        5,000ms       80ms          62x
1,000 docs      50,000ms      120ms         416x
10,000 docs     500,000ms     180ms         2,777x
```

**Ключевой вывод:** HNSW становится экспоненциально быстрее по мере роста набора данных!

---

## Пример 3: Производительность поиска при различных значениях k

```javascript
async function example3() {
    // Create 100 documents
    const largeDataset = [];
    for (let i = 0; i < 10; i++) {
        baseDocuments.forEach((doc, idx) => {
            largeDataset.push(new Document(doc.pageContent, {
                ...doc.metadata,
                id: `doc_${i}_${idx}`,
                batch: i,
            }));
        });
    }
    
    await addDocumentsToStore(vectorStore, context, largeDataset);
    
    const kValues = [1, 5, 10, 20, 50];
    for (const k of kValues) {
        const { results, embedTime, searchTime } = await searchWithTiming(
            vectorStore, context, query, k
        );
    }
}
```

**Что демонстрирует:** Производительность поиска при различных значениях k на более крупном наборе данных.

**Типичные результаты:**
```
k=1   → Embed: 45ms, Search: 3ms,  Total: 48ms  (1 result)
k=5   → Embed: 45ms, Search: 4ms,  Total: 49ms  (5 results)
k=10  → Embed: 45ms, Search: 5ms,  Total: 50ms  (10 results)
k=20  → Embed: 45ms, Search: 6ms,  Total: 51ms  (20 results)
k=50  → Embed: 45ms, Search: 8ms,  Total: 53ms  (50 results)
```

**Анализ:**
- Время эмбеддинга: постоянное (~45 мс), не зависит от k
- Время поиска: слегка возрастает с увеличением k (~3–8 мс)
- Общее время: определяется временем генерации эмбеддинга
- Поиск в 50 раз большего количества результатов добавляет всего 5 мс!

**Ключевой вывод:** kNN-поиск чрезвычайно быстр; оптимизацию следует направлять на генерацию эмбеддингов.

---

## Пример 4: Пакетные операции поиска

```javascript
async function example4() {
    // Create larger dataset (100 documents)
    const baseDocuments = createSampleDocuments();
    const largeDataset = [];
    for (let i = 0; i < 10; i++) {
        baseDocuments.forEach((doc, idx) => {
            largeDataset.push(new Document(doc.pageContent, {
                ...doc.metadata,
                id: `doc_${i}_${idx}`,
                batch: i,
            }));
        });
    }
    await addDocumentsToStore(vectorStore, context, largeDataset);
    
    const queries = [
        "programming languages",
        "artificial intelligence",
        "container deployment",
        "database systems",
    ];
    
    // Pre-compute embeddings for fair comparison
    const embeddings = [];
    for (const query of queries) {
        const emb = await context.getEmbeddingFor(query);
        embeddings.push({ query, vector: Array.from(emb.vector) });
    }
    
    // Sequential Search
    const startSeq = Date.now();
    for (const { query, vector } of embeddings) {
        const results = await vectorStore.search(NS, vector, 3);
    }
    const seqTime = Date.now() - startSeq;
    
    // Parallel Search
    const startPar = Date.now();
    const allResults = await Promise.all(
        embeddings.map(({ query, vector }) => 
            vectorStore.search(NS, vector, 3)
                .then(results => ({ query, results }))
        )
    );
    const parTime = Date.now() - startPar;
}
```

**Что демонстрирует:** Параллельные операции поиска быстрее последовательных, когда эмбеддинги предварительно вычислены.

**Ключевой вывод:** Справедливое сравнение производительности требует разделения генерации эмбеддингов и операций поиска.

**Последовательный подход:**
```javascript
// Pre-computed embeddings used
for (const { vector } of embeddings) {
    const res = await search(vector);    // 1-2ms each
}
// Total: 4 × 1-2ms = 4-8ms
```

**Параллельный подход:**
```javascript
// All searches execute concurrently
const results = await Promise.all(
    embeddings.map(({ vector }) => search(vector))
);
// Total: ~1-2ms (runs in parallel!)
```

**Почему параллельный подход быстрее:**
- Векторные поиски — независимые операции
- JavaScript async/await обеспечивает параллельное выполнение
- Между поисками нет общего состояния
- Каждый поиск обращается к заблокированному для чтения индексу параллельно

**Важные замечания:**
- Эмбеддинги должны быть предварительно вычислены для справедливого сравнения
- Ускорение за счёт параллелизма наиболее заметно на более крупных наборах данных
- При очень малых наборах данных (< 100 документов) поиск настолько быстр (< 1 мс), что разница во времени может быть пренебрежимо малой
- Реальная выгода проявляется при поиске в более крупных индексах или при большом количестве запросов

**Лучшая практика:**
```javascript
// Always separate embedding from search for batch operations
// 1. Pre-compute all embeddings
const embeddings = await Promise.all(queries.map(q => embed(q)));

// 2. Execute searches in parallel
const results = await Promise.all(embeddings.map(e => search(e)));
```

---

## Пример 5: Сравнение метрик расстояний

```javascript
async function example5() {
    const texts = [
        "Python programming language",
        "JavaScript programming language",
        "Cooking delicious pasta",
    ];
    
    const embeddings = [];
    for (const text of texts) {
        const emb = await context.getEmbeddingFor(text);
        embeddings.push(Array.from(emb.vector));
    }
    
    // Cosine Similarity Matrix
    for (let i = 0; i < embeddings.length; i++) {
        for (let j = 0; j < embeddings.length; j++) {
            const sim = cosineSimilarity(embeddings[i], embeddings[j]);
            console.log(sim.toFixed(4));
        }
    }
    
    // Euclidean Distance Matrix
    for (let i = 0; i < embeddings.length; i++) {
        for (let j = 0; j < embeddings.length; j++) {
            const dist = euclideanDistance(embeddings[i], embeddings[j]);
            console.log(dist.toFixed(4));
        }
    }
}
```

**Что демонстрирует:** Как различные метрики измеряют сходство.

**Пример вывода:**

**Косинусное сходство:**
```
        Text 1    Text 2    Text 3
Text 1  1.0000    0.8532    0.2145
Text 2  0.8532    1.0000    0.2389
Text 3  0.2145    0.2389    1.0000
```

**Евклидово расстояние:**
```
        Text 1    Text 2    Text 3
Text 1  0.0000    4.2314    15.8923
Text 2  4.2314    0.0000    16.2341
Text 3  15.8923   16.2341   0.0000
```

**Интерпретация:**
- Текст 1 и 2 (оба о программировании): высокое сходство (0.85), малое расстояние (4.2)
- Текст 3 (о готовке): низкое сходство (0.21), большое расстояние (15.9)
- Диагональ: тождество (сходство 1.0, расстояние 0.0)

**Какую метрику использовать?**
- **Косинусное сходство**: стандарт для семантического поиска
- **Евклидово расстояние**: альтернатива, учитывает длину вектора
- **Скалярное произведение**: быстрая, ненормированная версия косинусного
- **Расстояние Манхэттена**: L1-расстояние, используется редко

**embedded-vector-db по умолчанию использует косинусное сходство** (через hnswlib).

---

## Пример 6: Качество поиска vs производительность

```javascript
async function example6() {
    const query = "programming languages for beginners";
    
    const strategies = [
        { name: "Fast Search (k=3)", k: 3 },
        { name: "Balanced Search (k=5)", k: 5 },
        { name: "Comprehensive Search (k=10)", k: 10 },
    ];
    
    for (const strategy of strategies) {
        const { results, embedTime, searchTime } = await searchWithTiming(
            vectorStore, context, query, strategy.k
        );
        
        results.forEach((result, index) => {
            const score = result.similarity;
            const relevance = score > 0.6 ? "High" : 
                             score > 0.4 ? "Medium" : "Low";
            console.log(`${index + 1}. [${score}] ${relevance}`);
        });
    }
}
```

**Что демонстрирует:** Компромисс между скоростью поиска и качеством результатов.

**Сравнение стратегий:**

**Быстрый поиск (k=3):**
- **Плюсы**: самый быстрый, наиболее сфокусированные результаты
- **Минусы**: может пропустить релевантные документы
- **Сценарий использования**: быстрый поиск, поиск в реальном времени
- **Производительность**: ≈ 50 мс

**Сбалансированный поиск (k=5):**
- **Плюсы**: хорошее покрытие, управляемый объём результатов
- **Минусы**: может включать 1–2 пограничных результата
- **Сценарий использования**: самый распространённый, общего назначения
- **Производительность**: ≈ 51 мс

**Комплексный поиск (k=10):**
- **Плюсы**: максимальное покрытие, возможность переранжирования
- **Минусы**: больше шума, требуется фильтрация
- **Сценарий использования**: требования к высокой полноте, переранжирование
- **Производительность**: ≈ 53 мс

**Рекомендации:**
```javascript
// Quick search
const results = await search(query, 3);

// With re-ranking
const candidates = await search(query, 20);
const reranked = await reranker.rank(query, candidates);
const top = reranked.slice(0, 5);

// With threshold
const results = await search(query, 10);
const filtered = results.filter(r => r.similarity > 0.5);
```

---

## Пример 7: Оптимизация производительности поиска

```javascript
async function example7() {
    // Technique 1: Cache embeddings
    const cachedEmbedding = await context.getEmbeddingFor(query);
    const cachedVec = Array.from(cachedEmbedding.vector);
    
    for (let i = 0; i < 3; i++) {
        await vectorStore.search(NS, cachedVec, 5);
    }
    
    // Technique 2: Over-fetching for filtering
    const manyResults = await vectorStore.search(NS, queryVec, 10);
    const filtered = manyResults
        .filter(r => r.metadata.category === "programming")
        .slice(0, 3);
    
    // Technique 3: Batch processing
    const embeddings = await Promise.all(queries.map(q => embed(q)));
    await Promise.all(embeddings.map(e => search(e, 3)));
}
```

**Что демонстрирует:** Три ключевые техники оптимизации.

### Техника 1: Кэширование эмбеддингов

**Проблема:** Повторное вычисление эмбеддинга для одного и того же запроса тратит время.

**Решение:**
```javascript
const embedCache = new Map();

async function searchCached(query, k) {
    if (!embedCache.has(query)) {
        const emb = await embed(query);
        embedCache.set(query, Array.from(emb.vector));
    }
    return await search(embedCache.get(query), k);
}
```

**Когда использовать:**
- Повторяющиеся запросы (например, постраничная навигация)
- Популярные поисковые запросы
- Шаблонные запросы с параметрами

### Техника 2: Избыточная выборка для фильтрации

**Проблема:** Необходимо k результатов после фильтрации по метаданным.

**Решение:**
```javascript
// Instead of: search(k=3) → filter → might get 0-3 results
// Do: search(k=10) → filter → slice(0, 3) → always 3 results

const candidates = await search(query, 20);
const filtered = candidates
    .filter(r => r.metadata.category === "programming")
    .filter(r => r.similarity > 0.5)
    .slice(0, 5);
```

**Правило:**
- При ожидаемом коэффициенте фильтрации 50%: выбирать 2× k
- При ожидаемом коэффициенте фильтрации 25%: выбирать 4× k
- Мониторить реальные показатели и корректировать

### Техника 3: Пакетная обработка

**Проблема:** Последовательные операции медленные.

**Решение:**
```javascript
// Sequential: 4 queries × 50ms = 200ms
for (const q of queries) {
    await search(q);
}

// Parallel: max(50ms each running concurrently) ≈ 50-70ms
const embeddings = await Promise.all(queries.map(embed));
const results = await Promise.all(embeddings.map(search));
```

**Лучшие практики:**
- Используйте `Promise.all()` для независимых операций
- Ограничивайте параллелизм при необходимости (например, лимиты частоты запросов)
- Рассмотрите использование `p-limit` для управляемого параллелизма

---

## Сводка оптимизации производительности

### Анализ узких мест

**Разбивка времени для типичного поиска:**
```
Total: ~50ms
├── Embedding generation: ~45ms (90%)
└── kNN search: ~5ms (10%)
```

**Приоритеты оптимизации:**
1. **Кэширование эмбеддингов** (45 мс → 0 мс при кэшировании)
2. **Пакетные операции** (4 × 50 мс → 70 мс)
3. **Использование GPU для эмбеддингов** (45 мс → 5 мс)
4. Параметры kNN (минимальное влияние)

### Чек-лист оптимизации

**✓ Рекомендуется:**
- Кэшировать эмбеддинги запросов
- Использовать пакетные операции с `Promise.all()`
- Предварительно вычислять эмбеддинги offline, когда это возможно
- Мониторить раздельное время для эмбеддинга и поиска
- Использовать избыточную выборку при фильтрации по метаданным

**✗ Не рекомендуется:**
- Повторно вычислять эмбеддинг для одного и того же запроса
- Обрабатывать запросы последовательно, когда возможен параллелизм
- Сосредотачиваться на оптимизации kNN до оптимизации эмбеддингов
- Извлекать k=100, когда нужно только k=5

---

## Сравнение алгоритмов

### HNSW (иерархический навигируемый малый мир)

**Принцип работы:**
1. Построение многослойного графа векторов
2. Навигация от входной точки с использованием жадного поиска
3. Переход к более близким соседям на каждом шаге
4. Возврат приближённых ближайших соседей

**Преимущества:**
- Быстрый: время поиска O(log N)
- Масштабируемый: работает с миллионами векторов
- Хорошая полнота: обычно >95% точности

**Недостатки:**
- Приближённый, не точный
- Более высокое потребление памяти (графовая структура)
- Более медленные вставки по сравнению с полным перебором

**Лучше всего подходит для:**
- Продакшн-систем
- Крупных наборов данных (> 1 000 документов)
- Требований к поиску в реальном времени

### Полный перебор (точный поиск)

**Принцип работы:**
1. Сравнение запроса с каждым документом
2. Сортировка по сходству
3. Возврат top-k

**Преимущества:**
- Точные результаты (100% полнота)
- Простая реализация
- Отсутствие накладных расходов на индекс

**Недостатки:**
- Медленный: время поиска O(N)
- Не масштабируется
- Непрактичен для крупных наборов данных

**Лучше всего подходит для:**
- Небольших наборов данных (< 1 000 документов)
- Валидации и тестирования
- Базовых сравнений

---

## Сводка ключевых понятий

### 1. k ближайших соседей

**Определение:** Найти k векторов, наиболее похожих на вектор запроса.

**Рекомендации по выбору k:**
- **k=1–3**: быстрые, сфокусированные результаты
- **k=5–10**: сбалансированные, общего назначения
- **k=20–50**: комплексные, для переранжирования
- **k>50**: обычно избыточно, шум увеличивается

### 2. Приближённый vs точный поиск

**Компромисс:**
- **Точный**: медленнее, но 100% точность
- **Приближённый**: быстрее, но ~95% точность
- **Гибридный**: приближённый для кандидатов, точный для top-k

**Когда достаточно приближённого:**
- Семантический поиск (небольшая потеря точности допустима)
- Требования к поиску в реальном времени
- Крупные наборы данных

### 3. Метрики расстояний

**Косинусное сходство:**
- Измеряет угол между векторами
- Диапазон: от -1 до 1
- Стандарт для эмбеддингов

**Евклидово расстояние:**
- Измеряет прямолинейное расстояние
- Диапазон: от 0 до ∞
- Учитывает длину вектора

### 4. Оптимизация производительности

**Ключевые выводы:**
- Генерация эмбеддингов является узким местом (90% времени)
- kNN-поиск очень быстрый (10% времени)
- Параллелизация даёт наибольшее ускорение
- Кэширование устраняет избыточные вычисления

---

## Лучшие практики

### 1. Выбор подходящего k

```javascript
// Too small: might miss relevant results
const results = await search(query, 1);  // ❌ Only 1 result

// Good: balanced
const results = await search(query, 5);  // ✓ Common choice

// Too large: unnecessary noise
const results = await search(query, 100);  // ❌ Too many
```

### 2. Кэширование эмбеддингов

```javascript
const cache = new Map();

async function searchWithCache(query, k) {
    if (!cache.has(query)) {
        const emb = await embed(query);
        cache.set(query, emb);
    }
    return await search(cache.get(query), k);
}
```

### 3. Использование пакетных операций

```javascript
// ❌ Slow: sequential
for (const query of queries) {
    await search(query);
}

// ✓ Fast: parallel
await Promise.all(queries.map(q => search(q)));
```

### 4. Мониторинг производительности

```javascript
async function searchWithMetrics(query, k) {
    const start = Date.now();
    const embedStart = Date.now();
    const embedding = await embed(query);
    const embedTime = Date.now() - embedStart;
    
    const searchStart = Date.now();
    const results = await search(embedding, k);
    const searchTime = Date.now() - searchStart;
    
    logMetrics({ embedTime, searchTime, total: Date.now() - start });
    return results;
}
```

---

## Итог

### Что мы построили

Семь примеров, демонстрирующих:
1. ✅ Основы k ближайших соседей
2. ✅ Сравнение точного и приближённого поиска
3. ✅ Характеристики производительности при различных значениях k
4. ✅ Пакетные операции поиска
5. ✅ Сравнение метрик расстояний
6. ✅ Компромисс между качеством поиска и производительностью
7. ✅ Техники оптимизации производительности

### Ключевые выводы

- **kNN** находит k наиболее похожих векторов на запрос
- **HNSW** экспоненциально быстрее полного перебора для крупных наборов данных
- **Генерация эмбеддингов** является узким местом производительности, а не поиск
- **Косинусное сходство** — стандартная метрика для семантического поиска
- **Кэширование и пакетная обработка** обеспечивают наибольший прирост производительности
- **Параметр k** управляет компромиссом между количеством и качеством результатов

### Рекомендации по производительности

| Размер набора данных | Алгоритм | Ожидаемое время | Примечания |
|---------------------|----------|----------------|------------|
| < 1 000 | Полный перебор | < 100 мс | Точные результаты |
| 1 000–10 000 | HNSW | < 50 мс | >95% полноты |
| 10 000–100 000 | HNSW | < 100 мс | Отличный результат |
| > 100 000 | HNSW | < 200 мс | Всё ещё быстро |

### Следующие шаги

- **03_metadata_filtering**: Продвинутые стратегии фильтрации

Алгоритмы поиска ближайших соседей, которые вы изучили, являются основой всех векторных баз данных. Освойте эти понятия, и вы поймёте, как работает современный семантический поиск!
