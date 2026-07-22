# Гибридный поиск

## Что такое гибридный поиск?

### Простой ответ

Гибридный поиск — это как **две поисковые системы, работающие вместе**:

1. **Одна, которая понимает смысл** (семантический поиск)
2. **Другая, которая находит точные слова** (поисковый запрос)

Сочетая обе, вы получаете лучшие результаты, чем при использовании каждой по отдельности.

### Аналогия из реальной жизни

Представьте, что вы ищете ресторан:

**Друг №1** (семантический поисковик):
- Понимает: «Я хочу итальянскую еду» = пицца, паста, ризотто
- Хорош в: поиске по концепции и смыслу
- Слабость: может пропустить, если вы скажете «Я хочу Alfredo's Pizza» (конкретное название)

**Друг №2** (поисковый запрос):
- Ищет: точные слова вроде «Alfredo's Pizza»
- Хорош в: поиске конкретных названий и терминов
- Слабость: если вы скажете «итальянская еда», не свяжет это с пиццериями

**Оба друга вместе** (гибридный поиск):
- Друг №1 находит итальянские рестораны (концепция)
- Друг №2 находит «Alfredo's Pizza» (точное название)
- Вы получаете лучшее от обоих! 

---

## Настоящая проблема, о которой никто не говорит

Большинство туториалов говорят вам «объедините векторный поиск с поиском по ключевым словам» и показывают веса 0.5/0.5. Вот и всё. Но когда вы реализуете это с реальными данными о товарах, вы сразу столкнётесь с этими проблемами:

### Проблема №1: SKU-коды ломают векторный поиск

**Сценарий:**
```
Query: "MBP-M3MAX-32-1TB"
Vector search returns: Generic laptop descriptions ✗
Keyword search returns: Exact product match ✓

Why? Vector embeddings don't understand alphanumeric codes!
```

**Реальный пример:**
```
Product in your catalog:
  Title: "Apple MacBook Pro 16-inch M3 Max"
  SKU: "MBP-M3MAX-32-1TB"
  Description: "Powerful laptop with advanced M3 Max chip..."

Vector Search Result (similarity: 0.3):
  → "Dell Inspiron laptop with Intel processor..." ✗ WRONG
  
Keyword Search Result (BM25 score: 45.2):
  → "Apple MacBook Pro 16-inch M3 Max" ✓ CORRECT
```

Именно поэтому фиксированное разделение 0.5/0.5 катастрофически не работает для поиска товаров.

### Проблема №2: Диапазоны оценок не совпадают

**Несоответствие:**
```
Vector similarity scores: 0.3 to 0.4 (small range)
BM25 keyword scores: 15 to 50 (large range)

If you just add them together, BM25 dominates everything!
```

**Конкретный пример:**
```
Document A:
  Vector: 0.35
  BM25: 18.5
  Naive sum: 18.85 ← BM25 score dominates

Document B:
  Vector: 0.42  ← Actually a better semantic match!
  BM25: 15.0
  Naive sum: 15.42 ← Ranks LOWER despite better meaning

Result: Your vector search becomes completely useless!
```

**Вы ОБЯЗАНЫ нормализовать оценки перед объединением. Это необязательно.**

### Проблема №3: Разные запросы требуют разных весов

**Почему 0.5/0.5 не работает:**
```
Query 1: "MBP-M3MAX-32-1TB"
What it needs: Keyword-heavy (0.2 vector / 0.8 keyword)
Why: It's a product code, semantic search is useless here

Query 2: "What's the best laptop for video editing?"
What it needs: Vector-heavy (0.8 vector / 0.2 keyword)
Why: Natural language question needs understanding

Query 3: "Sony headphones"
What it needs: Balanced (0.5 / 0.5)
Why: Brand name + product category, both matter

One weight setting cannot possibly work for all three!
```

---

## Как создать гибридный поиск, который действительно работает

### Шаг 1: Нормализация оценок (КРИТИЧЕСКИ ВАЖНО)

У вас есть три варианта. Вот когда использовать каждый:

#### Вариант 1: Минимум-Максимум нормализация

**Формула:**
```
normalized_score = (score - min_score) / (max_score - min_score)
```

**Пример с оценками BM25:**
```
Raw scores: [15, 25, 50]
Min: 15, Max: 50, Range: 35

Normalized:
  15 → (15-15)/35 = 0.0
  25 → (25-15)/35 = 0.286
  50 → (50-15)/35 = 1.0
```

**Плюсы:** Просто, всегда даёт диапазон 0–1  
**Минусы:** Чувствительно к выбросам (одна экстремальная оценка искажает всё)  
**Использовать когда:** Быстрое прототипирование, хорошо структурированные данные

#### Вариант 2: Z-Score нормализация

**Формула:**
```
normalized_score = (score - mean) / standard_deviation
```

**Пример:**
```
Raw scores: [15, 25, 50]
Mean: 30, Std Dev: 14.79

Normalized:
  15 → (15-30)/14.79 = -1.01
  25 → (25-30)/14.79 = -0.34
  50 → (50-30)/14.79 = 1.35
```

**Плюсы:** Сохраняет форму распределения  
**Минусы:** Может давать отрицательные значения, требует дополнительного масштабирования  
**Использовать когда:** Важен статистический анализ

#### Вариант 3: Ранговая нормализация (РЕКОМЕНДУЕТСЯ)

**Формула:**
```
normalized_score = rank_position / total_results
```

**Пример:**
```
Sorted results:
  1st place (score 50) → 1/3 = 0.33
  2nd place (score 25) → 2/3 = 0.67
  3rd place (score 15) → 3/3 = 1.0
```

**Плюсы:** Наиболее устойчива, работает с любой шкалой оценок, используется в RRF  
**Минусы:** Теряет точную величину оценки  
**Использовать когда:** Продакшн-системы (это то, что действительно работает)

**Руководство по выбору:**
```
Score scales vastly different? → Rank-Based (safest choice)
Need exact precision? → Min-Max
Statistical analysis? → Z-Score
Not sure? → Rank-Based (production default)
```

---

### Шаг 2: Динамический выбор весов

**Хватит зашивать веса в код. Автоматически определяйте паттерны запросов.**

У вас есть два подхода: **на основе правил** (быстрый, предсказуемый) или **на основе LLM** (умный, гибкий).

#### Подход A: Определение паттернов на основе правил (Быстрый)

**Лучше всего подходит для:** Продакшн-систем, требований к низкой задержке, предсказуемых паттернов

```javascript
function analyzeQuery(query) {
    const upperCount = (query.match(/[A-Z]/g) || []).length;
    const digitCount = (query.match(/\d/g) || []).length;
    const hyphenCount = (query.match(/-/g) || []).length;
    
    // SKU/Product code pattern
    // Examples: "MBP-M3MAX-32-1TB", "PROD-12345-XL"
    if (hyphenCount >= 2 || (upperCount > 3 && digitCount > 0)) {
        return { vector: 0.2, text: 0.8 }; // keyword-heavy
    }
    
    // Natural language question
    // Examples: "What laptop is best?", "How do I...?"
    if (/^(what|how|which|why|when|where)/i.test(query)) {
        return { vector: 0.8, text: 0.2 }; // vector-heavy
    }
    
    // Brand + product (2 words, one capitalized)
    // Examples: "Sony headphones", "Apple laptop"
    const words = query.split(' ');
    if (words.length === 2 && upperCount > 0) {
        return { vector: 0.5, text: 0.5 }; // balanced
    }
    
    // Default to balanced
    return { vector: 0.5, text: 0.5 };
}
```

**Плюсы:**
- Быстро (< 1 мс)
- Предсказуемо
- Без затрат на API
- Легко отлаживать

**Минусы:**
- Ограничено определёнными паттернами
- Не может обрабатывать сложные запросы
- Требует ручной настройки

#### Подход B: Классификация запросов на основе LLM (Умный)

**Лучше всего подходит для:** Сложных запросов, когда требуется рассуждение, качественный пользовательский опыт

```javascript
async function analyzeQueryWithLLM(query) {
    const prompt = `Analyze this search query and classify it to determine optimal search weights.

Query: "${query}"

Classify into ONE of these categories:

1. PRODUCT_CODE: Contains SKU, product code, or alphanumeric identifier
   Examples: "MBP-M3MAX-32-1TB", "SKU-12345", "PROD-001-XL"
   Weights: vector=0.2, keyword=0.8

2. NATURAL_QUESTION: Conversational question seeking recommendations
   Examples: "What's the best laptop?", "How do I choose headphones?"
   Weights: vector=0.8, keyword=0.2

3. BRAND_CATEGORY: Brand name + product category
   Examples: "Sony headphones", "Apple laptop", "Dell monitor"
   Weights: vector=0.5, keyword=0.5

4. TECHNICAL_TERM: Technical jargon, acronyms, specific terminology
   Examples: "PostgreSQL ACID", "CNN architecture", "GraphQL schema"
   Weights: vector=0.3, keyword=0.7

5. FEATURE_SEARCH: Describes features or attributes
   Examples: "wireless noise canceling", "16GB RAM laptop", "4K monitor"
   Weights: vector=0.6, keyword=0.4

Respond ONLY with JSON:
{
  "category": "CATEGORY_NAME",
  "reasoning": "brief explanation",
  "weights": { "vector": 0.X, "keyword": 0.X }
}`;

    const response = await llm.complete(prompt);
    const result = JSON.parse(response);
    
    return {
        vector: result.weights.vector,
        text: result.weights.keyword,
        reasoning: result.reasoning
    };
}
```

**Примеры ответов LLM:**
```javascript
Query: "MBP-M3MAX-32-1TB"
Response: {
    "category": "PRODUCT_CODE",
    "reasoning": "Contains alphanumeric code with hyphens typical of SKU",
    "weights": { "vector": 0.2, "keyword": 0.8 }
  }

Query: "laptop for machine learning under $2000"
Response: {
    "category": "FEATURE_SEARCH",
    "reasoning": "Describes use case (ML) and constraint (price)",
    "weights": { "vector": 0.6, "keyword": 0.4 }
  }

Query: "how do Sony headphones compare to Bose"
Response: {
    "category": "NATURAL_QUESTION",
    "reasoning": "Comparative question needing conceptual understanding",
    "weights": { "vector": 0.8, "keyword": 0.2 }
  }
```

**Плюсы:**
- Обрабатывает сложные/нюансированные запросы
- Может рассуждать о пограничных случаях
- Адаптируется к новым паттернам без изменений кода
- Может объяснять свои решения

**Минусы:**
- Медленнее (100–500 мс)
- Стоимость за каждый запрос
- Нужен запасной вариант на случай сбоя LLM
- Менее предсказуемо

#### Подход C: Гибридный (лучшее от обоих)

Используйте правила с запасным вариантом LLM для сложных запросов:

```javascript
async function analyzeQueryHybrid(query) {
    // Fast path: Simple patterns
    const ruleBasedResult = analyzeQueryRuleBased(query);
    
    // If confident, return immediately
    if (ruleBasedResult.confidence > 0.8) {
        return ruleBasedResult;
    }
    
    // Slow path: LLM for complex queries
    try {
        const llmResult = await analyzeQueryWithLLM(query);
        return llmResult;
    } catch (error) {
        // Fallback to rule-based
        console.log("LLM failed, using rules");
        return ruleBasedResult;
    }
}

function analyzeQueryRuleBased(query) {
    const upperCount = (query.match(/[A-Z]/g) || []).length;
    const digitCount = (query.match(/\d/g) || []).length;
    const hyphenCount = (query.match(/-/g) || []).length;
    
    // High confidence patterns
    if (hyphenCount >= 2 || (upperCount > 3 && digitCount > 0)) {
        return { 
            vector: 0.2, 
            text: 0.8, 
            confidence: 0.95,
            method: 'rule' 
        };
    }
    
    if (/^(what|how|which|why)/i.test(query)) {
        return { 
            vector: 0.8, 
            text: 0.2, 
            confidence: 0.85,
            method: 'rule'
        };
    }
    
    // Low confidence - let LLM decide
    return { 
        vector: 0.5, 
        text: 0.5, 
        confidence: 0.5,
        method: 'rule'
    };
}
```

#### Какой подход когда использовать?

```
┌───────────────────────────────────────────────────────────┐
│              QUERY CLASSIFICATION DECISION                │
├───────────────────────────────────────────────────────────┤
│                                                           │
│ Simple, predictable queries (SKUs, brands)?               │
│   → Rule-Based (Approach A)                               │
│                                                           │
│ Complex, nuanced queries (comparisons, multi-intent)?     │
│   → LLM-Based (Approach B)                                │
│                                                           │
│ Mixed traffic, want best of both?                         │
│   → Hybrid (Approach C)                                   │
│                                                           │
│ Ultra-low latency required (< 10ms)?                      │
│   → Rule-Based only                                       │
│                                                           │
│ Cost is primary concern?                                  │
│   → Rule-Based (free)                                     │
│                                                           │
│ Quality is primary concern?                               │
│   → LLM-Based (best results)                              │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

#### Сравнение производительности в реальных условиях

Тестовые запросы и результаты:

```
Query: "MBP-M3MAX-32-1TB"
├─ Rule-Based: ✓ 0.2/0.8 (correct) - 0.5ms
└─ LLM-Based:  ✓ 0.2/0.8 (correct) - 180ms

Query: "Sony headphones"  
├─ Rule-Based: ✓ 0.5/0.5 (correct) - 0.3ms
└─ LLM-Based:  ✓ 0.5/0.5 (correct) - 150ms

Query: "laptop for machine learning under $2000 with good battery"
├─ Rule-Based: ~ 0.5/0.5 (generic fallback) - 0.4ms
└─ LLM-Based:  ✓ 0.6/0.4 (optimal feature search) - 220ms

Query: "compare Sony WH-1000XM5 to Bose QC45 for travel"
├─ Rule-Based: ~ 0.5/0.5 (generic fallback) - 0.3ms  
└─ LLM-Based:  ✓ 0.7/0.3 (comparison question) - 200ms
```

**Ключевой вывод:** Правила обрабатывают идеально около 70% запросов. LLM улучшает оставшиеся 30% сложных запросов.

#### Рекомендация для продакшна

Начните с правил, позже добавьте LLM:

```javascript
// Phase 1: Launch with rules (week 1)
const weights = analyzeQueryRuleBased(query);

// Phase 2: Add LLM for complex queries (week 4)
const weights = await analyzeQueryHybrid(query);

// Phase 3: Optimize with caching (week 8)
const weights = await analyzeQueryHybridCached(query);
```

**Почему именно так?**
1. Правила дают вам 80% результата с нулевыми затратами
2. Отслеживайте, какие запросы не работают
3. Добавьте LLM только для этих паттернов
4. Кэшируйте результаты LLM для повторяющихся запросов

#### Примеры паттернов запросов

**Паттерн 1: Коды товаров → Акцент на поисковом запросе (0.2/0.8)**
```
"MBP-M3MAX-32-1TB"
"SKU-99887-V2"
"PROD-001-XL-BLK"

Detection: Multiple hyphens + uppercase + digits
Why keyword-heavy: Vector embeddings fail on codes
```

**Паттерн 2: Вопросы → Акцент на векторном поиске (0.8/0.2)**
```
"What's the best laptop for video editing?"
"How do wireless headphones compare?"
"Which monitor is good for gaming?"

Detection: Starts with question words
Why vector-heavy: Need conceptual understanding
```

**Паттерн 3: Бренд + категория → Баланс (0.5/0.5)**
```
"Sony headphones"
"Apple laptop"
"Dell monitor"

Detection: Two words, one capitalized
Why balanced: Both brand name and concept matter
```

---

### Шаг 3: Мультиполевая индексация

Для поиска товаров индексация по одному полю не работает. Вы ОБЯЗАНЫ индексировать несколько полей.

#### Проблема

```
Product data:
  Title: "MacBook Pro 16-inch M3 Max"
  Brand: "Apple"
  SKU: "MBP-M3MAX-32-1TB"
  Description: "Powerful laptop with advanced chip..."
  Attributes: "M3 Max, 32GB RAM, 1TB SSD"

Query: "Apple wireless keyboard"

With single-field (description only):
  ✗ Misses "Apple" in brand field
  ✗ Misses exact title match
  ✗ Only searches description text
  Result: Poor results
```

#### Решение

```javascript
// Index ALL relevant fields
await vectorStore.setFullTextIndexedFields(namespace, [
    'content',    // Full product description (semantic)
    'title',      // Product name (keyword + semantic)
    'brand',      // Apple, Dell, Sony (exact match)
    'sku',        // Product codes (exact match)
    'attributes'  // Technical specs (mixed)
]);
```

#### Как это работает

```
Query: "Apple wireless keyboard"

Brand field matches:
  "Apple" → Exact brand match (high keyword score)

Title field matches:
  "Magic Keyboard" → Contains "keyboard" (medium score)

Description matches:
  "Wireless connectivity..." → Semantic match (medium score)

Combined result: Product surfaces because multiple fields contribute!
```

#### Структура каталога товаров

```javascript
// Structure your products like this:
new Document(
    "Apple MacBook Pro 16-inch with M3 Max chip, featuring...", 
    {
        id: "PROD-001",
        title: "MacBook Pro 16-inch M3 Max",
        brand: "Apple",
        category: "laptops",
        price: 3499,
        sku: "MBP-M3MAX-32-1TB",
        attributes: "M3 Max, 32GB RAM, 1TB SSD, 16-inch Liquid Retina XDR"
    }
)
```

**Почему каждое поле важно:**
- `brand`: Поиск по точному бренду («Apple»)
- `sku`: Поиск по коду товара («MBP-M3MAX-32-1TB»)
- `title`: Поиск по названию товара («MacBook Pro»)
- `attributes`: Поиск по характеристикам («32GB RAM»)
- `content`: Поиск на естественном языке («лучший ноутбук для программирования»)

---

### Шаг 4: Стратегии запасного варианта

Никогда не показывайте пустые результаты, когда существуют похожие товары.

#### Проблема

```
User searches: "Microsoft laptop"
Your catalog: Only Apple, Dell, HP laptops

Keyword search: 0 results (no "Microsoft" products)
Vector search: Finds similar laptops

Without fallback: User sees "No results found" (terrible UX!)
```

#### Решение

```javascript
async function searchWithFallback(query) {
    // Attempt 1: Balanced search
    let results = await hybridSearch(query, { 
        vector: 0.5, 
        text: 0.5,
        limit: 10
    });
    
    // Attempt 2: If few results, go vector-heavy
    if (results.length < 3) {
        console.log("Few results, expanding search...");
        results = await hybridSearch(query, { 
            vector: 0.8, 
            text: 0.2,
            limit: 10
        });
    }
    
    // Attempt 3: Pure semantic if still nothing
    if (results.length === 0) {
        console.log("No exact matches, finding similar...");
        results = await hybridSearch(query, { 
            vector: 1.0, 
            text: 0.0,
            limit: 10
        });
    }
    
    return results;
}
```

#### Пользовательский опыт

```
Query: "Microsoft laptop"

Step 1 (Balanced 0.5/0.5):
  → 0 results (no Microsoft in catalog)

Step 2 (Vector-heavy 0.8/0.2):
  → 3 results found!
     - Dell Inspiron 15 (similar specs)
     - HP Pavilion (similar price)
     - Apple MacBook (similar category)

Display message:
  "No Microsoft laptops found. Here are similar options:"
```

---

## Два метода объединения: Взвешенный и RRF

### Метод 1: Взвешенное объединение (на основе оценок)

**Как работает:** Использует фактические оценки с регулируемыми весами

**Формула:**
```
final_score = (vector_score × vector_weight) + (keyword_score × keyword_weight)
```

**Пример:**
```
Document A:
  Vector score: 0.8 (after normalization)
  Keyword score: 0.3 (after normalization)
  Weights: 0.7 vector / 0.3 keyword
  
Final: (0.8 × 0.7) + (0.3 × 0.3) = 0.56 + 0.09 = 0.65
```

**Плюсы:**
- Прост и интуитивно понятен
- Легко настраивать веса
- Полный контроль над балансом семантики и ключевых слов

**Минусы:**
- Требует качественной нормализации
- Чувствителен к распределению оценок

**Когда использовать:** Выбор по умолчанию, особенно когда нужен контроль

### Метод 2: Консенсусный рейтинг (RRF)

**Как работает:** Использует позицию/ранг вместо оценок

**Формула:**
```
RRF_score = 1 / (k + rank)
where k is a constant (typically 60)
```

**Пример:**
```
Semantic results:          Keyword results:
1. Doc A (rank 1)         1. Doc C (rank 1)
2. Doc B (rank 2)         2. Doc A (rank 2)
3. Doc C (rank 3)         3. Doc D (rank 3)

RRF Scores (k=60):
Doc A: 1/(60+1) + 1/(60+2) = 0.0164 + 0.0161 = 0.0325 ← Highest!
Doc C: 1/(60+3) + 1/(60+1) = 0.0159 + 0.0164 = 0.0323
Doc B: 1/(60+2) + 0        = 0.0161
Doc D: 0        + 1/(60+3) = 0.0159
```

**Плюсы:**
- Не требует нормализации
- Очень стабилен между запросами
- Менее чувствителен к проблемам с масштабом оценок

**Минусы:**
- Сложно регулировать баланс семантики и ключевых слов
- Менее интуитивен

**Когда использовать:** Диапазоны оценок сильно различаются, нужен принцип «настроил и забыл»

### Какой метод выбрать?

```
Use Weighted when:
  ✓ You want fine control over weights
  ✓ Scores are well-normalized
  ✓ You understand your query patterns

Use RRF when:
  ✓ Score ranges are very different
  ✓ You want maximum stability
  ✓ Normalization is problematic
```

---

## Примеры из реальной жизни

### Пример 1: Поиск SKU в электронной коммерции

**Запрос:** `"MBP-M3MAX-32-1TB"`

**Автоматически определённые веса:** Акцент на поисковом запросе (0.2/0.8)

**Что происходит:**
```
Vector search (20% weight):
  Similarity: 0.4 → Finds MacBook Pro category
  Normalized: 0.4
  
Keyword search (80% weight):
  BM25: 45.2 → Exact SKU match!
  Normalized: 0.95

Combined: (0.4 × 0.2) + (0.95 × 0.8) = 0.08 + 0.76 = 0.84 ✓
```

**Без динамических весов (0.5/0.5):**
```
Combined: (0.4 × 0.5) + (0.95 × 0.5) = 0.20 + 0.475 = 0.675
Generic products might rank higher! ✗
```

### Пример 2: Вопрос на естественном языке

**Запрос:** `"What's the best laptop for video editing?"`

**Автоматически определённые веса:** Акцент на векторном поиске (0.8/0.2)

**Что происходит:**
```
Vector search (80% weight):
  Understands: video editing = high performance, graphics, RAM
  Finds: MacBook Pro M3 Max, Dell XPS 15, etc.
  Normalized: 0.88

Keyword search (20% weight):
  Matches: "laptop", "video", "editing"
  Normalized: 0.6

Combined: (0.88 × 0.8) + (0.6 × 0.2) = 0.704 + 0.12 = 0.824 ✓
```

### Пример 3: Бренд + категория

**Запрос:** `"Sony headphones"`

**Автоматически определённые веса:** Баланс (0.5/0.5)

**Что происходит:**
```
Vector search (50% weight):
  Finds: Audio products, similar Sony items
  Normalized: 0.75

Keyword search (50% weight):
  Exact matches: "Sony" in brand + "headphones" in title
  Normalized: 0.90

Combined: (0.75 × 0.5) + (0.90 × 0.5) = 0.375 + 0.45 = 0.825 ✓
```

### Пример 4: Запасной вариант в действии

**Запрос:** `"Microsoft laptop"`  
**Проблема:** Нет товаров Microsoft в каталоге

**Последовательность запасного варианта:**
```
Attempt 1 (Balanced 0.5/0.5):
  Keyword: 0 matches (no "Microsoft")
  Vector: 0.3 similarity (finds laptops generally)
  Result: 0 strong matches

Attempt 2 (Vector-heavy 0.8/0.2):
  Vector: 0.8 similarity (finds similar laptops)
  Results: Dell Inspiron, HP Pavilion, etc.
  Display: "No Microsoft laptops. Similar options:"
```

---

## Комбинирование с фильтрами

Гибридный поиск + фильтры метаданных = готовое к продакшну решение

### Реализация

```javascript
async function searchProducts(query, filters = {}) {
    // Step 1: Auto-detect query type
    const weights = analyzeQuery(query);
    
    // Step 2: Perform hybrid search
    let results = await hybridSearch(query, weights);
    
    // Step 3: Apply filters
    if (filters.category) {
        results = results.filter(r => 
            r.metadata.category === filters.category
        );
    }
    
    if (filters.priceRange) {
        const [min, max] = filters.priceRange;
        results = results.filter(r => 
            r.metadata.price >= min && r.metadata.price <= max
        );
    }
    
    if (filters.brand) {
        results = results.filter(r => 
            r.metadata.brand === filters.brand
        );
    }
    
    return results;
}
```

### Пример использования

```javascript
// Natural language + price filter
const results = await searchProducts(
    "best laptop for gaming",
    {
        category: "laptops",
        priceRange: [1000, 2000]
    }
);

// Hybrid search: Finds relevant gaming laptops
// Filters: Only those priced $1000-$2000
```

---

## Понимание BM25

BM25 — это алгоритм поиска по ключевым словам. Вот что действительно важно:

### Три вопроса, которые задаёт BM25

**1. Содержит ли документ поисковые термины?**
```
Query: "machine learning"
Doc A: Contains both terms ✓
Doc B: Contains "machine" only
Doc C: Contains neither ✗
```

**2. Насколько редки термины?**
```
"the" appears in 90% of docs → Low importance
"GraphQL" appears in 5% of docs → High importance

Rare terms get higher scores!
```

**3. Насколько длин документ?**
```
Doc A: 100 words, "machine learning" appears 5 times (5% density)
Doc B: 500 words, "machine learning" appears 5 times (1% density)

Doc A scores higher (better density)!
```

### Два параметра BM25

**k1 (насыщение частоты термина):**
```
Low k1 (1.2): "machine machine machine" ≈ "machine"
High k1 (2.0): More repetition = much higher score
Default: 1.5 (balanced)
```

**b (нормализация длины):**
```
b = 0.0: No penalty for long documents
b = 1.0: Strong penalty for long documents
Default: 0.75 (mostly penalize length)
```

**Совет:** Значения по умолчанию работают хорошо в 99% случаев. Настраивайте только если результаты кажутся некорректными.

---

## Типовые паттерны и советы

### Паттерн 1: Начните с простого, затем оптимизируйте

```
Week 1: Use balanced (0.5/0.5) for everything
Week 2: Track which queries work/fail
Week 3: Identify patterns (SKUs, questions, brands)
Week 4: Implement auto-detection
```

### Паттерн 2: Правила автоматического определения

```
SKU codes (hyphens + caps + digits) → 0.2/0.8
Questions (what/how/why) → 0.8/0.2
Brand names (2 words, capitalized) → 0.5/0.5
Technical terms (multiple caps) → 0.3/0.7
Everything else → 0.5/0.5
```

### Паттерн 3: Позвольте пользователям настраивать

```
Search bar with mode selector:
○ Exact match (keyword-heavy)
○ Similar results (semantic-heavy)
○ Balanced (default)
```

### Паттерн 4: Отслеживайте и совершенствуйте

```
Log every search:
  - Query text
  - Detected weights
  - Result count
  - Click-through rate

Analyze weekly:
  - Which patterns work?
  - Which fail?
  - Adjust detection logic
```

---

## Когда гибридный поиск особенно эффективен

### Сценарий 1: Электронная коммерция

**Почему это идеально:**
- Коды товаров (поисковый запрос)
- Описания товаров (семантический поиск)
- Названия брендов (точные совпадения)
- Поиск по характеристикам (концептуальный)

**Пример:**
```
"wireless headphones under $100"
→ Multi-field search across title, attributes, price
→ Semantic understanding of "wireless"
→ Keyword match on "$100"
```

### Сценарий 2: Справочные центры

**Почему это идеально:**
- Пользователи задают вопросы по-разному
- Нужно концептуальное понимание
- Но также нужны точные коды ошибок

**Пример:**
```
"how do I reset my password"
→ Semantic: Understands intent
→ Keyword: Ensures "password" appears
→ Finds all password recovery articles
```

### Сценарий 3: Техническая документация

**Почему это идеально:**
- Конкретные термины важны (PostgreSQL, ACID)
- Но концепции тоже важны (транзакции)

**Пример:**
```
"PostgreSQL ACID transactions"
→ Keyword: Exact "PostgreSQL" and "ACID"
→ Semantic: Related transaction concepts
→ Perfect technical matches
```

### Сценарий 4: Поиск по коду

**Почему это идеально:**
- Имена функций (точные ключевые слова)
- Описания (семантический)
- Комментарии (концептуальные)

**Пример:**
```
"function to parse JSON"
→ Keyword: "parse", "JSON"
→ Semantic: Similar to "decode", "deserialize"
→ Finds all relevant code
```

---

### Золотые правила

1. **Всегда нормализуйте оценки** (ранговая — самая безопасная)
2. **Автоматически определяйте паттерны запросов** (не зашивайте веса)
3. **Индексируйте несколько полей** (особенно для товаров)
4. **Реализуйте запасные варианты** (никогда не показывайте пустые результаты)
5. **Начинайте с баланса, затем оптимизируйте** (измеряйте всё)

---

## Краткая справка

![Дерево решений для гибридного поиска](../../../images/hybrid-search-decision-tree.png)

### Методы нормализации:
```
┌─────────────────┬──────────────┬─────────────┬──────────────────┐
│ Метод           │ Лучше для    │ Плюсы       │ Минусы           │
├─────────────────┼──────────────┼─────────────┼──────────────────┤
│ Ранговая        │ Продакшн-    │ Наиболее    │ Теряет           │
│ (РЕКОМЕНДУЕТСЯ) │ системы      │ устойчива   │ величину         │
│                 │              │ Нет выбросов│                  │
├─────────────────┼──────────────┼─────────────┼──────────────────┤
│ Минимакс        │ Прототипи-   │ Проста      │ Чувствительна    │
│                 │ рование      │ Быстра      │ к выбросам       │
├─────────────────┼──────────────┼─────────────┼──────────────────┤
│ Z-Скор          │ Статисти-    │ Сохраняет   │ Может быть       │
│                 │ ческий       │ распределе- │ отрицательной    │
│                 │ анализ       │ ние         │                  │
└─────────────────┴──────────────┴─────────────┴──────────────────┘
```

### Методы объединения:
```
┌─────────────┬───────────────┬──────────────────┬────────────────┐
│ Метод       │ Уровень       │ Требует          │ Лучше для      │
│             │ контроля      │                  │                │
├─────────────┼───────────────┼──────────────────┼────────────────┤
│ Взвешенный  │ Полный контроль│ Нормализация    │ Точная         │
│             │ над балансом  │                  │ настройка      │
├─────────────┼───────────────┼──────────────────┼────────────────┤
│ RRF         │ Менее         │ Ничего           │ Стабильность,  │
│             │ настраиваемый │                  │ «настроил и     │
│             │               │                  │  забыл»        │
└─────────────┴───────────────┴──────────────────┴────────────────┘
```
