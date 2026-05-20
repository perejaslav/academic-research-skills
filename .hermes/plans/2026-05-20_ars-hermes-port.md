# ARS → Hermes Agent Porting Plan

**Цель**: Портировать Academic Research Skills (ARS) v3.9.4.2 из Claude Code плагина в Hermes Agent skill с Kanban-оркестрацией.

**Репо**: https://github.com/perejaslav/academic-research-skills (ветка `hermes-port`)

**Дата**: 2026-05-20

---

## Текущее состояние

| Метрика | Кол-во |
|---------|--------|
| Модулей | 4 (deep-research, academic-paper, academic-paper-reviewer, academic-pipeline) |
| Агентов | 38 (14 + 12 + 7 + 5) |
| Справочников | 83 файла |
| Шаблонов | 20 файлов |
| Скриптов | 127 файлов |
| Всего файлов | 765 |

---

## Архитектура решения

```
┌─────────────────────────────────────────────────────┐
│                 KANBAN ORCHESTRATOR                  │
│  (SKILL.md — макро-пайплайн, 10 стадий)             │
│  Карточки: T1→T2→T3→...→T10 с parents/dependencies │
│  kanban_block() для чекпоинтов человека              │
└────────────┬──────────────────────────┬─────────────┘
             │ delegate_task            │ delegate_task
     ┌───────▼───────┐        ┌────────▼────────┐
     │  deep-research │        │ academic-paper   │ ...
     │  (13 агентов)  │        │ (12 агентов)    │
     │  внутри 1      │        │ внутри 1        │
     │  воркера       │        │ воркера         │
     └────────────────┘        └─────────────────┘
```

**Принцип**: Kanban управляет МАКРО-пайплайном (10 стадий с зависимостями и чекпоинтами). Внутри каждой карточки — delegate_task() для МИКРО-работы (конкретные агенты).

---

## Фазы реализации

### ФАЗА 0: Подготовка инфраструктуры (сегодня)

**Задачи:**
- [ ] 0.1 — Создать Hermes-профили для Kanban-воркеров
- [ ] 0.2 — Создать структуру директорий в форке
- [ ] 0.3 — Настроить ветку и .gitignore

**Профили Hermes для Kanban:**

| Профиль | Назначение | Модель | Скиллы |
|---------|-----------|--------|--------|
| `ars-researcher` | deep-research (поиск литературы, источники) | mimo-v2.5-pro | academic-research, arxiv |
| `ars-writer` | academic-paper (написание, форматирование) | mimo-v2.5-pro | academic-research |
| `ars-reviewer` | academic-paper-reviewer (peer review) | mimo-v2.5-pro | academic-research |
| `ars-orchestrator` | Kanban-оркestrатор (декомпозиция, маршрутизация) | mimo-v2.5-pro | kanban-orchestrator, academic-research |

> Все профили на одной модели для начала. Потом можно развести на разные модели.

**Структура директорий (в форке):**

```
hermes/
  SKILL.md                          ← Оркestrатор (из academic-pipeline)
  agents/
    deep-research/
      research_question_agent.md    ← 14 агентов модуля
      bibliography_agent.md
      ...
    academic-paper/
      intake_agent.md               ← 12 агентов модуля
      draft_writer_agent.md
      ...
    academic-paper-reviewer/
      field_analyst_agent.md        ← 7 агентов модуля
      eic_agent.md
      ...
    academic-pipeline/
      pipeline_orchestrator_agent.md ← 5 агентов оркестрации
      integrity_verification_agent.md
      ...
  references/
    deep-research/                  ← 22 файла
    academic-paper/                 ← 25 файлов
    academic-paper-reviewer/        ← 12 файлов
    academic-pipeline/              ← 19 файлов
    shared/                         ← общие: handoff_schemas, mode_spectrum, etc.
  templates/
    deep-research/                  ← 6 файлов
    academic-paper/                 ← 11 файлов
    academic-paper-reviewer/        ← 3 файла
  scripts/
    adapters/                       ← Python-адаптеры (Zotero, etc.)
    validators/                     ← Скрипты валидации
```

---

### ФАЗА 1: Портирование deep-research (14 агентов)

**Приоритет**: Самый автономный модуль. Минимум зависимостей.

**Задачи:**
- [ ] 1.1 — Скопировать SKILL.md, адаптировать frontmatter под Hermes
- [ ] 1.2 — Скопировать 14 агентов в hermes/agents/deep-research/
  - Убрать Claude Code Phase Boundary v3.9.2 (не нужен — изоляция через delegate_task)
  - Сохранить: роль, принципы, пошаговые протоколы, Decision Tree
  - Адаптировать: триггеры (добавить русские ключевые слова)
- [ ] 1.3 — Скопировать 22 references/ файла
  - semantic_scholar_api_protocol.md — проверить, что curl-команды работают из Hermes
  - openalex_api_protocol.md, crossref_api_protocol.md — аналогично
  - Остальные — чистый markdown, копировать as-is
- [ ] 1.4 — Скопировать 6 templates/
- [ ] 1.5 — Переписать оркестрацию deep-research как delegate_task-цепочку:

```python
# Внутри SKILL.md deep-research:
# Phase 1: Scoping (параллельно)
rq_brief = delegate_task(
    goal="Сформулировать исследовательский вопрос",
    context=f"{agent_prompt}\n\nТЕМА: {topic}\n\nФОРМАТ: RQ Brief (Schema 1)"
)
methodology = delegate_task(
    goal="Разработать методологический план",
    context=f"{agent_prompt}\n\nRQ BRIEF: {rq_brief}"
)
# Phase 2: Investigation
bibliography = delegate_task(
    goal="Провести систематический поиск литературы",
    context=f"{agent_prompt}\n\nRQ: {rq_brief}\nMETHODOLOGY: {methodology}"
)
# ... и т.д. по 6 фазам
```

- [ ] 1.6 — Протестировать на реальной задаче (1 исследовательский вопрос)
- [ ] 1.7 — Зафиксировать проблемы, патчить агентов

**Что НЕ переносить:**
- `.claude/CLAUDE.md` routing rules
- Phase Boundary v3.9.2 проверки (замена: изоляция delegate_task)
- hooks/hooks.json

---

### ФАЗА 2: Портирование academic-paper (12 агентов)

**Зависит от**: deep-research (RQ Brief, Annotated Bibliography, Synthesis Report)

**Задачи:**
- [ ] 2.1 — Скопировать SKILL.md, адаптировать frontmatter
- [ ] 2.2 — Скопировать 12 агентов
  - intake_agent: оставить Deep Research Handoff Detection логику
  - draft_writer_agent: сохранить Writing Quality Check, Style Calibration
  - formatter_agent: адаптировать под Hermes-инструменты (terminal для pandoc/tectonic)
  - citation_compliance_agent: сохранить APA 7.0 протоколы
- [ ] 2.3 — Скопировать 25 references/
  - apa7_extended_guide.md — критически важный, as-is
  - writing_quality_check.md — as-is
  - anti_leakage_protocol.md — as-is
  - citation_format_switcher.md — as-is
- [ ] 2.4 — Скопировать 11 templates/
  - imrad_template.md — as-is
  - latex_article_template.tex — as-is
- [ ] 2.5 — Переписать оркестрацию (delegate_task-цепочка из 7 фаз)
- [ ] 2.6 — Протестировать: написать короткую статью (2000 слов) по готовой литературе
- [ ] 2.7 — Патч

---

### ФАЗА 3: Портирование academic-paper-reviewer (7 агентов)

**Зависит от**: academic-paper (draft)

**Задачи:**
- [ ] 3.1 — Скопировать SKILL.md
- [ ] 3.2 — Скопировать 7 агентов
  - field_analyst_agent: динамическая настройка рецензентов
  - eic_agent, methodology_reviewer_agent, domain_reviewer_agent, perspective_reviewer_agent: 4 параллельных рецензента
  - devils_advocate_reviewer_agent: адверсариальный обзор
  - editorial_synthesizer_agent: синтез решений
- [ ] 3.3 — Скопировать 12 references/ + 3 templates/
- [ ] 3.4 — Оркестрация: fan-out (4 рецензента параллельно) → fan-in (синтез)

```python
# Параллельные рецензенты
reviews = delegate_task(tasks=[
    {"goal": "Рецензия: методология", "context": f"{methodology_prompt}\n\n{paper}"},
    {"goal": "Рецензия: доменная экспертиза", "context": f"{domain_prompt}\n\n{paper}"},
    {"goal": "Рецензия: междисциплинарный взгляд", "context": f"{perspective_prompt}\n\n{paper}"},
    {"goal": "Адвокат дьявола: атака на основные аргументы", "context": f"{da_prompt}\n\n{paper}"},
])
# Синтез решений
decision = delegate_task(
    goal="Синтезировать рецензии в редакционное решение",
    context=f"{synthesizer_prompt}\n\nREVIEWS: {reviews}\n\nPAPER: {paper}"
)
```

- [ ] 3.5 — Протестировать: прогнать статью из Фазы 2 через ревью
- [ ] 3.6 — Патч

---

### ФАЗА 4: Создание Hermes SKILL.md-оркестратора

**Зависит от**: все 3 модуля (Фазы 1-3)

**Задачи:**
- [ ] 4.1 — Создать SKILL.md — главный файл скилла academic-research
  - Trigger conditions (русские + английские ключевые слова)
  - Pipeline overview (10 стадий)
  - Mode selection (full, plan, quick, revision, etc.)
  - Kanban task graph template
  - Delegate_task шаблоны для каждой стадии
- [ ] 4.2 — Адаптировать pipeline_orchestrator_agent.md
  - Убрать Claude Code специфику
  - Оставить: intent detection, mid-entry routing, Material Passport
  - Добавить: Kanban-специфичные инструкции (kanban_create, kanban_block, parents)
- [ ] 4.3 — Адаптировать integrity_verification_agent.md
  - 7-mode failure checklist → delegate_task с валидацией
- [ ] 4.4 — Адаптировать shared/ файлы:
  - handoff_schemas.md — as-is (критически важен)
  - style_calibration_protocol.md — as-is
  - mode_spectrum.md — as-is
  - compliance_checkpoint_protocol.md — as-is
- [ ] 4.5 — Адаптировать скрипты валидации (из scripts/)
  - check_pipeline_integrity.py — адаптировать под Hermes terminal
  - check_data_access_level.py — as-is
- [ ] 4.6 — Создать Kanban-специфичный раздел в SKILL.md:

```markdown
## Kanban Pipeline Mode

Когда запускается через Kanban, пайплайн разбивается на карточки:

| # | Карточка | Профиль | Parents | Блокировка |
|---|----------|---------|---------|------------|
| T1 | RESEARCH | ars-researcher | — | — |
| T2 | WRITE | ars-writer | T1 | — |
| T3 | INTEGRITY | ars-orchestrator | T2 | — |
| T4 | REVIEW | ars-reviewer | T3 | — |
| T5 | DECISION | — | T4 | kanban_block: человек решает |
| T6 | REVISE | ars-writer | T5 | — |
| T7 | RE-REVIEW | ars-reviewer | T6 | — |
| T8 | FINAL DECISION | — | T7 | kanban_block: человек |
| T9 | FINALIZE | ars-writer | T8 | — |
| T10 | SUMMARY | ars-orchestrator | T9 | — |
```

---

### ФАЗА 5: Интеграция с Kanban

**Задачи:**
- [ ] 5.1 — Создать Hermes-профили через CLI:
```bash
hermes profile create ars-researcher --model mimo-v2.5-pro
hermes profile create ars-writer --model mimo-v2.5-pro
hermes profile create ars-reviewer --model mimo-v2.5-pro
hermes profile create ars-orchestrator --model mimo-v2.5-pro
```
- [ ] 5.2 — Привязать скилл academic-research к каждому профилю
- [ ] 5.3 — Настроить notification_sources для кросс-профильных уведомлений
- [ ] 5.4 — Протестировать полный Kanban-пайплайн:
  1. Создать T1 (RESEARCH) → ars-researcher
  2. Проверить что T2 (WRITE) автоматически promoted из todo → ready после T1
  3. Проверить что T5 (DECISION) блокируется через kanban_block
  4. Разблокировать вручную, проверить продолжение
- [ ] 5.5 — Патч

---

### ФАЗА 6: Русский язык и полировка

**Задачи:**
- [ ] 6.1 — Добавить русские триггеры во все SKILL.md
  - "исследование", "обзор литературы", "написать статью", "рецензия", etc.
- [ ] 6.2 — Добавить русские шаблоны (если нужно)
- [ ] 6.3 — Перевести ключевые интерфейсные сообщения агентов
  - Приветствия, запросы подтверждения, отчёты
- [ ] 6.4 — Тест полного цикла на русском языке

---

### ФАЗА 7: Документация и коммит

**Задачи:**
- [ ] 7.1 — Написать README для hermes/ директории
- [ ] 7.2 — Обновить корневой README (добавить секцию Hermes)
- [ ] 7.3 — Создать CHANGELOG записи
- [ ] 7.4 — Squash-коммит, push в hermes-port
- [ ] 7.5 — PR в upstream (опционально)

---

## Файлы которые изменятся

### Новые файлы (создаются в форке):
```
hermes/
  SKILL.md
  agents/deep-research/*.md        (14 файлов)
  agents/academic-paper/*.md       (12 файлов)
  agents/academic-paper-reviewer/*.md (7 файлов)
  agents/academic-pipeline/*.md    (5 файлов)
  references/**/*.md               (83 файла)
  templates/**/*.{md,tex}          (20 файлов)
  scripts/**/*.py                  (адаптированные)
README.hermes.md
```

### Не трогаем (оригинал ARS остаётся как есть):
```
deep-research/           ← оригинальный Claude Code модуль
academic-paper/
academic-paper-reviewer/
academic-pipeline/
.claude/
.cclaude-plugin/
```

---

## Риски и митигации

| Риск | Вероятность | Митигация |
|------|-------------|-----------|
| Агенты работают хуже на mimo vs Claude | Средняя | Тестировать каждого агента отдельно, патчить промпты |
| Токен-бюджет превышен (38 агентов × контекст) | Высокая | Ленивая загрузка — только нужные агенты в контекст |
| Kanban overhead для коротких стадий | Низкая | Мелкие стадии (2.5, 4.5) объединять с предыдущими |
| Semantic Scholar API rate limit | Низкая | 1 req/sec без ключа — достаточно для пайплайна |
| Handoff schemas ломаются при передаче контекста | Средняя | Строгая валидация в каждом delegate_task |
| Потеря качества при fan-out (4 рецензента) | Средняя | Сохранять paper целиком в context каждого |

---

## Порядок тестирования

1. **Unit**: каждый агент отдельно (дать тему → проверить выход)
2. **Module**: deep-research целиком (тема → RQ + библиография + синтез + отчёт)
3. **Cross-module**: deep-research → academic-paper (RQ → черновик)
4. **Full pipeline**: Kanban T1→T10 на маленькой статье (1500 слов)
5. **Regression**: повторить тест с другим topic/domain

---

## Решения (приняты 2026-05-20)

| Вопрос | Решение |
|--------|---------|
| Модель | mimo-v2.5-pro (дефолтная) для всех 4 профилей |
| Формат вывода | Только Markdown (Pandoc/tectonic — позже) |
| Источники | Приоритет: русские (Киберленинка, eLibrary/РИНЦ). Международные — fallback |
| Язык агентов | Русский (промпты, протоколы, отчёты). Английский — только для терминов |
| Semantic Scholar | Нет ключа. Используем без ключа (1 req/sec) как fallback |
| Киберленинка | Нет API — парсинг через web_search + web_extract |
| eLibrary/РИНЦ | Используем открытый API elibrary.ru |
| Форк | perejaslav/academic-research-skills, ветка hermes-port |

## Источниковая стратегия (замена оригинальной)

Оригинальный ARS: Semantic Scholar → OpenAlex → Crossref (все англоязычные)

Наша стратегия (каскад fallback):
```
1. Киберленинка — основной источник (парсинг, не API)
   Поиск: web_search("site:cyberleninka.ru KEYWORDS")
   Чтение: web_extract(urls=["https://cyberleninka.ru/article/n/SLUG"])
   Возвращает: заголовок, аннотацию, авторов, журнал
   Рейтинг: приоритет №1 — крупнейшая русскоязычная OA-библиотека

2. eLibrary/РИНЦ — второй источник (ограниченный доступ)
   Официальный API есть, но требует регистрации на elibrary.ru
   Fallback: web_search("site:elibrary.ru KEYWORDS") + web_extract
   Возвращает: метаданные, РИНЦ-индекс, цитирования
   Рейтинг: приоритет №2 — главная наукометрическая база РФ

3. Google Scholar (русский сегмент) — третий источник
   Поиск: web_search("KEYWORDS научная статья")
   Рейтинг: приоритет №3 — широкий охват, но шум

4. Semantic Scholar — fallback для англоязычных
   API: curl api.semanticscholar.org (без ключа, 1 req/sec)
   Рейтинг: приоритет №4 — только если русских источников мало

5. arXiv — для препринтов и ML/AI тематики
   Через skill arxiv (встроенный)
   Рейтинг: приоритет №5 — англоязычный, узкая тематика
```

---

## Следующий шаг

Начать Фазу 0: создать профили и структуру директорий.
