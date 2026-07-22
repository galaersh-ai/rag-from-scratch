# Гибридный поиск — разбор кода

Это руководство пошагово объясняет код гибридного поиска для электронной коммерции. К его прочтению вы поймёте, как сочетать векторный поиск (семантическое понимание) с ключевым поиском (точное совпадение) для поиска товаров.

## Содержание

1. [Что такое гибридный поиск?](#что-такое-гибридный-поиск)
2. [Установка и импорты](#установка-и-импорты)
3. [Конфигурация](#конфигурация)
4. [Основные функции](#основные-функции)
5. [Пример 1: Проблемы с SKU и брендами](#пример-1-проблемы-с-sku-и-брендами)
6. [Пример 2: Нормализация оценок](#пример-2-нормализация-оценок)
7. [Пример 3: Многополевой поиск](#пример-3-многополевой-поиск)
8. [Пример 4: Динамическая настройка весов](#пример-4-динамическая-настройка-весов)
9. [Пример 5: Стратегии запасного поиска](#пример-5-стратегии-запасного-поиска)
10. [Пример 6: Поиск с учётом фильтров](#пример-6-поиск-с-учётом-фильтров)
11. [Пример 7: Оптимизация производительности](#пример-7-оптимизация-производительности)

---

## Что такое гибридный поиск?

**Гибридный поиск** сочетает два различных метода поиска:

1. **Векторный поиск** (семантический)
   - Понимает *значение* слов
   - Пример: «ноутбук для монтажа» совпадает с «MacBook Pro для работы с видео»
   - Подходит для: естественного языка, концепций, перефразированных запросов

2. **Ключевой поиск** (BM25)
   - Ищет *точные совпадения слов*
   - Пример: «MBP-M3MAX-32» совпадает только с соответствующим SKU
   - Подходит для: кодов товаров, названий брендов, технических характеристик

**Зачем их сочетать?** Каждый метод имеет свои сильные и слабые стороны. Гибридный поиск даёт вам лучшее из обоих миров!

---

## Установка и импорты

```javascript
import { fileURLToPath } from "url";
import path from "path";
import { VectorDB } from "embedded-vector-db";
import { getLlama } from "node-llama-cpp";
import { Document } from "../../../src/index.js";
import { OutputHelper } from "../../../helpers/output-helper.js";
import chalk from "chalk";
```

**Назначение каждого импорта:**

- `fileURLToPath` и `path`: помощь в поиске файлов на компьютере
- `VectorDB`: база данных для хранения эмбеддингов товаров (векторных представлений)
- `getLlama`: загружает ИИ-модель для создания эмбеддингов
- `Document`: обёртка для данных о товарах
- `OutputHelper`: утилита для красивого вывода в консоль
- `chalk`: делает текст в консоли цветным (необязательно, просто красиво!)

---

## Конфигурация

```javascript
const __dirname = path.dirname(fileURLToPath(import.meta.url));
const EMBEDDING_MODEL_PATH = path.join(__dirname, "..", "..", "..", "models", "bge-small-en-v1.5.Q8_0.gguf");

const DIM = 384;
const MAX_ELEMENTS = 10000;
const NS = "product_search";
```

**Значение этих констант:**

- `__dirname`: директория текущего файла
- `EMBEDDING_MODEL_PATH`: путь к файлу ИИ-модели
- `DIM`: размерность эмбеддинга (384 числа описывают каждый товар)
- `MAX_ELEMENTS`: максимальное количество товаров, которое может хранить база данных
- `NS`: пространство имён (аналог имени папки в базе данных)

---

## Основные функции

### Функция 1: Инициализация модели эмбеддингов

```javascript
async function initializeEmbeddingModel() {
    try {
        const llama = await getLlama({ logLevel: "error" });
        const model = await llama.loadModel({ modelPath: EMBEDDING_MODEL_PATH });
        return await model.createEmbeddingContext();
    } catch (error) {
        throw new Error(`Failed to initialize embedding model: ${error.message}`);
    }
}
```

**Что она делает:**
1. Загружает ИИ-модель с диска
2. Создаёт «контекст эмбеддингов» (инструмент для преобразования текста → числа)
3. Возвращает контекст эмбеддингов для дальнейшего использования

**Зачем это нужно:** Чтобы преобразовать описания товаров и поисковые запросы в векторы (массивы чисел), которые компьютер может сравнивать.

---

### Функция 2: Создание каталога товаров

```javascript
function createProductCatalog() {
    return [
        new Document("Apple MacBook Pro 16-inch with M3 Max chip...", {
            id: "PROD-001",
            title: "MacBook Pro 16-inch M3 Max",
            brand: "Apple",
            category: "laptops",
            price: 3499,
            sku: "MBP-M3MAX-32-1TB",
            attributes: "M3 Max, 32GB RAM, 1TB SSD..."
        }),
        // ... больше товаров
    ];
}
```

**Что она делает:**
Создаёт массив документов с товарами. Каждый товар содержит:
- **Содержимое** (первый параметр): полное описание
- **Метаданные** (второй параметр): структурированные данные — SKU, цена, бренд

**Почему структура именно такая:**
- Содержимое используется для семантического поиска (понимание смысла)
- Поля метаданных используются для точного совпадения и фильтрации

---

### Функция 3: Добавление товаров в хранилище

```javascript
async function addProductsToStore(vectorStore, embeddingContext, products) {
    try {
        // Enable full-text indexing on multiple fields
        await vectorStore.setFullTextIndexedFields(NS, 
            ['content', 'title', 'brand', 'sku', 'attributes']
        );

        for (const product of products) {
            // Convert product description to vector
            const embedding = await embeddingContext.getEmbeddingFor(product.pageContent);
            
            // Prepare metadata
            const metadata = {
                content: product.pageContent,
                ...product.metadata,
            };
            
            // Store in database
            await vectorStore.insert(
                NS,
                product.metadata.id,
                Array.from(embedding.vector),
                metadata
            );
        }
    } catch (error) {
        throw new Error(`Failed to add products to store: ${error.message}`);
    }
}
```

**Пошаговое описание:**

1. **`setFullTextIndexedFields`**: указывает базе данных, какие поля индексировать для ключевого поиска
   - Как создание указателя в книге для быстрого поиска

2. **Перебор товаров**: обработка каждого товара по очереди

3. **`getEmbeddingFor`**: преобразует описание товара в вектор (массив из 384 чисел)
   - Пример: «MacBook Pro 16-inch...» → [0.23, -0.45, 0.12, ...]

4. **`insert`**: сохраняет товар с указанием:
   - Пространства имён (папки)
   - Идентификатора товара
   - Вектора (для семантического поиска)
   - Метаданных (для ключевого поиска и фильтрации)

---

## Пример 1: Проблемы с SKU и брендами

Этот пример показывает, почему ключевой поиск критически важен для кодов товаров.

### Тест 1: Поиск по точному SKU

```javascript
const query1 = "MBP-M3MAX-32-1TB";
const queryEmbedding1 = await embeddingContext.getEmbeddingFor(query1);
const vectorResults1 = await vectorStore.search(NS, Array.from(queryEmbedding1.vector), 3);
const keywordResults1 = await vectorStore.fullTextSearch(NS, query1, 3);
```

**Что происходит:**

1. **Векторный поиск**:
   - Преобразует «MBP-M3MAX-32-1TB» в числа
   - Проблема: модель не понимает коды товаров!
   - Результат: может вернуть случайные товары

2. **Ключевой поиск (BM25)**:
   - Ищет точное совпадение текста
   - Находит товары с «MBP-M3MAX-32-1TB» в поле SKU
   - Результат: идеальное совпадение!

**Ключевой вывод:**
```
SKU-коды вроде «MBP-M3MAX-32-1TB» являются «вне словаря» (OOV) для ИИ-моделей.
Модель воспринимает их как случайные символы, но ключевой поиск находит их безупречно.
```

### Тест 2: Поиск по названию бренда

```javascript
const query2 = "Sony headphones";
```

**Что происходит:**

- **Векторный поиск**: находит товары Sony + похожие товары (например, Bose)
- **Ключевой поиск**: приоритизирует точные совпадения «Sony»

**Лучший подход:** использовать оба метода (гибридный поиск) для получения:
- Точного совпадения бренда (ключевой поиск)
- Семантического понимания категории (векторный поиск)

---

## Пример 2: Нормализация оценок

### Проблема

```javascript
const vecScores = vectorResults.map(r => r.similarity);
const keyScores = keywordResults.map(r => r.similarity);

// Vector scores might be: [0.85, 0.83, 0.80]
// Keyword scores might be: [12.5, 8.3, 5.1]
```

**Проблема:** разные шкалы! Как справедливо их объединить?

Если просто сложить: `0.85 + 12.5 = 13.35` (ключевой поиск доминирует!)

### Решение 1: Мин-макс нормализация

```javascript
const normalizeMinMax = (scores) => {
    const min = Math.min(...scores);
    const max = Math.max(...scores);
    const range = max - min || 1;
    return scores.map(s => (s - min) / range);
};
```

**Что она делает:**
- Масштабирует все оценки в диапазон [0, 1]
- Формула: `(оценка - минимум) / (максимум - минимум)`
- Пример: [12.5, 8.3, 5.1] → [1.0, 0.43, 0]

**Плюсы:** просто, интуитивно
**Минусы:** чувствительна к выбросам (одна очень высокая/низкая оценка влияет на всё)

### Решение 2: Z-нормализация

```javascript
const normalizeZScore = (scores) => {
    const mean = scores.reduce((a, b) => a + b, 0) / scores.length;
    const variance = scores.reduce((a, b) => a + Math.pow(b - mean, 2), 0) / scores.length;
    const stdDev = Math.sqrt(variance) || 1;
    return scores.map(s => (s - mean) / stdDev);
};
```

**Что она делает:**
- Центрирует оценки вокруг 0
- Формула: `(оценка - среднее) / стандартное_отклонение`
- Пример: [12.5, 8.3, 5.1] → [1.2, 0.1, -1.3]

**Плюсы:** сохраняет распределение, лучше работает с выбросами
**Минусы:** может давать отрицательные оценки (но это допустимо для сравнения)

### Решение 3: Нормализация по рангу

```javascript
const normalizeRank = (scores) => {
    const n = scores.length;
    return scores.map((_, idx) => (n - idx) / n);
};
```

**Что она делает:**
- Игнорирует фактические оценки, используя только ранжирование
- Формула: `(общее количество - ранг) / общее количество`
- Пример (для 5 результатов): [1.0, 0.8, 0.6, 0.4, 0.2]

**Плюсы:** наиболее устойчивая, используется в Reciprocal Rank Fusion (RRF)
**Минусы:** теряет информацию о разнице между оценками

---

## Пример 3: Многополевой поиск

### Зачем несколько полей?

```javascript
await vectorStore.setFullTextIndexedFields(NS, 
    ['content', 'title', 'brand', 'sku', 'attributes']
);
```

**Каждое поле выполняет свою задачу:**

| Поле | Назначение | Пример запроса |
|------|-----------|----------------|
| `content` | Семантическое понимание | «ноутбук для монтажа видео» |
| `title` | Совпадение по названию товара | «MacBook Pro» |
| `brand` | Точное совпадение по бренду | «Apple» |
| `sku` | Совпадение по коду товара | «MBP-M3MAX-32-1TB» |
| `attributes` | Технические характеристики | «32GB RAM» |

### Как это работает

```javascript
const query = "Apple wireless keyboard";
```

**Процесс поиска:**
1. Одновременно ищет по ВСЕМ индексированным полям
2. «Apple» сильно совпадает в поле `brand`
3. «wireless» совпадает в полях `title` и `attributes`
4. «keyboard» совпадает в полях `title` и `content`
5. Объединяет все сигналы для итоговой оценки

**Результат:** более комплексный и точный поиск!

---

## Пример 4: Динамическая настройка весов

### Функция analyzeQuery

```javascript
function analyzeQuery(query) {
    const upperCount = (query.match(/[A-Z]/g) || []).length;
    const digitCount = (query.match(/\d/g) || []).length;
    const hyphenCount = (query.match(/-/g) || []).length;
    const wordCount = query.split(/\s+/).length;
    const hasQuestionWord = /^(what|how|which|where|why|when|who)/i.test(query);

    // Detect query type and return optimal weights
    let queryType, weights;

    if (hyphenCount >= 2 || (upperCount > 3 && digitCount > 0)) {
        queryType = "SKU/Model Number";
        weights = { vector: 0.2, text: 0.8 };
    } else if (digitCount >= 3) {
        queryType = "Technical Specs";
        weights = { vector: 0.3, text: 0.7 };
    } else if (hasQuestionWord || wordCount >= 6) {
        queryType = "Natural Language Question";
        weights = { vector: 0.8, text: 0.2 };
    } else if (wordCount <= 2) {
        queryType = "Short Keyword";
        weights = { vector: 0.4, text: 0.6 };
    } else {
        queryType = "Mixed Query";
        weights = { vector: 0.5, text: 0.5 };
    }

    return { queryType, weights };
}
```

### Понимание логики

**Определение паттерна:**

1. **SKU/модель** (`hyphenCount >= 2`)
   - Пример: «ASUS-G14-R9-4070»
   - Много дефисов + заглавные буквы + цифры = код товара
   - Приоритет ключевого поиска: **0.2 векторный / 0.8 текстовый**

2. **Технические характеристики** (`digitCount >= 3`)
   - Пример: «32GB RAM 4K display»
   - Несколько чисел = технические требования
   - Приоритет ключевого поиска: **0.3 векторный / 0.7 текстовый**

3. **Вопрос на естественном языке** (начинается со слов-вопросов)
   - Пример: «What's the best laptop for video editing?»
   - Требует семантического понимания
   - Приоритет векторного поиска: **0.8 векторный / 0.2 текстовый**

4. **Короткое ключевое слово** (`wordCount <= 2`)
   - Пример: «Sony headphones»
   - Краткий запрос, вероятно содержит бренд/категорию
   - Приоритет ключевого поиска: **0.4 векторный / 0.6 текстовый**

5. **Смешанный запрос** (по умолчанию)
   - Пример: «portable charger fast charging»
   - Может выиграть от обоих подходов
   - Сбалансированный режим: **0.5 векторный / 0.5 текстовый**

### Как использовать

```javascript
const query = "ASUS-G14-R9-4070";
const analysis = analyzeQuery(query);
// Returns: { queryType: "SKU/Model Number", weights: { vector: 0.2, text: 0.8 } }

const results = await vectorStore.hybridSearch(
    NS,
    Array.from(queryEmbedding.vector),
    query,
    {
        vectorWeight: analysis.weights.vector,  // 0.2
        textWeight: analysis.weights.text,      // 0.8
        k: 2
    }
);
```

**Почему это важно:**
- Автоматическая оптимизация на основе типа запроса
- Лучший пользовательский опыт без ручной настройки
- Адаптация к различным паттернам поиска

---

## Пример 5: Стратегии запасного поиска

### Проблема: ноль результатов

```javascript
const query = "Microsoft Surface laptop touchscreen";
```

Проблема: в каталоге нет товаров Microsoft!

### Последовательность стратегий

#### Стратегия 1: Чистый ключевой поиск (скорее всего не сработает)

```javascript
const keywordOnly = await vectorStore.fullTextSearch(NS, query, 3);
// Result: 0 results (no exact "Microsoft" match)
```

**Почему не работает:** ни один товар не содержит слово «Microsoft»

#### Стратегия 2: Сбалансированный гибридный поиск

```javascript
const hybrid = await vectorStore.hybridSearch(
    NS,
    Array.from(queryEmbedding.vector),
    query,
    { k: 3 }  // Default: 0.5/0.5
);
// Result: Returns laptops with touchscreens (similar products)
```

**Почему работает лучше:** векторный поиск семантически понимает слова «ноутбук» и «сенсорный экран»

#### Стратегия 3: Запасной вариант с приоритетом векторного поиска

```javascript
const vectorFallback = await vectorStore.hybridSearch(
    NS,
    Array.from(queryEmbedding.vector),
    query,
    {
        vectorWeight: 0.9,
        textWeight: 0.1,
        k: 3
    }
);
// Result: Returns most semantically similar laptops
```

**Зачем это использовать:** когда ключевой поиск полностью не срабатывает, полагаемся на семантическое сходство

### Рекомендуемая последовательность запасного поиска

```
1. Попробовать сбалансированный гибридный поиск (0.5/0.5)
   ↓ (если < 3 результатов)
2. Попробовать с приоритетом векторного поиска (0.8/0.2)
   ↓ (если всё ещё < 3 результатов)
3. Использовать чистый векторный поиск (1.0/0.0)
   ↓
4. Показать пользователю сообщение «Похожие товары»
```

**В коде:**

```javascript
let results = await hybridSearch(query, { vectorWeight: 0.5, textWeight: 0.5, k: 10 });

if (results.length < 3) {
    // Fallback to vector-heavy
    results = await hybridSearch(query, { vectorWeight: 0.8, textWeight: 0.2, k: 10 });
}

if (results.length < 3) {
    // Final fallback: pure vector
    results = await hybridSearch(query, { vectorWeight: 1.0, textWeight: 0.0, k: 10 });
    // Show message: "No exact matches. Here are similar products:"
}
```

---

## Пример 6: Поиск с учётом фильтров

### Сочетание поиска с бизнес-логикой

```javascript
const query = "high-end laptop professional work";
const maxPrice = 2500;
```

**Двухэтапный процесс:**

#### Этап 1: Гибридный поиск

```javascript
const results = await vectorStore.hybridSearch(
    NS,
    Array.from(queryEmbedding.vector),
    query,
    { k: 5 }
);
```

Находит релевантные товары на основе запроса.

#### Этап 2: Применение бизнес-фильтров

```javascript
// Filter by price
const priceFiltered = results.filter(doc => doc.metadata.price <= maxPrice);

// Filter by category AND price
const categoryFiltered = results.filter(doc =>
    doc.metadata.category === 'laptops' && 
    doc.metadata.price <= maxPrice
);
```

### Типичные фильтры для электронной коммерции

```javascript
// Price range
const inBudget = results.filter(r => 
    r.metadata.price >= minPrice && 
    r.metadata.price <= maxPrice
);

// In stock
const available = results.filter(r => 
    r.metadata.inStock === true
);

// Category
const category = results.filter(r => 
    r.metadata.category === 'laptops'
);

// Brand
const brand = results.filter(r => 
    r.metadata.brand === 'Apple'
);

// Rating
const highRated = results.filter(r => 
    r.metadata.rating >= 4.0
);
```

### Предварительная фильтрация vs фильтрация после поиска

**Фильтрация после поиска (показано выше):**
```javascript
// Search first, filter after
const results = await hybridSearch(query, { k: 100 });
const filtered = results.filter(r => r.metadata.price <= maxPrice);
```

**Плюсы:** просто, гибко
**Минусы:** неэффективно для больших баз данных

**Предварительная фильтрация (продакшен-подход):**
```javascript
// Filter BEFORE searching (if database supports it)
const results = await hybridSearch(query, { 
    k: 100,
    metadataFilter: { 
        price: { $lte: maxPrice },
        category: 'laptops'
    }
});
```

**Плюсы:** гораздо быстрее, поиск по меньшему набору данных
**Минусы:** требует поддержки со стороны базы данных

---

## Пример 7: Оптимизация производительности

### Оптимизация 1: Кэширование результатов запросов

```javascript
const queryCache = new Map();

async function cachedHybridSearch(query, options = {}) {
    // Create cache key from query + options
    const cacheKey = `${query}:${JSON.stringify(options)}`;

    // Check cache first
    if (queryCache.has(cacheKey)) {
        return { results: queryCache.get(cacheKey), cached: true };
    }

    // Cache miss: perform search
    const queryEmbedding = await embeddingContext.getEmbeddingFor(query);
    const results = await vectorStore.hybridSearch(
        NS,
        Array.from(queryEmbedding.vector),
        query,
        options
    );

    // Store in cache
    queryCache.set(cacheKey, results);
    return { results, cached: false };
}
```

**Как это работает:**

1. **Первый поиск** по запросу «wireless headphones»:
   - Вычисление эмбеддинга (медленно)
   - Поиск в базе данных (медленно)
   - Сохранение результата в кэш
   - Время: ~100 мс

2. **Повторный поиск** по тому же запросу:
   - Нахождение в кэше (мгновенно)
   - Возврат кэшированных результатов
   - Время: ~1 мс
   - **Ускорение в 100 раз!**

**Важные рекомендации:**

```javascript
// Set cache expiration (Time-To-Live)
const CACHE_TTL = 5 * 60 * 1000; // 5 minutes

queryCache.set(cacheKey, {
    results: results,
    timestamp: Date.now()
});

// Check if cache entry is still valid
if (queryCache.has(cacheKey)) {
    const cached = queryCache.get(cacheKey);
    if (Date.now() - cached.timestamp < CACHE_TTL) {
        return cached.results;
    } else {
        queryCache.delete(cacheKey); // Expired
    }
}
```

### Оптимизация 2: Двухэтапный ретривер

**Проблема:** Вычисление эмбеддингов для большого числа товаров — медленный процесс.

**Решение:** поиск в два этапа:

```javascript
// Stage 1: Fast keyword search to get candidates
const candidates = await vectorStore.fullTextSearch(NS, query, 100);
// Returns 100 candidates in ~10ms

// Stage 2: Rerank top candidates with vector search
const candidateIds = candidates.map(c => c.metadata.id);
const reranked = await vectorStore.search(NS, queryEmbedding, 10, {
    filter: { id: { $in: candidateIds } }
});
// Only computes similarity for 100 products, not all 10,000!
```

**Почему это быстрее:**
- Этап 1: ключевой поиск выполняется очень быстро
- Этап 2: вычисление дорогостоящих векторов только для лучших кандидатов
- **Ускорение в 10–100 раз** для больших баз данных

### Оптимизация 3: Разбиение индекса

```javascript
// Create separate indices for categories
const laptopStore = new VectorDB({ dim: 384, namespace: "laptops" });
const audioStore = new VectorDB({ dim: 384, namespace: "audio" });
const accessoryStore = new VectorDB({ dim: 384, namespace: "accessories" });

// Search only relevant partition
async function partitionedSearch(query, category) {
    let store;
    switch(category) {
        case 'laptops': store = laptopStore; break;
        case 'audio': store = audioStore; break;
        case 'accessories': store = accessoryStore; break;
    }
    
    return await store.hybridSearch(/* ... */);
}
```

**Преимущества:**
- Меньшее пространство поиска = быстрее
- Возможность оптимизации каждого раздела отдельно
- Упрощённое горизонтальное масштабирование

### Оптимизация 4: Предварительно вычисленные эмбеддинги

**Плохо (медленно):**
```javascript
// Compute embedding every time
for (const product of products) {
    const embedding = await getEmbedding(product.description);
    await store.insert(id, embedding, metadata);
}
```

**Хорошо (быстро):**
```javascript
// Compute embeddings once, store them
const productsWithEmbeddings = products.map(async (p) => ({
    ...p,
    embedding: await getEmbedding(p.description)
}));

// Save embeddings to database or file
await saveEmbeddings(productsWithEmbeddings);

// Later: Load pre-computed embeddings
const cachedProducts = await loadEmbeddings();
for (const product of cachedProducts) {
    await store.insert(id, product.embedding, metadata);
}
```

**Почему:** вычисление эмбеддингов — самая медленная часть. Выполните это один раз и многократно используйте!

---

## Полная схема потока данных

Вот как все компоненты связаны воедино:

```
Запрос пользователя: «Apple wireless keyboard»
           ↓
    [Анализ запроса]
    Определено: Короткое ключевое слово
    Веса: 0.4 векторный / 0.6 текстовый
           ↓
    ┌──────────────────┐
    │ Гибридный поиск │
    └──────────────────┘
           ↓
    ┌──────┴──────┐
    ↓             ↓
[Векторный поиск] [Ключевой поиск]
    ↓             ↓
Семантическое совпадение  Точное совпадение
«keyboard»              «Apple»
    ↓             ↓
Оценка: 0.85     Оценка: 12.5
    ↓             ↓
[Нормализация оценок]
    ↓             ↓
Норм: 0.92       Норм: 0.88
    ↓             ↓
    └──────┬──────┘
           ↓
[Объединение с весами]
Общая = (0.92 × 0.4) + (0.88 × 0.6) = 0.896
           ↓
[Применение бизнес-фильтров]
- Цена <= $200
- В наличии = true
- Категория = accessories
           ↓
[Возврат лучших результатов]
1. Apple Magic Keyboard Touch ID (0.896)
2. Apple Magic Keyboard (0.874)
3. ...
```

---

## Сводка лучших практик

### 1. Выбирайте веса на основе типа запроса

| Тип запроса | Вес векторного поиска | Вес ключевого поиска |
|-------------|----------------------|---------------------|
| SKU/код товара | 0.2 | 0.8 |
| Технические характеристики | 0.3 | 0.7 |
| Естественный язык | 0.8 | 0.2 |
| Короткие ключевые слова | 0.4 | 0.6 |
| Смешанный/по умолчанию | 0.5 | 0.5 |

### 2. Индексируйте несколько полей

```javascript
await vectorStore.setFullTextIndexedFields(NS, 
    ['content', 'title', 'brand', 'sku', 'attributes']
);
```

### 3. Реализуйте стратегию запасного поиска

```javascript
// Try balanced → vector-heavy → pure vector
if (results.length < threshold) {
    // Shift toward vector search
}
```

### 4. Кэшируйте популярные запросы

```javascript
const cache = new Map();
// Cache results for 5-10 minutes
```

### 5. Используйте предварительную фильтрацию, когда это возможно

```javascript
// Filter BEFORE search, not after
const results = await search(query, { 
    metadataFilter: { price: { $lte: maxPrice } }
});
```

### 6. Следите за производительностью и оптимизируйте

```javascript
// Log slow queries
if (searchTime > 100) {
    console.log(`Slow query: ${query} (${searchTime}ms)`);
}
```

---

## Типичные ошибки и решения

### Ошибка 1: Забытая нормализация оценок

**Проблема:**
```javascript
// Keyword scores: [15.2, 12.8, 9.3]
// Vector scores: [0.85, 0.82, 0.79]
combined = vectorScore + keywordScore; // Keywords dominate!
```

**Решение:**
```javascript
const normVector = normalizeMinMax(vectorScores);
const normKeyword = normalizeMinMax(keywordScores);
combined = (normVector * 0.5) + (normKeyword * 0.5);
```

### Ошибка 2: Отсутствие обработки нулевых результатов

**Проблема:**
```javascript
const results = await search(query);
return results[0]; // Error if no results!
```

**Решение:**
```javascript
let results = await search(query, { vectorWeight: 0.5, textWeight: 0.5 });

if (results.length === 0) {
    // Fallback to vector-only
    results = await search(query, { vectorWeight: 1.0, textWeight: 0.0 });
}

if (results.length === 0) {
    return { message: "No products found", suggestions: getPopularProducts() };
}

return results[0];
```

### Ошибка 3: Чрезмерное кэширование

**Проблема:**
```javascript
// Cache never expires!
queryCache.set(query, results);
```

**Решение:**
```javascript
// Set TTL and max cache size
const MAX_CACHE_SIZE = 1000;
const CACHE_TTL = 5 * 60 * 1000;

if (queryCache.size >= MAX_CACHE_SIZE) {
    const firstKey = queryCache.keys().next().value;
    queryCache.delete(firstKey); // Remove oldest
}

queryCache.set(query, {
    results,
    timestamp: Date.now()
});
```

### Ошибка 4: Неиспользование анализа запросов

**Проблема:**
```javascript
// Always use same weights for all queries
const results = await hybridSearch(query, { vectorWeight: 0.5, textWeight: 0.5 });
```

**Решение:**
```javascript
// Adapt weights to query type
const analysis = analyzeQuery(query);
const results = await hybridSearch(query, {
    vectorWeight: analysis.weights.vector,
    textWeight: analysis.weights.text
});
```

---

## Заключение

Теперь вы понимаете:

✅ **Что такое гибридный поиск** и почему он эффективен  
✅ **Как настроить** векторную базу данных с многополевой индексацией  
✅ **Техники нормализации оценок** для справедливого объединения  
✅ **Динамическая настройка весов** на основе паттернов запросов  
✅ **Стратегии запасного поиска** для сценариев с нулевыми результатами  
✅ **Оптимизация производительности** с кэшированием и двухэтапным ретривером

### Следующие шаги

1. **Экспериментируйте**: пробуйте различные комбинации весов
2. **Отслеживайте**: фиксируйте медленные запросы
3. **Оптимизируйте**: внедряйте кэширование для популярных запросов
4. **Улучшайте**: корректируйте веса на основе обратной связи пользователей
5. **Масштабируйте**: добавьте разбиение индекса для больших каталогов

### Дополнительные ресурсы

- **Векторный поиск**: изучите HNSW и FAISS
- **Алгоритм BM25**: разберитесь в математике ключевого поиска
- **Классификация запросов**: используйте ML для автоматического определения типов запросов
- **Реранжирование**: добавьте третий этап для точной сортировки

Удачного поиска! 🚀