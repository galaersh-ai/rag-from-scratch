# Создание обёртки для LLM — Разбор кода

Подробное описание того, как создавать и использовать класс-обёртку для больших языковых моделей (LLM), охватывающее как реализацию (`LlamaCpp.js`), так и примеры использования (`example.js`).

## Обзор

В данном примере демонстрируется:
- Как спроектировать абстрактный базовый класс (`BaseLLM`)
- Как реализовать конкретную обёртку (`LlamaCpp`)
- Как использовать обёртку в реальных приложениях
- Лучшие практики интеграции LLM в системы RAG

---

## Часть 1: Реализация (LlamaCpp.js)

### Импорты

```javascript
import {BaseLLM} from './BaseLLM.js';
import {createLlamaContext, createLlamaGrammar, createLlamaModel, createLlamaSession,} from '../utils/llama_cpp.js';
import {getLlama} from 'node-llama-cpp';
```

**Что импортируется:**
- **`BaseLLM`**: Абстрактный базовый класс, определяющий интерфейс, который должны соблюдать все LLM
- **Вспомогательные функции**: Утилиты, инкапсулирующие шаги инициализации node-llama-cpp
- **`getLlama`**: Фабричная функция из node-llama-cpp для получения экземпляра Llama

**Зачем нужны вспомогательные функции?**
- Инкапсулируют сложную логику инициализации
- Переиспользуются в разных обёртках для LLM
- Упрощают тестирование и сопровождение

---

### Определение класса и конструктор

```javascript
export class LlamaCpp extends BaseLLM {
    constructor(inputs) {
        super(inputs);
        this.maxTokens = inputs?.maxTokens;
        this.temperature = inputs?.temperature;
        this.topK = inputs?.topK;
        this.topP = inputs?.topP;
        this.trimWhitespaceSuffix = inputs?.trimWhitespaceSuffix;

        // Private instances (initialized in initialize())
        this._model = null;
        this._context = null;
        this._session = null;
        this._grammar = null;
    }
}
```

**Что делает конструктор:**
1. **Наследует BaseLLM**: Получает общую функциональность и обеспечивает единый интерфейс
2. **Сохраняет конфигурацию**: Запоминает параметры генерации (температура, токены и т.д.)
3. **Инициализирует приватные поля**: Устанавливает заглушки для объектов node-llama-cpp

**Разбор ключевых параметров:**
- **`maxTokens`**: Максимальное количество токенов для генерации (управляет длиной ответа)
- **`temperature`**: Случайность при генерации (0.0 = детерминированно, 1.0+ = творчески)
- **`topK`**: Ограничивает сэмплирование K наиболее вероятными токенами
- **`topP`**: Ядерное сэмплирование (nucleus sampling) — учитывает токены с кумулятивной вероятностью до P
- **`trimWhitespaceSuffix`**: Удалять ли пробелы в конце ответов

**Почему приватные поля (`_model`, `_context` и т.д.)?**
- Инкапсуляция: Пользователям не нужно знать об этих внутренних деталях
- Безопасность: Предотвращает случайное изменение критических объектов
- Читаемость: Префикс с подчёркиванием сигнализирует о «внутренней детали реализации»

**Почему инициализация значениями `null`?**
- Конструктор в JavaScript не может быть асинхронным
- Фактическая инициализация происходит в статическом методе `initialize()`
- Этот паттерн гарантирует, что у нас никогда не будет частично инициализированного экземпляра

---

### Статический фабричный метод

```javascript
static async initialize(inputs) {
    const instance = new LlamaCpp(inputs);
    const llama = await getLlama();

    // Create model, context, and session using helper functions
    instance._model = await createLlamaModel(inputs, llama);
    instance._context = await createLlamaContext(instance._model, inputs);
    instance._session = await createLlamaSession(instance._context);

    // Optional: Load grammar
    if (inputs.gbnf) {
        instance._grammar = await createLlamaGrammar(inputs.gbnf, llama);
    }

    return instance;
}
```

**Фабричный паттерн в действии:**

**Проблема:** Конструкторы в JavaScript не могут быть асинхронными, но загрузка модели требует асинхронных операций.

**Решение:** Статический фабричный метод, который:
1. Создаёт экземпляр (синхронно)
2. Выполняет асинхронную инициализацию
3. Возвращает полностью готовый экземпляр

**Пошагово:**

**Шаг 1: Создание экземпляра**
```javascript
const instance = new LlamaCpp(inputs);
```
Создаёт объект с сохранённой конфигурацией, но ещё не инициализированный.

**Шаг 2: Получение экземпляра Llama**
```javascript
const llama = await getLlama();
```
Получает основной экземпляр Llama, управляющий нативными бинарными файлами.

**Шаг 3: Загрузка модели**
```javascript
instance._model = await createLlamaModel(inputs, llama);
```
Загружает файл модели GGUF в память. Это самый объёмный и медленный шаг инициализации.

**Ожидаемые входные данные:**
```javascript
{
    modelPath: "./models/llama-3.1-8b-q4_0.gguf",  // Required
    gpuLayers: 35,  // Optional - for GPU acceleration
}
```

**Шаг 4: Создание контекста**
```javascript
instance._context = await createLlamaContext(instance._model, inputs);
```
Создаёт контекстное окно («рабочую память» модели).

**Ожидаемые входные данные:**
```javascript
{
    contextSize: 2048,  // Optional - defaults to 2048 tokens
}
```

**Шаг 5: Создание сессии**
```javascript
instance._session = await createLlamaSession(instance._context);
```
Создаёт сессию, управляющую состоянием разговора в рамках контекста.

**Шаг 6: Опциональная загрузка грамматики**
```javascript
if (inputs.gbnf) {
    instance._grammar = await createLlamaGrammar(inputs.gbnf, llama);
}
```

**Что такое GBNF?**
- Формат грамматики, ограничивающий вывод модели
- Гарантирует, что вывод следует определённой структуре (например, JSON, списки)
- Полезен для извлечения структурированных данных

**Пример GBNF:**
```javascript
const gbnf = `
root ::= answer
answer ::= "yes" | "no"
`;
// Model can only output "yes" or "no"
```

**Почему этот паттерн лучше:**

❌ **Плохо (ручная инициализация):**
```javascript
const model = new LlamaCpp({ modelPath: "..." });
await model.initialize();  // Easy to forget!
const response = await model.invoke("Hello");
```

✅ **Хорошо (фабричный метод):**
```javascript
const model = await LlamaCpp.initialize({ modelPath: "..." });
const response = await model.invoke("Hello");  // Always ready!
```

---

### Метод invoke()

```javascript
async invoke(prompt, options = {}) {
    try {
        const promptOptions = {
            maxTokens: options.maxTokens ?? this.maxTokens,
            temperature: this.temperature,
            topK: this.topK,
            topP: this.topP,
            trimWhitespaceSuffix: this.trimWhitespaceSuffix,
            grammar: this._grammar,
            onToken: options.onToken,
        };

        return await this._session.prompt(prompt, promptOptions);
    } catch (error) {
        throw new Error(`LlamaCpp invoke failed: ${error.message}`);
    }
}
```

**Что делает:** Генерирует полный ответ для заданного промпта.

**Пошагово:**

**Шаг 1: Объединение опций**
```javascript
const promptOptions = {
    maxTokens: options.maxTokens ?? this.maxTokens,
    // ... other parameters
};
```

**Приоритет:**
1. Сначала проверяется `options.maxTokens` (переопределение для конкретного вызова)
2. Если не передано (`undefined`), используется `this.maxTokens` (значение по умолчанию экземпляра)
3. Оператор `??` — оператор объединения с null (возвращает правое значение только если левое — `null` или `undefined`)

**Почему это удобно:**
```javascript
// Uses instance defaults
await model.invoke("Hello");

// Overrides maxTokens for this call only
await model.invoke("Write a story", { maxTokens: 500 });
```

**Шаг 2: Вызов сессии**
```javascript
return await this._session.prompt(prompt, promptOptions);
```
Делегирует node-llama-cpp сессии фактическую генерацию.

**Шаг 3: Обработка ошибок**
```javascript
catch (error) {
    throw new Error(`LlamaCpp invoke failed: ${error.message}`);
}
```
Оборачивает ошибки низкого уровня информацией о том, какой метод завершился с ошибкой.

**Возвращаемое значение:** Строка, содержащая полный сгенерированный ответ.

**Пример использования:**
```javascript
const response = await model.invoke("What is 2+2?");
console.log(response);  // "4" or "2+2 equals 4" depending on model
```

---

### Метод stream()

```javascript
async *stream(prompt, options = {}) {
    const chunks = [];

    await this.invoke(prompt, {
        ...options,
        onToken: (chunk) => {
            chunks.push(chunk);
        },
    });

    for (const chunk of chunks) {
        yield chunk;
    }
}
```

**Что делает:** Возвращает асинхронный генератор, который отдаёт токены по мере их генерации.

**Синтаксис `async *`:**
- `async`: Эта функция возвращает Promise
- `*`: Это функция-генератор
- `async *`: Асинхронный генератор — отдаёт значения по мере поступления

**Как это работает:**

**Шаг 1: Сбор фрагментов**
```javascript
const chunks = [];

await this.invoke(prompt, {
    ...options,
    onToken: (chunk) => {
        chunks.push(chunk);
    },
});
```

- Вызывает `invoke()` с колбэком, собирающим каждый токен
- Колбэк `onToken` вызывается node-llama-cpp при генерации каждого токена
- Все фрагменты сохраняются в массиве

**Шаг 2: Отдача фрагментов**
```javascript
for (const chunk of chunks) {
    yield chunk;
}
```
Отдаёт каждый фрагмент вызывающему коду по одному.

**Пример использования:**
```javascript
for await (const token of model.stream("Count to 5")) {
    process.stdout.write(token);  // Prints tokens as they arrive
}
```

**Вывод:**
```
1
,
2
,
3
,
4
,
5
```
(Каждый токен печатается сразу по мере генерации)

**Почему именно такая реализация?**

Это простая реализация, которая:
- ✅ Переиспользует логику `invoke()`
- ✅ Работает с API колбэков node-llama-cpp
- ✅ Предоставляет потоковый интерфейс пользователям

**Более продвинутая реализация могла бы:**
- Отправлять фрагменты в реальном времени (без буферизации)
- Использовать асинхронную итерацию напрямую
- Предоставлять обновления прогресса

---

### Вспомогательные методы

#### getModelType()

```javascript
getModelType() {
    return 'llama_cpp';
}
```

**Что делает:** Возвращает строковый идентификатор для этого типа LLM.

**Почему это полезно:**
```javascript
// Can identify which LLM you're using
console.log(model.getModelType());  // "llama_cpp"

// Useful for logging and debugging
function logQuery(model, query, response) {
    console.log(`[${model.getModelType()}] ${query} -> ${response}`);
}
```

#### dispose()

```javascript
async dispose() {
    if (this._context) {
        await this._context.dispose();
    }
}
```

**Что делает:** Освобождает ресурсы, используемые моделью.

**Почему это важно:**
- LLM потребляют значительный объём памяти (гигабайты для крупных моделей)
- Контекст хранит токенизированные данные в памяти
- Отсутствие очистки может привести к утечкам памяти

**Лучшая практика использования:**
```javascript
const model = await LlamaCpp.initialize({ modelPath: "..." });
try {
    const response = await model.invoke("Hello");
    console.log(response);
} finally {
    await model.dispose();  // Always clean up!
}
```

**Что освобождается:**
- Контекст: Освобождает память, использованную для контекстного окна
- (Модель и сессия привязаны к контексту, поэтому они также очищаются)

---

## Часть 2: Примеры использования (example.js)

### Установка и импорты

```javascript
import {LlamaCpp} from "../../../src/llms/index.js";
```

Импортирует построенную нами обёртку. Обратите внимание, что путь поднимается до `/src/llms/`, где находится наша реализация.

### Пример 1: Базовое использование

```javascript
const model = await LlamaCpp.initialize({
    modelPath: process.env.MODEL_PATH || './models/llama-3.1-8b-q4_0.gguf',
    maxTokens: 100,
    temperature: 0.7,
});

const response1 = await model.invoke(
    'What are the three primary colors?'
);
console.log('Q: What are the three primary colors?');
console.log('A:', response1);
```

**Что происходит:**

**Шаг 1: Инициализация с конфигурацией**
```javascript
modelPath: process.env.MODEL_PATH || './models/llama-3.1-8b-q4_0.gguf',
```
- Сначала пытается использовать переменную окружения `MODEL_PATH`
- Если она не задана, использует путь по умолчанию
- Хорошая практика: Делает код конфигурируемым без его изменения

**Шаг 2: Установка значений по умолчанию**
```javascript
maxTokens: 100,
temperature: 0.7,
```
- `maxTokens: 100`: Относительно короткие ответы (подходит для фактических вопросов и ответов)
- `temperature: 0.7`: Умеренная творческость (сбалансированный вариант)

**Шаг 3: Простой вызов**
```javascript
const response1 = await model.invoke('What are the three primary colors?');
```
Один вызов метода — вся сложность скрыта!

**Ожидаемый вывод:**
```
Q: What are the three primary colors?
A: The three primary colors are red, blue, and yellow.
```

---

### Пример 2: Потоковая передача токенов

```javascript
console.log('Q: Count from 1 to 5.');
console.log('A: ');

for await (const chunk of model.stream('Count from 1 to 5.')) {
    process.stdout.write(chunk);
}
```

**Что происходит:**

**Синтаксис `for await...of`:**
```javascript
for await (const chunk of model.stream(...)) {
    // Process each chunk as it arrives
}
```
- Ожидает каждый фрагмент от асинхронного генератора
- Обрабатывает токены в реальном времени
- Идеально подходит для обновления пользовательского интерфейса

**Запись в stdout:**
```javascript
process.stdout.write(chunk);
```
- Использует `write()` вместо `console.log()` для избежания перевода строки
- Печатает токены непрерывно по мере их поступления
- Создаёт эффект печатной машинки

**Ожидаемый вывод:**
```
Q: Count from 1 to 5.
A: 1, 2, 3, 4, 5
```
(Но каждое число/запятая появляется постепенно, а не все сразу)

**Когда использовать потоковую передачу:**
- ✅ Интерфейсы чат-ботов (пользователи видят формирующийся ответ)
- ✅ Генерация длинного контента (показывает прогресс)
- ✅ Интерактивные приложения (мгновенная обратная связь)
- ❌ Пакетная обработка (накладные расходы не оправданы)
- ❌ API-ответы, где нужен готовый текст

---

### Пример 3: Пакетная обработка

```javascript
const questions = [
    'What is 2+2?',
    'What color is the sky?',
    'What is the capital of France?',
];

console.log('Processing multiple questions...\n');
const responses = await model.batch(questions);

questions.forEach((q, i) => {
    console.log(`Q${i + 1}: ${q}`);
    console.log(`A${i + 1}: ${responses[i]}\n`);
});
```

**Что происходит:**

**Метод batch():**
```javascript
const responses = await model.batch(questions);
```
- Унаследован от `BaseLLM`
- Обрабатывает все вопросы и возвращает массив ответов
- Порядок такой же, как у входных данных

**Реализация по умолчанию (в BaseLLM):**
```javascript
async batch(prompts) {
    return Promise.all(prompts.map(p => this.invoke(p)));
}
```
- Использует `Promise.all()` для параллельного выполнения промптов
- Ожидает завершения всех запросов
- Возвращает массив результатов

**Формат вывода:**
```
Processing multiple questions...

Q1: What is 2+2?
A1: 4

Q2: What color is the sky?
A2: Blue

Q3: What is the capital of France?
A3: Paris
```

**Когда использовать batch():**
- ✅ Обработка нескольких документов
- ✅ Запуск оценок/бенчмарков
- ✅ Массовые задачи вопросов и ответов
- ⚠️ Осторожно с очень большими пакетами (использование памяти)

---

### Пример 4: Пользовательские опции для каждого запроса

```javascript
const response2 = await model.invoke(
    'Write a creative story opening.',
    {
        maxTokens: 150,
        temperature: 0.9,  // More creative
    }
);

console.log('Q: Write a creative story opening.');
console.log('A:', response2);
```

**Что происходит:**

**Переопределение значений по умолчанию:**
```javascript
{
    maxTokens: 150,      // Override: longer response (default was 100)
    temperature: 0.9,    // Override: more creative (default was 0.7)
}
```

**Почему стоит переопределять для каждого запроса:**
- Разные задачи требуют разных параметров
- Фактические вопросы и ответы: Низкая температура (0.1–0.5)
- Творческое письмо: Высокая температура (0.8–1.2)
- Короткие ответы: Низкое maxTokens (50–100)
- Длинные тексты: Высокое maxTokens (500+)

**Ожидаемый вывод:**
```
Q: Write a creative story opening.
A: The old lighthouse stood defiant against the storm, its beam cutting 
through the darkness like a knife through butter. Sarah had always been 
drawn to this place, though she couldn't quite explain why...
```
(Более творческий и длинный, чем предыдущие ответы)

---

### Очистка ресурсов

```javascript
await model.dispose();
```

**Ключевой завершающий шаг:**
- Освобождает память, используемую моделью
- Предотвращает утечки памяти
- Всегда должен находиться в блоке `finally` или в конце программы

**Лучший паттерн:**
```javascript
const model = await LlamaCpp.initialize({ ... });
try {
    // Use model
} finally {
    await model.dispose();  // Always runs, even if errors occur
}
```

---

## Часть 3: Интерфейс BaseLLM

Хотя он не показан в файлах, `BaseLLM` выглядел бы примерно так:

```javascript
export class BaseLLM {
    constructor(inputs) {
        // Store common configuration
    }

    // Abstract method - must be implemented by subclasses
    async invoke(prompt, options = {}) {
        throw new Error('Subclass must implement invoke()');
    }

    // Abstract method - must be implemented by subclasses
    async *stream(prompt, options = {}) {
        throw new Error('Subclass must implement stream()');
    }

    // Concrete method - works for all subclasses
    async batch(prompts) {
        return Promise.all(prompts.map(p => this.invoke(p)));
    }

    // Abstract method - must be implemented by subclasses
    getModelType() {
        throw new Error('Subclass must implement getModelType()');
    }

    // Optional - subclasses can override if needed
    async dispose() {
        // Default: no-op
    }
}
```

**Ключевые моменты:**

**Абстрактные методы:**
- Определены, но выбрасывают ошибки
- Должны быть реализованы подклассами
- Гарантируют, что все LLM имеют одинаковый интерфейс

**Конкретные методы:**
- Реализованы в базовом классе
- Переиспользуются всеми подклассами
- Могут быть переопределены при необходимости

**Преимущества:**
- ✅ Единообразие: Все LLM работают одинаково
- ✅ Переиспользование: Общая логика в одном месте
- ✅ Гибкость: Легко добавлять новых поставщиков LLM
- ✅ Типизация: IDE знает, какие методы существуют

---

## Сводка ключевых концепций

### 1. Фабричный паттерн

**Проблема:** В JavaScript нельзя иметь асинхронные конструкторы.

**Решение:** Статический фабричный метод
```javascript
static async initialize(inputs) {
    const instance = new ClassName(inputs);
    await instance.doAsyncSetup();
    return instance;
}
```

### 2. Паттерн «Шаблонный метод»

**Паттерн:** Базовый класс определяет структуру, подклассы заполняют деталями

```javascript
// Base class
class BaseLLM {
    async batch(prompts) {
        return Promise.all(prompts.map(p => this.invoke(p)));
    }
    // Uses this.invoke() which subclasses implement
}

// Subclass
class LlamaCpp extends BaseLLM {
    async invoke(prompt) {
        // Specific implementation
    }
}
```

### 3. Инъекция зависимостей

**Паттерн:** Передавайте зависимости вместо жёсткой кодировки

```javascript
// Good: Flexible
async function processQuestions(questions, model) {
    return model.batch(questions);
}

// Can use any model:
processQuestions(questions, llamaModel);
processQuestions(questions, openaiModel);
```

### 4. Управление ресурсами

**Паттерн:** Всегда очищайте ресурсы

```javascript
const model = await LlamaCpp.initialize({ ... });
try {
    // Use model
} finally {
    await model.dispose();  // Guaranteed to run
}
```

---

## Справочник параметров конфигурации

### Параметры инициализации

```javascript
await LlamaCpp.initialize({
    // Required
    modelPath: string,        // Path to GGUF model file
    
    // Optional - Model loading
    gpuLayers: number,        // Number of layers to offload to GPU
    
    // Optional - Context
    contextSize: number,      // Context window size (default: 2048)
    
    // Optional - Generation defaults
    maxTokens: number,        // Max tokens to generate
    temperature: number,      // Sampling temperature (0.0-2.0)
    topK: number,            // Top-K sampling
    topP: number,            // Top-P (nucleus) sampling
    trimWhitespaceSuffix: boolean,  // Trim trailing whitespace
    
    // Optional - Structured output
    gbnf: string,            // GBNF grammar string
});
```

### Параметры invoke()

```javascript
await model.invoke(prompt, {
    maxTokens: number,       // Override instance default
    onToken: (chunk) => {},  // Callback for each token (for streaming)
});
```

---

## Итоги

### Что мы создали

1. **Класс-обёртку LlamaCpp**
   - Наследует BaseLLM для обеспечения единообразия
   - Использует фабричный паттерн для инициализации
   - Предоставляет методы `invoke()`, `stream()` и `batch()`
   - Обеспечивает очистку ресурсов

2. **Примеры использования**
   - Базовый вызов
   - Потоковая передача токенов
   - Пакетная обработка
   - Пользовательские опции

### Ключевые выводы

- ✅ Обёртки упрощают работу со сложными библиотеками
- ✅ Фабричный паттерн решает проблему асинхронных конструкторов
- ✅ Абстрактный базовый класс обеспечивает единообразие
- ✅ Управление ресурсами предотвращает утечки памяти
- ✅ Объекты конфигурации гибки и понятны

### Следующие шаги

С этой обёрткой вы можете:
- Строить пайплайны RAG
- Легко переключаться между поставщиками LLM
- Эффективно тестировать свой код

Обёртка — это основа для всех операций с LLM в вашем приложении.
