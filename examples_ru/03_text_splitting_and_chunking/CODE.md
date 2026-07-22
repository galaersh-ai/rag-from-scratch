# Разбор кода разбиения текста

## Содержание
1. [Разбор по классам](#разбор-по-классам)
2. [Детальный обзор алгоритмов](#детальный-обзор-алгоритмов)
3. [Разбор примеров](#разбор-примеров)
4. [Лучшие практики](#лучшие-практики)

---

## Разбор по классам

### 1. TextSplitter (базовый класс)

```javascript
export class TextSplitter {
    constructor({
        chunkSize = 1000,
        chunkOverlap = 200,
        lengthFunction = t => t.length,
        keepSeparator = false
    } = {}) {
        if (chunkOverlap >= chunkSize) {
            throw new Error('chunkOverlap must be less than chunkSize');
        }
        Object.assign(this, {chunkSize, chunkOverlap, lengthFunction, keepSeparator});
    }
```

#### Параметры конструктора

**`chunkSize`** (по умолчанию: 1000)
- Максимальный размер каждого чанка
- Измеряется с помощью `lengthFunction` (символы или токены)
- Представьте это как «целевой размер» для чанков

**`chunkOverlap`** (по умолчанию: 200)
- Сколько контента повторяется между чанками
- Создает непрерывность контекста
- Должно быть меньше `chunkSize`

**Пример:**
```javascript
Text: "ABCDEFGHIJ" (10 chars)
chunkSize = 4
chunkOverlap = 1

Result:
Chunk 1: "ABCD"
Chunk 2:    "DEFG"  (D is overlap)
Chunk 3:       "GHIJ" (G is overlap)
```

**`lengthFunction`**
- Способ измерения длины текста
- По умолчанию: подсчет символов `t => t.length`
- Для токенов: `t => Math.ceil(t.length / 4)`
- Позволяет использовать гибкие стратегии измерения

**`keepSeparator`**
- Включать ли разделители в чанки
- Обычно `false` (удалять разделители)
- Пример: сохранить `\n\n` или удалить его

#### Валидация

```javascript
if (chunkOverlap >= chunkSize) {
    throw new Error('chunkOverlap must be less than chunkSize');
}
```

**Почему?** Если перекрытие ≥ размера, чанки будут бесконечно повторять контент.

```
chunkSize = 5, overlap = 5
Chunk 1: "ABCDE"
Chunk 2: "ABCDE" (same content!)
Chunk 3: "ABCDE" (infinite loop!)
```

---

### 2. Основные методы

#### `splitText()`

```javascript
splitText() {
    throw new Error('splitText() must be implemented by subclass');
}
```

**Паттерн абстрактного метода**
- Заставляет подклассы реализовывать собственную логику
- Каждый сплиттер имеет уникальную стратегию разбиения
- Базовый класс предоставляет общую инфраструктуру

---

#### `splitDocuments()`

```javascript
async splitDocuments(documents) {
    const chunks = [];
    for (const doc of documents) {
        chunks.push(...await this.createDocuments([doc.pageContent], [doc.metadata]));
    }
    return chunks;
}
```

**Назначение:** Основная точка входа для разбиения нескольких документов

**Пошаговый процесс:**
```
Input:  [Document1, Document2, Document3]
           ↓
Loop:   Document1 → createDocuments() → [Chunk1, Chunk2, Chunk3]
        Document2 → createDocuments() → [Chunk4, Chunk5]
        Document3 → createDocuments() → [Chunk6, Chunk7, Chunk8]
           ↓
Output: [Chunk1, Chunk2, ..., Chunk8]
```

**Ключевые моменты:**
- Сохраняет метаданные из исходных документов
- Обрабатывает каждый документ независимо
- Объединяет результаты в один массив
- `async` для возможных будущих асинхронных операций

---

#### `createDocuments()`

```javascript
async createDocuments(texts, metadatas = []) {
    const documents = [];
    for (let i = 0; i < texts.length; i++) {
        const text = texts[i];
        const metadata = metadatas[i] || {};
        const chunks = this.splitText(text);

        for (let j = 0; j < chunks.length; j++) {
            documents.push(
                new Document(chunks[j], {
                    ...metadata,
                    chunk: j,
                    totalChunks: chunks.length
                })
            );
        }
    }
    return documents;
}
```

**Назначение:** Преобразование текстовых строк в объекты Document с метаданными

**Пошаговый процесс:**
```
1. Input: texts=["full text"], metadatas=[{source: "pdf", page: 5}]
   
2. Split text:
   chunks = this.splitText("full text")
   → ["chunk1", "chunk2", "chunk3"]

3. Create Documents:
   For each chunk, create Document with:
   - pageContent: the chunk text
   - metadata: original + chunk info
   
4. Metadata enrichment:
   {
       source: "pdf",        // from original
       page: 5,              // from original
       chunk: 0,             // NEW: which chunk (0-indexed)
       totalChunks: 3        // NEW: total count
   }
```

**Зачем добавлять метаданные чанков?**
- Отслеживание позиции в исходном документе
- Возможность навигации «следующий/предыдущий чанк»
- Полезно для отслеживания и отладки
- Помогает при дедупликации

---

#### `mergeSplits()` — основной алгоритм

```javascript
mergeSplits(splits, separator) {
    const chunks = [];
    let current = [];
    let length = 0;

    for (const split of splits) {
        const splitLength = this.lengthFunction(split);
        const extraLength = current.length ? separator.length : 0;

        // finalize current chunk if it exceeds size
        if (length + splitLength + extraLength > this.chunkSize) {
            if (current.length) {
                chunks.push(this.joinSplits(current, separator));
            }

            // maintain overlap
            while (length > this.chunkOverlap && current.length) {
                length -= this.lengthFunction(current.shift()) + separator.length;
            }
        }

        current.push(split);
        length += splitLength + (current.length > 1 ? separator.length : 0);
    }

    if (current.length) {
        chunks.push(this.joinSplits(current, separator));
    }

    return chunks.filter(Boolean);
}
```

**Назначение:** Основной алгоритм, создающий чанки с перекрытием

**Наглядный пример:**
```
splits = ["A", "B", "C", "D", "E", "F"]
chunkSize = 3 (chars)
chunkOverlap = 1 (char)
separator = ""

Iteration 1: Add "A"
  current = ["A"], length = 1

Iteration 2: Add "B"
  current = ["A", "B"], length = 2

Iteration 3: Add "C"
  current = ["A", "B", "C"], length = 3

Iteration 4: Try to add "D"
  length + "D" = 3 + 1 = 4 > chunkSize (3)
  → Finalize chunk: "ABC"
  → Remove splits until length ≤ overlap (1)
  → Remove "A", "B" → Keep "C"
  → current = ["C"], length = 1
  → Add "D": current = ["C", "D"], length = 2

Iteration 5: Add "E"
  current = ["C", "D", "E"], length = 3

Iteration 6: Try to add "F"
  length + "F" = 4 > chunkSize
  → Finalize chunk: "CDE"
  → Maintain overlap: current = ["E"]
  → Add "F": current = ["E", "F"]

Final: Add remaining: "EF"

Result: ["ABC", "CDE", "EF"]
         overlap→ C
                overlap→ E
```

**Ключевые шаги алгоритма:**

1. **Фаза накопления:**
   ```javascript
   current.push(split);
   length += splitLength + (current.length > 1 ? separator.length : 0);
   ```
    - Добавление разбиения в текущий чанк
    - Отслеживание суммарной длины
    - Учет разделителей между разбиениями

2. **Проверка переполнения:**
   ```javascript
   if (length + splitLength + extraLength > this.chunkSize) {
   ```
    - Превысит ли добавление этого разбиения лимит?
    - Если да, сначала завершить текущий чанк

3. **Завершение:**
   ```javascript
   if (current.length) {
       chunks.push(this.joinSplits(current, separator));
   }
   ```
    - Объединение накопленных разбиений
    - Добавление в массив чанков

4. **Поддержание перекрытия:**
   ```javascript
   while (length > this.chunkOverlap && current.length) {
       length -= this.lengthFunction(current.shift()) + separator.length;
   }
   ```
    - Удаление разбиений из начала
    - Удаление до тех пор, пока длина ≤ chunkOverlap
    - Это создает перекрытие для следующего чанка

**Зачем `filter(Boolean)` в конце?**
- `joinSplits()` может вернуть `null` для пустых чанков
- Удаляет любые записи null/undefined
- Гарантирует, что возвращаются только валидные чанки

---

#### `joinSplits()`

```javascript
joinSplits(splits, separator) {
    const text = splits.join(separator).trim();
    return text || null;
}
```

**Назначение:** Объединение текстовых фрагментов обратно в единую строку

**Примеры:**
```javascript
// With separator
joinSplits(["Hello", "World"], " ")
→ "Hello World"

// With newline separator
joinSplits(["Paragraph 1", "Paragraph 2"], "\n\n")
→ "Paragraph 1\n\nParagraph 2"

// Empty result
joinSplits(["", ""], " ")
→ null (because trim() makes it empty)
```

**Почему возвращать `null` для пустого результата?**
- Сигнализирует «нет валидного контента»
- Отфильтровывается методом `mergeSplits()`
- Предотвращает наличие пустых чанков в финальном результате

---

### 3. CharacterTextSplitter

```javascript
export class CharacterTextSplitter extends TextSplitter {
    constructor({
        separator = '\n\n',
        chunkSize = 1000,
        chunkOverlap = 200,
        lengthFunction,
        keepSeparator = false
    } = {}) {
        super({chunkSize, chunkOverlap, lengthFunction, keepSeparator});
        this.separator = separator;
    }

    splitText(text) {
        const splits = text.split(this.separator).filter(s => s.trim().length > 0);
        return this.mergeSplits(splits, this.separator);
    }
}
```

**Назначение:** Простое разбиение по одному разделителю

**Как работает:**
```
Input text:
"Paragraph 1\n\nParagraph 2\n\nParagraph 3"

Step 1: Split by separator ('\n\n')
→ ["Paragraph 1", "Paragraph 2", "Paragraph 3"]

Step 2: Filter empty strings
→ ["Paragraph 1", "Paragraph 2", "Paragraph 3"]

Step 3: Merge with overlap
→ Uses base class mergeSplits() algorithm
```

**Сценарии использования:**
- Разбиение по абзацам (`\n\n`)
- Разбиение по строкам (`\n`)
- Разбиение по секциям (пользовательский разделитель)

**Пример:**
```javascript
const splitter = new CharacterTextSplitter({
    separator: '\n\n',
    chunkSize: 500,
    chunkOverlap: 50
});

const text = "Para 1\n\nPara 2\n\nPara 3\n\nPara 4";
const chunks = splitter.splitText(text);
// If each para is ~150 chars:
// Chunk 1: "Para 1\n\nPara 2\n\nPara 3" (450 chars)
// Chunk 2: "Para 3\n\nPara 4" (overlap from Para 3)
```

---

### 4. RecursiveCharacterTextSplitter (наиболее важный)

```javascript
export class RecursiveCharacterTextSplitter extends TextSplitter {
    constructor({
        separators = ['\n\n', '\n', '. ', ' ', ''],
        chunkSize = 1000,
        chunkOverlap = 200,
        lengthFunction,
        keepSeparator = false
    } = {}) {
        super({chunkSize, chunkOverlap, lengthFunction, keepSeparator});
        this.separators = separators;
    }
```

**Назначение:** Умное разбиение, которое иерархически пробует несколько разделителей

**Иерархия разделителей:**
```
1. '\n\n'  → Абзацы (крупнейшая семантическая единица)
2. '\n'    → Строки
3. '. '    → Предложения
4. ' '     → Слова
5. ''      → Символы (крайний случай)
```

**Почему именно такой порядок?**
- Сохраняет более крупные семантические единицы, когда это возможно
- Переходит к более мелким единицам только при необходимости
- Сохраняет смысл и контекст

---

#### Алгоритм `splitText()`

```javascript
splitText(text) {
    const finalChunks = [];
    let separator = this.separators.at(-1);  // Default to last (empty string)
    let nextSeparators = [];

    // Choose appropriate separator
    for (let i = 0; i < this.separators.length; i++) {
        const sep = this.separators[i];
        if (text.includes(sep)) {
            separator = sep;
            nextSeparators = this.separators.slice(i + 1);
            break;
        }
    }

    const splits = text.split(separator).filter(Boolean);
    let temp = [];

    for (const s of splits) {
        if (this.lengthFunction(s) <= this.chunkSize) {
            temp.push(s);  // Accumulate good splits
        } else {
            // Split is too large
            if (temp.length) {
                finalChunks.push(...this.mergeSplits(temp, separator));
                temp = [];
            }

            if (nextSeparators.length === 0) {
                finalChunks.push(s);  // No more separators, keep as-is
            } else {
                // Recursively split with next separator
                const recursiveSplitter = new RecursiveCharacterTextSplitter({
                    ...this,
                    separators: nextSeparators
                });
                finalChunks.push(...recursiveSplitter.splitText(s));
            }
        }
    }

    if (temp.length) {
        finalChunks.push(...this.mergeSplits(temp, separator));
    }

    return finalChunks;
}
```

**Пошаговый разбор:**

**Пример текста:**
```
"This is paragraph 1.\n\nThis is a very long paragraph that exceeds the chunk size limit and needs to be split further. It has multiple sentences.\n\nThis is paragraph 3."
```

**Выполнение:**

**Шаг 1: Выбор разделителя**
```javascript
for (let i = 0; i < this.separators.length; i++) {
    const sep = this.separators[i];
    if (text.includes(sep)) {
        separator = sep;
        nextSeparators = this.separators.slice(i + 1);
        break;
    }
}
```

- Проверка `\n\n`: ✓ Найдено!
- `separator = '\n\n'`
- `nextSeparators = ['\n', '. ', ' ', '']`

**Шаг 2: Разбиение по выбранному разделителю**
```javascript
const splits = text.split(separator).filter(Boolean);
```

Результат:
```
splits = [
    "This is paragraph 1.",
    "This is a very long paragraph...",  // Too long!
    "This is paragraph 3."
]
```

**Шаг 3: Обработка каждого разбиения**

```javascript
for (const s of splits) {
    if (this.lengthFunction(s) <= this.chunkSize) {
        temp.push(s);  // Good size, accumulate
    } else {
        // Too large, needs further splitting
    }
}
```

**Обработка:**

1. **Разбиение 1: "This is paragraph 1."** (22 символа)
    - ✓ В пределах лимита (при chunkSize = 100)
    - Добавление в `temp`: `temp = ["This is paragraph 1."]`

2. **Разбиение 2: "This is a very long paragraph..."** (150 символов)
    - ✗ Превышает лимит!
    - Завершение `temp`: `finalChunks.push(...mergeSplits(temp))`
    - `temp = []`
    - **Рекурсивное разбиение** со следующими разделителями `['\n', '. ', ' ', '']`

3. **Рекурсивный вызов:**
   ```javascript
   const recursiveSplitter = new RecursiveCharacterTextSplitter({
       ...this,
       separators: ['\n', '. ', ' ', '']  // Without '\n\n'
   });
   finalChunks.push(...recursiveSplitter.splitText(largeParagraph));
   ```

    - Проверка `\n`: Не найдено (нет переносов строк)
    - Проверка `. `: ✓ Найдено!
    - Разбиение на предложения
    - Каждое теперь в пределах лимита

4. **Разбиение 3: "This is paragraph 3."** (22 символа)
    - ✓ В пределах лимита
    - Добавление в `temp`

**Шаг 4: Завершение оставшегося**
```javascript
if (temp.length) {
    finalChunks.push(...this.mergeSplits(temp, separator));
}
```

**Наглядная схема:**
```
Input:
┌─────────────────────────────────────────┐
│ Para1 \n\n HUGE_Para2 \n\n Para3        │
└─────────────────────────────────────────┘
              ↓ split by '\n\n'
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Para1   │  │ HUGE_Para│  │  Para3   │
│  (OK)    │  │   (TOO   │  │  (OK)    │
│          │  │   BIG!)  │  │          │
└──────────┘  └──────────┘  └──────────┘
                    ↓ recursive split by '. '
              ┌─────┬─────┬─────┐
              │Sent1│Sent2│Sent3│
              │(OK) │(OK) │(OK) │
              └─────┴─────┴─────┘
                    ↓
Final Chunks: [Para1, Sent1, Sent2, Sent3, Para3]
```

**Ключевые выводы:**

1. **Жадный выбор разделителя:**
    - Всегда используется наиболее крупный разделитель, присутствующий в тексте
    - Сохраняются семантические единицы

2. **Рекурсия для крупных разбиений:**
    - Если разбиение слишком велико после использования текущего разделителя
    - Пробуются более мелкие разделители рекурсивно
    - Процесс продолжается, пока все чанки не впишутся в лимит

3. **Стратегия накопления:**
    - «Хорошие» разбиения собираются в `temp`
    - Объединяются вместе (с перекрытием)
    - Рекурсия используется только при крайней необходимости

---

### 5. TokenTextSplitter

```javascript
export class TokenTextSplitter extends TextSplitter {
    constructor({
        encodingName = 'cl100k_base',
        chunkSize = 1000,
        chunkOverlap = 200
    } = {}) {
        const lengthFunction = text => Math.ceil(text.length / 4);
        super({chunkSize, chunkOverlap, lengthFunction});
        this.encodingName = encodingName;
    }

    splitText(text) {
        const splitter = new RecursiveCharacterTextSplitter({
            separators: ['\n\n', '\n', '. ', ' ', ''],
            chunkSize: this.chunkSize,
            chunkOverlap: this.chunkOverlap,
            lengthFunction: this.lengthFunction
        });
        return splitter.splitText(text);
    }
}
```

**Назначение:** Разбиение на основе подсчета токенов вместо символов

**Оценка количества токенов:**
```javascript
const lengthFunction = text => Math.ceil(text.length / 4);
```

**Правило приближения:**
- 1 токен ≈ 4 символа (грубая оценка)
- Точное значение зависит от языка и токенизатора
- В продакшене: используйте полноценный токенизатор (tiktoken, GPT-tokenizer)

**Как работает:**
```
Text: "Hello world" (11 chars)
Token estimate: Math.ceil(11 / 4) = 3 tokens

Actual GPT tokens: ["Hello", " world"] = 2 tokens
(Approximation is close enough for chunking)
```

**Зачем приближать?**
- Полноценная токенизация медленна
- Достаточно точно для определения границ чанков
- Точный подсчет токенов происходит позже, при создании эмбеддингов

**Сценарии использования:**
- Модели эмбеддингов с ограничениями на количество токенов (например, 512 токенов)
- Ограничения токенов в API OpenAI
- Обеспечение вписывания чанков в контекстные окна

**Пример:**
```javascript
const splitter = new TokenTextSplitter({
    encodingName: 'cl100k_base',  // GPT-4 encoding
    chunkSize: 500,                // 500 tokens
    chunkOverlap: 50               // 50 token overlap
});

// Internally uses RecursiveCharacterTextSplitter
// but with token-based length measurement
```

---

## Детальный обзор алгоритмов

### Алгоритм перекрытия, объясненный

**Задача:** Как эффективно создавать чанки с перекрытием?

**Решение:** Подход «Скользящее окно с памятью»

```javascript
while (length > this.chunkOverlap && current.length) {
    length -= this.lengthFunction(current.shift()) + separator.length;
}
```

**Наглядный пример:**
```
Text: "A B C D E F G H I J"
chunkSize = 3 words
chunkOverlap = 1 word

Step 1: Build first chunk
current = ["A", "B", "C"]
length = 3
→ Finalize: "A B C"

Step 2: Maintain overlap
Remove from beginning until length ≤ 1:
  Remove "A": current = ["B", "C"], length = 2
  Remove "B": current = ["C"], length = 1 ✓
  
Step 3: Build next chunk starting with overlap
current = ["C"]  ← This is the overlap!
Add "D": current = ["C", "D"], length = 2
Add "E": current = ["C", "D", "E"], length = 3
→ Finalize: "C D E"

Result:
Chunk 1: "A B C"
Chunk 2:     "C D E"  ← "C" is shared
           ↑ overlap
```

**Почему это работает:**
1. Сохраняются последние N элементов из предыдущего чанка
2. N определяется настройкой `chunkOverlap`
3. Обеспечивается непрерывность контекста
4. Предотвращается потеря информации на границах

---

### Сложность рекурсивного разбиения

**Анализ временной сложности:**

```
Best Case: O(n)
- Text splits cleanly with first separator
- No recursion needed

Worst Case: O(n * m)
- n = text length
- m = number of separators
- Must try all separators for each large chunk

Average Case: O(n * log m)
- Usually splits well with first 2-3 separators
```

**Пространственная сложность:**
```
O(n) - stores all chunks in memory
```

**Возможности оптимизации:**
1. Кэширование проверок разделителей
2. Ограничение глубины рекурсии
3. Потоковая обработка крупных документов
4. Параллельная обработка нескольких документов

---

## Разбор примеров

### Пример 1: Простое символьное разбиение

```javascript
async function example1() {
    const url = 'https://arxiv.org/pdf/2402.19473';
    const documents = await OutputHelper.withSpinner(
        'Loading PDF...', 
        () => extractTextFromPDF(url, {splitPages: true})
    );

    OutputHelper.log.info('Using CharacterTextSplitter (500/50)');
    const splitter = new CharacterTextSplitter({
        chunkSize: 500, 
        chunkOverlap: 50
    });

    const chunks = await OutputHelper.withSpinner(
        'Splitting text...', 
        () => splitter.splitDocuments(documents)
    );

    OutputHelper.formatStats({
        Pages: documents.length,
        Chunks: chunks.length,
        AvgPerPage: (chunks.length / documents.length).toFixed(1),
        Splitter: 'CharacterTextSplitter'
    });
    
    chunks.slice(0, 3).forEach(OutputHelper.formatChunkPreview);
}
```

**Что делает этот код:**

1. **Загрузка PDF**: Загружает и парсит PDF, разбивая по страницам
2. **Создание сплиттера**: CharacterTextSplitter с чанками по 500 символов и перекрытием 50 символов
3. **Разбиение документов**: Каждая страница → несколько чанков
4. **Отображение статистики**: Показывает количество чанков и средние значения
5. **Предпросмотр чанков**: Показывает первые 3 чанка

**Пример вывода:**
```
Loading PDF... ✓ Done
ℹ Using CharacterTextSplitter (500/50)
Splitting text... ✓ Done

Statistics:
  Pages: 22
  Chunks: 156
  AvgPerPage: 7.1
  Splitter: CharacterTextSplitter

[Chunk 1]
Chunk 1/7 | Page 1 | 487 chars
Retrieval-Augmented Generation for AI-Generated Content...
```

**Почему 156 чанков?**
```
22 pages × ~7 chunks/page = 154 chunks
(varies based on page content density)
```

---

### Пример 2: Рекурсивное символьное разбиение

```javascript
async function example2() {
    const docs = await OutputHelper.withSpinner(
        'Loading PDF...', 
        () => extractTextFromPDF(url, {splitPages: true})
    );

    const splitter = new RecursiveCharacterTextSplitter({
        chunkSize: 1000, 
        chunkOverlap: 200
    });

    const chunks = await splitter.splitDocuments(docs);
    const {avg, min, max, median} = OutputHelper.analyzeChunks(chunks);

    OutputHelper.formatStats({
        'Total Chunks': chunks.length,
        'Average Size': `${avg} chars`,
        'Min Size': `${min} chars`,
        'Max Size': `${max} chars`,
        'Median Size': `${median} chars`,
        'Chunk Size Limit': '1000',
        'Overlap': '200'
    });
}
```

**Ключевое отличие от примера 1:**
- Используется **RecursiveCharacterTextSplitter** (более умный)
- Более крупные чанки (1000 вместо 500)
- Более большое перекрытие (200 вместо 50)
- Анализ распределения размеров чанков

**Почему медиана важна:**
```
If chunks = [100, 200, 950, 980, 1000, 1000]
Average = 705 chars
Median = 965 chars

→ Median shows most chunks are near the limit
→ Average affected by a few small chunks
```

---

### Пример 3: Токенное разбиение

```javascript
async function example3() {
    const splitter = new TokenTextSplitter({
        chunkSize: 500,   // 500 TOKENS (not chars)
        chunkOverlap: 50
    });

    const chunks = await splitter.splitDocuments(docs);
    
    const tokens = chunks.map(c => splitter.lengthFunction(c.pageContent));
    const avg = Math.round(tokens.reduce((a, b) => a + b, 0) / tokens.length);
}
```

**Отличие от символьного разбиения:**
```
Character-based: chunkSize = 500 chars
Token-based:     chunkSize = 500 tokens ≈ 2000 chars

Same chunkSize number, different meaning!
```

**Зачем использовать этот подход?**
- Эмбеддинги OpenAI: лимит 8191 токен
- Контекст GPT-4: лимит 128k токенов
- Необходимость вписывания в ограничения по токенам

---

### Пример 4: Фильтрация по метаданным

```javascript
async function example4() {
    const chunks = await splitter.splitDocuments(docs);
    
    // Filter by chunk position
    const firstTen = chunks.filter(c => c.metadata.chunk < 10);
    
    // Simulated retrieval
    const query = 'retrieval augmented generation';
    const relevant = chunks.filter(c => 
        c.pageContent.toLowerCase().includes(query)
    ).slice(0, 3);
}
```

**Что демонстрирует этот пример:**
1. **Метаданные сохраняются** при разбиении
2. **Можно фильтровать по позиции чанка** (полезно для пагинации)
3. **Симулируется поиск по ключевым словам** (реальный RAG использует эмбеддинги)

**Структура метаданных:**
```javascript
{
    source: "https://arxiv.org/pdf/2402.19473",
    pdf: {...},
    loc: {pageNumber: 5},
    chunk: 2,          // This is chunk 2
    totalChunks: 8     // Out of 8 total from this page
}
```

---

### Пример 5: Сравнение стратегий

```javascript
async function example5() {
    const strategies = [
        {name: 'Large (1500/150)', size: 1500, overlap: 150},
        {name: 'Medium (1000/200)', size: 1000, overlap: 200},
        {name: 'Small (500/50)', size: 500, overlap: 50}
    ];

    for (const s of strategies) {
        const splitter = new RecursiveCharacterTextSplitter({
            chunkSize: s.size, 
            chunkOverlap: s.overlap
        });
        const chunks = await splitter.splitDocuments(docs);
        // Compare results...
    }
}
```

**Сравнение результатов:**

| Стратегия | Чанки | Средний размер | Сценарий использования |
|----------|--------|----------|----------|
| Крупные | 45 | 1350 | Максимальный контекст, суммаризация |
| Средние | 67 | 920 | Сбалансированные, универсальные |
| Мелкие | 156 | 480 | Максимальная точность, Q&A |

**Компромиссы:**
```
More Chunks:
  ✓ Better precision
  ✓ More granular retrieval
  ✗ Less context per chunk
  ✗ More compute (embeddings)

Fewer Chunks:
  ✓ More context
  ✓ Better for comprehension
  ✗ Less precise retrieval
  ✗ May exceed limits
```

---

## Лучшие практики

### 1. Выбор размера чанка

```javascript
// Question Answering (need precision)
const splitter = new RecursiveCharacterTextSplitter({
    chunkSize: 500,
    chunkOverlap: 50
});

// Summarization (need context)
const splitter = new RecursiveCharacterTextSplitter({
    chunkSize: 2000,
    chunkOverlap: 400
});

// General Purpose (balanced)
const splitter = new RecursiveCharacterTextSplitter({
    chunkSize: 1000,
    chunkOverlap: 200
});
```

### 2. Выбор размера перекрытия

```javascript
// Rule of thumb: 10-20% of chunk size
chunkSize: 1000 → chunkOverlap: 100-200
chunkSize: 500  → chunkOverlap: 50-100
chunkSize: 2000 → chunkOverlap: 200-400
```

### 3. Тестирование различных стратегий

```javascript
async function testStrategies(docs) {
    const configs = [
        {size: 500, overlap: 50},
        {size: 1000, overlap: 100},
        {size: 1000, overlap: 200},
        {size: 1500, overlap: 150}
    ];
    
    for (const config of configs) {
        const splitter = new RecursiveCharacterTextSplitter(config);
        const chunks = await splitter.splitDocuments(docs);
        
        // Measure retrieval quality
        const quality = await evaluateRetrieval(chunks, queries);
        console.log(`${config.size}/${config.overlap}: ${quality}`);
    }
}
```

### 4. Учёт структуры документа

```javascript
// For markdown documents
const splitter = new RecursiveCharacterTextSplitter({
    separators: [
        '\n## ',      // H2 headers
        '\n### ',     // H3 headers
        '\n\n',       // Paragraphs
        '\n',         // Lines
        '. '          // Sentences
    ]
});

// For code
const splitter = new RecursiveCharacterTextSplitter({
    separators: [
        '\n\nclass ',  // Class definitions
        '\n\ndef ',    // Function definitions
        '\n\n',        // Blank lines
        '\n',          // Lines
        ' '            // Words
    ]
});
```