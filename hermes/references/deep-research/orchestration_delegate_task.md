# Оркестрация deep-research через delegate_task

**Статус**: v1.0
**Используется**: основной агент (Hermes), загрузивший skill academic-research

---

## Принцип

В оригинальном ARS каждый агент — это persona prompt, который Claude Code подменяет
через PreToolUse hook. В Hermes агенты изолированы через `delegate_task()`: каждый
агент получает свой контекст, свою сессию, свой terminal.

---

## Полный пайплайн (режим full)

### Фаза 1: Определение (интерактивная)

Агент-оркестратор (вы) работает напрямую с пользователем.

```
delegate_task(
    goal="Прочитай файл agents/deep-research/research_question_agent.md и выполни его инструкции.
          Тема: {topic}. Сформируй RQ Brief по указанному формату.",
    context="Тема исследования: {topic}. Режим: full. Язык вывода: русский.",
    toolsets=["web", "file"]
)

# Результат: RQ Brief

delegate_task(
    goal="Прочитай файл agents/deep-research/research_architect_agent.md и выполни его инструкции.
          RQ Brief: {rq_brief}. Сформируй методологический план.",
    context="RQ Brief:\n{rq_brief_content}\nЯзык вывода: русский.",
    toolsets=["web", "file"]
)

# Результат: Methodology Blueprint

delegate_task(
    goal="Прочитай файл agents/deep-research/devils_advocate_agent.md и выполни Чекпоинт 1.
          Оцени RQ Brief и методологический план.",
    context="RQ Brief:\n{rq_brief_content}\n\nМетодологический план:\n{blueprint_content}\nЯзык вывода: русский.",
    toolsets=["file"]
)

# Результат: Вердикт PASS/REVISE

# >>> ЧЕКПОИНТ: показать результат пользователю, получить подтверждение <<<
```

### Фаза 2: Исследование

```
delegate_task(
    goal="Прочитай файл agents/deep-research/bibliography_agent.md и выполни его инструкции.
          Проведи систематический поиск литературы по стратегии из references/shared/source_search_strategy.md.
          Используй web_search для поиска, web_extract для чтения.
          Сформируй аннотированную библиографию.",
    context="RQ: {rq}\nКлючевые слова: {keywords}\nДиапазон дат: {date_range}\nЯзык вывода: русский.
             Источники: каскадный поиск — Киберленинка → eLibrary → Google Scholar → авторитетные неакадемические → Semantic Scholar → arXiv.
             Для каждого источника указывай тип: [А] академический, [О] отчёт, [Ан] аналитика, [С] СМИ.",
    toolsets=["web", "file"]
)

# Результат: Аннотированная библиография

delegate_task(
    goal="Прочитай файл agents/deep-research/source_verification_agent.md и выполни его инструкции.
          Проверь каждый источник из библиографии.",
    context="Библиография:\n{bibliography_content}\nЯзык вывода: русский.",
    toolsets=["web", "file"]
)

# Результат: Верифицированные источники
```

### Фаза 3: Анализ

```
delegate_task(
    goal="Прочитай файл agents/deep-research/synthesis_agent.md и выполни его инструкции.
          Проведи синтез по верифицированным источникам.",
    context="Верифицированные источники:\n{verified_sources}\nRQ: {rq}\nЯзык вывода: русский.",
    toolsets=["file"]
)

# Результат: Отчёт о синтезе

delegate_task(
    goal="Прочитай файл agents/deep-research/devils_advocate_agent.md и выполни Чекпоинт 2.
          Оцени синтез на предвзятость и логические ошибки.",
    context="Синтез:\n{synthesis_content}\nRQ: {rq}\nЯзык вывода: русский.",
    toolsets=["file"]
)

# Результат: Вердикт PASS/REVISE
```

### Фаза 4: Написание

```
delegate_task(
    goal="Прочитай файл agents/deep-research/report_compiler_agent.md и выполни его инструкции.
          Собери полный отчёт по APA 7.0 из всех предыдущих материалов.",
    context="RQ Brief: {rq_brief}\nМетодология: {blueprint}\nБиблиография: {bibliography}\nСинтез: {synthesis}\nЯзык вывода: русский.",
    toolsets=["file"]
)

# Результат: Полный черновик
```

### Фаза 5: Ревью (параллельно)

```
# Запускаем 3 агентов параллельно через batch delegate_task
delegate_task(
    tasks=[
        {
            "goal": "Прочитай agents/deep-research/editor_in_chief_agent.md. Проведи редакторскую оценку черновика.",
            "context": "Черновик:\n{draft_content}\nЯзык вывода: русский.",
            "toolsets": ["file"]
        },
        {
            "goal": "Прочитай agents/deep-research/ethics_review_agent.md. Проведи этическую проверку черновика.",
            "context": "Черновик:\n{draft_content}\nЯзык вывода: русский.",
            "toolsets": ["file"]
        },
        {
            "goal": "Прочитай agents/deep-research/devils_advocate_agent.md. Выполни Чекпоинт 3.",
            "context": "Черновик:\n{draft_content}\nЯзык вывода: русский.",
            "toolsets": ["file"]
        }
    ]
)

# Результат: 3 отчёта (редактор + этика + адвокат дьявола)
```

### Фаза 6: Доработка

```
delegate_task(
    goal="Прочитай файл agents/deep-research/report_compiler_agent.md.
          Доработай черновик по замечаниям рецензентов. Максимум 2 цикла доработки.
          Нерешённые проблемы → раздел 'Признанные ограничения'.",
    context="Черновик:\n{draft_content}\n\nЗамечания редактора:\n{editor_feedback}\n\nЭтическая проверка:\n{ethics_feedback}\n\nАдвокат дьявола:\n{da_feedback}\n\nЯзык вывода: русский.",
    toolsets=["file"]
)

# Результат: Финальный отчёт
```

---

## Правила передачи контекста

1. **Каждый агент получает ВСЕ необходимые данные через context** — у агента нет
   доступа к предыдущим сессиям
2. **Не передавайте сырые tool outputs** — суммаризируйте их перед передачей
3. **Формат контекста**: plain text с чёткими секциями (заголовки markdown)
4. **Язык**: всегда указывайте "Язык вывода: русский" в context
5. **Размер контекста**: если данные слишком большие, передавайте ссылки на файлы
   и просите агента прочитать их самостоятельно

---

## Обработка ошибок

| Ситуация | Действие |
|----------|----------|
| Агент таймаутится (>600с) | Уменьшите scope задачи, попробуйте снова |
| Агент возвращает пустой результат | Проверьте контекст, возможно не хватает данных |
| Вердикт REVISE | Покажите замечания пользователю, получите решение |
| Вердикт CRITICAL/BLOCKED | Остановите пайплайн, покажите проблему |
| Мало источников (<5) | Расширьте поисковые термины, добавьте источники |

---

## Режим quick (ускоренный)

Для режима quick пропустите фазы 3, 5, 6:

```
Фаза 1: RQ Brief (только research_question_agent)
Фаза 2: Библиография + верификация (bibliography_agent + source_verification_agent)
Фаза 4: Сборка краткого отчёта (report_compiler_agent, режим quick)
```

## Режим lit-review (только обзор литературы)

```
Фаза 2: Библиография + верификация
Фаза 3: Синтез
```
