# Генерация эмбеддингов — разбор кода

Пошаговое руководство по эффективной генерации, хранению и управлению эмбеддингами для систем RAG.

## Обзор

В этом примере вы научитесь масштабироваться от 10 документов (01_text_similarity_basics) до тысяч документов, внедрив персистентность, пакетную обработку и инкрементальные обновления.

---

## Основные функции

### 1. Инициализация модели эмбеддингов
```javascript
async function initializeEmbeddingModel() {
    const llama = await getLlama({
        logLevel: 'error'
    });
    const model = await llama.loadModel({
        modelPath: MODEL_PATH
    });
    return await model.createEmbeddingContext();
}
```

**Что делает:** Загружает модель эмбеддингов BGE и создаёт контекст для генерации эмбеддингов.

**Важный момент:** Это ресурсоёмкая операция (занимает ~1–2 секунды), поэтому мы выполняем её только один раз для каждого примера.

---

### 2. Генерация эмбеддингов с прогрессом
```javascript
async function generateEmbeddings(context, documents, onProgress = null) {
    const embeddings = [];
    let processed = 0;

    for (const document of documents) {
        const embedding = await context.getEmbeddingFor(document.pageContent);
        
        embeddings.push({
            id: document.metadata.id || `doc_${processed}`,
            content: document.pageContent,
            metadata: document.metadata,
            embedding: Array.from(embedding.vector), // Convert to plain array
            timestamp: Date.now()
        });

        processed++;
        if (onProgress) {
            onProgress(processed, documents.length); // Report progress
        }
    }

    return embeddings;
}
```

**Что делает:**
1. Перебирает документы
2. Генерирует эмбеддинг для текста каждого документа
3. Сохраняет эмбеддинг вместе с метаданными
4. Отслеживает прогресс (опциональный callback)

**Ключевой момент:** `Array.from(embedding.vector)` преобразует нативный объект эмбеддинга в обычный JavaScript-массив, который можно сохранить в JSON.

---

### 3. Сохранение эмбеддингов (JSON)
```javascript
async function saveEmbeddingsJSON(embeddings, filename) {
    await fs.mkdir(STORAGE_DIR, {recursive: true});
    const filepath = path.join(STORAGE_DIR, filename);

    const data = {
        version: '1.0',
        model: 'bge-small-en-v1.5',
        dimensions: (embeddings.length > 0 && embeddings[0]?.embedding)
            ? embeddings[0].embedding.length
            : 384,
        count: embeddings.length,
        created: new Date().toISOString(),
        embeddings: embeddings
    };

    await fs.writeFile(filepath, JSON.stringify(data, null, 2), 'utf-8');
    
    const stats = await fs.stat(filepath);
    return {filepath, size: stats.size};
}
```

**Что делает:**
1. Создаёт каталог для хранения (если он ещё не существует)
2. Оборачивает эмбеддинги в метаданные (версия, модель, размерность, количество, временная метка)
3. Сохраняет в форматированном JSON
4. Возвращает путь к файлу и его размер

**Зачем нужны метаданные?** При последующей загрузке эмбеддингов вы сможете узнать:
- Какая модель их сгенерировала
- Когда они были созданы
- Совместимы ли они с текущей конфигурацией

---

### 4. Загрузка эмбеддингов
```javascript
async function loadExistingEmbeddings(filename) {
    try {
        const embeddings = await loadEmbeddingsJSON(filename);
        const embeddingMap = new Map();

        for (const item of embeddings) {
            embeddingMap.set(item.id, item);
        }

        return embeddingMap;
    } catch (error) {
        return new Map(); // No existing embeddings
    }
}
```

**Что делает:** Читает JSON-файл и извлекает массив эмбеддингов.

**Производительность:** Загрузка в 100–1000 раз быстрее, чем повторная генерация эмбеддингов!

---

### 5. Инкрементальные обновления
```javascript
async function incrementalEmbedding(context, newDocuments, existingFilename) {
    const existingMap = await loadExistingEmbeddings(existingFilename);

    // Filter out documents that already have embeddings
    const documentsToEmbed = newDocuments.filter(doc => {
        const id = doc.metadata.id || doc.pageContent.substring(0, 50);
        return !existingMap.has(id);
    });

    console.log(`Existing embeddings: ${existingMap.size}`);
    console.log(`New documents to embed: ${documentsToEmbed.length}`);
    console.log(`Skipped (already embedded): ${newDocuments.length - documentsToEmbed.length}`);

    if (documentsToEmbed.length === 0) {
        console.log(chalk.green('All documents already embedded!'));
        return Array.from(existingMap.values());
    }

    // Generate embeddings only for new documents
    const newEmbeddings = await generateEmbeddings(
        context,
        documentsToEmbed,
        (current, total) => {
            process.stdout.write(`\rEmbedding: ${current}/${total}`);
        }
    );
    console.log(); // New line

    // Merge with existing
    return [
        ...Array.from(existingMap.values()),
        ...newEmbeddings
    ];
}
```

**Что делает:**
1. Загружает существующие эмбеддинги в Map (для быстрого поиска)
2. Сверяет каждый новый документ с уже имеющимися идентификаторами
3. Генерирует эмбеддинги только для документов, которых ещё нет
4. Объединяет новые эмбеддинги с существующими

**Почему это важно:** Если у вас 1000 документов с эмбеддингами и вы добавляете 10 новых, вы генерируете эмбеддинги только для 10 (а не для 1010).

---

## Разбор примеров

### Пример 1: Пакетная обработка

**Цель:** Продемонстрировать эффективную генерацию эмбеддингов для 100 документов.

**Ключевой код:**
```javascript
const embeddings = await generateEmbeddings(
    context,
    documents,
    (current, total) => {
        const percent = ((current / total) * 100).toFixed(1);
        process.stdout.write(`\rProgress: ${current}/${total} (${percent}%)`);
    }
);
```

**Вывод:**
```
Progress: 100/100 (100.0%)
✓ Completed in 0.49s
Throughput: 204.1 docs/sec
```

**Вывод:** С помощью пакетной обработки вы можете обработать сотни документов за считанные секунды.

---

### Пример 2: Сохранение и загрузка

**Цель:** Продемонстрировать значительное ускорение за счёт кэширования эмбеддингов.

**Тест:**
1. Генерация эмбеддингов: ~0,75 с
2. Сохранение на диск: ~5 мс
3. Загрузка с диска: ~2 мс

**Результат:** Загрузка выполняется **в 373 раза быстрее**, чем генерация!

**Шаблон кода:**
```javascript
// Generate once
const embeddings = await generateEmbeddings(context, documents);
await saveEmbeddingsJSON(embeddings, 'embeddings.json');

// Load many times (fast!)
const loadedEmbeddings = await loadEmbeddingsJSON('embeddings.json');
```

**Практическое применение:**
```javascript
// On first run or when documents change
if (!fs.existsSync('embeddings.json')) {
    embeddings = await generateEmbeddings(context, documents);
    await saveEmbeddingsJSON(embeddings, 'embeddings.json');
}

// On subsequent runs (normal case)
embeddings = await loadEmbeddingsJSON('embeddings.json');
```

---

### Пример 3: Инкрементальные обновления

**Цель:** Генерировать эмбеддинги только для новых документов, пропуская уже существующие.

**Сценарий:**
- Начинаем с 30 документов (с эмбеддингами, сохранёнными на диск)
- Добавляем 20 новых документов + 5 дубликатов
- Ожидаемый результат: генерация эмбеддингов только для 20 новых документов

**Вывод:**
```
Existing embeddings: 30
New documents to embed: 20
Skipped (already embedded): 5
✓ Update Time: 0.08s
```

**Почему это важно:** В продакшене коллекция документов растёт со временем. Инкрементальные обновления экономят часы повторных вычислений.

---

### Пример 4: Сравнение форматов хранения

**Цель:** Сравнить производительность JSON-формата.

**Результаты:**
- **JSON**: 1,10 МБ, время загрузки 5 мс
- Читаем человеком
- Легко отлаживать
- Подходит для разработки

**Когда использовать JSON:**
- Разработка и отладка
- Небольшие наборы данных (<1000 документов)
- Когда необходимо проверить содержимое эмбеддингов
- Переносимость между системами

---

### Пример 5: Подготовка к хранилищам векторов

**Цель:** Структурировать эмбеддинги для импорта в базу данных.

**Формат хранилища векторов:**
```javascript
{
  "id": "doc_0",
  "vector": [0.023, -0.156, 0.891, ...], // 384 numbers
  "metadata": {
    "content": "Introduction to machine learning...",
    "source": "ml_guide.pdf",
    "page": 1,
    "chunk": 0,
    "timestamp": 1234567890
  }
}
```

**Почему такая структура?**
- `id`: уникальный идентификатор для поиска
- `vector`: эмбеддинг (то, по чему осуществляется поиск)
- `metadata`: всё остальное (отображается в результатах)

**Следующий шаг:** импорт в LanceDB, Qdrant, Chroma или Milvus.

---

### Пример 6: Практический конвейер

**Цель:** Полный рабочий процесс от PDF до сохранённых эмбеддингов.

**Конвейер:**
```
1. Загрузка PDF
   ↓ PDFLoader
   [22 страницы текста]

2. Разбиение на фрагменты
   ↓ RecursiveCharacterTextSplitter()
   [334 текстовых фрагмента]

3. Генерация эмбеддингов
   ↓ generateEmbeddings() (первые 20 фрагментов для демонстрации)
   [20 векторных эмбеддингов]

4. Сохранение на диск
   ↓ saveEmbeddingsJSON()
   [pdf_embeddings.json — 1070 КБ]
```

**Код:**
```javascript
const loader = new PDFLoader(pdfUrl, {splitPages: true});
const docs = await loader.load();
const splitter = new RecursiveCharacterTextSplitter({chunkSize: 500, chunkOverlap: 50});
const chunks = await splitter.splitDocuments(docs);
const embeddings = await generateEmbeddings(context, chunks.slice(0, 20));
await saveEmbeddingsJSON(embeddings, 'pdf_embeddings.json');
```

---

## Ключевые шаблоны

### Шаблон 1: Проверка перед генерацией
```javascript
// Smart caching pattern
let embeddings;
try {
    embeddings = await loadEmbeddingsJSON('embeddings.json');
    console.log('Loaded cached embeddings');
} catch {
    embeddings = await generateEmbeddings(context, documents);
    await saveEmbeddingsJSON(embeddings, 'embeddings.json');
    console.log('Generated and cached embeddings');
}
```

### Шаблон 2: Отслеживание прогресса
```javascript
await generateEmbeddings(context, documents, (current, total) => {
    process.stdout.write(`\rProgress: ${current}/${total}`);
});
console.log(); // New line after progress
```

### Шаблон 3: Обогащение метаданными
```javascript
// Add metadata during embedding generation
embeddings.push({
    id: document.metadata.id,
    content: document.pageContent,
    metadata: {
        ...document.metadata,
        embedded_at: Date.now(),
        model: 'bge-small-en-v1.5',
        chunk_size: document.pageContent.length
    },
    embedding: Array.from(embedding.vector)
});
```

---

## Показатели производительности

На основе результатов примеров:

| Операция | Время | Примечания |
|-----------|------|------------|
| Генерация 100 эмбеддингов | 0,49 с | ~5 мс на документ |
| Сохранение в JSON | 5 мс | Пренебрежимо малые накладные расходы |
| Загрузка из JSON | 2 мс | В 373 раза быстрее, чем генерация |
| Инкрементальное обновление (20 новых) | 0,08 с | Пропуск существующих эмбеддингов |

**Производительность:** ~200 документов в секунду на CPU

**Размер хранения:** ~11 КБ на документ (формат JSON)

---

## Распространённые ошибки

### ❌ Повторная генерация всех эмбеддингов
```javascript
// BAD: Regenerate all embeddings every time
const embeddings = await generateEmbeddings(context, allDocuments);
```

### ✅ Используйте кэширование
```javascript
// GOOD: Load if available, generate if needed
const embeddings = await loadEmbeddingsJSON('cache.json')
    .catch(() => generateEmbeddings(context, allDocuments));
```

### ❌ Отсутствие отслеживания идентификаторов документов
```javascript
// BAD: No way to know what's embedded
embeddings.push({ embedding: [...] });
```

### ✅ Всегда включайте идентификаторы
```javascript
// GOOD: Track source and content
embeddings.push({
    id: doc.metadata.id,
    content: doc.pageContent,
    embedding: [...]
});
```

### ❌ Забытые метаданные
```javascript
// BAD: Just the vector
{ embedding: [...] }
```

### ✅ Добавляйте контекст
```javascript
// GOOD: Vector + metadata for retrieval
{
    embedding: [...],
    metadata: {
        source: 'document.pdf',
        page: 5,
        timestamp: Date.now()
    }
}
```