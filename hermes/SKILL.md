---
name: academic-research
description: "Быстрый академический пайплайн: от темы до статьи за 30-45 мин. 3 режима: instant (быстро), full (глубоко), legacy (Kanban-пайплайн)."
version: "2.0.0"
metadata:
  hermes:
    tags: [academic, research, writing, review, paper]
    platforms: [linux, macos, windows]
---

# Academic Research Skills v2

**Что делает:** от темы до готовой статьи за 30-45 минут. Поиск источников → написание → ревью → файл на Desktop.

**Не делает:** имитацию академического процесса с 38 агентами и 10 Kanban-карточками. Для этого есть `legacy`-режим.

---

## Режимы

| Режим | Когда | Время | Агентов |
|-------|-------|-------|---------|
| `instant` | Есть тема, нужна статья | ~30-45 мин | 3 (bibliography → writer → editor) |
| `full` | Сложная тема, нужна глубина | ~1-2 ч | 5-8 (core + situational) |
| `legacy` | Нужен Kanban-пайплайн | 3+ ч | 38 |

`instant` — по умолчанию. Используй его, если не сказано иное.

---

## Режим «instant» (основной)

### Схема

```
Тема → [bibliography: web_search, 5-10 источников]
     → [draft_writer: статья, APA 7.0]
     → [editor: рецензия, 3-5 замечаний]
     → [доработка если major/reject]
     → [file_output: Desktop]
```

### Команда для запуска

```
Напиши статью на тему: [тема]
```

Или явно:

```
instant: [тема], стиль: научно-публицистический, объём: ~3000 слов
```

### Агенты (core/)

- `agents/core/bibliography_agent.md` — поиск 5-10 источников (каскад: Киберленинка → eLibrary → Google Scholar → отраслевые)
- `agents/core/draft_writer_agent.md` — написание статьи от источников
- `agents/core/editor_agent.md` — рецензия, 3-5 конкретных замечаний
- `agents/core/file_output_agent.md` — запись на Desktop (Windows-safe)

### Референсы (references/core/)

- `references/core/source_search_strategy.md` — каскадная стратегия поиска
- `references/core/apa7_russian_guide.md` — краткий гайд APA 7.0
- `references/core/pipeline_rules.md` — правила пайплайна

### Процесс (внутренний)

```python
def instant_paper(topic: str, style: str = "academic", word_count: str = "3000-5000"):
    # Шаг 1: поиск источников
    sources = delegate_task(
        goal="Найти 5-10 источников по теме: " + topic,
        context=f"Тема: {topic}\nСтратегия: {source_search_strategy}",
        toolsets=["web"],
    )
    
    # Шаг 2: написание статьи
    draft = delegate_task(
        goal="Написать статью по теме: " + topic,
        context=f"Тема: {topic}\nИсточники: {sources['summary']}\n"
                f"Стиль: {style}\nОбъём: {word_count} слов\n"
                f"{apa7_guide}\n{pipeline_rules}",
        toolsets=[],
    )
    
    # Шаг 3: рецензия
    review = delegate_task(
        goal="Провести рецензию статьи",
        context=f"Статья:\n{draft['summary']}\n"
                f"Правила рецензии: {editor_agent}",
        toolsets=[],
    )
    
    # Шаг 4: доработка (если major/reject)
    if "major" in review['summary'].lower() or "reject" in review['summary'].lower():
        draft = delegate_task(
            goal="Доработать статью по замечаниям рецензента",
            context=f"Статья:\n{draft['summary']}\n"
                    f"Замечания:\n{review['summary']}",
            toolsets=[],
        )
    
    # Шаг 5: запись файла (через execute_code)
    save_to_desktop(draft['summary'], topic)
    
    return {
        "article": draft['summary'],
        "sources": sources['summary'],
        "review": review['summary'],
        "file": f"~/Desktop/СТАТЬЯ_{topic[:30]}.md"
    }
```

---

## Режим «full» (ситуационные агенты)

Когда instant-статьи недостаточно (объём >5000 слов, нужна методология, этическая экспертиза):

```
full: [тема], с методологическим планом и этической проверкой
```

Дополнительно загружаются агенты из `agents/situational/`:
- research_question_agent — формулировка RQ по FINER
- structure_architect_agent — детальная структура
- ethics_review_agent — этическая проверка
- и другие по необходимости

---

## Режим «legacy» (Kanban-пайплайн)

Полный порт оригинального ARS. 10 Kanban-карточек, 4 профиля, gateway, чекпоинты.

Для запуска:
```
legacy: [тема]
```

Требует:
- 4 профиля (ars-researcher, ars-writer, ars-reviewer, ars-orchestrator)
- Ручное создание Kanban-карточек
- Вмешательство на T5 и T8

Архивная документация в `agents/obsolete/` и `references/archive/`.

---

## Стратегия поиска источников

Каскад (русские академические → авторитетные → международные):

1. **Киберленинка** — `web_search("site:cyberleninka.ru КЛЮЧЕВЫЕ_СЛОВА")` + `web_extract`
2. **eLibrary/РИНЦ** — `web_search("site:elibrary.ru КЛЮЧЕВЫЕ_СЛОВА")`
3. **Google Scholar** — `web_search("КЛЮЧЕВЫЕ_СЛОВА научная статья")`
4. **Авторитетные отраслевые** — vc.ru, ppc.world, seonews, habr.com и т.д.
5. **Semantic Scholar** — fallback (без ключа, 1 req/sec)
6. **arXiv** — для препринтов ML/AI

Подробнее: `references/core/source_search_strategy.md`

---

## Установка (v2)

```bash
# Удалить профили (если были v1)
hermes config delete profiles.ars-researcher
hermes config delete profiles.ars-writer
hermes config delete profiles.ars-reviewer
hermes config delete profiles.ars-orchestrator

# Установить скилл
hermes skills add academic-research

# Готово — больше ничего не нужно
```
