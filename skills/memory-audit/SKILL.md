---
name: memory-audit
description: >
  Самоаудит памяти агента по регламенту конституции Orgrimmar. Используй этот скилл
  когда: (1) принц пишет «аудит памяти», «проверь память», «memory-audit»,
  (2) еженедельно по крону (воскресенье), (3) после крупных изменений в конфигурации.
  Скилл проверяет личную память, shared-слой, QMD, cron-расписание и obsidian-sync.
  Выдаёт структурированный отчёт с оценкой по каждому разделу.
---

# Memory Audit — Самоаудит памяти по регламенту

Ты проводишь независимый аудит своей памяти. Проверяй факты через shell-команды.
Не полагайся на допущения — только на то, что реально существует на диске.

---

## Шаг 0: Определи свой контекст

Прежде чем начать — установи параметры текущего агента:

```bash
# Узнай свой workspace
grep -A5 '"list"' /home/openclaw/.openclaw/openclaw.json | grep workspace | head -3

# Узнай server hostname
hostname
cat /etc/hostname
```

На основе этого заполни переменные для следующих шагов:
- `AGENT_ID` — твой id из openclaw.json (silvana / arthas / main)
- `WS` — путь к workspace агента
- `SERVER` — sylvanas / thrall / illidan

---

## Раздел А: Личная память (Слой 1)

**Регламент:** обязательные файлы в `$WS/memory/`:

| Файл | Назначение |
|------|-----------|
| `$WS/MEMORY.md` | COLD: архив решений |
| `$WS/SOUL.md` | Идентичность агента |
| `$WS/memory/hot/HOT_MEMORY.md` | Now / Next / Blockers |
| `$WS/memory/warm/WARM_MEMORY.md` | Операционная база |
| `$WS/memory/warm/WATCHLIST.md` | Личные наблюдения |
| `$WS/memory/warm/LEARNINGS.md` | Уроки из ошибок |

**Проверка:**
```bash
WS="<твой_workspace_путь>"
for f in \
  "$WS/MEMORY.md" \
  "$WS/SOUL.md" \
  "$WS/memory/hot/HOT_MEMORY.md" \
  "$WS/memory/warm/WARM_MEMORY.md" \
  "$WS/memory/warm/WATCHLIST.md" \
  "$WS/memory/warm/LEARNINGS.md"
do
  if [ -f "$f" ]; then
    SIZE=$(wc -c < "$f")
    echo "✅ $(basename $f) — ${SIZE} bytes"
    # Warn if approaching rotation limit
    [ "$SIZE" -gt 8192 ] && echo "  ⚠️  ВНИМАНИЕ: >8KB, скоро ротация"
    [ "$SIZE" -gt 10240 ] && echo "  ❌ КРИТИЧНО: >10KB, flush заблокирован!"
  else
    echo "❌ ОТСУТСТВУЕТ: $f"
  fi
done

# Проверь дневник (последние 3 дня)
for i in 0 1 2; do
  D=$(date -u -d "$i days ago" +%Y-%m-%d 2>/dev/null || date -u -v-${i}d +%Y-%m-%d)
  [ -f "$WS/memory/$D.md" ] && echo "✅ diary: $D.md" || echo "⚠️  diary: $D.md — нет"
done

# Проверь archive/
ls "$WS/memory/archive/" 2>/dev/null | wc -l | xargs -I{} echo "archive: {} файлов"
```

**Оценивай:**
- Все 6 файлов присутствуют → ✅
- Есть файлы >10KB → ❌ CRITICAL (flush заблокирован)
- Есть файлы 8-10KB → ⚠️ скоро ротация
- Дневника за сегодня нет → ⚠️ (создать при следующем flush)

---

## Раздел Б: Compaction (memoryFlush)

**Регламент:** промпт должен содержать все 6 элементов.

**Проверка:**
```bash
python3 -c "
import json
with open('/home/openclaw/.openclaw/openclaw.json') as f:
    d = json.load(f)
defaults = d.get('agents',{}).get('defaults',{})
mf = defaults.get('compaction',{}).get('memoryFlush',{})
prompt = mf.get('prompt','')
threshold = mf.get('softThresholdTokens', 0)

checks = {
  'softThresholdTokens=30000': threshold == 30000,
  'HOT_MEMORY.md в промпте': 'HOT_MEMORY.md' in prompt,
  'MEMORY.md в промпте': 'MEMORY.md' in prompt,
  'LEARNINGS.md в промпте': 'LEARNINGS.md' in prompt,
  'YYYY-MM-DD в промпте': 'YYYY-MM-DD' in prompt or 'дневник' in prompt.lower(),
  'NO_FLUSH в промпте': 'NO_FLUSH' in prompt,
  'ВАЖНО: проверь размер': 'ВАЖНО' in prompt or '10KB' in prompt,
}
for k, v in checks.items():
    print('✅' if v else '❌', k)
"
```

---

## Раздел В: Shared память (Слой 2)

### В1: Источник правды — `_shared/` (только Sylvanas)

Если ты на Sylvanas:
```bash
SHARED="/home/openclaw/.openclaw/workspace/_shared"
REQUIRED="USER.md USER_COGNITIVE_PROFILE.md ROSTER.md AGENTS-ROSTER.md CONVENTIONS.md COSTS.md CHATS.md TELEGRAM-CHATS.md"
for f in $REQUIRED; do
  [ -f "$SHARED/$f" ] && echo "✅ _shared/$f" || echo "❌ _shared/$f — ОТСУТСТВУЕТ"
done
```

### В2: Obsidian-зеркало — `agent-memory/shared/`

```bash
MIRROR="/home/openclaw/.openclaw/agent-memory/shared"
REQUIRED_MIRROR="USER.md USER_COGNITIVE_PROFILE.md ROSTER.md AGENTS-ROSTER.md CONVENTIONS.md COSTS.md CHATS.md TELEGRAM-CHATS.md LEARNINGS.md IDEAS.md"
for f in $REQUIRED_MIRROR; do
  [ -f "$MIRROR/$f" ] && echo "✅ agent-memory/shared/$f" || echo "❌ agent-memory/shared/$f — ОТСУТСТВУЕТ"
done

# Проверь свежесть (должен обновляться каждый час)
if [ -d "$MIRROR" ]; then
  LAST=$(stat -c %Y "$MIRROR" 2>/dev/null || stat -f %m "$MIRROR")
  AGE=$(( ($(date +%s) - LAST) / 3600 ))
  [ "$AGE" -lt 2 ] && echo "✅ Зеркало свежее (${AGE}ч назад)" || echo "⚠️  Зеркало устарело (${AGE}ч назад)"
fi
```

### В3: learnings-merge.py и shared/LEARNINGS.md (только Sylvanas)

```bash
[ -f /home/openclaw/.openclaw/shared/LEARNINGS.md ] && \
  echo "✅ shared/LEARNINGS.md существует" || \
  echo "❌ shared/LEARNINGS.md — ОТСУТСТВУЕТ"

[ -f /home/openclaw/.openclaw/workspace/scripts/learnings-merge.py ] && \
  echo "✅ learnings-merge.py существует" || \
  echo "❌ learnings-merge.py — ОТСУТСТВУЕТ"
```

---

## Раздел Г: QMD (Слой 3)

**Регламент:** 5 коллекций в index.yml, `embedInterval: "30m"`.

**Проверка:**
```bash
# Путь к index.yml — адаптируй под свой agent_id
AGENT_ID="<silvana|arthas|main>"
IDX="/home/openclaw/.openclaw/agents/$AGENT_ID/qmd/xdg-config/qmd/index.yml"

echo "=== QMD index.yml ==="
if [ -f "$IDX" ]; then
  echo "✅ index.yml существует"
  grep -q "shared" "$IDX" && echo "✅ коллекция shared присутствует" || echo "❌ коллекция shared ОТСУТСТВУЕТ"
  grep -q "memory-dir" "$IDX" && echo "✅ memory-dir присутствует" || echo "❌ memory-dir ОТСУТСТВУЕТ"
  grep -q "sessions" "$IDX" && echo "✅ sessions присутствует" || echo "❌ sessions ОТСУТСТВУЕТ"
  cat "$IDX"
else
  echo "❌ index.yml НЕ НАЙДЕН: $IDX"
fi

echo ""
echo "=== embedInterval ==="
python3 -c "
import json
with open('/home/openclaw/.openclaw/openclaw.json') as f:
    d = json.load(f)
ei = d.get('memory',{}).get('qmd',{}).get('update',{}).get('embedInterval','NOT SET')
ok = ei not in ['0', '0s', '', 'NOT SET']
print('✅ embedInterval:', ei) if ok else print('❌ embedInterval:', ei, '— QMD не строит эмбеддинги!')
"

echo ""
echo "=== QMD SQLite (наполненность) ==="
DB=$(find /home/openclaw/.openclaw/agents -name "main.sqlite" 2>/dev/null | head -1)
if [ -n "$DB" ]; then
  python3 -c "
import sqlite3
conn = sqlite3.connect('$DB')
cur = conn.cursor()
try:
    cur.execute('SELECT COUNT(*) FROM chunks')
    n = cur.fetchone()[0]
    print('✅ chunks в БД:', n) if n > 0 else print('⚠️  БД пустая — эмбеддинги ещё не созданы')
except Exception as e:
    print('⚠️  Ошибка БД:', e)
conn.close()
"
else
  echo "⚠️  SQLite не найдена — QMD ещё не запускался"
fi
```

---

## Раздел Д: Obsidian sync (только Тралл)

Если ты Тралл, проверь:

```bash
SYNC_SCRIPT="/home/openclaw/.openclaw/scripts/obsidian-sync.sh"
LOG="/tmp/obsidian-sync.log"

[ -f "$SYNC_SCRIPT" ] && echo "✅ obsidian-sync.sh существует" || echo "❌ ОТСУТСТВУЕТ"

# Последний запуск
if [ -f "$LOG" ]; then
  LAST_RUN=$(tail -1 "$LOG")
  echo "Последний запуск: $LAST_RUN"
  grep -q "Pushed\|No changes" "$LOG" && echo "✅ Последний sync успешен" || echo "⚠️  Проверь лог: $LOG"
fi

# Покрытие агентов
for agent in thrall silvana arthas kaelthas illidan; do
  grep -q "sync.*$agent\|Synced: $agent" "$LOG" 2>/dev/null && \
    echo "✅ $agent синхронизирован" || echo "⚠️  $agent — нет данных в логе"
done

# Проверь что IDEAS.md В репо (не перезаписывается из _shared/)
REPO="/home/openclaw/.openclaw/agent-memory"
[ -f "$REPO/shared/IDEAS.md" ] && echo "✅ agent-memory/shared/IDEAS.md существует" || echo "❌ IDEAS.md отсутствует в repo"
```

---

## Раздел Е: Cron-расписание

**Регламент:**

| Сервер | Обязательные кроны |
|--------|-------------------|
| Sylvanas | `constitution-sync.sh` (*/6ч), `learnings-merge.py` (*/6ч), `memory-rotate.sh` ×3 (21:00) |
| Тралл | `obsidian-sync.sh` (каждый час), `constitution-sync.sh` (*/6ч), `memory-rotate.sh` (21:00) |
| Иллидан | `constitution-sync.sh` (*/6ч), `memory-rotate.sh` (21:00) |

**Проверка:**
```bash
echo "=== Crontab (openclaw) ==="
crontab -u openclaw -l 2>/dev/null

echo ""
echo "=== Проверка обязательных кронов ==="
CRON=$(crontab -u openclaw -l 2>/dev/null)

check_cron() {
  local name="$1"
  local pattern="$2"
  echo "$CRON" | grep -q "$pattern" && \
    echo "✅ $name" || echo "❌ $name — ОТСУТСТВУЕТ"
}

check_cron "constitution-sync.sh" "constitution-sync"
check_cron "memory-rotate.sh" "memory-rotate"

# Только Sylvanas:
SERVER=$(hostname)
if echo "$SERVER" | grep -qi "sylvanas\|164\.92"; then
  check_cron "learnings-merge.py" "learnings-merge"
fi

# Только Тралл:
if echo "$SERVER" | grep -qi "thrall\|46\.101\|orgrimmar"; then
  check_cron "obsidian-sync.sh" "obsidian-sync"
fi
```

---

## Раздел Ж: Маршрутизация идей

**Проверка корректности idea-capture.sh:**
```bash
SCRIPT="/home/openclaw/.openclaw/shared/tasks/idea-capture.sh"
if [ -f "$SCRIPT" ]; then
  echo "✅ idea-capture.sh существует"
  grep -q "IDEAS.md\|ideas-md\|agent-memory" "$SCRIPT" && \
    echo "✅ обновляет IDEAS.md" || echo "❌ НЕ обновляет IDEAS.md"
  grep -q "git push\|git commit" "$SCRIPT" && \
    echo "✅ пушит в GitHub" || echo "❌ НЕ пушит в GitHub"
else
  echo "⚠️  idea-capture.sh недоступен (возможно, не на Sylvanas)"
fi

# Проверь структуру tasks/
TASKS="/home/openclaw/.openclaw/shared/tasks"
for d in inbox active done ideas; do
  [ -d "$TASKS/$d" ] && echo "✅ tasks/$d/" || echo "❌ tasks/$d/ — ОТСУТСТВУЕТ"
done
```

---

## Финальный отчёт

После выполнения всех проверок составь отчёт в формате:

```markdown
# Memory Audit Report — <agent_name> — <YYYY-MM-DD>

## Итог

| Раздел | Статус | Детали |
|--------|--------|--------|
| А. Личная память | ✅/⚠️/❌ | ... |
| Б. Compaction | ✅/⚠️/❌ | ... |
| В. Shared память | ✅/⚠️/❌ | ... |
| Г. QMD | ✅/⚠️/❌ | ... |
| Д. Obsidian sync | ✅/N/A/❌ | ... |
| Е. Cron | ✅/⚠️/❌ | ... |
| Ж. Маршрутизация | ✅/⚠️/❌ | ... |

**Общий балл: X/7 разделов ✅**

## Критические проблемы (❌)
<список, если есть>

## Предупреждения (⚠️)
<список, если есть>

## Рекомендации
<что исправить>
```

**Если найдены ❌ CRITICAL:**
- Немедленно сообщи принцу
- Исправь если в твоих полномочиях (только безопасные, обратимые действия)
- Запиши в LEARNINGS.md

**Если только ⚠️:**
- Запиши в HOT_MEMORY.md как Blockers
- Предложи принцу план исправления

**Если всё ✅:**
- Кратко: «memory-audit пройден, X/7 ✅»
- Запиши в дневник (`memory/YYYY-MM-DD.md`)
