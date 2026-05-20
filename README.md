# Academic Research Skills — Hermes Agent

Портированная версия [ARS v3.9.4.2](https://github.com/Imbad0202/academic-research-skills) для Hermes Agent.

**Что это:** 38 агентов для академического исследования — от темы до готовой статьи с рецензированием.

**Язык:** русский (промпты, термины на английском). Формат: только Markdown.

---

## Быстрый старт

### 1. Установка

```bash
# Клонировать репо
git clone https://github.com/perejaslav/academic-research-skills.git
cd academic-research-skills

# Скопировать скилл в Hermes
cp -r hermes/skills/academic-research ~/.hermes/skills/

# Скопировать в профили (если используете несколько)
for p in ars-researcher ars-writer ars-reviewer ars-orchestrator; do
    mkdir -p ~/.hermes/profiles/$p/skills/
    cp -r hermes/skills/academic-research ~/.hermes/profiles/$p/skills/
done

# Проверить установку
hermes skills list | grep academic
```

### 2. Профили

Создайте 4 профиля для полного пайплайна:

```bash
hermes profile create ars-researcher --model mimo-v2.5-pro
hermes profile create ars-writer --model mimo-v2.5-pro
hermes profile create ars-reviewer --model mimo-v2.5-pro
hermes profile create ars-orchestrator --model mimo-v2.5-pro
```

### 3. Запуск

**Быстрый запрос (один агент):**
```
hermes chat -q "Проведи исследование по теме: LLM в Яндекс.Директ"
```

**Через Kanban (полный пайплайн T1→T10):**
```bash
hermes gateway start
hermes kanban ls  # посмотреть карточки
```

---

## Архитектура

```
┌──────────────────────────────────────────────┐
│         KANBAN ORCHESTRATOR (SKILL.md)        │
│         10 карточек с зависимостями           │
└──────────────┬─────────────────┬──────────────┘
               │                 │
    ┌──────────▼──────┐   ┌──────▼──────────┐
    │  ars-researcher │   │   ars-writer    │
    │  (T1: Research)  │   │  (T2: Write)    │
    └──────────┬───────┘   └──────┬──────────┘
               │                 │
    ┌──────────▼──────┐   ┌──────▼──────────┐
    │ ars-orchestrator │   │  ars-reviewer   │
    │ (T3: Integrity) │   │  (T4: Review)   │
    └──────────────────┘   └─────────────────┘

Внутри каждой карточки: delegate_task() → микро-агенты
```

### Модули (38 агентов)

| Модуль | Агентов | Назначение |
|--------|---------|------------|
| deep-research | 14 | Поиск литературы, синтез, отчёт |
| academic-paper | 12 | Написание статьи (intake→formatter) |
| academic-paper-reviewer | 7 | Мультиперспективная рецензия |
| academic-pipeline | 5 | Оркестрация, целостность, трекинг |

### Режимы работы

| Режим | Когда | Агенты | Выход |
|-------|-------|--------|-------|
| `socratic` | Нет темы | RQ + Socratic Mentor | План исследования |
| `full` | Есть тема | Все 9 | Статья 3000-8000 слов |
| `quick` | Нужна справка | RQ + Bibliography | Бриф 500-1500 слов |
| `lit-review` | Только обзор | Bibliography + Synthesis | Аннотированная библиография |
| `review` | Есть текст | Editor + DA + Ethics | Рецензия |
| `fact-check` | Проверка фактов | Source Verification | Отчёт 300-800 слов |

---

## Источники

Каскадный поиск (академические → авторитетные):

1. **Киберленинка** — `web_search("site:cyberleninka.ru KEYWORDS")`
2. **eLibrary/РИНЦ** — `web_search("site:elibrary.ru KEYWORDS")`
3. **Google Scholar** — `web_search("KEYWORDS научная статья")`
4. **Авторитетные неакадемические** — vedomosti.ru, rbc.ru, gov.ru
5. **Semantic Scholar** — fallback, API без ключа (1 req/sec)
6. **arXiv** — для ML/AI препринтов

---

## Структура файлов

```
hermes/
├── SKILL.md                    # Главный оркестратор
├── agents/
│   ├── deep-research/         # 14 агентов исследования
│   ├── academic-paper/        # 12 агентов написания
│   ├── academic-paper-reviewer/  # 7 агентов рецензии
│   └── academic-pipeline/     # 5 агентов пайплайна
├── references/                # 74 файла (справочники)
└── templates/                 # 19 файлов (шаблоны)
```

---

## Конфигурация

**Модель:** mimo-v2.5-pro (по умолчанию)

**Формат вывода:** только Markdown (без Pandoc/DOCX/PDF)

**Язык агентов:** русский. Термины на английском (APA, IMRaD, FINER, etc.)

---

## Kanban Pipeline (10 карточек)

```
T1  RESEARCH    [ars-researcher]  — нет родителей
T2  WRITE       [ars-writer]     — parents: [T1]
T3  INTEGRITY    [ars-orchestrator] — parents: [T2]
T4  REVIEW       [ars-reviewer]   — parents: [T3]
T5  DECISION     [—]              — parents: [T4]  ← kanban_block
T6  REVISE       [ars-writer]     — parents: [T5]
T7  RE-REVIEW    [ars-reviewer]   — parents: [T6]
T8  FINAL DEC    [—]              — parents: [T7]  ← kanban_block
T9  FINALIZE     [ars-writer]     — parents: [T8]
T10 SUMMARY      [ars-orchestrator] — parents: [T9]
```

Чекпоинты T5 и T8 — `kanban_block()` для решения человека (accept/minor/major/reject).

---

## Тесты

Тесты модулей пройдены (delegate_task, 4 модуля, 12 агентов):

| Модуль | Тест | Статус |
|--------|------|--------|
| academic-paper | intake_agent | ✅ 32с |
| academic-paper | draft_writer_agent | ✅ 288с, ~1500 слов |
| reviewer | field_analyst_agent | ✅ 91с |
| reviewer | editorial_synthesizer_agent | ✅ 42с |
| pipeline | state_tracker_agent | ✅ 25с |
| pipeline | integrity_verification_agent | ✅ 226с |

---

## Разработка

**Ветка:** `hermes-port` (форк: perejaslav/academic-research-skills)

**Commits:** b5b80fa, 8ece9bf (финальная версия)

```bash
git clone https://github.com/perejaslav/academic-research-skills.git
git checkout hermes-port
```

---

## Оригинал

[Imbad0202/academic-research-skills](https://github.com/Imbad0202/academic-research-skills) — Claude Code плагин, ARS v3.9.4.2