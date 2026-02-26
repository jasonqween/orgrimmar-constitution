# Конвенции кода и операций

_Стандарты кодирования и операционные конвенции для всех агентов._

---

## Shell Security

Все переменные в shell-командах должны быть в двойных кавычках.

```bash
# Правильно
rm -f "${file}"
ssh root@"${server}" "cat \"${path}\""

# Неправильно
rm -f $file
ssh root@${server} 'cat ${path}'
```

### Правила:

| Контекст | Формат | Пример |
|----------|--------|--------|
| Локальная команда | `"${var}"` | `cat "${file}"` |
| SSH команда | `ssh root@"${server}" "cmd \"${arg}\""` | `ssh root@"${ip}" "cat \"${path}\""` |
| Python open() | `sys.argv`, не f-string | `python3 -c "import sys; open(sys.argv[1])" "${file}"` |
| Имена переменных | Только `_`, без `-` | `pipeline_name` (не `pipeline-name`) |

**Почему:** `${pipeline-name}` в bash = `${pipeline:-name}` (parameter expansion с default). Одинарные кавычки не защищают от `'` внутри значения переменной.

---

## Artifact Delivery (воркеры)

Воркеры (sessions_spawn) записывают результат в файл, а не возвращают текстом.

### Стандарт:

```
Запиши результат в /tmp/worker-{label}.md
```

### Freshness Check (обязательно):

Перед спавном воркера:
```bash
rm -f "/tmp/worker-${label}.md"
```

После завершения воркера:
```bash
# Проверить что файл свежий (не старше 10 мин)
stat -c %Y "/tmp/worker-${label}.md"
```

### Параллельные pipeline'ы:

Если возможен параллельный запуск -- добавить timestamp суффикс:
```bash
label="gather-$(date +%s)"
```

---

## Timeout для воркеров

Если задача воркера включает `sleep N`, timeout должен быть:
```
timeout = sleep_time + work_time + buffer
```

Пример: safe-update VERIFY (sleep 300) → timeout = 660 (300 + 300 + 60).

---

## Git

- Коммиты на русском языке
- Workflow: ветки + PR, не пуш в main напрямую
- Git user: настроен на каждом сервере (Тралл, Иллидан, Сильвана)

---

## Язык и форматирование

- Код и комментарии: английский
- Коммиты, документация, отчёты: русский
- Тире: короткое `–` (не длинное `—`)
- Кавычки: русские `«»` (не `""`)
