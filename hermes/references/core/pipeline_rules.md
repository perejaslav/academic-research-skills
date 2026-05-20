---
name: pipeline_rules
description: "Правила работы instant-пайплайна: роли, передача контекста, ограничения"
---

# Правила ARS-пайплайна (instant-режим)

## Роли в пайплайне
1. **bibliography_agent** — поиск 5-10 источников, каскадный метод
2. **draft_writer_agent** — написание статьи от источников
3. **editor_agent** — однораундовая рецензия
4. **file_output_agent** — запись на Desktop через execute_code

## Передача контекста
- Не использовать файлы — передача через `summary` и `context` delegate_task
- bibliography → draft_writer: тезисы + URL источников
- draft_writer → editor: полный текст статьи
- editor → доработка: список замечаний

## Ограничения
- 1 цикл ревью (не более). Если major/reject → доработка → финал. Второй ревью не делаем.
- Макс. 3 delegate_task в одном прогоне (лимит платформы)
- Если нужно больше — разбить на 2 запуска

## Формат итогового файла
- UTF-8, Markdown
- Имя: `СТАТЬЯ_<тема>.md` на Desktop
- Структура: Title → Abstract (рус+англ) → разделы → References → (опц.) Appendix
