# Глоссарий терминов RAG from Scratch (RU)

Словарь утверждённых переводов технических терминов.
Все агенты переводческого конвейера **обязаны** следовать этому глоссарию.

---

## Основные термины RAG

| Английский | Русский | Примечание |
|-----------|---------|------------|
| RAG (Retrieval-Augmented Generation) | RAG | не переводится |
| retrieval | ретривер / извлечение | контекстно |
| generation | генерация | |
| augmentation | аугментация | |
| embedding | эмбеддинг | НЕ «вложение» |
| vector | вектор | |
| vector store / vector database | векторное хранилище / векторная БД | |
| similarity | сходство / similarity | |
| cosine similarity | косинусное сходство | |
| chunk / chunking | чанк / нарезка | разбиение текста на фрагменты |
| context window | контекстное окно | |
| prompt | промпт | |
| prompt template | шаблон промпта | |
| hallucination | галлюцинация | |
| grounding | привязка к данным | |
| token | токен | |
| tokenizer | токенизатор | |
| LLM | LLM | не переводится |
| inference | инференс | НЕ «вывод» |
| model | модель | |
| pipeline | конвейер / пайплайн | |
| knowledge base | база знаний | |
| document | документ | |
| query | запрос | |
| answer | ответ | |

## Поиск и ретривер

| Английский | Русский | Примечание |
|-----------|---------|------------|
| keyword search | поисковый запрос / keyword search | |
| semantic search | семантический поиск | |
| hybrid search | гибридный поиск | |
| nearest neighbor | ближайший сосед | |
| k-nearest neighbors (KNN) | k ближайших соседей (KNN) | |
| top-k | top-k | не переводится |
| BM25 | BM25 | не переводится |
| reranking | переупорядочивание / reranking | |
| re-ranking | переупорядочивание | |
| reciprocal rank fusion (RRF) | консенсусный рейтинг (RRF) | |
| score normalization | нормализация оценок | |
| relevance | релевантность | |
| precision | точность (precision) | |
| recall | полнота (recall) | |

## Векторные хранилища

| Английский | Русский | Примечание |
|-----------|---------|------------|
| vector store | векторное хранилище | |
| in-memory store | хранилище в памяти | |
| LanceDB | LanceDB | не переводится |
| Qdrant | Qdrant | не переводится |
| Pinecone | Pinecone | не переводится |
| Weaviate | Weaviate | не переводится |
| Chroma | Chroma | не переводится |
| metadata | метаданные | |
| metadata filtering | фильтрация по метаданным | |
| index | индекс | |

## Текстовая обработка

| Английский | Русский | Примечание |
|-----------|---------|------------|
| text splitting | разбиение текста | |
| text chunking | нарезка текста | |
| recursive character text splitter | рекурсивный символьный сплиттер | |
| token text splitter | токенный сплиттер | |
| overlap | перекрытие | |
| separator | разделитель | |
| delimiter | разделитель | |
| preprocessing | предобработка | |
| normalization | нормализация | |
| stopword | стоп-слово | |

## Загрузка данных

| Английский | Русский | Примечание |
|-----------|---------|------------|
| loader | загрузчик | |
| PDF loader | загрузчик PDF | |
| text loader | загрузчик текста | |
| directory loader | загрузчик директории | |
| data source | источник данных | |
| ingestion | загрузка / поглощение | |

## Модели и LLM

| Английский | Русский | Примечание |
|-----------|---------|------------|
| LLM | LLM | не переводится |
| local LLM | локальный LLM | |
| foundation model | фундаментальная модель | |
| embedding model | модель эмбеддингов | |
| cross-encoder | кросс-энкодер | |
| bi-encoder | би-энкодер | |
| quantization | квантизация | |
| GGUF | GGUF | не переводится |
| llama.cpp | llama.cpp | не переводится |
| node-llama-cpp | node-llama-cpp | не переводится |

## Цепочки и конвейеры

| Английский | Русский | Примечание |
|-----------|---------|------------|
| chain | цепочка | |
| retrieval chain | цепочка ретривера | |
| RAG chain | RAG-цепочка | |
| conversational chain | диалоговая цепочка | |
| conversational memory | диалоговая память | |
| context stuffing | наполнение контекста | |
| context compression | сжатие контекста | |

## Оптимизация запросов

| Английский | Русский | Примечание |
|-----------|---------|------------|
| query rewriting | перезапрос / рефразировка запроса | |
| query expansion | расширение запроса | |
| query decomposition | декомпозиция запроса | |
| intent classification | классификация намерений | |
| multi-query retrieval | мультизапросный ретривер | |

## Программирование (Node.js)

| Английский | Русский | Примечание |
|-----------|---------|------------|
| JavaScript | JavaScript | не переводится |
| Node.js | Node.js | не переводится |
| npm | npm | не переводится |
| ES modules | ES-модули | |
| class | класс | |
| function | функция | |
| array | массив | |
| object | объект | |

---

## Правила перевода терминов

1. **Устоявшийся перевод есть** → используем его (embedding → эмбеддинг, similarity → сходство)
2. **Нет устоявшегося** → транслитерация (token → токен, chunk → чанк)
3. **Аббревиатуры** → без перевода (LLM, GPU, RAG, KNN, BM25)
4. **Названия библиотек/инструментов** → без перевода (LanceDB, Qdrant, llama.cpp)
5. **Глаголы** → адаптируем (to retrieve → извлекать, to chunk → нарезать)

---

*Глоссарий адаптирован из ai-engineering-from-scratch*
