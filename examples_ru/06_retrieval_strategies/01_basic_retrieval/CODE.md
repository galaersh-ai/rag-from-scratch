# Базовая стратегия извлечения — разбор кода

Подробное объяснение реализации базовой стратегии извлечения в RAG, совмещающей векторный поиск и генерацию LLM.

## Обзор

В данном примере демонстрируется:
- Полная реализация пайплайна RAG
- Top-k извлечение из векторного хранилища
- Сборка и форматирование контекста
- Генерация ответов на основе LLM
- Обработка крайних случаев (отсутствие результатов, низкое качество)
- Аспекты производительности
- Сравнение с/без RAG

**Используемые модели:**
- **Эмбеддинг**: `bge-small-en-v1.5.Q8_0.gguf` (384 измерения)
- **LLM**: `hf_Qwen_Qwen3-1.7B.Q8_0.gguf` (1,7 млрд параметров)

---

## Установка и конфигурация

### Импорт

```javascript
import { fileURLToPath } from "url";
import path from "path";
import { VectorDB } from "embedded-vector-db";
import { getLlama, LlamaChatSession } from "node-llama-cpp";
import { Document } from "../../../src/index.js";
import { OutputHelper } from "../../../helpers/output-helper.js";
import chalk from "chalk";
```

**Ключевые импорты:**
- `VectorDB`: Векторная база данных в памяти на основе HNSW
- `getLlama, LlamaChatSession`: Инференс LLM
- `Document`: Абстракция документа с содержимым и метаданными
- `OutputHelper`: Утилиты форматирования и индикатора загрузки
- `chalk`: Цвета терминала для наглядного вывода

### Конфигурация

```javascript
const EMBEDDING_MODEL_PATH = path.join(__dirname, "..", "..", "..", "models", "bge-small-en-v1.5.Q8_0.gguf");
const LLM_MODEL_PATH = path.join(__dirname, "..", "..", "..", "models", "hf_Qwen_Qwen3-1.7B.Q8_0.gguf");

const DIM = 384;
const MAX_ELEMENTS = 10000;
const NS = "basic_retrieval";
```

**Параметры конфигурации:**
- `DIM = 384`: Размерность эмбеддингов BGE-small
- `MAX_ELEMENTS`: Ёмкость векторной БД
- `NS`: Пространство имён для изоляции

---

## Основные функции

### 1. Инициализация модели эмбеддингов

```javascript
async function initializeEmbeddingModel() {
    const llama = await getLlama({ logLevel: "error" });
    const model = await llama.loadModel({ modelPath: EMBEDDING_MODEL_PATH });
    return await model.createEmbeddingContext();
}
```

**Что делает:**
- Загружает модель эмбеддингов BGE-small-en-v1.5
- Создаёт контекст эмбеддингов для генерации векторов
- Возвращает контекст, используемый как для документов, так и для запросов

**Почему отдельно от LLM:**
- Модели эмбеддингов отличаются от генеративных LLM
- Можно использовать разные модели для эмбеддингов и генерации
- Контекст эмбеддингов можно переиспользовать

**Производительность:**
- Время загрузки: ~2–5 секунд
- Генерация эмбеддинга: ~45 мс на текст

### 2. Инициализация LLM

```javascript
async function initializeLLM() {
    const llama = await getLlama({ logLevel: "error" });
    const model = await llama.loadModel({ modelPath: LLM_MODEL_PATH });
    const context = await model.createContext();
    return new LlamaChatSession({ contextSequence: context.getSequence() });
}
```

**Что делает:**
- Загружает модель Qwen3 1.7B
- Создаёт контекст для инференса
- Возвращает сессию чата для диалогового взаимодействия

**Преимущества LlamaChatSession:**
- Поддерживает историю диалога
- Автоматически форматирует промпт
- Управляет окном контекста

**Производительность:**
- Время загрузки: ~5–10 секунд (модель 1,7 млрд)
- Генерация: ~300–500 мс для типичного ответа

### 3. Создание базы знаний

```javascript
function createKnowledgeBase() {
    return [
        new Document("Python is a high-level programming language...", {
            id: "doc_1",
            category: "programming",
            topic: "python",
        }),
        // ... больше документов
    ];
}
```

**Что делает:**
- Создаёт демонстрационные документы
- Каждый документ содержит содержимое и метаданные
- Охватывает различные темы для разнообразных запросов

**Структура документа:**
```javascript
{
    pageContent: "...",  // Основное текстовое содержимое
    metadata: {
        id: "doc_1",           // Уникальный идентификатор
        category: "...",       // Высокоуровневая категория
        topic: "...",          // Конкретная тема
        // ... пользовательские поля
    }
}
```

**Рекомендации:**
- Используйте осмысленные ID для отслеживания
- Добавляйте метаданные для фильтрации
- Сохраняйте фокус документов (одна тема на документ)
- Разделяйте большие документы на чанки

### 4. Добавление документов в векторное хранилище

```javascript
async function addDocumentsToStore(vectorStore, embeddingContext, documents) {
    for (const doc of documents) {
        const embedding = await embeddingContext.getEmbeddingFor(doc.pageContent);
        const metadata = {
            content: doc.pageContent,
            ...doc.metadata,
        };
        await vectorStore.insert(
            NS,
            doc.metadata.id,
            Array.from(embedding.vector),
            metadata
        );
    }
}
```

**Что делает:**
- Генерирует эмбеддинг содержимого каждого документа
- Сохраняет вектор и метаданные в векторную БД
- Использует пространство имён для организации

**Пошагово:**
1. Генерация эмбеддинга содержимого документа
2. Объединение содержимого с метаданными
3. Вставка в векторное хранилище с ID

**Почему хранить содержимое в метаданных:**
- Векторная БД возвращает метаданные вместе с результатами поиска
- Избегает дополнительного поиска содержимого
- Удобно для сборки контекста

**Производительность:**
- Эмбеддинг: ~45 мс на документ
- Вставка: ~1 мс на документ
- Итого: ~46 мс на документ

### 5. Извлечение документов

```javascript
async function retrieveDocuments(vectorStore, embeddingContext, query, k = 3) {
    const queryEmbedding = await embeddingContext.getEmbeddingFor(query);
    const results = await vectorStore.search(NS, Array.from(queryEmbedding.vector), k);
    return results;
}
```

**Что делает:**
- Генерирует эмбеддинг запроса
- Выполняет поиск top-k наиболее похожих документов в векторном хранилище
- Возвращает ранжированные результаты с оценками сходства

**Формат возврата:**
```javascript
[
    {
        id: "doc_1",
        similarity: 0.8734,
        metadata: {
            content: "...",
            category: "...",
            topic: "...",
        }
    },
    // ... k результатов
]
```

**Ключевые параметры:**
- `k`: Количество извлекаемых документов
- Значение по умолчанию `k=3`: хороший баланс для большинства сценариев

**Оценки сходства:**
- Диапазон: 0,0 до 1,0 (косинусное сходство)
- > 0,7: Высокая релевантность
- 0,5–0,7: Релевантность
- < 0,5: Низкая релевантность

### 6. Сборка контекста

```javascript
function buildContext(retrievedDocs) {
    if (retrievedDocs.length === 0) {
        return "";
    }
    
    const contextParts = retrievedDocs.map((doc, idx) => 
        `[${idx + 1}] ${doc.metadata.content}`
    );
    
    return contextParts.join("\n\n");
}
```

**Что делает:**
- Форматирует извлечённые документы в единую строку контекста
- Добавляет нумерацию для ссылок
- Разделяет документы пустыми строками

**Пример вывода:**
```
[1] Python is a high-level programming language known for its simplicity...

[2] JavaScript is the primary language for web browsers...

[3] Machine learning is a subset of artificial intelligence...
```

**Почему такой формат:**
- Чёткое разделение между документами
- LLM легко обрабатывает такой формат
- Пронумерованные ссылки для цитирования
- Пустые строки повышают читаемость

**Альтернативы:**
```javascript
// С оценками сходства
`[${idx + 1}] (Score: ${doc.similarity.toFixed(2)}) ${doc.metadata.content}`

// С метаданными
`[${idx + 1}] Category: ${doc.metadata.category}
Content: ${doc.metadata.content}`
```

### 7. Генерация ответа

```javascript
async function generateAnswer(chatSession, query, context) {
    if (!context || context.trim().length === 0) {
        const prompt = `Question: ${query}\n\nYou don't have any relevant information to answer this question. Please say so politely.`;
        return await chatSession.prompt(prompt, { maxTokens: 150 });
    }
    
    const prompt = `You are a helpful assistant. Use the following context to answer the question. If the context doesn't contain relevant information, say so.

Context:
${context}

Question: ${query}

Answer:`;
    
    return await chatSession.prompt(prompt, { maxTokens: 200 });
}
```

**Что делает:**
- Обрабатывает два случая: с контекстом и без
- Форматирует промпт с контекстом и вопросом
- Генерирует ответ с помощью LLM

**Случай 1: Без контекста**
```javascript
if (!context || context.trim().length === 0) {
    // Просьба LLM указать на отсутствие информации
}
```

**Случай 2: С контекстом**
```javascript
const prompt = `Системная инструкция + Контекст + Вопрос`;
```

**Инженерия промптов:**
- Чёткая системная инструкция
- Контекст размещается перед вопросом
- Явное указание использовать контекст
- Защита: «Если контекст не помогает, об этом нужно сказать»

**Параметр maxTokens:**
- 150–200 токенов: Краткие ответы
- 500+ токенов: Подробные ответы
- Баланс: полнота vs задержка

### 8. Полный пайплайн RAG

```javascript
async function ragPipeline(vectorStore, embeddingContext, chatSession, query, k = 3) {
    // Шаг 1: Извлечение релевантных документов
    const retrievedDocs = await retrieveDocuments(vectorStore, bindingContext, query, k);
    
    // Шаг 2: Сборка контекста из извлечённых документов
    const context = buildContext(retrievedDocs);
    
    // Шаг 3: Генерация ответа с помощью LLM на основе контекста
    const answer = await generateAnswer(chatSession, query, context);
    
    return { retrievedDocs, context, answer };
}
```

**Что делает:**
- Оркестрирует трёхэтапный процесс RAG
- Возвращает все промежуточные результаты для анализа
- Обеспечивает полную прозрачность

**Три шага:**
1. **Извлечение**: Векторный поиск релевантных документов
2. **Сборка**: Форматирование документов в контекст
3. **Генерация**: LLM создаёт ответ на основе контекста

**Возвращаемый объект:**
```javascript
{
    retrievedDocs: [...],  // Массив извлечённых документов
    context: "...",        // Форматированная строка контекста
    answer: "..."          // Ответ, сгенерированный LLM
}
```

**Почему возвращается всё:**
- Отладка: проверка того, что было извлечено
- Логирование: отслеживание качества извлечения
- Интерфейс: отображение источников пользователям
- Тестирование: проверка каждого шага

---

## Пример 1: Базовый пайплайн RAG

```javascript
async function example1() {
    const vectorStore = new VectorDB({ dim: DIM, maxElements: MAX_ELEMENTS });
    const embeddingContext = await initializeEmbeddingModel();
    const chatSession = await initializeLLM();
    const documents = createKnowledgeBase();
    await addDocumentsToStore(vectorStore, embeddingContext, documents);

    const query = "What is Python?";
    const { retrievedDocs, context, answer } = await ragPipeline(
        vectorStore, embeddingContext, chatSession, query, 3
    );

    // Отображение результатов...
}
```

**Что демонстрирует:**
- Полный сквозной пайплайн RAG
- Визуализация всех трёх шагов
- Чёткое разделение ответственности

**Ожидаемый вывод:**
```
Query: What is Python?

Step 1 - Retrieved Documents: (k=3)
1. [0.8734] Python is a high-level programming language known for its...
2. [0.6521] JavaScript is the primary language for web browsers...
3. [0.5892] Machine learning is a subset of artificial intelligence...

Step 2 - Context Assembled:
[1] Python is a high-level programming language...
[2] JavaScript is the primary language...
[3] Machine learning is a subset...

Step 3 - Generated Answer:
Python is a high-level programming language known for its simplicity and 
readability, widely used in data science, web development, and automation.
```

**Ключевой вывод:**
- Первый документ имеет высокое сходство (0,87)
- Ответ основан на извлечённом содержимом
- RAG обеспечивает точную и конкретную информацию

---

## Пример 2: Изменение параметра k

```javascript
async function example2() {
    // ... код инициализации
    
    const query = "Tell me about artificial intelligence";
    const kValues = [1, 3, 5];

    for (const k of kValues) {
        const { retrievedDocs, answer } = await ragPipeline(
            vectorStore, embeddingContext, chatSession, query, k
        );
        // Сравнение результатов...
    }
}
```

**Что демонстрирует:**
- Влияние k на извлечение и ответы
- Компромисс между точностью и полнотой
- Как выбрать k для вашего сценария

**Сравнение:**

**k=1:**
```
Извлечено: 1 документ
Плюсы: Быстро, сфокусировано
Минусы: Может упустить важный контекст
Ответ: Краткий, основанный на одном источнике
```

**k=3:**
```
Извлечено: 3 документа
Плюсы: Сбалансировано, хорошее покрытие
Минусы: Отсутствуют (рекомендуемое значение по умолчанию)
Ответ: Комплексный, многоаспектный
```

**k=5:**
```
Извлечено: 5 документов
Плюсы: Максимальный контекст
Минусы: Может включать менее релевантные документы, больше токенов
Ответ: Подробный, но потенциально шумный
```

**Рекомендация:**
- Начните с k=3
- Увеличьте, если ответам не хватает детализации
- Отслеживайте оценки сходства последних документов
- Если сходство последнего документа < 0,4, уменьшите k

---

## Пример 3: Влияние качества извлечения

```javascript
async function example3() {
    const queries = [
        { query: "What is machine learning?", expected: "high relevance" },
        { query: "How does Docker work?", expected: "medium relevance" },
        { query: "What's the weather today?", expected: "no relevance" },
    ];

    for (const { query, expected } of queries) {
        { retrievedDocs, answer } = await ragPipeline(
            vectorStore, embeddingContext, chatSession, query, 3
        );
        // Анализ оценок сходства...
    }
}
```

**Что демонстрирует:**
- Как оценки сходства указывают на качество извлечения
- Поведение LLM при высоком и низком качестве извлечения
- Важность обработки запросов вне предметной области

**Сценарий 1: Высокая релевантность**
```
Запрос: "What is machine learning?"
Сходство первого документа: 0,87
Ответ: Точный, подробный, основанный на контексте
```

**Сценарий 2: Средняя релевантность**
```
Запрос: "How does Docker work?"
Сходство первого документа: 0,62
Ответ: Приемлемый, но может не хватать конкретики
```

**Сценарий 3: Нет релевантности**
```
Запрос: "What's the weather today?"
Сходство первого документа: 0,18
Ответ: "У меня нет информации по данному вопросу..."
```

**Ключевой вывод:**
- Сходство > 0,7: Высокая достоверность
- Сходство 0,5–0,7: Средняя достоверность
- Сходство < 0,5: Рассмотрите фильтрацию или возврат «нет информации»

---

## Пример 4: С извлечением и без

```javascript
async function example4() {
    const query = "What is React used for?";
    
    // Без извлечения (только LLM)
    const answerNoRAG = await chatSession.prompt(query, { maxTokens: 150 });
    
    // С извлечением (RAG)
    const { retrievedDocs, answer } = await ragPipeline(
        vectorStore, embeddingContext, chatSession, query, 3
    );
}
```

**Что демонстрирует:**
- Прямое сравнение только LLM vs RAG
- Ценность опоры на конкретную базу знаний
- Как RAG снижает галлюцинации

**Без RAG:**
```
Q: "What is React used for?"
A: "React is a popular JavaScript library for building user interfaces. 
    It was created by Facebook and is widely used in web development..."
```
Обобщённый ответ, применимый ко многим библиотекам

**С RAG:**
```
Q: "What is React used for?"
Retrieved: [React documentation from knowledge base]
A: "According to the context, React is a JavaScript library for building 
    user interfaces, developed by Facebook. It uses a component-based 
    architecture and virtual DOM."
```
Конкретный ответ, основанный на вашей базе знаний

**Ключевые различия:**
- RAG: Опирается на конкретную базу знаний
- RAG: Соответствует вашей документации
- RAG: Может включать специфические для компании детали
- RAG: Снижает риск галлюцинаций

---

## Пример 5: Фильтрация некачественных извлечений

```javascript
async function example5() {
    const query = "What's the capital of Australia?";
    const retrievedDocs = await retrieveDocuments(vectorStore, embeddingContext, query, 5);

    // Применение порога сходства
    const threshold = 0.3;
    const filtered = retrievedDocs.filter(doc => doc.similarity > threshold);
    
    const context = buildContext(filtered);
    const answer = await generateAnswer(chatSession, query, context);
}
```

**Что демонстрирует:**
- Важность фильтрации некачественных результатов
- Как установить и применить пороги сходства
- Корректная обработка запросов вне предметной области

**Без фильтрации:**
```
Извлечено 5 документов:
1. [0.28] Python is a programming language...
2. [0.25] JavaScript runs in browsers...
3. [0.23] Docker containers provide...
4. [0.21] SQL databases use...
5. [0.18] Git is a version control...

Проблема: Все документы — шум, LLM пытается ответить всё равно → Неверный ответ
```

**С фильтрацией (порог = 0,3):**
```
Извлечено 5 документов:
После фильтрации: 0 документов (все ниже порога)

Результат: "В моей базе знаний нет информации по данному вопросу."
```

**Выбор порога:**
- **0,5+**: Очень строгий, высокая точность
- **0,4**: Сбалансированный (рекомендуется)
- **0,3**: Мягкий, более высокая полнота
- **< 0,3**: Слишком мягкий, много шума

**Рекомендация:**
```javascript
const SIMILARITY_THRESHOLD = 0.4;

const filtered = docs.filter(d => d.similarity > SIMILARITY_THRESHOLD);

if (filtered.length === 0) {
    return "No relevant information found in knowledge base.";
}
```

---

## Пример 6: Управление окном контекста

```javascript
async function example6() {
    const query = "programming languages";
    const retrievedDocs = await retrieveDocuments(vectorStore, embeddingContext, query, 5);

    // Отображение размеров контекста
    for (let k = 1; k <= 5; k++) {
        const context = buildContext(retrievedDocs.slice(0, k));
        const tokens = Math.ceil(context.length / 4); // Приблизительная оценка
        console.log(`k=${k}: ${context.length} chars (~${tokens} tokens)`);
    }
}
```

**Что демонстрирует:**
- Как растёт размер контекста с увеличением k
- Методы оценки количества токенов
- Баланс между размером контекста и ограничениями модели

**Пример вывода:**
```
k=1: 142 chars (~36 tokens)
k=2: 296 chars (~74 tokens)
k=3: 451 chars (~113 tokens)
k=4: 608 chars (~152 tokens)
k=5: 762 chars (~191 tokens)
```

**Соображения по окну контекста:**

**Llama 3.2 1B (контекст 2048 токенов):**
```
Общий бюджет: 2048 токенов
- Системный промпт: ~50 токенов
- Вопрос: ~20 токенов
- Контекст (k=3): ~150 токенов
- Место для ответа: ~1828 токенов ✅
```

**При k=10:**
```
Общий бюджет: 2048 токенов
- Системный промпт: ~50 токенов
- Вопрос: ~20 токенов
- Контекст (k=10): ~500 токенов
- Место для ответа: ~1478 токенов ✅ (всё ещё допустимо)
```

**Рекомендации:**
- Держите контекст < 25% от общего окна для небольших моделей
- Отслеживайте фактическое использование токенов
- Уменьшайте k при достижении лимитов контекста
- Рассмотрите усечение длинных документов

**Оценка количества токенов:**
```javascript
// Приблизительная оценка: 1 токен ≈ 4 символа
const estimatedTokens = text.length / 4;

// Лучше: использовать токенизатор
const tokens = tokenizer.encode(text).length;
```

---

## Пример 7: Пакетная обработка

```javascript
async function example7() {
    const queries = [
        "What is Python?",
        "Explain neural networks",
        "What is Docker?",
    ];

    const startTime = Date.now();

    for (const query of queries) {
        const { retrievedDocs, answer } = await ragPipeline(
            vectorStore, embeddingContext, chatSession, query, 2
        );
    }

    const totalTime = Date.now() - startTime;
    console.log(`Total time: ${totalTime}ms`);
}
```

**Что демонстрирует:**
- Обработка нескольких запросов
- Производительность при масштабировании
- Последовательная vs параллельная обработка

**Последовательная обработка:**
```javascript
for (const query of queries) {
    await ragPipeline(query);
}
// Время: 3 × 500мс = 1500мс
```

**Почему последовательная (текущий пример):**
- Сохраняется состояние контекста LLM
- Более простой код
- Приемлемо для умеренных нагрузок

**Когда применять параллелизацию:**
```javascript
// Параллельное извлечение (эмбеддинги + поиск)
const embeddings = await Promise.all(
    queries.map(q => embed(q))
);
const retrievals = await Promise.all(
    embeddings.map(e => search(e))
);

// Последовательная генерация (управление состоянием LLM)
for (const [query, docs] of zip(queries, retrievals)) {
    await generate(query, docs);
}
```

**Рекомендация:**
- Параллелизуйте эмбеддинги (независимые операции)
- Параллелизуйте поиски (только чтение)
- Сохраняйте генерацию последовательной (с поддержкой состояния)

---

## Оптимизация производительности

### Разбивка по задержке

```
Типичный запрос RAG: ~500мс
├── Эмбеддинг запроса: 45мс (9%)
├── Векторный поиск: 5мс (1%)
├── Сборка контекста: 2мс (<1%)
└── Генерация LLM: 448мс (90%)
```

**Ключевой вывод:** Генерация LLM — это узкое место!

### Стратегии оптимизации

**1. Используйте более маленькую/быструю LLM**
```javascript
// Вместо модели 7B: ~800мс
// Используйте модель 1B: ~400мс
// Ускорение: 2x
```

**2. Кэшируйте эмбеддинги запросов**
```javascript
const embedCache = new Map();

async function getEmbedding(query) {
    if (!embedCache.has(query)) {
        embedCache.set(query, await embed(query));
    }
    return embedCache.get(query);
}
// Экономия 45мс на каждый кэшированный запрос
```

**3. Уменьшайте размер контекста**
```javascript
// k=5 с длинными документами: ~500мс генерация
// k=3 с теми же документами: ~450мс генерация
// Ускорение: 10%
```

**4. Пакетная генерация эмбеддингов**
```javascript
// Последовательно: 3 × 45мс = 135мс
const embs = [];
for (const q of queries) {
    embs.push(await embed(q));
}

// Параллельно: max(45мс) = 45мс
const embs = await Promise.all(queries.map(embed));
// Ускорение: 3x
```

---

## Сводка рекомендаций

### 1. Настройка параметров

```javascript
// Хорошие значения по умолчанию
const k = 3;                           // Количество документов
const similarityThreshold = 0.4;       // Фильтр качества
const maxTokens = 200;                 // Длина ответа
```

### 2. Обработка ошибок

```javascript
try {
    const { retrievedDocs, answer } = await ragPipeline(query, k);
    
    if (retrievedDocs.length === 0) {
        return "Документы не найдены.";
    }
    
    if (retrievedDocs[0].similarity < 0.3) {
        return "В базе знаний нет релевантной информации.";
    }
    
    return answer;
} catch (error) {
    console.error("Ошибка RAG:", error);
    return "Извините, при обработке вашего вопроса произошла ошибка.";
}
```

### 3. Логирование и мониторинг

```javascript
console.log({
    query,
    retrieved: docs.length,
    topSimilarity: docs[0]?.similarity,
    contextLength: context.length,
    latency: {
        embedding: embedTime,
        search: searchTime,
        generation: genTime,
        total: totalTime
    }
});
```

### 4. Проверки качества

```javascript
// Проверка качества извлечения
if (docs[0].similarity < 0.5) {
    console.warn("Низкое качество извлечения для запроса:", query);
}

// Проверка размера контекста
if (context.length > 2000) {
    console.warn("Большой контекст:", context.length, "символов");
}

// Проверка ответа
if (answer.includes("I don't know") && docs.length > 0) {
    console.warn("LLM не смог ответить при наличии извлечённых документов");
}
```

---

## Типовые паттерны

### Паттерн 1: Фильтрация по порогу сходства

```javascript
const docs = await retrieve(query, k=10);
const filtered = docs.filter(d => d.similarity > 0.4);
const context = buildContext(filtered);
```

### Паттерн 2: Фильтрация по метаданным

```javascript
const docs = await retrieve(query, k=10);
const filtered = docs
    .filter(d => d.similarity > 0.3)
    .filter(d => d.metadata.category === "programming");
const context = buildContext(filtered);
```

### Паттерн 3: Плавная деградация

```javascript
let k = 5;
let docs = await retrieve(query, k);

while (docs.length > 0 && docs[0].similarity < 0.3) {
    k--;
    if (k === 0) return "Релевантная информация не найдена.";
    docs = await retrieve(query, k);
}
```

### Паттерн 4: Указание источников

```javascript
const context = docs.map((doc, idx) => 
    `[Source ${idx + 1}: ${doc.id}]\n${doc.metadata.content}`
).join("\n\n");

const prompt = `${context}\n\nQuestion: ${query}\n\nAnswer (cite sources):`;
```

---

## Итог

### Что мы создали

Семь примеров, демонстрирующих:
1. ✅ Базовый пайплайн RAG (3 шага)
2. ✅ Влияние параметра k
3. ✅ Анализ качества извлечения
4. ✅ Сравнение с/без RAG
5. ✅ Фильтрация по порогу сходства
6. ✅ Управление окном контекста
7. ✅ Пакетная обработка запросов

### Основные компоненты

- **Модель эмбеддингов**: Преобразует текст в векторы
- **Векторное хранилище**: Хранит и ищет векторы документов
- **Извлечение**: Top-k поиск по сходству
- **Сборка контекста**: Форматирует документы для LLM
- **Генерация LLM**: Создаёт ответы на основе контекста

### Ключевые выводы

**RAG = Извлечение + Усиленная генерация**

**Три шага:**
1. Извлечение: Векторный поиск (k=3)
2. Сборка: Форматирование контекста
3. Генерация: LLM + контекст → ответ

**Рекомендации:**
- Начните с k=3
- Фильтруйте по сходству (> 0,4)
- Отслеживайте размер контекста
- Обрабатывайте крайние случаи
- Ведите логирование на начальном этапе
- Сравнивайте результаты с/без RAG

**Производительность:**
- Извлечение: ~50мс (10%)
- Генерация: ~450мс (90%)
- Оптимизируйте генерацию в первую очередь

### Следующие шаги

- **Продвинутое извлечение**: Переупорядочивание, расширение запросов, гибридный поиск
- **Промышленное использование**: Кэширование, обработка ошибок, мониторинг
- **Оценка**: Метрики, A/B тестирование, обратная связь пользователей

Освоив базовое извлечение, вы освоите основу систем RAG!
