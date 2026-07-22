# Продвинутая фильтрация по метаданным — разбор кода

Подробное описание стратегий фильтрации по метаданным для векторных баз данных, включая базовые и продвинутые шаблоны фильтрации и методы оптимизации производительности.

## Обзор

В данном примере демонстрируются:
- Базовая фильтрация по одному полю
- Сложная фильтрация по нескольким полям
- Фильтрация по диапазону (даты, оценки, числовые значения)
- Фильтрация по массивам/тегам с логикой OR и AND
- Оптимизация производительности фильтрации (стратегия избыточной выборки)
- Динамическая композиция фильтров
- Сложные шаблоны фильтрации и лучшие практики

**Векторная база данных:** `embedded-vector-db` (beta)
- Фильтрация на стороне клиента после семантического поиска
- Гибкая фильтрация по метаданным
- Поддержка сложных условий фильтрации

---

## Установка и конфигурация

### Импорты

```javascript
import { fileURLToPath } from "url";
import path from "path";
import { VectorDB } from "embedded-vector-db";
import { getLlama } from "node-llama-cpp";
import { Document } from "../../../src/index.js";
import { OutputHelper } from "../../../helpers/output-helper.js";
import chalk from "chalk";
```

**Та же основа, что и в предыдущих примерах**, с акцентом на стратегии фильтрации.

### Константы конфигурации

```javascript
const MODEL_PATH = path.join(__dirname, "..", "..", "..", "models", "bge-small-en-v1.5.Q8_0.gguf");
const DIM = 384;
const MAX_ELEMENTS = 10000;
const NS = "metadata_filter";
```

**Ключевое отличие:** Пространство имён установлено на `"metadata_filter"` для данного набора примеров.

---

## Документы с расширенными метаданными

### Примеры документов с расширенными метаданными

```javascript
function createSampleDocuments() {
    return [
        new Document("Python is a high-level programming language...", {
            id: "doc_1",
            category: "programming",
            language: "python",
            difficulty: "beginner",
            year: 2023,
            rating: 4.5,
            tags: ["backend", "scripting", "data-science"],
            author: "Alice",
            views: 1500,
        }),
        // ... 12 total documents with rich metadata
    ];
}
```

**Что делает эти метаданные богатыми:**

**Категориальные поля:**
- `category`: Высокоуровневая классификация (programming, ai, devops, database)
- `language`: Язык программирования (python, javascript, typescript)
- `difficulty`: Уровень сложности (beginner, intermediate, advanced)

**Числовые поля:**
- `year`: Год публикации (2022–2024)
- `rating`: Оценка качества (4.2–4.9)
- `views`: Показатель популярности (1500–7000)

**Поля-массивы:**
- `tags`: Несколько категоризаций для каждого документа

**Строковые поля:**
- `author`: Идентификация автора
- `topic`: Конкретная тема

**Почему богатые метаданные важны:**
- Позволяют точную фильтрацию
- Поддерживают множественные паттерны доступа
- Обеспечивают фасетированный поиск
- Питают рекомендательные системы

---

## Вспомогательные функции

### Поиск с замером времени

```javascript
async function searchWithTiming(vectorStore, embeddingContext, query, k = 5) {
    const startEmbed = Date.now();
    const queryEmbedding = await embeddingContext.getEmbeddingFor(query);
    const embedTime = Date.now() - startEmbed;

    const startSearch = Date.now();
    const results = await vectorStore.search(NS, Array.from(queryEmbedding.vector), k);
    const searchTime = Date.now() - startSearch;

    return { results, embedTime, searchTime };
}
```

**Назначение:** Измерение производительности операций эмбеддинга и поиска раздельно.

**Почему замер времени важен для фильтрации:**
- Фильтрация происходит после поиска (на стороне клиента)
- Время фильтрации обычно пренебрежимо мало (< 1 мс)
- Избыточная выборка добавляет минимальное время поиска
- Время эмбеддинга доминирует в общем времени

---

## Пример 1: Базовая фильтрация по одному полю

```javascript
async function example1() {
    // Setup...
    const query = "programming languages";
    
    // Without filter
    const allResults = await vectorStore.search(NS, queryVector, 8);
    
    // With category filter
    const filteredResults = allResults
        .filter(r => r.metadata.category === "programming")
        .slice(0, 5);
}
```

**Что демонстрирует:** Простейший шаблон фильтрации — точное совпадение по одному полю.

### Как это работает

**Шаг 1: Семантический поиск**
```javascript
const allResults = await vectorStore.search(NS, queryVector, 8);
// Возвращает: Все семантически похожие документы
```

**Шаг 2: Применение фильтра**
```javascript
const filtered = allResults.filter(r => r.metadata.category === "programming");
// Возвращает: Только документы, где category === "programming"
```

**Шаг 3: Получение топ-результатов**
```javascript
const top5 = filtered.slice(0, 5);
// Возвращает: Первые 5 отфильтрованных результатов
```

### Пример вывода

**Без фильтра:**
```
1. [0.8234] programming - Python is a high-level...
2. [0.7891] programming - JavaScript is essential...
3. [0.7543] ai - Machine learning models require...
4. [0.7234] programming - TypeScript adds static...
5. [0.6891] ai - Neural networks are inspired...
```

**С фильтром (category = "programming"):**
```
1. [0.8234] programming - Python is a high-level...
2. [0.7891] programming - JavaScript is essential...
3. [0.7234] programming - TypeScript adds static...
4. [0.6543] programming - React is a popular...
5. [0.6234] programming - GraphQL is a query...
```

**Ключевой вывод:** Фильтрация сохраняет семантический ранжированный порядок, сужая выборку до конкретной категории.

---

## Пример 2: Фильтрация по нескольким полям

```javascript
async function example2() {
    const allResults = await vectorStore.search(NS, queryVector, 10);
    
    // Filter 1: Category only
    const filter1 = allResults.filter(r => 
        r.metadata.category === "programming"
    );
    
    // Filter 2: Category + Difficulty
    const filter2 = allResults.filter(r => 
        r.metadata.category === "programming" && 
        r.metadata.difficulty === "beginner"
    );
    
    // Filter 3: Multiple conditions
    const filter3 = allResults.filter(r => 
        r.metadata.category === "programming" && 
        r.metadata.difficulty === "intermediate" &&
        r.metadata.year === 2023
    );
}
```

**Что демонстрирует:** Комбинирование нескольких полей метаданных для точной фильтрации.

### Селективность фильтра

**Понятие:** Насколько каждый фильтр уменьшает набор результатов.

```javascript
// Начинаем с 10 результатов

// После фильтра 1 (category = "programming")
// → 6 результатов (селективность 60%)

// После фильтра 2 (category + difficulty)
// → 2 результата (селективность 20%)

// После фильтра 3 (category + difficulty + year)
// → 1 результат (селективность 10%)
```

**Правило:** Больше фильтров = выше точность, меньше результатов.

### Логика AND

**Все условия должны быть истинными:**

```javascript
r.metadata.category === "programming" &&    // Должно быть true
r.metadata.difficulty === "beginner" &&      // И должно быть true
r.metadata.year === 2023                     // И должно быть true
```

**Применение:** «Мне нужны статьи по программированию для начинающих за 2023 год»

### Оптимизация порядка фильтров

**Сначала наименее селективный (плохо):**
```javascript
r.metadata.year === 2023 &&                // 50% документов
r.metadata.category === "programming" &&   // 30% оставшихся
r.metadata.difficulty === "beginner"       // 10% оставшихся
```

**Сначала наиболее селективный (хорошо):**
```javascript
r.metadata.difficulty === "beginner" &&    // 20% документов
r.metadata.category === "programming" &&   // 10% оставшихся  
r.metadata.year === 2023                   // 5% оставшихся
```

**Почему это важно:** JavaScript использует ленивое вычисление — при применении селективных фильтров сначала происходит быстрый отказ.

---

## Пример 3: Фильтрация по диапазону

```javascript
async function example3() {
    const allResults = await vectorStore.search(NS, queryVector, 12);
    
    // Filter by rating range
    const highRated = allResults.filter(r => r.metadata.rating >= 4.5);
    
    // Filter by views range
    const popularDocs = allResults.filter(r => 
        r.metadata.views >= 3000 && r.metadata.views <= 5000
    );
    
    // Filter by year (recent content)
    const recentDocs = allResults.filter(r => r.metadata.year === 2024);
}
```

**Что демонстрирует:** Фильтрацию по числовым полям с условиями диапазона.

### Шаблоны фильтрации по диапазону

**Нижний порог:**
```javascript
r.metadata.rating >= 4.5              // Как минимум 4.5 звезды
r.metadata.views >= 1000              // Как минимум 1000 просмотров
r.metadata.year >= 2023               // 2023 или новее
```

**Верхний порог:**
```javascript
r.metadata.rating <= 4.0              // Как максимум 4.0 звезды
r.metadata.views <= 1000              // Как максимум 1000 просмотров
r.metadata.year <= 2022               // 2022 или старше
```

**Диапазон (от до):**
```javascript
r.metadata.views >= 3000 && r.metadata.views <= 5000    // От 3K до 5K
r.metadata.rating >= 4.0 && r.metadata.rating <= 4.5    // От 4.0 до 4.5
```

### Типичные случаи применения

**Фильтрация по качеству:**
```javascript
// Только качественный контент
const quality = results.filter(r => r.metadata.rating >= 4.7);
```

**Фильтрация по давности:**
```javascript
// Только последние статьи
const currentYear = new Date().getFullYear();
const recent = results.filter(r => r.metadata.year >= currentYear - 1);
```

**Фильтрация по популярности:**
```javascript
// Набирающий популярность контент (много просмотров, недавний)
const trending = results.filter(r => 
    r.metadata.views >= 5000 &&
    r.metadata.year === 2024
);
```

**Фильтрация по ценовому диапазону (электронная коммерция):**
```javascript
const affordable = results.filter(r => 
    r.metadata.price >= 10 && r.metadata.price <= 50
);
```

---

## Пример 4: Фильтрация по массивам/тегам

```javascript
async function example4() {
    const allResults = await vectorStore.search(NS, queryVector, 12);
    
    // Filter by tag inclusion (OR)
    const frontendDocs = allResults.filter(r => 
        r.metadata.tags && r.metadata.tags.includes("frontend")
    );
    
    // Filter by multiple tags (OR logic)
    const aiDocs = allResults.filter(r => 
        r.metadata.tags && (
            r.metadata.tags.includes("deep-learning") || 
            r.metadata.tags.includes("nlp")
        )
    );
    
    // Filter by multiple tags (AND logic)
    const containerDocs = allResults.filter(r => 
        r.metadata.tags && 
        r.metadata.tags.includes("containers") && 
        r.metadata.tags.includes("deployment")
    );
}
```

**Что демонстрирует:** Работу с полями-массивами и многозначными метаданными.

### Шаблоны фильтрации по тегам

**Один тег (OR — содержит этот тег):**
```javascript
r.metadata.tags && r.metadata.tags.includes("frontend")
// Содержит тег "frontend"
```

**Несколько тегов (OR — содержит любой из этих тегов):**
```javascript
r.metadata.tags && (
    r.metadata.tags.includes("react") ||
    r.metadata.tags.includes("vue") ||
    r.metadata.tags.includes("angular")
)
// Содержит "react" ИЛИ "vue" ИЛИ "angular"
```

**Несколько тегов (AND — содержит все эти теги):**
```javascript
r.metadata.tags &&
r.metadata.tags.includes("javascript") &&
r.metadata.tags.includes("frontend") &&
r.metadata.tags.includes("framework")
// Содержит ВСЕ три тега
```

**Продвинутое: Любой из нескольких тегов:**
```javascript
const targetTags = ["react", "vue", "angular"];
r.metadata.tags && r.metadata.tags.some(tag => targetTags.includes(tag))
// Содержит хотя бы один из целевых тегов
```

**Продвинутое: Все из нескольких тегов:**
```javascript
const requiredTags = ["javascript", "frontend"];
r.metadata.tags && requiredTags.every(tag => r.metadata.tags.includes(tag))
// Содержит все обязательные теги
```

### Почему теги важны

**Гибкая категоризация:**
- Один документ может иметь несколько тегов
- Отсутствует жёсткая иерархия
- Легко добавлять новые теги

**Фасетированный поиск:**
```javascript
// Пользователь выбрал: "frontend" AND "react"
const results = allResults.filter(r =>
    r.metadata.tags?.includes("frontend") &&
    r.metadata.tags?.includes("react")
);
```

**Типичные стратегии тегов:**
- **Технологические теги:** javascript, python, react
- **Тематические теги:** tutorial, guide, reference
- **Теги сложности:** beginner, advanced
- **Теги функций:** video, code-examples, interactive

---

## Пример 5: Производительность фильтрации — стратегия избыточной выборки

```javascript
async function example5() {
    // Strategy 1: Fetch exactly k (risky)
    const results1 = await searchWithTiming(vectorStore, context, query, 5);
    const filtered1 = results1.filter(r => r.metadata.category === "programming");
    // May get 0-5 results
    
    // Strategy 2: Over-fetch (safe)
    const results2 = await searchWithTiming(vectorStore, context, query, 15);
    const filtered2 = results2
        .filter(r => r.metadata.category === "programming")
        .slice(0, 5);
    // Guaranteed up to 5 results
}
```

**Что демонстрирует:** Стратегию избыточной выборки для надёжной фильтрации.

### Проблема

**Сценарий:** Вам нужно 5 документов по «программированию».

**Наивный подход:**
```javascript
const results = await search(query, 5);
const filtered = results.filter(r => r.metadata.category === "programming");
// Может вернуть 0, 1, 2, 3, 4 или 5 результатов — непредсказуемо!
```

**Проблема:** Если только 2 из 5 результатов соответствуют фильтру, вы получите 2 вместо 5.

### Решение: Избыточная выборка

**Вычисление множителя избыточной выборки:**
```
Если селективность фильтра = 50% (половина совпадений)
→ Запрашивать 2x целевого k

Если селективность фильтра = 33% (треть совпадений)
→ Запрашивать 3x целевого k

Если селективность фильтра = 25% (четверть совпадений)
→ Запрашивать 4x целевого k
```

**Реализация:**
```javascript
// Хотим 5 результатов, ожидаем селективность 40%
const overFetchMultiplier = 1 / 0.4;  // 2.5x
const fetchK = Math.ceil(5 * overFetchMultiplier);  // 13

const results = await search(query, fetchK);
const filtered = results.filter(condition).slice(0, 5);
// Теперь гарантированно получим до 5 результатов
```

### Влияние на производительность

**Сравнение по времени:**
```
Стратегия 1 (k=5):
  Embed: 45мс
  Поиск: 3мс
  Итого: 48мс
  Результатов: 2 (недостаточно!)

Стратегия 2 (k=15):
  Embed: 45мс  (то же)
  Поиск: 4мс  (всего +1мс!)
  Итого: 49мс  (+1мс)
  Результатов: 5 (как нужно)
```

**Ключевой вывод:** Избыточная выборка добавляет минимальное время (< 1 мс на каждые 10 дополнительных результатов), но гарантирует достаточное количество отфильтрованных результатов.

### Адаптивная избыточная выборка

**Мониторинг селективности:**
```javascript
const stats = {
    totalFetched: 0,
    totalFiltered: 0,
};

function adaptiveSearch(query, targetK, filter) {
    // Calculate selectivity from history
    const selectivity = stats.totalFiltered / stats.totalFetched || 0.5;
    
    // Adjust fetch k
    const fetchK = Math.ceil(targetK / selectivity);
    
    const results = await search(query, fetchK);
    const filtered = results.filter(filter).slice(0, targetK);
    
    // Update stats
    stats.totalFetched += results.length;
    stats.totalFiltered += filtered.length;
    
    return filtered;
}
```

---

## Пример 6: Динамическая композиция фильтров

```javascript
async function example6() {
    // Filter builder function
    function buildFilter(conditions) {
        return (result) => {
            for (const [field, value] of Object.entries(conditions)) {
                if (value === undefined) continue;
                
                if (typeof value === 'object' && value !== null) {
                    // Handle range filters
                    if (value.min !== undefined && result.metadata[field] < value.min) return false;
                    if (value.max !== undefined && result.metadata[field] > value.max) return false;
                } else {
                    // Handle exact match
                    if (result.metadata[field] !== value) return false;
                }
            }
            return true;
        };
    }
    
    // Scenario 1: Beginner content
    const filter1 = buildFilter({ difficulty: "beginner" });
    
    // Scenario 2: High-quality AI content
    const filter2 = buildFilter({ 
        category: "ai",
        rating: { min: 4.7 }
    });
    
    // Scenario 3: Recent programming with good ratings
    const filter3 = buildFilter({ 
        category: "programming",
        year: { min: 2023 },
        rating: { min: 4.5 }
    });
}
```

**Что демонстрирует:** Создание повторно используемых, композируемых функций фильтрации.

### Паттерн построителя фильтров

**Преимущества:**
1. **Повторное использование:** Определить один раз, использовать многократно
2. **Композируемость:** Динамическое комбинирование фильтров
3. **Сопровождаемость:** Централизованная логика фильтрации
4. **Типобезопасность:** Возможность добавить типы TypeScript

**Как это работает:**

**Шаг 1: Определение условий**
```javascript
const conditions = {
    category: "programming",
    year: { min: 2023 },
    rating: { min: 4.5 }
};
```

**Шаг 2: Построение функции фильтра**
```javascript
const filter = buildFilter(conditions);
// Возвращает: (result) => boolean
```

**Шаг 3: Применение к результатам**
```javascript
const filtered = results.filter(filter);
```

### Продвинутая композиция фильтров

**Комбинирование фильтров (AND):**
```javascript
const isRecent = buildFilter({ year: { min: 2023 } });
const isHighQuality = buildFilter({ rating: { min: 4.7 } });

const filtered = results.filter(r => isRecent(r) && isHighQuality(r));
```

**Комбинирование фильтров (OR):**
```javascript
const isBeginnerProgramming = buildFilter({ 
    category: "programming",
    difficulty: "beginner"
});
const isAdvancedAI = buildFilter({ 
    category: "ai",
    difficulty: "advanced"
});

const filtered = results.filter(r => 
    isBeginnerProgramming(r) || isAdvancedAI(r)
);
```

**Отрицание:**
```javascript
const notAdvanced = (r) => !buildFilter({ difficulty: "advanced" })(r);
const filtered = results.filter(notAdvanced);
```

### Практическое применение

**Фильтрация по предпочтениям пользователя:**
```javascript
function buildUserFilter(userPreferences) {
    const conditions = {};
    
    if (userPreferences.minRating) {
        conditions.rating = { min: userPreferences.minRating };
    }
    
    if (userPreferences.categories) {
        // Handle array of categories (requires OR logic)
        return (result) => 
            userPreferences.categories.includes(result.metadata.category) &&
            buildFilter(conditions)(result);
    }
    
    return buildFilter(conditions);
}

// Usage
const userFilter = buildUserFilter({
    categories: ["programming", "ai"],
    minRating: 4.5
});

const personalized = results.filter(userFilter);
```

---

## Пример 7: Сложные шаблоны фильтрации

```javascript
async function example7() {
    const allResults = await vectorStore.search(NS, queryVector, 12);
    
    // Pattern 1: Exclude filter
    const exclude = allResults.filter(r => r.metadata.difficulty !== "advanced");
    
    // Pattern 2: OR conditions
    const orFilter = allResults.filter(r => 
        r.metadata.category === "programming" || r.metadata.category === "ai"
    );
    
    // Pattern 3: Complex nested conditions
    const complex = allResults.filter(r => 
        (r.metadata.category === "ai" && r.metadata.difficulty === "advanced") ||
        (r.metadata.category === "programming" && r.metadata.difficulty === "beginner")
    );
    
    // Pattern 4: Sort filtered results
    const sorted = allResults
        .filter(r => r.metadata.category === "programming")
        .sort((a, b) => b.metadata.rating - a.metadata.rating);
    
    // Pattern 5: Similarity threshold + metadata filter
    const threshold = allResults.filter(r => 
        r.similarity > 0.3 && r.metadata.year === 2024
    );
}
```

**Что демонстрирует:** Продвинутые шаблоны фильтрации для сложных задач.

### Шаблон 1: Исключение/отрицание

**Назначение:** Удаление нежелательных результатов.

```javascript
// Исключить сложный контент
r.metadata.difficulty !== "advanced"

// Исключить несколько значений
!["advanced", "expert"].includes(r.metadata.difficulty)

// Исключить по условию
r.metadata.views < 10000  // Исключить очень популярные
```

**Случаи применения:**
- Фильтрация уже просмотренного контента
- Удаление неприемлемых результатов
- Исключение устаревших/архивных записей

### Шаблон 2: Условия OR

**Назначение:** Соответствие любому из нескольких критериев.

```javascript
// Категория — programming ИЛИ ai
r.metadata.category === "programming" || r.metadata.category === "ai"

// Лучше: использовать includes()
["programming", "ai"].includes(r.metadata.category)

// Сложный OR
r.metadata.rating > 4.7 || r.metadata.views > 5000
```

### Шаблон 3: Вложенные условия

**Сложная логика:**
```javascript
(condition1 && condition2) || (condition3 && condition4)
```

**Пример:**
```javascript
// (AI И advanced) ИЛИ (programming И beginner)
(r.metadata.category === "ai" && r.metadata.difficulty === "advanced") ||
(r.metadata.category === "programming" && r.metadata.difficulty === "beginner")
```

**Читается как:** «Дайте либо сложный контент по ИИ, либо базовый контент по программированию»

**Таблица истинности:**
```
category     difficulty    Результат
ai           advanced      ✓ Совпадение
ai           beginner      ✗ Нет совпадения
programming  advanced      ✗ Нет совпадения
programming  beginner      ✓ Совпадение
devops       beginner      ✗ Нет совпадения
```

### Шаблон 4: Сортировка после фильтрации

**Назначение:** Переупорядочивание отфильтрованных результатов по метаданным.

**Сортировка по одному полю:**
```javascript
filtered.sort((a, b) => b.metadata.rating - a.metadata.rating)  // По убыванию
filtered.sort((a, b) => a.metadata.year - b.metadata.year)      // По возрастанию
```

**Сортировка по нескольким полям:**
```javascript
filtered.sort((a, b) => {
    // Primary: rating (descending)
    if (a.metadata.rating !== b.metadata.rating) {
        return b.metadata.rating - a.metadata.rating;
    }
    // Secondary: views (descending)
    return b.metadata.views - a.metadata.views;
});
```

**Почему сортировать после фильтрации?**
- Векторный поиск ранжирует по сходству
- Сортировка по метаданным предоставляет альтернативное ранжирование
- Полезно для «лучше оценённых» или «самых свежих»

### Шаблон 5: Комбинирование сходства и метаданных

**Назначение:** Пороговое значение качества + критерии метаданных.

```javascript
r.similarity > 0.5 &&              // Семантическая релевантность
r.metadata.rating >= 4.5 &&        // Качество
r.metadata.year === 2024           // Актуальность
```

**Случай применения:** «Недавний, качественный, релевантный контент»

**Пример вывода:**
```
Без порогового значения:
1. [0.8234] rating: 4.2, year: 2023  ✓ Релевантно, но старое
2. [0.6543] rating: 4.8, year: 2024  ✓ Идеальное совпадение
3. [0.4123] rating: 4.7, year: 2024  ✗ Слишком низкое сходство
4. [0.7891] rating: 4.1, year: 2024  ✗ Слишком низкая оценка

С пороговым значением (similarity > 0.5, rating >= 4.5, year = 2024):
1. [0.6543] rating: 4.8, year: 2024  ✓ Только это совпадает
```

---

## Оптимизация производительности фильтрации

### Стратегии оптимизации

**1. Сначала наиболее селективные фильтры**
```javascript
// Плохо: Сначала наименее селективный
r.metadata.year >= 2020 &&              // 80% проходят
r.metadata.category === "programming" && // 40% проходят
r.metadata.difficulty === "expert"       // 5% проходят

// Хорошо: Сначала наиболее селективный
r.metadata.difficulty === "expert" &&    // 5% проходят (быстрый отказ!)
r.metadata.category === "programming" && // 3% проходят
r.metadata.year >= 2020                  // 3% проходят
```

**2. Кэширование функций фильтров**
```javascript
const filterCache = new Map();

function getCachedFilter(conditions) {
    const key = JSON.stringify(conditions);
    if (!filterCache.has(key)) {
        filterCache.set(key, buildFilter(conditions));
    }
    return filterCache.get(key);
}
```

**3. Мониторинг селективности**
```javascript
function trackSelectivity(filter, name) {
    return (results) => {
        const before = results.length;
        const filtered = results.filter(filter);
        const after = filtered.length;
        const selectivity = after / before;
        
        console.log(`${name}: ${before} → ${after} (${(selectivity * 100).toFixed(1)}%)`);
        
        return filtered;
    };
}
```

### Сравнение производительности

```javascript
// Неэффективно: Несколько проходов
let results = allResults;
results = results.filter(r => r.metadata.category === "programming");
results = results.filter(r => r.metadata.difficulty === "beginner");
results = results.filter(r => r.metadata.year === 2023);

// Эффективно: Один проход
const results = allResults.filter(r =>
    r.metadata.category === "programming" &&
    r.metadata.difficulty === "beginner" &&
    r.metadata.year === 2023
);
```

**Почему один проход лучше:**
- Одна итерация вместо трёх
- Меньше выделений памяти
- Лучшее использование кэша процессора

---

## Типичные случаи применения фильтрации

### 1. Поиск товаров в интернет-магазине

```javascript
// User filters
const filters = {
    category: "electronics",
    price: { min: 100, max: 500 },
    rating: { min: 4.0 },
    inStock: true,
    brand: ["Apple", "Samsung", "Sony"]
};

const products = searchResults.filter(r =>
    r.metadata.category === filters.category &&
    r.metadata.price >= filters.price.min &&
    r.metadata.price <= filters.price.max &&
    r.metadata.rating >= filters.rating.min &&
    r.metadata.inStock === filters.inStock &&
    filters.brand.includes(r.metadata.brand)
);
```

### 2. Система управления контентом

```javascript
// Editorial filters
const articles = searchResults.filter(r =>
    r.metadata.status === "published" &&
    r.metadata.publishDate >= lastWeek &&
    r.metadata.author !== "deleted-user" &&
    r.metadata.category !== "archived"
);
```

### 3. Права пользователей

```javascript
// Access control
const accessible = searchResults.filter(r => {
    if (r.metadata.visibility === "public") return true;
    if (r.metadata.visibility === "private") {
        return r.metadata.owner === userId;
    }
    if (r.metadata.visibility === "team") {
        return user.teams.includes(r.metadata.teamId);
    }
    return false;
});
```

### 4. Фильтрация по времени

```javascript
// Content lifecycle
const currentDate = new Date();
const active = searchResults.filter(r => {
    const startDate = new Date(r.metadata.startDate);
    const endDate = new Date(r.metadata.endDate);
    return currentDate >= startDate && currentDate <= endDate;
});
```

---

## Лучшие практики

### 1. Всегда проверяйте наличие поля

```javascript
// ❌ Плохо: Предполагает наличие поля
r.metadata.tags.includes("react")

// ✓ Хорошо: Сначала проверить наличие
r.metadata.tags && r.metadata.tags.includes("react")

// ✓ Лучше: Опциональная цепочка вызовов
r.metadata.tags?.includes("react")
```

### 2. Используйте типобезопасные фильтры

```javascript
// Define expected metadata structure
interface Metadata {
    category: string;
    rating?: number;
    tags?: string[];
    year?: number;
}

function isValidCategory(value: unknown): value is string {
    return typeof value === "string" && value.length > 0;
}
```

### 3. Логика фильтрации документов

```javascript
/**
 * Filters results for beginner-friendly programming content
 * - Must be category "programming"
 * - Must be difficulty "beginner"
 * - Must have rating >= 4.0
 * - Must be published in last 2 years
 */
function beginnerProgrammingFilter(result) {
    const currentYear = new Date().getFullYear();
    return (
        result.metadata.category === "programming" &&
        result.metadata.difficulty === "beginner" &&
        result.metadata.rating >= 4.0 &&
        result.metadata.year >= currentYear - 2
    );
}
```

### 4. Обработка граничных случаев

```javascript
function safeFilter(result) {
    // Handle missing metadata
    if (!result.metadata) return false;
    
    // Handle undefined fields
    const rating = result.metadata.rating ?? 0;
    const tags = result.metadata.tags ?? [];
    
    // Apply filter logic
    return rating >= 4.0 && tags.length > 0;
}
```

### 5. Тестирование фильтров независимо

```javascript
describe("Category filter", () => {
    it("filters by single category", () => {
        const results = [...]; // Test data
        const filtered = results.filter(r => r.metadata.category === "programming");
        expect(filtered).toHaveLength(3);
        expect(filtered.every(r => r.metadata.category === "programming")).toBe(true);
    });
});
```

---

## Резюме

### Что было создано

Семь примеров, демонстрирующих:
1. ✅ Базовая фильтрация по одному полю
2. ✅ Фильтрация по нескольким полям с условиями AND
3. ✅ Фильтрация по диапазону для числовых полей
4. ✅ Фильтрация по массивам/тегам с логикой OR и AND
5. ✅ Стратегия избыточной выборки для надёжных результатов
6. ✅ Динамическая композиция фильтров
7. ✅ Сложные шаблоны фильтрации

### Ключевые выводы

- **Фильтрация по метаданным** сужает результаты семантического поиска
- **Многополевые фильтры** обеспечивают точность
- **Фильтры по диапазону** отлично работают для дат и оценок
- **Фильтры по тегам** обеспечивают гибкую категоризацию
- **Избыточная выборка** гарантирует достаточное количество отфильтрованных результатов
- **Динамические фильтры** поддерживают запросы пользователей
- **Композиция фильтров** позволяет реализовать сложную логику

### Шпаргалка по стратегиям фильтрации

| Потребность | Стратегия | Пример |
|------|----------|---------|
| Точное совпадение | Равенство | `r.metadata.category === "ai"` |
| Несколько полей | Условия AND | `category === "ai" && difficulty === "beginner"` |
| Любое из нескольких | Условия OR | `category === "ai" \|\| category === "programming"` |
| Числовой диапазон | Сравнения | `rating >= 4.5 && rating <= 5.0` |
| Содержит тег | Array includes | `tags?.includes("frontend")` |
| Содержит любой тег | Array some | `tags?.some(t => ["a", "b"].includes(t))` |
| Содержит все теги | Несколько includes | `tags?.includes("a") && tags?.includes("b")` |
| Исключение | Отрицание | `difficulty !== "expert"` |
| Сложная логика | Вложенные условия | `(a && b) \|\| (c && d)` |

### Советы по производительности

1. **Избыточная выборка в 2–3 раза** при фильтрации (минимальные затраты)
2. **Сначала селективные фильтры** (быстрый отказ)
3. **Один проход** лучше, чем несколько итераций
4. **Кэширование функций фильтров** для повторного использования
5. **Мониторинг селективности** для оптимизации коэффициента избыточной выборки

### Дальнейшие шаги

- **Примените** к вашим доменным метаданным
- **Масштабируйте** на большие объёмы данных с оптимизацией

Фильтрация по метаданным — это мост между семантическим поиском и точными результатами. Освойте эти шаблоны, и вы создадите мощный, удобный для пользователей поисковый интерфейс!
