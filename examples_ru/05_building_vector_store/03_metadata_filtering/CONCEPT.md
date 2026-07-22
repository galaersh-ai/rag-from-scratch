# Продвинутая фильтрация по метаданным — обзор концепции

## Что такое фильтрация по метаданным?

**Фильтрация по метаданным** — это процесс сужения результатов поиска на основе структурированных данных (метаданных), связанных с документами, а не только их семантического содержания.

Представьте это как добавление возможности **«расширенного поиска»** к семантическому векторному поиску.

### Основная идея

**Только семантический поиск:**
```javascript
query: "programming tutorials"
→ Returns all semantically similar documents about programming
```

**Семантический поиск + фильтрация по метаданным:**
```javascript
query: "programming tutorials"
filters: { difficulty: "beginner", year: 2024, rating: >= 4.5 }
→ Returns only beginner-friendly, recent, high-quality programming content
```

**Ключевой вывод:** Сочетайте мощь семантического понимания с точностью структурированной фильтрации.

---

## Почему фильтрация по метаданным важна

### Проблема только семантического поиска

Семантический поиск находит **релевантный** контент, но не может различить:
- Актуальные и устаревшие статьи
- Материалы для начинающих и продвинутых
- Бесплатный и платный контент
- Опубликованные и черновые документы
- Публичные и приватные документы

### Решение: богатые метаданные

Добавив структурированные метаданные, вы можете:
- Фильтровать по категории, сложности, цене, статусу
- Сортировать по дате, рейтингу, популярности
- Применять контроль доступа и разрешения
- Создавать интерфейсы фасетного поиска
- Строить персонализированные рекомендации

### Практическое значение

**Пример из электронной коммерции:**

**Без фильтрации:**
```
User searches: "laptop"
→ Returns 1000 laptops
→ User overwhelmed, leaves site
```

**С фильтрацией по метаданным:**
```
User searches: "laptop"
Filters: price $500-$1000, rating >= 4 stars, in stock
→ Returns 15 relevant laptops
→ User finds what they need, purchases
```

**Результат:** Лучший пользовательский опыт = более высокий коэффициент конверсии!

---

## Типы метаданных

### 1. Категориальные метаданные

**Назначение:** Классифицировать документы на отдельные группы.

**Примеры:**
- `category`: "electronics", "books", "clothing"
- `language`: "python", "javascript", "java"
- `difficulty`: "beginner", "intermediate", "advanced"
- `status`: "published", "draft", "archived"

**Примеры использования:**
- «Покажите мне только опубликованные статьи»
- «Отфильтровать по категории программирования»
- «Только учебники для начинающих»

### 2. Числовые метаданные

**Назначение:** Количественные атрибуты, поддерживающие запросы по диапазону.

**Примеры:**
- `price`: 19.99, 49.99, 99.99
- `rating`: 4.5, 4.8, 3.2
- `year`: 2022, 2023, 2024
- `views`: 1500, 5000, 10000
- `word_count`: 500, 1000, 2000

**Примеры использования:**
- «От $50 до $100»
- «Рейтинг не менее 4 звёзд»
- «Опубликовано в последний год»
- «Популярные статьи (>5000 просмотров)»

### 3. Метаданные даты и времени

**Назначение:** Фильтрация и сортировка по времени.

**Примеры:**
- `created_at`: "2024-01-15T10:30:00Z"
- `updated_at`: "2024-03-20T14:00:00Z"
- `publish_date`: "2024-02-01"
- `expires_at`: "2024-12-31"

**Примеры использования:**
- «За последние 7 дней»
- «Контент за текущий месяц»
- «Не истёкший»
- «Недавно обновлённый»

### 4. Метаданные массивов/тегов

**Назначение:** Многозначная категоризация.

**Примеры:**
- `tags`: ["javascript", "frontend", "react"]
- `features`: ["video", "code-examples", "quiz"]
- `platforms`: ["web", "mobile", "desktop"]

**Примеры использования:**
- «Содержит тег "react"»
- «Содержит любой из: vue, react, angular»
- «Содержит и "javascript", и "frontend"»

### 5. Булевы метаданные

**Назначение:** Двоичные флаги.

**Примеры:**
- `is_featured`: true/false
- `in_stock`: true/false
- `is_free`: true/false
- `requires_auth`: true/false

**Примеры использования:**
- «Только избранные товары»
- «Товары в наличии»
- «Бесплатный контент»

### 6. Вложенные/сложные метаданные

**Назначение:** Структурированные иерархические данные.

**Примеры:**
```javascript
author: {
    id: "user_123",
    name: "John Doe",
    verified: true
}

permissions: {
    visibility: "public",
    allowed_teams: ["engineering", "product"]
}
```

**Примеры использования:**
- «От проверенных авторов»
- «Видимо для моей команды»
- «Доступ к премиум-уровню»

---

## Стратегии фильтрации

### Стратегия 1: Фильтрация по одному полю

**Что это:** Фильтрация по одному полю метаданных.

**Когда использовать:**
- Простая категоризация
- Пользователь выбирает один фильтр
- Чёткие критерии

**Пример:**
```javascript
// Show only AI articles
results.filter(r => r.metadata.category === "ai")
```

**Преимущества:**
- Простая реализация
- Быстрое выполнение
- Легко понять

**Недостатки:**
- Ограниченная точность
- Может возвращать слишком много результатов

### Стратегия 2: Фильтрация по нескольким полям (И)

**Что это:** Все условия должны быть истинными.

**Когда использовать:**
- Требуется высокая точность
- Сужение до определённого подмножества
- Множественные требования

**Пример:**
```javascript
// Beginner AI articles from 2024
results.filter(r =>
    r.metadata.category === "ai" &&
    r.metadata.difficulty === "beginner" &&
    r.metadata.year === 2024
)
```

**Преимущества:**
- Высокая точность
- Целевые результаты
- Снижение шума

**Недостатки:**
- Может возвращать слишком мало результатов
- Требует точного знания критериев

### Стратегия 3: Фильтрация по нескольким полям (ИЛИ)

**Что это:** Любое условие может быть истинным.

**Когда использовать:**
- Расширенный поиск
- Несколько допустимых вариантов
- Увеличение полноты охвата

**Пример:**
```javascript
// AI or programming articles
results.filter(r =>
    r.metadata.category === "ai" ||
    r.metadata.category === "programming"
)
```

**Преимущества:**
- Широкий охват
- Больше результатов
- Гибкие критерии

**Недостатки:**
- Низкая точность
- Может включать нерелевантные результаты

### Стратегия 4: Фильтрация по диапазону

**Что это:** Фильтрация числовых значений по диапазону.

**Когда использовать:**
- Цены, даты, рейтинги
- Запросы «между»
- Пороги качества

**Пример:**
```javascript
// Products $50-$100, rated 4+ stars
results.filter(r =>
    r.metadata.price >= 50 &&
    r.metadata.price <= 100 &&
    r.metadata.rating >= 4.0
)
```

**Преимущества:**
- Точность для числовых данных
- Естественно для пользователей
- Гибкие границы

**Недостатки:**
- Работает только с числами
- Требует значений мин/макс

### Стратегия 5: Фильтрация по тегам

**Что это:** Фильтрация по принадлежности к массиву.

**Когда использовать:**
- Многокатегорийные элементы
- Гибкие таксономии
- Пользовательские теги

**Пример:**
```javascript
// Has "react" OR "vue" tag
results.filter(r =>
    r.metadata.tags?.includes("react") ||
    r.metadata.tags?.includes("vue")
)

// Has BOTH "javascript" AND "frontend"
results.filter(r =>
    r.metadata.tags?.includes("javascript") &&
    r.metadata.tags?.includes("frontend")
)
```

**Преимущества:**
- Очень гибкая
- Многогранная классификация
- Удобна для пользователей

**Недостатки:**
- Может быть непоследовательной
- Требует стандартизации тегов

### Стратегия 6: Исключающая фильтрация

**Что это:** Удаление нежелательных результатов.

**Когда использовать:**
- Фильтрация известных плохих результатов
- Удаление уже просмотренного контента
- Исключение категорий

**Пример:**
```javascript
// Exclude advanced difficulty
results.filter(r => r.metadata.difficulty !== "advanced")

// Exclude multiple categories
results.filter(r => !["archived", "deleted"].includes(r.metadata.status))
```

**Преимущества:**
- Убирает шум
- Повышает релевантность
- Удобно для пользователей («скрыть X»)

**Недостатки:**
- Снижает количество результатов
- Может быть чрезмерно ограничительной

---

## Стратегия избыточной выборки (Over-Fetching)

### Проблема

**Сценарий:** Вы хотите получить 5 статей по программированию после фильтрации.

**Наивный подход:**
```javascript
// Fetch 5 results
const results = await search(query, 5);

// Filter to programming
const filtered = results.filter(r => r.metadata.category === "programming");

// Result: Might get 0, 1, 2, 3, 4, or 5 - unpredictable!
```

**Проблема:** Если только 40% результатов относятся к «программированию», вы получите в среднем только 2 результата.

### Решение

**Извлекайте больше результатов, оценивая избирательность фильтра:**
```javascript
// Want 5 results, expect 40% selectivity
const fetchMultiplier = 1 / 0.4;  // 2.5x
const fetchCount = Math.ceil(5 * fetchMultiplier);  // 13

// Fetch 13 results
const results = await search(query, fetchCount);

// Filter and take top 5
const filtered = results
    .filter(r => r.metadata.category === "programming")
    .slice(0, 5);

// Result: Reliably get 5 results
```

### Оценка избирательности

**Формула:**
```
selectivity = (filtered results) / (total results)
over-fetch multiplier = 1 / selectivity
fetch count = desired count × multiplier
```

**Примеры:**
```
50% selectivity → fetch 2x
33% selectivity → fetch 3x
25% selectivity → fetch 4x
20% selectivity → fetch 5x
```

### Почему это работает

**Влияние на производительность:**
```
Fetching 5 vs 15 results:
  Embedding time: 45ms (same for both)
  Search time: 3ms vs 4ms (+1ms)
  Total: 48ms vs 49ms (+1ms)
```

**Ключевой вывод:** Избыточная выборка практически ничего не стоит! Дополнительная 1 мс стоит надёжного количества результатов.

### Адаптивная избыточная выборка

**Обучение на основе истории:**
```javascript
// Track selectivity over time
let totalFetched = 0;
let totalFiltered = 0;

function search(query, targetCount, filter) {
    // Use historical selectivity
    const selectivity = totalFiltered / totalFetched || 0.5;
    const fetchCount = Math.ceil(targetCount / selectivity);
    
    const results = await vectorSearch(query, fetchCount);
    const filtered = results.filter(filter).slice(0, targetCount);
    
    // Update statistics
    totalFetched += results.length;
    totalFiltered += filtered.length;
    
    return filtered;
}
```

**Преимущества:**
- Автоматическая подстройка под реальную избирательность
- Минимизация избыточной выборки со временем
- Обработка различных критериев фильтрации

---

## Паттерны композиции фильтров

### Паттерн 1: Конструктор фильтров

**Назначение:** Создание переиспользуемых, композитных фильтров.

```javascript
function buildFilter(conditions) {
    return (result) => {
        for (const [field, value] of Object.entries(conditions)) {
            if (typeof value === 'object' && value.min) {
                if (result.metadata[field] < value.min) return false;
            } else if (typeof value === 'object' && value.max) {
                if (result.metadata[field] > value.max) return false;
            } else {
                if (result.metadata[field] !== value) return false;
            }
        }
        return true;
    };
}

// Usage
const filter = buildFilter({
    category: "programming",
    year: { min: 2023 },
    rating: { min: 4.5 }
});

const filtered = results.filter(filter);
```

**Преимущества:**
- Декларативный API
- Легко тестировать
- Переиспользуемость в приложении

### Паттерн 2: Комбинаторы фильтров

**Назначение:** Комбинирование нескольких фильтров с логикой И/ИЛИ.

```javascript
function and(...filters) {
    return (result) => filters.every(f => f(result));
}

function or(...filters) {
    return (result) => filters.some(f => f(result));
}

function not(filter) {
    return (result) => !filter(result);
}

// Usage
const isRecent = buildFilter({ year: { min: 2023 } });
const isHighQuality = buildFilter({ rating: { min: 4.5 } });
const isAdvanced = buildFilter({ difficulty: "advanced" });

const filter = and(
    isRecent,
    isHighQuality,
    not(isAdvanced)
);

const filtered = results.filter(filter);
```

**Преимущества:**
- Композитность
- Читаемость
- Тестируемость изолированно

### Паттерн 3: Динамические фильтры

**Назначение:** Построение фильтров на основе пользовательского ввода.

```javascript
function buildUserFilter(userPreferences) {
    const conditions = {};
    
    if (userPreferences.category) {
        conditions.category = userPreferences.category;
    }
    
    if (userPreferences.minRating) {
        conditions.rating = { min: userPreferences.minRating };
    }
    
    if (userPreferences.yearRange) {
        conditions.year = {
            min: userPreferences.yearRange.start,
            max: userPreferences.yearRange.end
        };
    }
    
    return buildFilter(conditions);
}

// Usage
const userFilter = buildUserFilter({
    category: "ai",
    minRating: 4.5,
    yearRange: { start: 2023, end: 2024 }
});

const personalized = results.filter(userFilter);
```

**Преимущества:**
- Определяется пользователем
- Гибкость
- Отсутствие зашитой логики

---

## Фильтрация + семантический поиск

### Синергия

**Семантический поиск:** Находит релевантный контент по смыслу
**Фильтрация по метаданным:** Сужает до конкретных критериев

**Вместе:** Релевантный И соответствующий требованиям

### Рабочие паттерны

**Паттерн A: Сначала фильтр (предфильтрация)**
```
1. User specifies filters (category, date, etc.)
2. Search only within filtered subset
3. Return top k semantic matches
```

**Преимущества:**
- Поиск только в релевантном подмножестве
- Быстрее для Highly Selective фильтров

**Недостатки:**
- Может пропустить хорошие семантические совпадения вне фильтра
- Требует поддержки фильтрованного поиска со стороны базы данных

**Паттерн B: Сначала поиск (постфильтрация)**
```
1. Semantic search across all documents
2. Filter results by metadata
3. Take top k filtered results
```

**Преимущества:**
- Рассматриваются лучшие семантические совпадения
- Гибкость (можно менять фильтры без повторного поиска)

**Недостатки:**
- Необходима избыточная выборка для обеспечения достаточного количества отфильтрованных результатов

**Паттерн C: Гибридный**
```
1. Apply broad filters (category, status)
2. Semantic search within broad subset
3. Apply fine filters (rating, date)
4. Return top k
```

**Преимущества:**
- Баланс эффективности и гибкости
- Хорошо подходит для производственных систем

---

## Соображения по производительности

### Сложность фильтров

**Простые фильтры (быстрые):**
- Одинарная проверка на равенство
- Булевы проверки
- Наличие поля

**Умеренные фильтры (средние):**
- Сравнения диапазонов
- Множественные условия И
- Принадлежность к массиву

**Сложные фильтры (медленные):**
- Вложенные условия
- Множественные условия ИЛИ
- Регулярные выражения
- Сложные вычисления

### Методы оптимизации

**1. Ставьте наиболее избирательные фильтры первыми**

JavaScript использует ленивое вычисление:
```javascript
// Bad: 80% pass first check
r.metadata.year >= 2020 &&
r.metadata.category === "programming" &&
r.metadata.difficulty === "expert"

// Good: 5% pass first check (fail fast!)
r.metadata.difficulty === "expert" &&
r.metadata.category === "programming" &&
r.metadata.year >= 2020
```

**2. Используйте однопроходную фильтрацию**

```javascript
// Bad: Multiple iterations
let results = allResults;
results = results.filter(f1);
results = results.filter(f2);
results = results.filter(f3);

// Good: Single iteration
const results = allResults.filter(r => f1(r) && f2(r) && f3(r));
```

**3. Кэшируйте функции фильтров**

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

**4. Мониторьте избирательность**

Отслеживайте, какие фильтры наиболее эффективны:
```javascript
const stats = {
    category: { total: 1000, passed: 300 },  // 30% selectivity
    difficulty: { total: 1000, passed: 200 }, // 20% selectivity
    year: { total: 1000, passed: 500 }        // 50% selectivity
};

// Use most selective (difficulty) first
```

---

## Типичные сценарии использования

### Сценарий 1: Поиск товаров в электронной коммерции

**Требования:**
- Фильтрация по ценовому диапазону
- Фильтрация по бренду
- Фильтрация по рейтингу
- Фильтрация по наличию
- Сортировка по цене или популярности

**Реализация:**
```javascript
const productFilter = {
    category: "electronics",
    price: { min: 100, max: 500 },
    brand: ["Apple", "Samsung", "Sony"],
    rating: { min: 4.0 },
    inStock: true
};

const products = results.filter(r =>
    r.metadata.category === productFilter.category &&
    r.metadata.price >= productFilter.price.min &&
    r.metadata.price <= productFilter.price.max &&
    productFilter.brand.includes(r.metadata.brand) &&
    r.metadata.rating >= productFilter.rating.min &&
    r.metadata.inStock === true
);
```

### Сценарий 2: Система управления контентом

**Требования:**
- Фильтрация по статусу публикации
- Фильтрация по автору
- Фильтрация по диапазону дат
- Фильтрация по категории
- Контроль доступа

**Реализация:**
```javascript
const cmsFilter = (result, currentUser) => {
    // Must be published
    if (result.metadata.status !== "published") return false;
    
    // Date range
    const publishDate = new Date(result.metadata.publishDate);
    if (publishDate < startDate || publishDate > endDate) return false;
    
    // Access control
    if (result.metadata.visibility === "private") {
        return result.metadata.author === currentUser.id;
    }
    if (result.metadata.visibility === "team") {
        return currentUser.teams.includes(result.metadata.team);
    }
    
    return true;
};
```

### Сценарий 3: Платформа обучения

**Требования:**
- Фильтрация по уровню сложности
- Фильтрация по теме/тегам
- Фильтрация по длительности
- Фильтрация по статусу прохождения
- Персонализированные рекомендации

**Реализация:**
```javascript
const learningFilter = (result, userProfile) => {
    // Match user's skill level
    if (!userProfile.skillLevels.includes(result.metadata.difficulty)) {
        return false;
    }
    
    // Has relevant tags
    const hasRelevantTag = result.metadata.tags?.some(tag =>
        userProfile.interests.includes(tag)
    );
    if (!hasRelevantTag) return false;
    
    // Appropriate duration
    if (result.metadata.duration > userProfile.maxDuration) {
        return false;
    }
    
    // Not already completed
    if (userProfile.completed.includes(result.metadata.id)) {
        return false;
    }
    
    return true;
};
```

### Сценарий 4: Агрегатор новостей

**Требования:**
- Фильтрация по актуальности
- Фильтрация по источнику
- Фильтрация по теме
- Фильтрация по языку
- Скрытие прочитанных статей

**Реализация:**
```javascript
const newsFilter = (result, userPrefs, readArticles) => {
    // Recent only (last 7 days)
    const articleDate = new Date(result.metadata.publishDate);
    const weekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
    if (articleDate < weekAgo) return false;
    
    // From preferred sources
    if (!userPrefs.sources.includes(result.metadata.source)) {
        return false;
    }
    
    // Matches topics of interest
    const hasInterestingTopic = result.metadata.topics?.some(topic =>
        userPrefs.topics.includes(topic)
    );
    if (!hasInterestingTopic) return false;
    
    // Not already read
    if (readArticles.has(result.metadata.id)) return false;
    
    return true;
};
```

---

## Рекомендуемые практики

### 1. Проектируйте богатую схему метаданных

**Рекомендации:**
- Включайте все фильтруемые атрибуты
- Используйте единообразные соглашения об именовании
- Выбирайте подходящие типы данных
- Документируйте структуру метаданных

**Пример схемы:**
```javascript
{
    // Required fields
    id: string,
    category: string,
    created_at: ISO8601 date string,
    
    // Optional categorical
    status?: "draft" | "published" | "archived",
    difficulty?: "beginner" | "intermediate" | "advanced",
    
    // Optional numerical
    rating?: number (0-5),
    price?: number,
    views?: number,
    
    // Optional arrays
    tags?: string[],
    authors?: string[],
    
    // Optional nested
    permissions?: {
        visibility: "public" | "private" | "team",
        teams?: string[]
    }
}
```

### 2. Валидируйте метаданные

**Всегда проверяйте наличие полей:**
```javascript
// ❌ Bad: Assumes field exists
r.metadata.tags.includes("react")

// ✓ Good: Safe check
r.metadata.tags?.includes("react")

// ✓ Better: With fallback
(r.metadata.tags || []).includes("react")
```

### 3. Предоставляйте интерфейс фильтров

**Удобные для пользователя интерфейсы:**
- Чекбоксы для категорий
- Ползунки для диапазонов
- Выбор даты для дат
- Выбор тегов для множественного выбора
- Кнопка сброса фильтров

### 4. Показывайте статистику фильтров

**Помогайте пользователям понимать влияние фильтров:**
```javascript
// Show result counts per filter option
{
    category: {
        "programming": 45,    // 45 results
        "ai": 32,             // 32 results
        "devops": 18          // 18 results
    },
    difficulty: {
        "beginner": 30,
        "intermediate": 40,
        "advanced": 25
    }
}
```

### 5. Поддерживайте сохранение фильтров

**Сохраняйте пользовательские настройки:**
```javascript
// Save to localStorage
localStorage.setItem('searchFilters', JSON.stringify(filters));

// Restore on page load
const savedFilters = JSON.parse(localStorage.getItem('searchFilters'));
```

### 6. Тестируйте логику фильтров

**Модульные тесты для фильтров:**
```javascript
describe('beginnerProgrammingFilter', () => {
    it('accepts beginner programming content', () => {
        const doc = { metadata: { category: "programming", difficulty: "beginner" } };
        expect(filter(doc)).toBe(true);
    });
    
    it('rejects advanced content', () => {
        const doc = { metadata: { category: "programming", difficulty: "advanced" } };
        expect(filter(doc)).toBe(false);
    });
});
```

---

## Сводка ключевых понятий

### 1. Фильтрация по метаданным

**Основная концепция:** Сужение результатов семантического поиска с помощью структурированных данных.

**Почему это важно:** Обеспечивает точность, которую не может дать только семантический поиск.

### 2. Типы фильтров

- **Категориальные:** Точное совпадение по дискретным значениям
- **Числовые:** Запросы по диапазону для чисел
- **Массивные:** Принадлежность к многозначным полям
- **Булевы:** Флаги true/false
- **Вложенные:** Сложные структурированные данные

### 3. Избыточная выборка

**Стратегия:** Извлекайте больше результатов, чем необходимо, перед фильтрацией.

**Формула:** `fetchCount = targetCount / expectedSelectivity`

**Почему:** Обеспечивает надёжное количество результатов при минимальных затратах.

### 4. Композиция фильтров

**Паттерн:** Строите сложные фильтры из простых строительных блоков.

**Преимущества:** Переиспользуемость, тестируемость, удобство сопровождения.

### 5. Производительность

**Ключевые оптимизации:**
- Ставьте избирательные фильтры первыми
- Однопроходная фильтрация
- Кэширование функций фильтров
- Мониторинг избирательности

---

## Какую стратегию выбрать

| Сценарий | Стратегия | Пример |
|----------|----------|--------|
| Простая категоризация | Одно поле | category = "programming" |
| Точное целевое попадание | Несколько полей И | category + difficulty + year |
| Широкое совпадение | Несколько полей ИЛИ | category = X or Y |
| Цены, даты | Фильтрация по диапазону | price between 50-100 |
| Теги, многокатегорийность | Фильтрация по массиву | has tag "react" |
| Удаление нежелательного | Исключающая фильтрация | difficulty != "advanced" |
| Сложные требования | Вложенные условия | (A && B) \|\| (C && D) |
| Пользовательский ввод | Динамические фильтры | Build from user input |

---

## Итог

### Что мы узнали

**Фильтрация по метаданным позволяет:**
- ✅ Точное сужение результатов
- ✅ Многомерный поиск
- ✅ Пользовательская настройка
- ✅ Контроль доступа и разрешения
- ✅ Фильтры по качеству и актуальности

**Ключевые стратегии:**
1. **Одно поле:** Простая категоризация
2. **Несколько полей:** Комбинации И/ИЛИ
3. **Диапазон:** Числовые границы
4. **Теги:** Многозначная фильтрация
5. **Исключение:** Удаление нежелательного
6. **Избыточная выборка:** Надёжное количество результатов

**Советы по производительности:**
- Сначала избирательные фильтры
- Однопроходная фильтрация
- Кэширование функций фильтров
- Мониторинг статистики

### Рекомендуемые практики

1. **Проектируйте богатые метаданные** со всеми фильтруемыми атрибутами
2. **Проверяйте наличие полей** перед обращением к ним
3. **Извлекайте на 2–3 раза больше** при фильтрации
4. **Комбинируйте фильтры** для переиспользуемости
5. **Тестируйте логику фильтров** независимо
6. **Предоставляйте интерфейс** для удобной фильтрации

### Следующие шаги

1. **Текущий:** Стратегии фильтрации по метаданным ✓
2. **Следующий:** Применение к вашей конкретной области
3. **В дальнейшем:** Производственная RAG-система с персистентностью

Сочетание семантического поиска и фильтрации по метаданным является основой современных поисковых систем. Независимо от того, создаёте ли вы систему электронной коммерции, контент-платформу или базу знаний, эти паттерны вам отлично послужат.

Освойте фильтрацию по метаданным — и вы создадите поисковые системы, которые будут одновременно интеллектуальными и точными!
