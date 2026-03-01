---
name: shared-memory
description: >
  Чтение shared-слоя памяти Orgrimmar. Обязателен при старте каждой сессии.
  Содержит: профиль принца, реестр агентов, конвенции, бюджеты, чаты, идеи, уроки.
  Используй когда: (1) старт сессии, (2) задача касается другого агента или сервера,
  (3) вопрос о принце, команде, стоимости, чатах или конвенциях.
---

# Shared Memory — Чтение общего контекста

Shared-слой — единственный источник правды о принце, команде и операционных стандартах.
Обновляется автоматически: Тралл синхронизирует каждый час через obsidian-sync.sh.

**Путь:** `/home/openclaw/.openclaw/agent-memory/shared/`

---

## Порядок чтения при старте сессии

Читай в этом порядке — от критичного к контекстному:

### 1. Кто принц (обязательно)
```bash
cat /home/openclaw/.openclaw/agent-memory/shared/USER.md
cat /home/openclaw/.openclaw/agent-memory/shared/USER_COGNITIVE_PROFILE.md
```
**Что применять:**
- Обращение: «принц» или «мой принц» (никогда «шеф», «пользователь», «вы»)
- Коммуникационный стиль из когнитивного профиля
- Временная зона и язык

### 2. Команда и возможности
```bash
cat /home/openclaw/.openclaw/agent-memory/shared/ROSTER.md
cat /home/openclaw/.openclaw/agent-memory/shared/AGENTS-ROSTER.md
```
**Что применять:**
- Кто какой агент, на каком сервере, с каким Telegram-ботом
- Знать кому делегировать, кого вовлекать в задачу

### 3. Стандарты и правила
```bash
cat /home/openclaw/.openclaw/agent-memory/shared/CONVENTIONS.md
```
**Что применять:**
- Конвенции кода, git, shell-безопасность
- Обязательны для всех агентов, выше локальных предпочтений

### 4. Бюджет и стоимость
```bash
cat /home/openclaw/.openclaw/agent-memory/shared/COSTS.md
```
**Что применять:**
- Не превышать лимиты без согласования с принцем
- Знать текущие расходы при выборе моделей

### 5. Каналы связи
```bash
cat /home/openclaw/.openclaw/agent-memory/shared/CHATS.md
cat /home/openclaw/.openclaw/agent-memory/shared/TELEGRAM-CHATS.md
```
**Что применять:**
- ID чатов для отправки сообщений
- Какой бот в каком чате работает

### 6. Контекст (по необходимости)
```bash
cat /home/openclaw/.openclaw/agent-memory/shared/IDEAS.md      # идеи проекта
cat /home/openclaw/.openclaw/agent-memory/shared/LEARNINGS.md  # уроки всех агентов
```

---

## Когда перечитывать в ходе сессии

- Задача затрагивает другого агента → перечитай `ROSTER.md`
- Вопрос о стоимости/модели → перечитай `COSTS.md`
- Принц упоминает чат или бота → перечитай `CHATS.md`
- Принц устанавливает новое правило → немедленно обнови `memory/warm/LEARNINGS.md` (скилл `learnings`)

---

## Проверка свежести

Данные обновляются Траллом каждый час. Если работаешь офлайн или после инцидента:
```bash
stat /home/openclaw/.openclaw/agent-memory/shared/USER.md
# Если mtime > 2 часов назад — данные могут быть устаревшими
```

---

## Что НИКОГДА не делать

- ❌ Не редактировать файлы в `agent-memory/shared/` напрямую
- ❌ Не писать в `shared/LEARNINGS.md` вручную (только через learnings-merge.py)
- ❌ Не обращаться к принцу как «шеф», «вы», «пользователь»
- ❌ Не игнорировать `CONVENTIONS.md` в пользу личных предпочтений
