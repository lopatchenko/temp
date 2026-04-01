# AI Trends Agent — Инструкция по настройке

Пошаговая инструкция для развёртывания агента на VPS.
Выполняй последовательно, каждый шаг проверяй перед переходом к следующему.

---

## 1. СТРУКТУРА ПРОЕКТА

```bash
mkdir -p ~/agents/ai-trends/{config,data,scripts,memory}
```

Итоговая структура:
```
~/agents/ai-trends/
├── SKILL.md                    # Скилл агента (уже готов)
├── config/
│   ├── sources.yaml            # Список источников (RSS, YouTube, Reddit)
│   ├── telegram_channels.yaml  # Каналы из папки ИИ (автозаполнение через MCP)
│   └── .env                    # Токены и секреты
├── data/
│   └── trends.db               # SQLite база (создаётся автоматически)
├── scripts/
│   ├── collector.py            # Сбор данных из внешних источников
│   ├── telegram_reader.py      # Чтение ТГ-каналов через MCP
│   ├── log_analyzer.py         # Анализ логов Джарвиса
│   ├── report_builder.py       # Формирование дайджеста
│   ├── content_engine.py       # Генерация постов
│   ├── publisher.py            # Публикация в ТГ + Threads
│   ├── db.py                   # Работа с SQLite
│   └── cleanup.py              # Очистка старых записей
└── memory/
    └── last_report.json        # Кэш последнего отчёта
```

---

## 2. КОНФИГ И СЕКРЕТЫ

### 2.1 Файл `.env`

```bash
cat > ~/agents/ai-trends/config/.env << 'EOF'
# Telegram Bot
TG_BOT_TOKEN=<токен бота для отправки дайджестов>
TG_CHAT_ID=<ID личного чата для одобрения постов>
TG_CHANNEL_ID=<ID канала для публикации>

# Telegram MCP (для чтения каналов)
TG_MCP_ENDPOINT=<URL MCP-сервера Telegram на VPS>

# Threads API (Meta)
THREADS_USER_ID=<user_id>
THREADS_ACCESS_TOKEN=<long-lived token>

# Claude API
ANTHROPIC_API_KEY=<ключ API>

# Пути к логам
JARVIS_MEMORY_DIR=~/jarvis/memory
JARVIS_REPO_DIR=~/jarvis
NORMDOCS_REPO_DIR=~/projects/normdocs
EOF

chmod 600 ~/agents/ai-trends/config/.env
```

### 2.2 Файл `sources.yaml`

```bash
cat > ~/agents/ai-trends/config/sources.yaml << 'EOF'
rss_feeds:
  - name: Anthropic Blog
    url: https://www.anthropic.com/feed.xml
  - name: OpenAI Blog
    url: https://openai.com/blog/rss.xml
  - name: Google AI Blog
    url: https://blog.google/technology/ai/rss/
  - name: Hugging Face Blog
    url: https://huggingface.co/blog/feed.xml
  - name: Simon Willison
    url: https://simonwillison.net/atom/everything/
  - name: Latent Space
    url: https://www.latent.space/feed

# YouTube: будет добавлен позже через YouTube Data API v3

reddit_subs:
  - MachineLearning
  - artificial
  - LocalLLaMA
  - ClaudeAI

hackernews:
  min_score: 50
  keywords:
    - AI
    - LLM
    - Claude
    - GPT
    - machine learning
    - neural
    - transformer
EOF
```

---

## 3. БАЗА ДАННЫХ

```bash
cat > ~/agents/ai-trends/scripts/db.py << 'PYEOF'
import sqlite3
import os
from contextlib import contextmanager

DB_PATH = os.path.expanduser("~/agents/ai-trends/data/trends.db")

@contextmanager
def get_db():
    conn = sqlite3.connect(DB_PATH, timeout=10)
    conn.row_factory = sqlite3.Row
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()

def init_db():
    with get_db() as conn:
        conn.executescript("""
            CREATE TABLE IF NOT EXISTS items (
                id TEXT PRIMARY KEY,
                source TEXT NOT NULL,
                title TEXT NOT NULL,
                description TEXT,
                url TEXT,
                score INTEGER DEFAULT 0,
                collected_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                published_at DATETIME,
                processed BOOLEAN DEFAULT 0
            );
            CREATE INDEX IF NOT EXISTS idx_collected ON items(collected_at);
            CREATE INDEX IF NOT EXISTS idx_source ON items(source);

            CREATE TABLE IF NOT EXISTS reports (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                report_json TEXT NOT NULL,
                report_type TEXT DEFAULT 'daily'
            );

            CREATE TABLE IF NOT EXISTS published_posts (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                topic TEXT NOT NULL,
                source TEXT NOT NULL,
                tg_message_id INTEGER,
                threads_post_id TEXT,
                status TEXT DEFAULT 'draft'
            );
        """)

if __name__ == "__main__":
    init_db()
    print("Database initialized at", DB_PATH)
PYEOF

python3 ~/agents/ai-trends/scripts/db.py
```

---

## 4. THREADS API — НАСТРОЙКА

### 4.1 Получение токена

1. Зарегистрируй приложение на https://developers.facebook.com/
2. Добавь продукт «Threads API»
3. Получи short-lived token через OAuth:
   ```
   https://threads.net/oauth/authorize
     ?client_id={APP_ID}
     &redirect_uri={REDIRECT_URI}
     &scope=threads_basic,threads_content_publish
     &response_type=code
   ```
4. Обменяй code на short-lived token:
   ```bash
   curl -X POST "https://graph.threads.net/oauth/access_token" \
     -d "client_id={APP_ID}" \
     -d "client_secret={APP_SECRET}" \
     -d "grant_type=authorization_code" \
     -d "redirect_uri={REDIRECT_URI}" \
     -d "code={CODE}"
   ```
5. Обменяй на long-lived token (60 дней):
   ```bash
   curl "https://graph.threads.net/access_token\
     ?grant_type=th_exchange_token\
     &client_secret={APP_SECRET}\
     &access_token={SHORT_LIVED_TOKEN}"
   ```
6. Запиши `THREADS_ACCESS_TOKEN` и `THREADS_USER_ID` в `.env`

### 4.2 Публикация (логика для `publisher.py`)

```python
import httpx
import os

THREADS_USER_ID = os.environ["THREADS_USER_ID"]
THREADS_TOKEN = os.environ["THREADS_ACCESS_TOKEN"]
BASE_URL = "https://graph.threads.net/v1.0"

async def publish_to_threads(text: str) -> str | None:
    """Публикует текстовый пост в Threads. Возвращает post_id или None."""
    async with httpx.AsyncClient() as client:
        # Шаг 1: создать media container
        resp = await client.post(
            f"{BASE_URL}/{THREADS_USER_ID}/threads",
            params={
                "media_type": "TEXT",
                "text": text[:500],  # лимит Threads
                "access_token": THREADS_TOKEN,
            }
        )
        if resp.status_code != 200:
            print(f"Threads container error: {resp.text}")
            return None
        creation_id = resp.json()["id"]

        # Шаг 2: опубликовать
        resp = await client.post(
            f"{BASE_URL}/{THREADS_USER_ID}/threads_publish",
            params={
                "creation_id": creation_id,
                "access_token": THREADS_TOKEN,
            }
        )
        if resp.status_code != 200:
            print(f"Threads publish error: {resp.text}")
            return None
        return resp.json()["id"]
```

### 4.3 Обновление токена (каждые 50 дней)

```bash
# Добавить в crontab:
# 0 3 */50 * * ~/agents/ai-trends/scripts/refresh_threads_token.sh

cat > ~/agents/ai-trends/scripts/refresh_threads_token.sh << 'SHEOF'
#!/bin/bash
source ~/agents/ai-trends/config/.env

NEW_TOKEN=$(curl -s "https://graph.threads.net/refresh_access_token\
?grant_type=th_refresh_token\
&access_token=${THREADS_ACCESS_TOKEN}" | jq -r '.access_token')

if [ "$NEW_TOKEN" != "null" ] && [ -n "$NEW_TOKEN" ]; then
    sed -i "s|THREADS_ACCESS_TOKEN=.*|THREADS_ACCESS_TOKEN=${NEW_TOKEN}|" \
        ~/agents/ai-trends/config/.env
    echo "$(date): Threads token refreshed" >> ~/agents/ai-trends/data/token_refresh.log
else
    echo "$(date): FAILED to refresh Threads token" >> ~/agents/ai-trends/data/token_refresh.log
fi
SHEOF

chmod +x ~/agents/ai-trends/scripts/refresh_threads_token.sh
```

---

## 5. TELEGRAM — РАЗБИВКА СООБЩЕНИЙ

Ключевая утилита для правильной отправки длинных дайджестов:

```python
# ~/agents/ai-trends/scripts/telegram_sender.py

import asyncio
import httpx
import os
import re

TG_BOT_TOKEN = os.environ["TG_BOT_TOKEN"]
TG_API = f"https://api.telegram.org/bot{TG_BOT_TOKEN}"
MAX_LEN = 4096
SECTION_SEP = "━━━"


def split_message(text: str) -> list[str]:
    """
    Разбивает длинное сообщение на части по разделителям секций.
    Никогда не разрывает тему — режет только между блоками.
    """
    if len(text) <= MAX_LEN:
        return [text]

    sections = re.split(rf"(?=\n{SECTION_SEP})", text)
    parts = []
    current = ""

    for section in sections:
        if len(current) + len(section) <= MAX_LEN - 20:  # запас на «продолжение»
            current += section
        else:
            if current:
                parts.append(current.strip() + "\n\n👇 продолжение...")
            current = section

    if current:
        parts.append(current.strip())

    return parts[:3]  # максимум 3 сообщения


async def send_digest(chat_id: str, text: str):
    """Отправляет дайджест, разбивая на сообщения при необходимости."""
    parts = split_message(text)
    async with httpx.AsyncClient() as client:
        for part in parts:
            await client.post(
                f"{TG_API}/sendMessage",
                json={
                    "chat_id": chat_id,
                    "text": part,
                    "parse_mode": "HTML",
                    "disable_web_page_preview": True,
                },
            )
            await asyncio.sleep(1)  # пауза между сообщениями
```

---

## 6. CRON / РАСПИСАНИЕ

```bash
# Добавить в crontab (crontab -e):

# Сбор данных каждые 4 часа
0 */4 * * * cd ~/agents/ai-trends && python3 scripts/collector.py >> data/collector.log 2>&1

# Чтение ТГ-каналов (08:00 МСК)
0 5 * * * cd ~/agents/ai-trends && python3 scripts/telegram_reader.py >> data/tg_reader.log 2>&1

# Анализ логов (08:30 МСК)
30 5 * * * cd ~/agents/ai-trends && python3 scripts/log_analyzer.py >> data/log_analyzer.log 2>&1

# Ежедневный дайджест (09:00 МСК)
0 6 * * * cd ~/agents/ai-trends && python3 scripts/report_builder.py >> data/report.log 2>&1

# Очистка БД (23:00 МСК)
0 20 * * * cd ~/agents/ai-trends && python3 scripts/cleanup.py >> data/cleanup.log 2>&1

# Обновление Threads токена (каждые 50 дней)
0 0 */50 * * ~/agents/ai-trends/scripts/refresh_threads_token.sh
```

> **Примечание:** время в cron — UTC. МСК = UTC+3.
> Проверь свой часовой пояс сервера: `timedatectl` или `date`.
> Если сервер в UTC, вычти 3 часа из МСК.

---

## 7. ЧЕКЛИСТ ЗАПУСКА

Перед запуском пройди по каждому пункту:

```
[ ] 1. Создана структура папок ~/agents/ai-trends/
[ ] 2. Заполнен .env (все токены и ID)
[ ] 3. Инициализирована БД (python3 scripts/db.py)
[ ] 4. sources.yaml заполнен актуальными источниками
[ ] 5. MCP Telegram сервер запущен и доступен
[ ] 6. Threads API: приложение создано, токен получен
[ ] 7. Threads: тестовая публикация прошла успешно
[ ] 8. collector.py: тестовый сбор (/collect) — данные в БД
[ ] 9. telegram_reader.py: тестовое чтение каналов
[ ] 10. report_builder.py: тестовый дайджест (/report)
[ ] 11. log_analyzer.py: тестовый анализ логов
[ ] 12. content_engine.py: тестовая генерация поста
[ ] 13. publisher.py: тестовая публикация в ТГ
[ ] 14. publisher.py: тестовая публикация в Threads
[ ] 15. Cron задачи добавлены
[ ] 16. Логи пишутся в ~/agents/ai-trends/data/
```

---

## 8. МОНИТОРИНГ

Команда `/stats` должна показывать:

```
📊 AI Trends Agent — статистика

🗄 База данных:
  Всего записей: {N}
  За последние 24ч: {N}
  По источникам:
    HackerNews: {N}
    arXiv: {N}
    Reddit: {N}
    RSS: {N}
    Telegram: {N}

📤 Публикации:
  Черновиков: {N}
  Опубликовано за неделю: {N}
  Последняя публикация: {дата}

⏰ Последний сбор: {дата время}
⏰ Последний дайджест: {дата время}

🔑 Токены:
  Threads: {действителен до}
  TG MCP: ✅/❌
```

---

## 9. TROUBLESHOOTING

| Проблема | Решение |
|----------|---------|
| `collector.py` не собирает Reddit | Reddit блокирует без User-Agent. Добавь: `headers={"User-Agent": "AITrendsBot/1.0"}` |
| Threads API 403 | Токен истёк. Запусти `refresh_threads_token.sh` |
| Telegram MCP timeout | Проверь что MCP-сервер запущен: `systemctl status tg-mcp` |
| SQLite locked | Два процесса пишут одновременно. Добавь `timeout=10` в `sqlite3.connect()` |
| Дайджест обрезается | Проверь `split_message()` — лимит 4096 символов на сообщение |
| Git log пустой | Проверь пути к репозиториям в `.env` |
| arXiv rate limit | Пауза 3 сек между запросами. Не чаще 1 раза в 3 часа |
