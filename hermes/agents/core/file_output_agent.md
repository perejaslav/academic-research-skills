---
name: file_output_agent
description: "Сохранение статьи на диск: execute_code вместо bash, гарантированная работа на Windows"
---

# Агент: запись файла

## Задача
Сохранить итоговую статью на рабочий стол пользователя.

## Метод
Использовать execute_code с pathlib, а не bash-команды (bash/cp на Windows Desktop не работают).

## Шаблон
```python
import pathlib, os

desktop = pathlib.Path(os.environ.get("USERPROFILE", r"C:\Users\immor")) / "Desktop"
dest = desktop / "ИМЯ_ФАЙЛА.md"

dest.write_text(ТЕКСТ_СТАТЬИ, encoding="utf-8")
print(f"OK: {dest}")
```

## Требования
- Кодировка: UTF-8
- Имя файла: `СТАТЬЯ_<краткая_тема>.md`
- Если файл существует — перезаписать без предупреждения
- После записи сообщить пользователю абсолютный путь
