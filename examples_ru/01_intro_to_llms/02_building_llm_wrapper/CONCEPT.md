# Создание обёртки для LLM — концептуальный обзор

## Что такое обёртка для LLM?

**Обёртка для LLM** — это класс, который предоставляет упрощённый, единообразный интерфейс для взаимодействия с большими языковыми моделями.

Представьте её как **переводчика и упорядочивателя** между вашим приложением и сложной нижележащей библиотекой LLM.

### Какую проблему она решает

**Без обёртки:**
```javascript
// Direct use of node-llama-cpp - verbose and repetitive
const llama = await getLlama();
const model = await llama.loadModel({ modelPath: "..." });
const context = await model.createContext({ contextSize: 2048 });
const session = new LlamaChatSession({ contextSequence: context.getSequence() });
const response = await session.prompt("Hello", { temperature: 0.7 });
// ... repeat this setup everywhere you need the model
```

**С обёрткой:**
```javascript
// Clean, simple interface
const model = await LlamaCpp.initialize({ modelPath: "..." });
const response = await model.invoke("Hello");
```

---

## Зачем создавать обёртку?

### 1. **Упрощение**

**До:** вам нужно понимать и управлять множеством концепций node-llama-cpp:
- Экземпляр Llama
- Загрузка модели
- Создание контекста
- Управление сессиями
- Парсинг грамматики

**После:** один класс берёт на себя всю сложность:
```javascript
const model = await LlamaCpp.initialize({ modelPath: "..." });
// Everything is ready to use!
```

### 2. **Единообразие**

Если вы создаёте RAG-систему, вы будете использовать LLM во многих местах:
- Резюмирование документов
- Ответы на вопросы
- Переранжирование результатов
- Расширение запросов

Без обёртки каждое место повторяет один и тот же код инициализации. С обёрткой у вас есть **единый единообразный интерфейс** повсюду.

### 3. **Гибкость**

Сегодня вы можете использовать `node-llama-cpp` (локальная модель). Завтра вы можете захотеть:
- Перейти на API OpenAI
- Использовать Claude от Anthropic
- Попробовать другую библиотеку для локальных моделей

**При грамотном проекте обёртки:**
```javascript
// All implementations share the same interface
const model1 = await LlamaCpp.initialize({ ... });
const model2 = await OpenAI.initialize({ apiKey: *** });
const model3 = await Claude.initialize({ apiKey: *** });

// They all work the same way:
await model1.invoke("Hello");
await model2.invoke("Hello");
await model3.invoke("Hello");
```

Измените одну строку кода — и полностью смените провайдера!

### 4. **Тестируемость**

**Без обёртки:**
```javascript
// Hard to test - tightly coupled to node-llama-cpp
async function answerQuestion(question) {
    const llama = await getLlama();
    const model = await llama.loadModel({ ... });
    // ... lots of setup
    return await session.prompt(question);
}
```

**С обёрткой:**
```javascript
// Easy to test - can mock the wrapper
async function answerQuestion(question, model) {
    return await model.invoke(question);
}

// In tests, use a mock:
const mockModel = { invoke: async (q) => "test response" };
```

---

## Ключевые паттерны проектирования

### Паттерн 1: Абстрактный базовый класс

```javascript
class BaseLLM {
    async invoke(prompt, options) {
        throw new Error("Must implement invoke()");
    }
    
    async batch(prompts) {
        // Default implementation works for all subclasses
        return Promise.all(prompts.map(p => this.invoke(p)));
    }
}
```

**Что делает:**
- Определяет интерфейс, который все LLM должны реализовать
- Предоставляет общую функциональность (такую как `batch()`)
- Обеспечивает единообразие среди различных реализаций LLM

**Преимущества:**
- Любой код, работающий с одной LLM, работает со всеми
- Добавление новых провайдеров LLM без изменения существующего кода
- Проверка типов и автодополнение в IDE

### Паттерн 2: Фабричный метод

```javascript
// Instead of:
const model = new LlamaCpp({ modelPath: "..." });
await model.initialize();  // Easy to forget!

// Use:
const model = await LlamaCpp.initialize({ modelPath: "..." });
// Already initialized and ready to use
```

**Что делает:**
- Обрабатывает асинхронную инициализацию в конструкторе
- Возвращает полностью готовый экземпляр
- Предотвращает использование неинициализированных объектов

**Почему это важно:**
- Конструкторы JavaScript не могут быть асинхронными
- Загрузка модели требует асинхронных операций
- Фабричный метод обеспечивает корректную инициализацию

### Паттерн 3: Объекты конфигурации

```javascript
const model = await LlamaCpp.initialize({
    modelPath: "./model.gguf",
    contextSize: 2048,
    maxTokens: 500,
    temperature: 0.7,
    topK: 40,
    topP: 0.9,
});
```

**Преимущества:**
- Именованные параметры (ясно, что означает каждое значение)
- Необязательные параметры с разумными значениями по умолчанию
- Легко добавлять новые опции без нарушения существующего кода
- Самодокументируемость

### Паттерн 4: Управление ресурсами

```javascript
const model = await LlamaCpp.initialize({ ... });
try {
    // Use the model
    const response = await model.invoke("Hello");
} finally {
    // Always clean up resources
    await model.dispose();
}
```

**Почему это важно:**
- LLM потребляют значительный объём памяти
- Корректная очистка предотвращает утечки памяти
- Освобождает ресурсы GPU при их использовании
- Соответствует лучшим практикам управления ресурсами

---

## Архитектура

### Обзор компонентов

```
┌─────────────────────────────────────────────┐
│           Ваше приложение                   │
│  (RAG-пайплайн, чат-бот и т. д.)           │
└─────────────────┬───────────────────────────┘
                  │
                  │ Простой API
                  │
┌─────────────────▼───────────────────────────┐
│            BaseLLM                          │
│  (Абстрактный интерфейс)                    │
│  - invoke(prompt)                           │
│  - stream(prompt)                           │
│  - batch(prompts)                           │
└─────────────────┬───────────────────────────┘
                  │
                  │ Реализует
                  │
┌─────────────────▼───────────────────────────┐
│           LlamaCpp                          │
│  (Конкретная реализация)                    │
│  - Управляет загрузкой модели               │
│  - Обрабатывает контекст/сессии             │
│  - Предоставляет invoke/stream              │
└─────────────────┬───────────────────────────┘
                  │
                  │ Использует
                  │
┌─────────────────▼───────────────────────────┐
│         node-llama-cpp                      │
│  (Внешняя библиотека)                       │
│  - Низкоуровневые операции LLM              │
└─────────────────────────────────────────────┘
```

### Как это работает

1. **Ваш код** вызывает простой API обёртки
2. **Обёртка LlamaCpp** преобразует вызовы для node-llama-cpp
3. **node-llama-cpp** выполняет фактические операции LLM
4. **Результаты** возвращаются обратно через слои

Каждый слой добавляет ценность:
- **BaseLLM**: обеспечивает единообразие и общие возможности
- **LlamaCpp**: обрабатывает специфику node-llama-cpp
- **Ваш код**: сосредоточен на логике вашего приложения

---

## Практические преимущества

### До: разрозненная сложность

**В retrieval.js:**
```javascript
const llama = await getLlama();
const model = await llama.loadModel({ modelPath: "..." });
// ... setup code ...
```

**В summarization.js:**
```javascript
const llama = await getLlama();
const model = await llama.loadModel({ modelPath: "..." });
// ... same setup code repeated ...
```

**В generation.js:**
```javascript
const llama = await getLlama();
const model = await llama.loadModel({ modelPath: "..." });
// ... same setup code again ...
```

**Проблемы:**
- Дублирование кода
- Легко допускать ошибки
- Сложно изменять конфигурацию
- Трудно проводить тестирование

### После: централизованная обёртка

**Во всех файлах:**
```javascript
import { LlamaCpp } from './llms/LlamaCpp.js';

const model = await LlamaCpp.initialize({ modelPath: "..." });
const response = await model.invoke(prompt);
```

**Преимущества:**
- ✅ Сложный код пишется один раз
- ✅ Простой API используется повсюду
- ✅ Конфигурация изменяется в одном месте
- ✅ Легко тестировать и подменять

---

## Демонстрируемые возможности обёртки

### 1. Базовый вызов

```javascript
const response = await model.invoke("What are the primary colors?");
```

**Что делает:** отправляет промпт и ожидает полный ответ.

**Когда использовать:** простые вопросы и ответы, одноразовые дополнения, пакетная обработка.

### 2. Стриминг

```javascript
for await (const chunk of model.stream("Count from 1 to 5")) {
    process.stdout.write(chunk);
}
```

**Что делает:** выдаёт токены по мере их генерации.

**Когда использовать:**
- Обновление интерфейса в реальном времени
- Длинные ответы, когда пользователь хочет видеть прогресс
- Интерфейсы чат-ботов

### 3. Пакетная обработка

```javascript
const questions = ["Q1?", "Q2?", "Q3?"];
const answers = await model.batch(questions);
```

**Что делает:** эффективно обрабатывает несколько промптов одновременно.

**Когда использовать:**
- Обработка нескольких документов
- Оценка модели на тестовых наборах
- Массовые операции

### 4. Пользовательские параметры

```javascript
await model.invoke("Write a story", {
    maxTokens: 500,      // Longer response
    temperature: 0.9,    // More creative
});
```

**Что делает:** переопределяет настройки по умолчанию для конкретного вызова.

**Когда использовать:** разные задачи требуют различных параметров.

---

## Часто задаваемые вопросы

### В: Почему бы просто не использовать node-llama-cpp напрямую?

**Ответ:** Можно! Но обёртка предоставляет:
- Более простой API
- Лучшую повторяемость
- Упрощённое тестирование
- Гибкость на будущее

Для разового скрипта прямое использование вполне подходит. Для более крупного приложения (такого как RAG-система) обёртка экономит время и снижает количество ошибок.

### В: Добавляет ли обёртка накладные расходы?

**Ответ:** минимальные. Обёртка — это тонкий слой, который:
- Не обрабатывает текст самостоятельно
- Только упорядочивает вызовы API
- Добавляет пренебрежимо малую задержку (<1 мс)

Время фактического инференса LLM (100 мс – 10 с) значительно превышает любые накладные расходы обёртки.

### В: Могу ли я добавить пользовательские методы в обёртку?

**Конечно!** Типичные дополнения:
```javascript
class LlamaCpp extends BaseLLM {
    // Your custom method
    async summarize(text) {
        return this.invoke(`Summarize: ${text}`, { maxTokens: 100 });
    }
    
    // Another helper
    async answerWithContext(question, context) {
        const prompt = `Context: ${context}\n\nQuestion: ${question}`;
        return this.invoke(prompt);
    }
}
```

### В: Как переключиться на другого провайдера LLM?

**Шаг 1:** создайте новую обёртку, реализующую `BaseLLM`:
```javascript
class OpenAI extends BaseLLM {
    static async initialize(inputs) {
        // Initialize OpenAI API
    }
    
    async invoke(prompt, options) {
        // Call OpenAI API
    }
}
```

**Шаг 2:** измените одну строку в вашем коде:
```javascript
// Before:
const model = await LlamaCpp.initialize({ ... });

// After:
const model = await OpenAI.initialize({ ... });
```

Всё остальное работает так же!

---

## Принципы проектирования

### 1. **Разделение ответственности**

- **BaseLLM**: определяет, что все LLM могут делать
- **LlamaCpp**: знает, как использовать node-llama-cpp
- **Ваше приложение**: сосредоточено на бизнес-логике

Каждый класс выполняет одну задачу и делает это хорошо.

### 2. **Не повторяйся (DRY)**

Общая функциональность сосредоточена в одном месте:
- `batch()` в BaseLLM работает для всех реализаций
- Парсинг конфигурации в одном месте
- Централизованная обработка ошибок

### 3. **Принцип открытости/закрытости**

- **Открытость для расширения:** легко добавлять новых провайдеров LLM
- **Закрытость для изменения:** существующий код не изменяется

Добавляйте новые реализации, не ломая старые.

### 4. **Сегрегация интерфейсов**

Обёртка предоставляет только то, что вам нужно:
- `invoke()` для базового использования
- `stream()` для стриминга
- `batch()` для нескольких промптов
- `dispose()` для очистки

Внутренняя сложность (модель, контекст, сессия) скрыта.

---

## Практический пример: RAG-система

### Без обёртки

```javascript
// Retrieval component
const llama1 = await getLlama();
const model1 = await llama1.loadModel({ ... });
const context1 = await model1.createContext({ ... });
const session1 = new LlamaChatSession({ ... });

// Generation component  
const llama2 = await getLlama();
const model2 = await llama2.loadModel({ ... });
const context2 = await model2.createContext({ ... });
const session2 = new LlamaChatSession({ ... });

// Lots of duplicated setup!
```

### С обёрткой

```javascript
// Retrieval component
const model = await LlamaCpp.initialize({ modelPath: "..." });

// Generation component (reuses the same model)
const answer = await model.invoke(query + retrievedContext);

// Clean, simple, maintainable!
```

---

## Резюме

**Обёртка для LLM необходима для:**
- Создания поддерживаемых RAG-систем
- Поддержки множества провайдеров LLM
- Написания тестируемого кода
- Сохранения чистоты кода приложения

**Ключевые концепции:**
1. **Абстракция:** скрытие сложности за простым интерфейсом
2. **Паттерн «Фабрика»:** обеспечение корректной асинхронной инициализации
3. **Базовый класс:** определение единообразного API среди реализаций
4. **Управление ресурсами:** корректная очистка

**Этот пример демонстрирует:**
- ✅ Как спроектировать класс обёртки
- ✅ Почему это лучше, чем использовать библиотеки напрямую
- ✅ Как поддерживать стриминг и пакетную обработку
- ✅ Как сделать код гибким и поддерживаемым

**Далее в RAG-пайплайне:**
- Загрузка данных и обработка документов
- Стратегии нарезки текста
- Эмбеддинги для семантического поиска
- Создание полной RAG-системы

Обёртка, которую вы создаёте сейчас, будет использоваться на протяжении всей вашей реализации RAG, поэтому этот фундамент критически важен для успеха.
