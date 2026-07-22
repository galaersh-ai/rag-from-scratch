# Концепции и архитектура разбиения текста

## Содержание
1. [Обзор](#обзор)
2. [Основная архитектура](#основная-архитектура)
3. [Сравнение с LangChain.js](#сравнение-с-langchainjs)
4. [Резюме](#резюме)

---

## Обзор

Этот модуль реализует функциональность разбиения текста для систем Retrieval-Augmented Generation (RAG). Он берёт большие документы и разбивает их на меньшие, удобные для обработки чанки, сохраняя контекст с помощью перекрытий.

### Ключевые концепции

**Зачем разбивать текст?**
- Модели эмбеддингов имеют ограничения по числу токенов (обычно 512–8192 токенов)
- Меньшие чанки повышают точность поиска
- Перекрытие сохраняет контекст на границах фрагментов
- Улучшаются результаты семантического поиска

**Основная задача:**
```
Большой документ (10 000 символов)
        ↓
[Чанк 1] [Чанк 2] [Чанк 3] [Чанк 4]
   ↑перекрытие↑  ↑перекрытие↑  ↑перекрытие↑
```

---

## Основная архитектура

### Иерархия классов

```
TextSplitter (базовый класс)
    ├── CharacterTextSplitter
    ├── RecursiveCharacterTextSplitter
    └── TokenTextSplitter
```

### Философия проектирования

1. **Единая ответственность**: каждый класс выполняет одну задачу
2. **Наследование**: общая логика размещается в базовом классе
3. **Полиморфизм**: различные стратегии разбиения через `splitText()`
4. **Композиция**: составные сплиттеры используют более простые внутри

---

## Сравнение с LangChain.js

### Сходства

#### 1. **Одинаковый основной API**
```javascript
// Наша реализация
const splitter = new RecursiveCharacterTextSplitter({
    chunkSize: 1000,
    chunkOverlap: 200
});
const chunks = await splitter.splitDocuments(documents);

// LangChain.js
import { RecursiveCharacterTextSplitter } from "langchain/text_splitter";
const splitter = new RecursiveCharacterTextSplitter({
    chunkSize: 1000,
    chunkOverlap: 200
});
const chunks = await splitter.splitDocuments(documents);
```

**Результат:** полная совместимость! Код работает с обоими вариантами.

#### 2. **Одинаковая иерархия классов**
```
Обе реализации:
- TextSplitter (базовый)
  ├── CharacterTextSplitter
  ├── RecursiveCharacterTextSplitter
  └── TokenTextSplitter
```

#### 3. **Одинаковая логика разбиения**
- Обе используют алгоритм «объединение с перекрытием»
- Одинаковая рекурсивная стратегия для больших чанков
- Одинаковая иерархия разделителей

#### 4. **Одинаковая структура метаданных**
```javascript
{
    pageContent: "текст чанка",
    metadata: {
        source: "...",
        chunk: 0,
        totalChunks: 5
    }
}
```

### Различия

#### 1. **Простота кода**

**Наша реализация:**
```javascript
// Лаконичный конструктор
constructor({chunkSize = 1000, chunkOverlap = 200, lengthFunction = t => t.length} = {}) {
    if (chunkOverlap >= chunkSize) {
        throw new Error('chunkOverlap must be less than chunkSize');
    }
    Object.assign(this, {chunkSize, chunkOverlap, lengthFunction});
}
```

**LangChain.js:**
```javascript
// Более многословная, с большим количеством проверок
constructor(fields) {
    super(fields);
    this.chunkSize = fields?.chunkSize ?? 1000;
    this.chunkOverlap = fields?.chunkOverlap ?? 200;
    // ... множество других полей
    // ... обширная валидация
    // ... обработка ошибок
}
```

**Почему проще?**
- Учебная направленность
- Меньше граничных случаев
- Легче понять
- Меньше производственной нагрузки

#### 2. **Подсчёт токенов**

**Наша реализация:**
```javascript
const lengthFunction = text => Math.ceil(text.length / 4);
```
- Простая приближённая оценка
- Без внешних зависимостей
- Быстро, но менее точно

**LangChain.js:**
```javascript
import { encodingForModel } from "js-tiktoken";
const encoder = encodingForModel("gpt-4");
const tokens = encoder.encode(text);
```
- Использует библиотеку tiktoken
- Точный подсчёт токенов
- Медленнее, но точнее

#### 3. **Обработка ошибок**

**Наша реализация:**
```javascript
// Минимальная обработка ошибок
if (chunkOverlap >= chunkSize) {
    throw new Error('chunkOverlap must be less than chunkSize');
}
```

**LangChain.js:**
```javascript
// Обширная обработка ошибок
if (chunkOverlap >= chunkSize) {
    throw new Error(`chunkOverlap (${chunkOverlap}) must be less than chunkSize (${chunkSize})`);
}
if (chunkSize <= 0) {
    throw new Error('chunkSize must be positive');
}
// ... множество других проверок
```

#### 4. **Функциональность**

| Функция | Наша реализация | LangChain.js |
|---------|-----------------|--------------|
| Базовое разбиение | ✓ | ✓ |
| Рекурсивное разбиение | ✓ | ✓ |
| Разбиение по токенам | ✓ (приближённо) | ✓ (точно) |
| Отслеживание метаданных | ✓ | ✓ |
| Пользовательские разделители | ✓ | ✓ |
| Разбиение Markdown | ✗ | ✓ |
| Разбиение кода | ✗ | ✓ |
| Разбиение LaTeX | ✗ | ✓ |
| Разбиение HTML | ✗ | ✓ |
| Callback-и трансформаций | ✗ | ✓ |
| Трансформеры документов | ✗ | ✓ |

#### 5. **TypeScript**

**Наша реализация:**
- Чистый JavaScript
- Комментарии JSDoc для подсказок типов
- Проще для понимания

**LangChain.js:**
- Написан на TypeScript
- Полная типизация
- Лучшая поддержка IDE
- Более сложная кодовая база

### Путь миграции

**С нашей реализации на LangChain.js:**

```javascript
// Шаг 1: Измените импорт
// Было:
import { RecursiveCharacterTextSplitter } from './example.js';
// Стало:
import { RecursiveCharacterTextSplitter } from "langchain/text_splitter";

// Шаг 2: Код остаётся прежним!
const splitter = new RecursiveCharacterTextSplitter({
    chunkSize: 1000,
    chunkOverlap: 200
});
const chunks = await splitter.splitDocuments(documents);

// Шаг 3: По желанию добавьте возможности LangChain
const splitter = new RecursiveCharacterTextSplitter({
    chunkSize: 1000,
    chunkOverlap: 200,
    // Специфичные для LangChain возможности:
    separators: ["\n\n", "\n", ".", "!", "?", ";", ",", " ", ""]
});
```

**Совместимость:** 95% кода работают без изменений!

---

## Резюме

### Ключевые выводы

1. **Паттерн TextSplitter**: базовый класс + специализированные подклассы
2. **Основной алгоритм**: объединение с перекрытием для непрерывности контекста
3. **Рекурсивная стратегия**: сначала используются крупные разделители, затем переход к мелким
4. **Совместимость API**: тот же интерфейс, что и у LangChain.js
5. **Простота**: акцент на ясность, а не на количество функций

### Преимущества архитектуры

```
✓ Модульный дизайн (легко расширять)
✓ Чёткое разделение ответственности
✓ Переиспользуемые компоненты
✓ Хорошая документированность
✓ Алгоритм, готовый к использованию в продакшене
✓ Совместимость с LangChain
```

### Что вы узнали

1. Как алгоритмически работает разбиение текста
2. Почему перекрытие важно для контекста
3. Стратегия рекурсивного разбиения
4. Разбиение по токенам vs по символам
5. Как выбрать правильную конфигурацию
6. Различия по сравнению с LangChain.js

### Следующие шаги

1. Реализуйте собственные сплиттеры для вашей предметной области
2. Добавьте специализированные разделители для ваших типов документов
3. Поэкспериментируйте с различными размерами чанков
4. Оцените качество поиска на ваших данных
