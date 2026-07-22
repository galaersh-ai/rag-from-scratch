# Основы текстового сходства — Концептуальный обзор

Документ объясняет ключевые концепции текстового сходства, их связь с RAG (Retrieval Augmented Generation — генерацией с расширением поиска), а также сравнивает данный подход с использованием LangChain.

## Содержание

1. [Основные концепции](#основные-концепции)
2. [Связь с RAG](#связь-с-rag)
3. [Наш подход vs LangChain](#наш-подход-vs-langchain)
4. [Когда использовать тот или иной подход](#когда-использовать-тот-или-иной-подход)

---

## Основные концепции

### Что такое текстовое сходство?

**Текстовое сходство** — это процесс измерения того, насколько семантически похожи два фрагмента текста, независимо от их точной формулировки.

**Традиционный подход (поиск по ключевым словам):**
```
Query:    "What is the highest mountain?"
Document: "Mount Everest is the tallest peak"
Match:    ❌ No shared keywords (highest ≠ tallest, mountain ≠ peak)
```

**Современный подход (семантическое сходство):**
```
Query:    "What is the highest mountain?"
Document: "Mount Everest is the tallest peak"
Match:    ✅ Same meaning (similarity score: 0.85)
```

### Как это работает?

Процесс включает три ключевых шага:

#### 1. **Эмбеддинги: текст → числа**

Преобразование текста в числовые вектора (массивы чисел), которые отражают семантический смысл.

```javascript
Text:      "The sky is blue"
Embedding: [0.23, -0.45, 0.89, ..., 0.12]  // 384 numbers
```

**Ключевые свойства:**
- Похожие значения → похожие вектора
- Разные значения → разные вектора
- Фиксированный размер (384 измерения для bge-small)

**Аналогия из реальной мира:** Это как присвоить каждому предложению GPS-координаты в «пространстве смысла»

#### 2. **Векторное пространство: где живут значения**

Эмбеддинги существуют в многомерном пространстве, где:
- Расстояние = семантическое различие
- Близкие точки = похожие значения
- Далекие точки = разные значения

```
        Weather cluster
            ★ "sunny day"
            ★ "clear sky"
            
                           Geography cluster
                               ★ "Paris"
                               ★ "France"

    Food cluster
        ★ "pizza"
        ★ "cheese"
```

#### 3. **Сходство: измерение близости**

**Косинусное сходство** измеряет угол между двумя векторами:

```
Same direction (similar meaning):
Vector A: ────────→
Vector B: ────────→
Similarity = 1.0

Perpendicular (unrelated):
Vector A: ────────→
Vector B:     ↓
Similarity = 0.0

Opposite direction (opposite meaning):
Vector A: ────────→
Vector B: ←────────
Similarity = -1.0
```

**Формула (обрабатывается библиотекой):**
```
similarity = cos(θ) = (A · B) / (||A|| × ||B||)
```

Где:
- `A · B` = скалярное произведение (сумма поэлементного умножения)
- `||A||` = норма/длина вектора A
- `||B||` = норма/длина вектора B

---

## Связь с RAG

### Что такое RAG?

**RAG (Retrieval Augmented Generation)** — техника, которая улучшает ответы LLM за счёт извлечения релевантной информации из базы знаний.

```
┌─────────────────────────────────────────────────────────┐
│                      RAG Pipeline                       │
└─────────────────────────────────────────────────────────┘

1. INDEXING (One-time setup)
   Documents → Embeddings → Vector Database
   
2. RETRIEVAL (Per query)
   User Query → Embedding → Find Similar Docs
   
3. GENERATION (Per query)
   Query + Retrieved Docs → LLM → Answer
```

### Где используется текстовое сходство

Текстовое сходство — это **«R» (Retrieval — извлечение)** в RAG:

```
┌──────────────────────────────────────────────────────────┐
│  RAG Component Breakdown                                 │
└──────────────────────────────────────────────────────────┘

R - RETRIEVAL ← [THIS EXAMPLE FOCUSES HERE]
├─ Convert documents to embeddings
├─ Store embeddings in searchable format
├─ Compare query embedding to document embeddings
└─ Return most similar documents

A - AUGMENTED
├─ Combine user query with retrieved documents
├─ Create enriched prompt for LLM
└─ Provide context that wasn't in training data

G - GENERATION
├─ Send augmented prompt to LLM
├─ Generate answer based on retrieved context
└─ Return final response to user
```

### Пример потока RAG

**Без RAG (ограниченный данными обучения):**
```
User: "What's our Q4 revenue?"
LLM:  "I don't have access to your company's financial data."
```

**С RAG (расширенный вашими данными):**
```
User: "What's our Q4 revenue?"
  ↓
[RETRIEVAL - Text Similarity]
  → Embed query: [0.23, -0.45, ...]
  → Search vector DB for similar docs
  → Find: "Q4 Revenue Report: $2.3M"
  ↓
[AUGMENTATION]
  → Context: "Based on this document: 'Q4 Revenue Report: $2.3M...'"
  → Prompt: "Answer the user's question using the context provided"
  ↓
[GENERATION]
  → LLM: "According to the Q4 Revenue Report, our revenue was $2.3M."
```

### Почему этот подход работает

**Проблемы традиционного поиска:**
- ❌ Поиск по ключевым словам пропускает семантически похожий контент
- ❌ Требует точного совпадения фраз
- ❌ Плохо работает с синонимами и перефразировками
- ❌ Не понимает контекст и намерение

**Решения на основе эмбеддингов:**
- ✅ Находит семантически похожий контент независимо от формулировки
- ✅ Понимает синонимы («car» ≈ «automobile»)
- ✅ Учитывает контекст и намерение
- ✅ Работает на разных языках (с мультиязычными моделями)
- ✅ Масштабируется на миллионы документов

---

## Наш подход vs LangChain

### Наш подход: node-llama-cpp (прямой)

**Философия:** Учиться, создавая всё с нуля с минимальными абстракциями.

```javascript
// 1. Load model directly
const llama = await getLlama();
const model = await llama.loadModel({
    modelPath: "bge-small-en-v1.5.Q8_0.gguf"
});
const context = await model.createEmbeddingContext();

// 2. Embed documents manually
const embeddings = new Map();
for (const doc of documents) {
    const embedding = await context.getEmbeddingFor(doc.pageContent);
    embeddings.set(doc, embedding);
}

// 3. Search manually
const queryEmbedding = await context.getEmbeddingFor(query);
const results = [];
for (const [doc, embedding] of embeddings) {
    const similarity = queryEmbedding.calculateCosineSimilarity(embedding);
    results.push({ doc, similarity });
}
results.sort((a, b) => b.similarity - a.similarity);
```

**Преимущества:**
- ✅ **Прозрачность**: Видно, что происходит на каждом шаге
- ✅ **Контроль**: Полный контроль над каждой операцией
- ✅ **Обучение**: Глубокое понимание основ
- ✅ **Без «магии»**: Нет скрытых абстракций
- ✅ **Лёгкость**: Минимальное количество зависимостей
- ✅ **Локально**: Полностью работает оффлайн с локальными моделями

**Недостатки:**
- ❌ Больше кода для написания
- ❌ Необходимо самостоятельно реализовывать утилиты
- ❌ Приходится вручную обрабатывать граничные случаи
- ❌ Ограничено моделями формата GGUF через llama.cpp

**Лучше всего подходит для:**
- Изучения основ RAG
- Образовательных целей
- Полного контроля над пайплайном
- Локальных приложений
- Понимания того, что делают библиотеки «под капотом»

---

### Подход с LangChain (абстрагированный)

**Философия:** Высокоуровневые абстракции для быстрой разработки.

```javascript
import { HuggingFaceTransformersEmbeddings } from "@langchain/community/embeddings/***";
import { MemoryVectorStore } from "langchain/vectorstores/memory";
import { Document } from "@langchain/core/documents";

// 1. Initialize embeddings (abstracts model loading)
const embeddings = new HuggingFaceTransformersEmbeddings({
    modelName: "Xenova/bge-small-en-v1.5",
});

// 2. Create documents
const docs = [
    new Document({ pageContent: "The sky is clear and blue today" }),
    new Document({ pageContent: "Mount Everest is the tallest mountain" }),
    // ...
];

// 3. Create vector store (abstracts embedding + storage)
const vectorStore = await MemoryVectorStore.fromDocuments(
    docs,
    embeddings
);

// 4. Search (abstracts similarity calculation + sorting)
const results = await vectorStore.similaritySearch(
    "What is the tallest mountain on Earth?",
    3  // top k
);

// Results are already sorted and formatted
console.log(results);
```

**С оценками сходства:**
```javascript
const resultsWithScores = await vectorStore.similaritySearchWithScore(
    "What is the tallest mountain on Earth?",
    3
);

// Returns: [{ document, score }, ...]
resultsWithScores.forEach(([doc, score]) => {
    console.log(`Score: ${score}, Content: ${doc.pageContent}`);
});
```

**Преимущества:**
- ✅ **Быстрая разработка**: Меньше кода, больше функциональности
- ✅ **Полный комплект**: Векторные хранилища, ретриверы, цепочки
- ✅ **Экосистема**: Интеграция со множеством инструментов и сервисов
- ✅ **Готовность к продакшену**: Обработка ошибок, логика повторных попыток и т.д.
- ✅ **Гибкость**: Лёгкая замена моделей эмбеддингов или векторных хранилищ
- ✅ **Сообщество**: Большое сообщество, множество примеров

**Недостатки:**
- ❌ **Накладные расходы абстракции**: Сложнее понять, что происходит
- ❌ **«Чёрный ящик»**: «Магия» за кулисами
- ❌ **Зависимости**: Тяжёлое дерево зависимостей
- ❌ **Меньше контроля**: Сложнее настраивать поведение нижнего уровня
- ❌ **Кривая обучения**: Необходимо изучать абстракции LangChain

**Лучше всего подходит для:**
- Продакшен-приложений
- Быстрого прототипирования
- Когда нужны проверенные решения
- Интеграции с множеством сервисов
- Команд, знакомых с паттернами LangChain

---

### Полный пример на LangChain

Вот как будет выглядеть пример из всего руководства на LangChain:

```javascript
import { HuggingFaceTransformersEmbeddings } from "@langchain/community/embeddings/***";
import { MemoryVectorStore } from "langchain/vectorstores/memory";
import { Document } from "@langchain/core/documents";

// Sample data
const sampleTexts = [
    "The sky is clear and blue today",
    "I love eating pizza with extra cheese",
    "Dogs love to play fetch with their owners",
    "The capital of France is Paris",
    "Drinking water is important for staying hydrated",
    "Mount Everest is the tallest mountain in the world",
    "A warm cup of tea is perfect for a cold winter day",
    "Painting is a form of creative expression",
    "Not all the things that shine are made of gold",
    "Cleaning the house is a good way to keep it tidy"
];

async function main() {
    // 1. Initialize embeddings model
    const embeddings = new HuggingFaceTransformersEmbeddings({
        modelName: "Xenova/bge-small-en-v1.5",
    });

    // 2. Create documents
    const docs = sampleTexts.map((text, i) => 
        new Document({ 
            pageContent: text,
            metadata: { id: i, source: 'sample_data' }
        })
    );

    // 3. Create vector store (embeds documents automatically)
    const vectorStore = await MemoryVectorStore.fromDocuments(
        docs,
        embeddings
    );

    // 4. Example 1: Basic similarity search
    console.log("Example 1: Basic Search");
    const query1 = "What is the tallest mountain on Earth?";
    const results1 = await vectorStore.similaritySearchWithScore(query1, 3);
    
    results1.forEach(([doc, score]) => {
        console.log(`Score: ${score.toFixed(4)}`);
        console.log(`Content: ${doc.pageContent}\n`);
    });

    // 5. Example 2: Multiple queries (reuses embeddings)
    console.log("Example 2: Multiple Queries");
    const queries = [
        "Tell me about hydration",
        "What's a good winter drink?",
        "Information about European capitals"
    ];

    for (const query of queries) {
        const results = await vectorStore.similaritySearch(query, 1);
        console.log(`Query: ${query}`);
        console.log(`Best match: ${results[0].pageContent}\n`);
    }

    // 6. Example 3: Using a retriever (more advanced)
    const retriever = vectorStore.asRetriever({
        k: 3,  // top 3 results
        searchType: "similarity",
    });

    const retrieved = await retriever.getRelevantDocuments(
        "Tell me about nature and weather"
    );
    
    console.log("Example 3: Using Retriever");
    retrieved.forEach(doc => {
        console.log(doc.pageContent);
    });

    // 7. Example 4: Filtering by metadata
    const filteredResults = await vectorStore.similaritySearch(
        "Tell me about geography",
        5,
        (doc) => doc.metadata.id < 5  // Filter function
    );
    
    console.log("Example 4: Filtered Results");
    console.log(filteredResults);
}

main();
```

### Ключевые различия в реализации

| Аспект | Наш подход | LangChain |
|--------|------------|-----------|
| **Загрузка модели** | Вручную через `getLlama()` | Абстрагировано через `HuggingFaceTransformersEmbeddings` |
| **Эмбеддинг** | Явный цикл с `getEmbeddingFor()` | Автоматически в `fromDocuments()` |
| **Хранение** | Ручной `Map` | `MemoryVectorStore` или внешняя БД |
| **Сходство** | Ручной `calculateCosineSimilarity()` | Автоматически в `similaritySearch()` |
| **Сортировка** | Ручная сортировка | Автоматическая |
| **Строк кода** | ~150 строк (с примерами) | ~50 строк (эквивалентная функциональность) |
| **Уровень абстракции** | Низкий (видно всё) | Высокий (скрытая сложность) |
| **Гибкость** | Полный контроль | Ограничено паттернами LangChain |
| **Образовательная ценность** | Высокая (понимание внутренностей) | Средняя (изучение паттернов) |

---

## Концептуальная модель

### Пайплайн эмбеддингов

```
┌─────────────────────────────────────────────────────────────┐
│                    EMBEDDING PIPELINE                       │
└─────────────────────────────────────────────────────────────┘

INPUT                   PROCESSING                    OUTPUT
─────                   ──────────                    ──────

"The sky               1. Tokenization              [0.234,
 is blue"      →          ["the", "sky",    →       -0.156,
                           "is", "blue"]              0.891,
                                                      ...,
                       2. Neural Network             -0.023]
                          (Transformer)               
                          - 384 dimensions            Vector
Text                      - Learned patterns          (384 floats)
(Human                    - Semantic features         (Machine
Readable)                                             Readable)

                       3. Normalization
                          (Unit length)
```

### Пайплайн поиска по сходству

```
┌─────────────────────────────────────────────────────────────┐
│                  SIMILARITY SEARCH PIPELINE                 │
└─────────────────────────────────────────────────────────────┘

INDEXING (One-time)
─────────────────────
Documents  →  Embeddings  →  Vector Store
   [D1]          [E1]            DB
   [D2]    →     [E2]      →    {E1, E2, E3, ...}
   [D3]          [E3]

QUERYING (Real-time)
────────────────────
Query      →  Embedding   →  Similarity     →  Top-K      →  Results
"mountain"     [Eq]          Calculation        Results
                             cos(Eq, E1)        [D3]          [D3]
                             cos(Eq, E2)    →   [D1]    →     [D1]
                             cos(Eq, E3)        [D5]          [D5]
                                ↓
                             Ranking
```

### Почему это важно для RAG

```
┌─────────────────────────────────────────────────────────────┐
│                   COMPLETE RAG SYSTEM                       │
└─────────────────────────────────────────────────────────────┘

[TEXT SIMILARITY]              [LLM GENERATION]
      ↓                              ↓
┌─────────────────┐          ┌─────────────────┐
│   Documents     │          │   User Query    │
│  - 10,000 docs  │          │  "What is...?"  │
└────────┬────────┘          └────────┬────────┘
         │                            │
         ↓                            ↓
┌─────────────────┐          ┌─────────────────┐
│   Embeddings    │          │ Query Embedding │
│  - Pre-computed │          │  - Real-time    │
└────────┬────────┘          └────────┬────────┘
         │                            │
         └────────────┬───────────────┘
                      ↓
             ┌─────────────────┐
             │ Similarity      │
             │ Search          │
             │ (THIS TUTORIAL) │
             └────────┬────────┘
                      ↓
             ┌─────────────────┐
             │ Top 3 Relevant  │
             │ Documents       │
             └────────┬────────┘
                      ↓
             ┌─────────────────┐
             │ Augmented       │
             │ Prompt          │
             │ "Based on..."   │
             └────────┬────────┘
                      ↓
             ┌─────────────────┐
             │ LLM             │
             │ (GPT/Claude)    │
             └────────┬────────┘
                      ↓
             ┌─────────────────┐
             │ Final Answer    │
             └─────────────────┘
```

---

## Резюме

1. **Текстовое сходство** — измерение семантической близости между текстами
2. **Эмбеддинги** — преобразование текста в числовые вектора
3. **Косинусное сходство** — измерение угла между векторами
4. **Извлечение (Retrieval)** — поиск релевантных документов с помощью сходства
