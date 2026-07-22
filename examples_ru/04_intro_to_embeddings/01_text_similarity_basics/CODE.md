# Основы сходства текстов — разбор кода

Документ содержит подробное, пошаговое объяснение кода примера сходства текстов.

## Содержание

1. [Обзор](#overview)
2. [Импорт и настройка](#imports-and-setup)
3. [Пример данных](#sample-data)
4. [Основные функции](#core-functions)
5. [Разбор примеров](#examples-breakdown)
6. [Ключевые понятия](#key-concepts)

---

## Обзор

Данный пример демонстрирует **семантическое сходство текстов** с использованием эмбеддингов. Вместо сопоставления ключевых слов мы преобразуем тексты в числовые векторы, которые передают смысл. Похожие тексты имеют схожие векторы, что позволяет находить релевантные документы даже при использовании разных слов.

**Что вы узнаете:**
- Как преобразовать текст в эмбеддинги (числовые векторы)
- Как измерять сходство между текстами с помощью косинусного сходства
- Почему эмбеддинги являются фундаментальной основой систем RAG (Retrieval Augmented Generation)

---

## Импорт и настройка

```javascript
import {fileURLToPath} from "url";
import path from "path";
import {getLlama} from "node-llama-cpp";
import {Document} from '../../02_data_loading/example.js';
import {OutputHelper} from "../../helpers/output-helper.js";
import chalk from "chalk";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
```

### Что здесь происходит?

- **`fileURLToPath` и `path`**: Обработка путей файлов в ES-модулях
- **`getLlama`**: Основной интерфейс node-llama-cpp для загрузки моделей эмбеддингов
- **`Document`**: Класс из предыдущих частей туториала, оборачивающий текст с метаданными
- **`OutputHelper`**: Утилита для красивого вывода в консоль (спиннеры, форматирование, цвета)
- **`chalk`**: Библиотека для цветного вывода в терминале
- **`__dirname`**: Получение текущей директории (необходимо, поскольку ES-модули по умолчанию не имеют `__dirname`)

---

## Пример данных

```javascript
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

const documents = sampleTexts.map((text, i) =>
    new Document(text, { source: 'sample_data' })
);
```

### Почему именно эти тексты?

Эти 10 разнообразных предложений охватывают различные темы:
- Погода (небо)
- Еда (пицца)
- Животные (собаки)
- География (Франция, гора Эверест)
- Здоровье (вода)
- и т. д.

Такое разнообразие помогает продемонстрировать, как эмбеддинги могут различать разные семантические категории.

### Почему оборачивать в объекты Document?

Использование объектов `Document` (вместо простых строк) обеспечивает единообразие с остальным туториалом по RAG. Каждый документ содержит:
- **`pageContent`**: Сам текст
- **`metadata`**: Дополнительная информация (id, источник и т. д.)

Этот паттерн подготавливает вас к работе с реальными документами в дальнейшем.

---

## Основные функции

### 1. Инициализация модели эмбеддингов

```javascript
async function initializeEmbeddingModel() {
    const llama = await getLlama({
        logLevel: 'error' // Only show errors, not warnings or info
    });
    const model = await llama.loadModel({
        modelPath: path.join(__dirname, "bge-small-en-v1.5.Q8_0.gguf")
    });
    return await model.createEmbeddingContext();
}
```

**Пошагово:**

1. **`getLlama()`**: Инициализация среды выполнения llama.cpp
    - `logLevel: 'error'` подавляет подробные логи для более чистого вывода

2. **`loadModel()`**: Загрузка модели эмбеддингов с диска
    - Используется `bge-small-en-v1.5` (BAAI General Embedding)
    - `.Q8_0.gguf` = 8-битный квантизованный формат GGUF (меньше размером, быстрее)

3. **`createEmbeddingContext()`**: Создание контекста для генерации эмбеддингов
    - Этот контекст используется для преобразования текста → векторы

**Почему именно эта модель?**
- **Малый размер**: ~37 МБ, работает на любом компьютере
- **Качество**: Хорошая точность для английского текста
- **Удобство для обучения**: Достаточно быстрая для изучения

---

### 2. Эмбеддинги документов

```javascript
async function embedDocuments(context, documents) {
    const embeddings = new Map();

    await Promise.all(
        documents.map(async (document) => {
            const embedding = await context.getEmbeddingFor(document.pageContent);
            embeddings.set(document, embedding);
        })
    );

    return embeddings;
}
```

**Пошагово:**

1. **Создание Map**: Будем хранить пары `Документ → Эмбеддинг`
    - Использование Map (а не массива) обеспечивает эффективный поиск

2. **`Promise.all()`**: Обработка всех документов параллельно
    - Гораздо быстрее, чем последовательная обработка
    - Для 10 документов это примерно в 10 раз быстрее

3. **`getEmbeddingFor()`**: Преобразование каждого текста в вектор эмбеддинга
    - Вход: `"Mount Everest is the tallest mountain in the world"`
    - Выход: массив из 384 чисел, например, `[0.023, -0.156, 0.891, ...]`

4. **Сохранение в Map**: Связывание каждого документа с его эмбеддингом

**Почему параллельно?**
```javascript
// Sequential (slow)
for (const doc of documents) {
    await embedDocument(doc); // Wait for each
}

// Parallel (fast)
await Promise.all(
    documents.map(doc => embedDocument(doc)) // All at once
);
```

---

### 3. Поиск похожих документов

```javascript
function findSimilarDocuments(queryEmbedding, documentEmbeddings, topK = 3) {
    const similarities = [];

    for (const [document, embedding] of documentEmbeddings) {
        const similarity = queryEmbedding.calculateCosineSimilarity(embedding);
        similarities.push({ document, similarity });
    }

    similarities.sort((a, b) => b.similarity - a.similarity);

    return similarities.slice(0, topK);
}
```

**Пошагово:**

1. **Перебор всех документов**: Сравнение запроса с каждым документом

2. **Вычисление косинусного сходства**:
   ```javascript
   queryEmbedding.calculateCosineSimilarity(embedding)
   ```
    - Возвращает число от -1 до 1
    - **1** = идентичный смысл
    - **0** = нет сходства
    - **-1** = противоположный смысл (редко)

3. **Сохранение результатов**: Сохранение документа и оценки сходства вместе

4. **Сортировка по сходству**: Наибольшие оценки первыми
   ```javascript
   .sort((a, b) => b.similarity - a.similarity)
   ```

5. **Возврат top K**: Возврат только лучших совпадений
   ```javascript
   .slice(0, topK) // Get first 3 results
   ```

**Пояснение косинусного сходства:**

Представьте две стрелки в 384-мерном пространстве:
- Если они указывают в одном направлении → сходство = 1
- Если они перпендикулярны → сходство = 0
- Если они указывают в противоположных направлениях → сходство = -1

```
Query:    ────────→
Document: ────────→  (similarity = 1.0, same direction)

Query:    ────────→
Document:     ↓      (similarity = 0.0, perpendicular)
```

---

## Разбор примеров

### Пример 1: Основы сходства текстов

```javascript
async function example1() {
    // 1. Load model
    const context = await OutputHelper.withSpinner(
        'Loading embedding model...',
        () => initializeEmbeddingModel()
    );

    // 2. Embed all documents
    const documentEmbeddings = await OutputHelper.withSpinner(
        'Creating embeddings...',
        () => embedDocuments(context, documents)
    );

    // 3. Define query
    const query = "What is the tallest mountain on Earth?";

    // 4. Embed query
    const queryEmbedding = await context.getEmbeddingFor(query);

    // 5. Find similar documents
    const topResults = findSimilarDocuments(queryEmbedding, documentEmbeddings, 3);

    // 6. Display results
    topResults.forEach((result, index) => {
        console.log(`${index + 1}. [Similarity: ${result.similarity.toFixed(4)}]`);
        console.log(`   "${result.document.pageContent}"`);
    });
}
```

**Момент волшебства:**

Запрос: `"What is the tallest mountain on Earth?"`

Лучший результат: `"Mount Everest is the tallest mountain in the world"` (сходство: ~0.65)

**Почему это впечатляет?**
- В запросе используется: "tallest mountain on **Earth**"
- В документе используется: "tallest mountain in the **world**"
- Точное совпадения ключевых слов нет, но эмбеддинги понимают, что эти выражения означают одно и то же!

**Традиционный поиск по ключевым словам здесь не сработал бы**, поскольку "Earth" ≠ "world" как строки.

---

### Пример 2: Несколько запросов

```javascript
const queries = [
    "Tell me about hydration",
    "What's a good winter drink?",
    "Information about European capitals"
];
```

**Что это демонстрирует:**

1. **Многократное использование**: Документы эмбеддингируются ОДИН раз, затем используются повторно
   ```javascript
   // Embed documents (expensive, do once)
   const documentEmbeddings = await embedDocuments(context, documents);
   
   // Query multiple times (cheap)
   for (const query of queries) {
       const queryEmbedding = await context.getEmbeddingFor(query);
       // Find matches in pre-computed embeddings
   }
   ```

2. **Шаблон эффективности для RAG**:
    - Хранение эмбеддингов документов в базе данных
    - Для каждого пользовательского запроса эмбеддингируется только запрос (быстро)
    - Поиск по предварительно вычисленным эмбеддингам (очень быстро)

**Аналогия из реального мира:**
- Вместо чтения 1000 книг для каждого вопроса (медленно)
- Вы индексируете книги один раз (единовременные затраты)
- Затем быстро находите нужные страницы (быстро)

---

### Пример 3: Понимание векторов эмбеддингов

```javascript
const sampleDoc = documents[0];
const embedding = await context.getEmbeddingFor(sampleDoc.pageContent);

console.log('Vector Dimensions:', embedding.vector.length); // 384
console.log('First 10 values:', embedding.vector.slice(0, 10));
// [0.0234, -0.1567, 0.8912, -0.0123, ...]
```

**Что на самом деле содержится в эмбеддинге?**

```javascript
Document: "The sky is clear and blue today"

Embedding: [
    0.0234,   // Position 0
   -0.1567,   // Position 1
    0.8912,   // Position 2
   -0.0123,   // Position 3
    // ... 380 more numbers
]
```

Каждое из 384 чисел представляет «признак», выученный моделью:
- Позиция 0 может отражать «позитивность»
- Позиция 1 может отражать «временную характеристику»
- Позиция 2 может отражать «связь с природой»
- и т. д.

**Расчёт объёма хранилища:**
```javascript
384 dimensions × 4 bytes per float = 1,536 bytes ≈ 1.5 KB per document
```

Для 1 миллиона документов: 1,5 ГБ эмбеддингов

---

### Пример 4: Распределение оценок сходства

```javascript
const allResults = findSimilarDocuments(
    queryEmbedding, 
    documentEmbeddings, 
    documents.length  // Return ALL documents, not just top 3
);

allResults.forEach((result, index) => {
    const score = result.similarity.toFixed(4);
    const bar = '█'.repeat(Math.round(result.similarity * 30));
    console.log(`${index + 1}. ${score} ${bar}`);
});
```

**Пример вывода:**
```
Query: "Tell me about nature and weather"

 1. 0.6234 ██████████████████
    "The sky is clear and blue today"

 2. 0.5421 ████████████████
    "A warm cup of tea is perfect for a cold winter day"

 3. 0.2134 ██████
    "Mount Everest is the tallest mountain in the world"

 4. 0.1023 ███
    "The capital of France is Paris"
```

**Что это показывает:**

1. **Высокие оценки (>0.5)**: Сильное семантическое совпадение
2. **Средние оценки (0.2–0.5)**: Частичная релевантность
3. **Низкие оценки (<0.2)**: Слабая или отсутствующая релевантность

**Инсайт о распределении:**
- Не все документы одинаково релевантны
- Чёткое разделение между релевантными и нерелевантными
- Именно так работает RAG: мы можем отфильтровать шум

---

### Пример 5: Сравнение различных запросов

```javascript
const queryPairs = [
    {
        type: 'Keyword vs Semantic',
        queries: [
            "mountain tallest",                        // Keyword-style
            "What is the highest peak in the world?"   // Natural language
        ]
    }
];
```

**Тестирование формулировок запросов:**

| Тип запроса | Запрос | Лучшее совпадение | Оценка |
|------------|-------|-------------------|--------|
| Ключевое слово | "mountain tallest" | Mount Everest... | 0.6523 |
| Естественный язык | "What is the highest peak?" | Mount Everest... | 0.6498 |

**Ключевой вывод:** Оба стиля работают хорошо!
- Ключевые слова: Просто и прямо
- Естественный язык: Больше контекста, но тоже эффективно

**Почему это важно:**
- Пользователи могут искать естественным языком («Какой хороший напиток на зиму?»)
- Нет необходимости подбирать идеальные ключевые слова
- Эмбеддинги понимают намерение, а не просто слова
