---
name: academic-research
description: "Академический исследовательский пайплайн: поиск литературы, написание статей, рецензирование, доработка. 38 агентов, 6 режимов, Kanban-оркестрация. Триггеры: исследование, написать статью, обзор литературы, рецензия, научная работа, research, write paper, literature review."
version: "1.0.0"
metadata:
  hermes:
    tags: [academic, research, writing, review, kanban, literature]
    platforms: [linux, macos, windows]
---

# Academic Research Skills — Hermes Agent

Академический исследовательский пайплайн для Hermes Agent. Портировано из
[ARS v3.9.4.2](https://github.com/Imbad0202/academic-research-skills) (Claude Code).

**Что делает**: помогает исследователю пройти путь от темы до готовой статьи —
с поиском источников, написанием, ревью и доработкой. AI — соавтор, а не автор.

---

## Быстрый старт

**Полный пайплайн** (от темы до статьи):
```
Хочу написать исследовательскую статью о влиянии ИИ на маркетинг
```

**Только исследование** (поиск литературы):
```
Проведи исследование по теме: применение машинного обучения в рекламе
```

**Сократовский режим** (нужна помощь сформулировать тему):
```
Помоги определить тему исследования, я интересуюсь цифровым маркетингом
```

**Рецензия статьи**:
```
Проведи рецензию этой статьи: [текст или файл]
```

---

## Режимы работы

| Режим | Когда использовать | Агенты | Выход |
|-------|-------------------|--------|-------|
| `socratic` | Нет чёткой темы, нужна помощь | RQ + Socratic Mentor + DA | План исследования |
| `full` | Есть тема, нужна полная статья | Все 9 основных | APA отчёт 3000-8000 слов |
| `quick` | Нужна быстрая справка | RQ + Bibliography + Verification | Бриф 500-1500 слов |
| `lit-review` | Нужен только обзор литературы | Bibliography + Verification + Synthesis | Аннотированная библиография |
| `review` | Есть текст, нужна оценка | Editor + DA + Ethics | Рецензия |
| `fact-check` | Проверить конкретные утверждения | Source Verification | Отчёт 300-800 слов |

**Как выбрать**: не знаете → `socratic`. Знаете тему → `full`. Нужно быстро → `quick`.

---

## Пайплайн (6 фаз)

```
Фаза 1: ОПРЕДЕЛЕНИЕ (интерактивная)
  [research_question_agent]    → RQ Brief (вопрос, подвопросы, оценка FINER)
  [research_architect_agent]   → Методологический план
  [devils_advocate_agent]      → Проверка: вопрос ясен? метод подходит?
  >>> ЧЕКПОИНТ: подтверждение пользователем <<<

Фаза 2: ИССЛЕДОВАНИЕ
  [bibliography_agent]         → Аннотированная библиография (APA 7.0)
  [source_verification_agent]  → Верификация и градация источников

Фаза 3: АНАЛИЗ
  [synthesis_agent]            → Синтез, выявление пробелов
  [devils_advocate_agent]      → Проверка на предвзятость

Фаза 4: НАПИСАНИЕ
  [report_compiler_agent]      → Полный черновик (APA 7.0)

Фаза 5: РЕВЬЮ (параллельно)
  [editor_in_chief_agent]      → Редакторская оценка
  [ethics_review_agent]        → Этическая проверка
  [devils_advocate_agent]      → Финальная проверка уязвимостей

Фаза 6: ДОРАБОТКА
  [report_compiler_agent]      → Финальный отчёт (макс. 2 цикла доработки)
```

---

## Kanban-режим (для полного пайплайна)

Когда запускается через Kanban, пайплайн разбивается на карточки:

```python
# Карточки создаются воркером ars-orchestrator
T1 = kanban_create(
    title="Исследование: [тема]",
    assignee="ars-researcher",
    body="Провести deep-research по теме: [тема]. Режим: full.",
)

T2 = kanban_create(
    title="Написание статьи: [тема]",
    assignee="ars-writer",
    parents=[T1],  # ждёт завершения исследования
    body="Написать статью по результатам исследования T1.",
)

T3 = kanban_create(
    title="Рецензия статьи: [тема]",
    assignee="ars-reviewer",
    parents=[T2],  # ждёт завершения статьи
    body="Провести мультиперспективную рецензию статьи из T2.",
)

T4 = kanban_create(
    title="Доработка статьи: [тема]",
    assignee="ars-writer",
    parents=[T3],  # ждёт завершения ревью
    body="Доработать статью по замечаниям рецензентов из T3.",
)
```

Чекпоинты между стадиями: `kanban_block()` для подтверждения человеком.

---

## Стратегия поиска источников

Каскад (русские академические → авторитетные неакадемические → международные):

1. **Киберленинка** — `web_search("site:cyberleninka.ru KEYWORDS")` + `web_extract`
2. **eLibrary/РИНЦ** — `web_search("site:elibrary.ru KEYWORDS")` + `web_extract`
3. **Google Scholar** — `web_search("KEYWORDS научная статья")`
4. **Авторитетные неакадемические** — отчёты, СМИ, аналитика (vedomosti.ru, gov.ru и т.д.)
5. **Semantic Scholar** — fallback для англоязычных (API без ключа)
6. **arXiv** — для препринтов ML/AI

Подробнее: `references/shared/source_search_strategy.md`

---

## Состав команды агентов

| # | Агент | Роль | Фаза | Файл |
|---|-------|------|------|------|
| 1 | research_question_agent | Формулировка исследовательского вопроса | 1 | `agents/deep-research/research_question_agent.md` |
| 2 | research_architect_agent | Методологический план | 1 | `agents/deep-research/research_architect_agent.md` |
| 3 | bibliography_agent | Систематический поиск литературы | 2 | `agents/deep-research/bibliography_agent.md` |
| 4 | source_verification_agent | Верификация источников | 2 | `agents/deep-research/source_verification_agent.md` |
| 5 | synthesis_agent | Межисточниковый синтез | 3 | `agents/deep-research/synthesis_agent.md` |
| 6 | report_compiler_agent | Сборка отчёта (APA 7.0) | 4, 6 | `agents/deep-research/report_compiler_agent.md` |
| 7 | editor_in_chief_agent | Редакторская ревью | 5 | `agents/deep-research/editor_in_chief_agent.md` |
| 8 | devils_advocate_agent | Адвокат дьявола | 1, 3, 5 | `agents/deep-research/devils_advocate_agent.md` |
| 9 | ethics_review_agent | Этическая проверка | 5 | `agents/deep-research/ethics_review_agent.md` |
| 10 | socratic_mentor_agent | Сократовский диалог | Socratic | `agents/deep-research/socratic_mentor_agent.md` |
| 11 | risk_of_bias_agent | Оценка рисков предвзятости | SR | `agents/deep-research/risk_of_bias_agent.md` |
| 12 | meta_analysis_agent | Мета-анализ | SR | `agents/deep-research/meta_analysis_agent.md` |
| 13 | monitoring_agent | Мониторинг новых публикаций | Опц. | `agents/deep-research/monitoring_agent.md` |
| 14 | intake_agent | Конфигурация статьи | 0 | `agents/academic-paper/intake_agent.md` |
| 15 | literature_strategist_agent | Стратегия поиска | 1 | `agents/academic-paper/literature_strategist_agent.md` |
| 16 | structure_architect_agent | Структура статьи | 2 | `agents/academic-paper/structure_architect_agent.md` |
| 17 | argument_builder_agent | Аргументация | 3 | `agents/academic-paper/argument_builder_agent.md` |
| 18 | draft_writer_agent | Написание черновика | 4 | `agents/academic-paper/draft_writer_agent.md` |
| 19 | citation_compliance_agent | Проверка цитат | 5a | `agents/academic-paper/citation_compliance_agent.md` |
| 20 | abstract_bilingual_agent | Аннотация | 5b | `agents/academic-paper/abstract_bilingual_agent.md` |
| 21 | peer_reviewer_agent | Имитация ревью | 6 | `agents/academic-paper/peer_reviewer_agent.md` |
| 22 | revision_coach_agent | Коучинг доработки | 6→7 | `agents/academic-paper/revision_coach_agent.md` |
| 23 | formatter_agent | Форматирование | 7 | `agents/academic-paper/formatter_agent.md` |
| 24 | visualization_agent | Графики и таблицы | 4 | `agents/academic-paper/visualization_agent.md` |
| 25 | field_analyst_agent | Определение области | 0 | `agents/academic-paper-reviewer/field_analyst_agent.md` |
| 26 | eic_agent | Главный редактор | 1 | `agents/academic-paper-reviewer/eic_agent.md` |
| 27 | methodology_reviewer_agent | Рецензент-методолог | 1 | `agents/academic-paper-reviewer/methodology_reviewer_agent.md` |
| 28 | domain_reviewer_agent | Доменный эксперт | 1 | `agents/academic-paper-reviewer/domain_reviewer_agent.md` |
| 29 | perspective_reviewer_agent | Междисциплинарный взгляд | 1 | `agents/academic-paper-reviewer/perspective_reviewer_agent.md` |
| 30 | devils_advocate_reviewer_agent | Адвокат дьявола (ревью) | 1 | `agents/academic-paper-reviewer/devils_advocate_reviewer_agent.md` |
| 31 | editorial_synthesizer_agent | Синтез решений | 2 | `agents/academic-paper-reviewer/editorial_synthesizer_agent.md` |
| 32 | pipeline_orchestrator_agent | Оркестратор пайплайна | — | `agents/academic-pipeline/pipeline_orchestrator_agent.md` |
| 33 | integrity_verification_agent | Проверка целостности | 2.5, 4.5 | `agents/academic-pipeline/integrity_verification_agent.md` |
| 34 | claim_ref_alignment_audit_agent | Аудит утверждений | 4→5 | `agents/academic-pipeline/claim_ref_alignment_audit_agent.md` |
| 35 | state_tracker_agent | Трекинг состояния | — | `agents/academic-pipeline/state_tracker_agent.md` |
| 36 | collaboration_depth_agent | Оценка коллаборации | 6 | `agents/academic-pipeline/collaboration_depth_agent.md` |

---

## Портирование из ARS

Оригинальный репо: https://github.com/Imbad0202/academic-research-skills
План: `.hermes/plans/2026-05-20_ars-hermes-port.md`

**Что изменено при портировании:**
- Язык агентов: английский → русский (термины на английском)
- Источники: Semantic Scholar/OpenAlex/Crossref → Киберленинка/eLibrary/Google Scholar
- Оркестрация: Claude Code persona switching → delegate_task()
- Контекст: shared session → isolated context passing
- Хуки: PreToolUse/PostToolUse → изоляция через delegate_task
- Формат: только Markdown (Pandoc/tectonic — позже)
