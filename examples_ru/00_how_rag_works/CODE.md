# Как работает RAG — разбор кода

Пошаговое объяснение минимальной системы RAG (Retrieval-Augmented Generation, генерация с усилением ретривером) с использованием наивного поискового запроса.

## Обзор

Этот пример демонстрирует **основную концепцию RAG** максимально просто:
- Без эмбеддингов (embedding) и векторных баз данных
- Без внешних API или моделей
- Всего 3 функции, показывающие, как ретривер (retrieval) и генерация (generation) работают вместе

Цель: увидеть, как работает RAG, менее чем за 70 строк кода, до погружения в векторы и эмбеддинги.

---

## База знаний

```javascript
const knowledge = [
    "Underwhelming Spatula is a kitchen tool that redefines expectations by fusing whimsy with functionality.",
    "Lisa Melton wrote Dubious Parenting Tips.",
    "The Almost-Perfect Investment Guide is 210 pages long.",
    "Quantum computing uses qubits instead of classical bits.",
    "The capital of France is Paris."
];
```

**Что это:** простой массив строк, представляющий вашу «базу данных» фактов.

**В реальной системе RAG:** здесь было бы:
- Тысячи фрагментов документов
- Хранимых в векторной базе данных
- Каждый с эмбеддингом (числовым представлением)

**В этом примере:** мы giữаем всё просто, чтобы сосредоточиться на паттерне «ретривер + генерация».

---

## Шаг 1: Ретривер (наивный поисковый запрос)

```javascript
function naiveKeywordSearch(query, documents, topK = 2) {
    const queryWords = query.toLowerCase().split(/\s+/);
    
    // Score each document by counting keyword matches
    const scored = documents.map(doc => {
        const docWords = doc.toLowerCase().split(/\s+/);
        const matches = queryWords.filter(word => docWords.includes(word)).length;
        return { doc, score: matches };
    });
    
    // Sort by score (highest first) and return top K
    return scored
        .sort((a, b) => b.score - a.score)
        .slice(0, topK)
        .filter(item => item.score > 0)
        .map(item => item.doc);
}
```

### Как это работает:

1. **Парсинг запроса (query)**
   ```javascript
   const queryWords = query.toLowerCase().split(/\s+/);
   ```
   - Преобразует запрос в нижний регистр
   - Разбивает на отдельные слова
   - Пример: «What is Underwhelming Spatula?» → `["what", "is", "underwhelming", "spatula?"]`

2. **Оценка каждого документа**
   ```javascript
   const scored = documents.map(doc => {
       const docWords = doc.toLowerCase().split(/\s+/);
       const matches = queryWords.filter(word => docWords.includes(word)).length;
       return { doc, score: matches };
   });
   ```
   - Для каждого документа разбивает текст на слова
   - Считает, сколько слов из запроса встречается в документе
   - Чем выше оценка (score), тем больше совпадений ключевых слов

3. **Ранжирование и фильтрация**
   ```javascript
   return scored
       .sort((a, b) => b.score - a.score)  // Сортировка по оценке (убывание)
       .slice(0, topK)                      // Берём топ K результатов
       .filter(item => item.score > 0)     // Убираем документы без совпадений
       .map(item => item.doc);              // Возвращаем только документы
   ```

### Пример:

**Запрос:** «What is Underwhelming Spatula?»

**Оценка:**
- Документ 1: «Underwhelming Spatula is a kitchen...» → Оценка: 3 («underwhelming», «spatula», «is»)
- Документ 2: «Lisa Melton wrote...» → Оценка: 0
- Документ 3: «The Almost-Perfect Investment...» → Оценка: 1 («is»)
- Документ 4: «Quantum computing uses...» → Оценка: 0
- Документ 5: «The capital of France...» → Оценка: 1 («is»)

**Результат:** Возвращает Документ 1 (наивысшая оценка)

### Ограничения наивного поискового запроса:

❌ **Нет семантического понимания**
- Запрос: «Who is the author of Dubious Parenting Tips?»
- Не связывает «wrote» с «author»

❌ **Порядок слов не важен**
- «dog bites man» получает ту же оценку, что и «man bites dog»

❌ **Нет обработки синонимов**
- «automobile» не совпадёт с «car»

**Почему мы используем его здесь:** он прост и демонстрирует основную концепцию ретривера без необходимости в эмбеддингах или моделях машинного обучения.

**Что лучше:** эмбеддинги + косинусное сходство (cosine similarity) — обсуждается в последующих примерах.

---

## Шаг 2: Генерация (симуляция)

```javascript
function generateAnswer(query, context) {
    if (context.length === 0) {
        return "I don't have enough information to answer that question.";
    }
    
    // In a real RAG system, this would call an LLM with the context
    // For now, we just return the most relevant context
    return `Based on the available information:\n\n${context.join('\n\n')}`;
}
```

### Как это работает:

1. **Проверка наличия контекста**
   ```javascript
   if (context.length === 0) {
       return "I don't have enough information to answer that question.";
   }
   ```
   - Если ретривер ничего не нашёл, честно признаём это
   - Предотвращает галлюцинации (hallucination) — выдумывание ответов

2. **Возврат контекста**
   ```javascript
   return `Based on the available information:\n\n${context.join('\n\n')}`;
   ```
   - Форматирует найденные документы как ответ
   - В реальной системе это передавалось бы в LLM

### В реальной системе RAG:

```javascript
function generateAnswer(query, context) {
    const prompt = `
        Answer the question based on the following context:
        
        Context:
        ${context.join('\n\n')}
        
        Question: ${query}
        
        Answer:
    `;
    
    return await callLLM(prompt);  // e.g., OpenAI, Claude, or local LLM
}
```

**LLM (большая языковая модель) would:**
- Прочитать извлечённый контекст
- Понять вопрос
- Синтезировать ответ на основе предоставленной информации
- Оставаться привязанным к данным (grounding) — не выдумывать

**В этом примере:** мы пропускаем LLM, чтобы сохранить простоту и отсутствие зависимостей.

---

## Шаг 3: Конвейер RAG (RAG Pipeline)

```javascript
function ragPipeline(query) {
    console.log(`\n📝 Question: ${query}`);
    console.log(`─────────────────────────────────────`);
    
    // Retrieve relevant documents
    const relevantDocs = naiveKeywordSearch(query, knowledge);
    console.log(`\n🔍 Retrieved ${relevantDocs.length} relevant document(s)`);
    
    // Generate answer using retrieved context
    const answer = generateAnswer(query, relevantDocs);
    console.log(`\n💡 Answer:\n${answer}\n`);
    
    return answer;
}
```

### Паттерн RAG:

**Вход:** запрос пользователя

↓

**Шаг 1 — Ретривер:** найти релевантные документы из базы знаний

↓

**Шаг 2 — Генерация:** использовать извлечённый контекст для создания ответа

↓

**Выход:** привязанный к данным ответ (на основе реальной информации)

### Почему это важно:

**Без RAG (только LLM):**
```
User: «What is Underwhelming Spatula?»
LLM: «I'm not sure, that sounds like a made-up product...»
```

**С RAG:**
```
User: «What is Underwhelming Spatula?»
System: [Извлекает релевантный документ]
LLM: «Underwhelming Spatula is a kitchen tool that redefines expectations...»
```

### Преимущества:
- ✅ **Фактические ответы:** основаны на ваших данных, а не на обучающих данных LLM
- ✅ **Актуальность:** добавляйте новые документы без переобучения
- ✅ **Прозрачность:** можно видеть, какие документы были использованы
- ✅ **Доменная специфика:** работает с проприетарными/специализированными знаниями

---

## Примеры вывода

### Запрос 1: «What is Underwhelming Spatula?»
```
📝 Question: What is Underwhelming Spatula?
─────────────────────────────────────

🔍 Retrieved 1 relevant document(s)

💡 Answer:
Based on the available information:

Underwhelming Spatula is a kitchen tool that redefines expectations by fusing whimsy with functionality.
```

**Почему это работает:** ключевые слова «Underwhelming» и «Spatula» напрямую совпадают с документом.

---

### Запрос 2: «Who wrote Dubious Parenting Tips?»
```
📝 Question: Who wrote Dubious Parenting Tips?
─────────────────────────────────────

🔍 Retrieved 1 relevant document(s)

💡 Answer:
Based on the available information:

Lisa Melton wrote Dubious Parenting Tips.
```

**Почему это работает:** ключевые слова «Dubious Parenting Tips» точно совпадают.

---

### Запрос 3: «What is the weather today?»
```
📝 Question: What is the weather today?
─────────────────────────────────────

🔍 Retrieved 0 relevant document(s)

💡 Answer:
I don't have enough information to answer that question.
```

**Почему это не работает:** ни один документ в нашей базе знаний не упоминает «weather» или «today».

**Это правильное поведение:** система не галлюцинирует ответ.

---

## Ключевые концепции

### 1. Ретривер ≠ поисковые системы

**Традиционный поиск:**
- Возвращает ссылки на документы
- Пользователь читает документы сам

**Ретривер в RAG:**
- Возвращает содержимое документов
- Система использует содержимое для генерации ответа

---

### 2. Контекстное окно имеет значение

**Проблема:** LLM имеют ограниченные контекстные окна (context window) (например, 4K–128K токенов)

**Почему ретривер помогает:**
- У вас может быть миллионы документов
- LLM может прочитать только несколько за раз
- Ретривер находит **самые релевантные**

**Пример:**
```
Knowledge base: 10,000 документов (10M токенов)
Контекстное окно LLM: 8K токенов
Ретривер: находит топ 5 наиболее релевантных (2K токенов)
Результат: LLM получает ровно то, что нужно
```

---

### 3. Формула RAG

```
Ответ = LLM(Запрос + Извлечённый_контекст)
```

**Без контекста:**
```
Ответ = LLM(Запрос)  ← Возможны галлюцинации
```

**С RAG:**
```
Ответ = LLM(Запрос + Извлечённый_контекст)  ← Привязано к фактам
```

---

## Что дальше?

Этот пример показывает **концепцию**, но реальный RAG требует:

1. **Лучший ретривер** → эмбеддинги + векторный поиск (следующие примеры)
2. **Реальная генерация** → интеграция LLM (OpenAI, Claude, локальные модели)
3. **Стратегия нарезки (chunking)** → разбиение больших документов на поисковые фрагменты
4. **Ранжирование/переупорядочивание (reranking)** → улучшение качества ретривера
5. **Инжиниринг промптов (prompt engineering)** → научить LLM эффективно использовать контекст

Всё это рассматривается в последующих примерах этого репозитория.

---

## Запуск примера

```bash
node examples/00_how_rag_works/example.js
```

**Никаких зависимостей не требуется!** Этот пример использует только встроенные возможности Node.js.

---

## Резюме

Этот 69-строчный пример демонстрирует:
- ✅ Основной паттерн RAG (ретривер + генерация)
- ✅ Как ретривер находит релевантную информацию
- ✅ Как контекст улучшает качество ответов
- ✅ Почему RAG предотвращает галлюцинации

**Следующий шаг:** узнайте, что такое LLM и как использовать node-llama-cpp для запуска LLM локально — ядро каждой системы RAG.
