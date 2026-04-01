# Travel Copywriter — Инструкция по настройке

Пошаговая инструкция для развёртывания агента-копирайтера на VPS.
Выполняй последовательно, каждый шаг проверяй перед переходом к следующему.

---

## 1. СТРУКТУРА ПРОЕКТА

```bash
mkdir -p ~/agents/travel-copywriter/{config,scripts,drafts,media}
```

Итоговая структура:
```
~/agents/travel-copywriter/
├── SKILL.md                     # Скилл агента (уже готов)
├── config/
│   └── .env                     # Токены и секреты
├── scripts/
│   ├── publisher.py             # Основной модуль публикации
│   ├── telegram_pub.py          # Публикация в Telegram
│   ├── vk_pub.py                # Публикация в VK
│   ├── instagram_pub.py         # Публикация в Instagram
│   ├── threads_pub.py           # Публикация в Threads
│   ├── photo_handler.py         # Обработка и загрузка фото
│   └── token_refresh.sh         # Обновление токенов Meta
├── drafts/                      # Черновики постов (текущий + история)
│   └── current_draft.json       # Текущий одобренный пост
└── media/                       # Временное хранение фото перед публикацией
```

---

## 2. КОНФИГ И СЕКРЕТЫ

```bash
cat > ~/agents/travel-copywriter/config/.env << 'EOF'
# ===== TELEGRAM =====
TG_BOT_TOKEN=<токен бота>
TG_CHANNEL_ID=<ID канала «Путешествия со смыслом»>
TG_ADMIN_CHAT_ID=<ID личного чата для одобрения>

# ===== VK =====
VK_ACCESS_TOKEN=<токен с правами wall,photos,offline>
VK_GROUP_ID=<ID группы/страницы, без минуса>
VK_API_VERSION=5.199

# ===== INSTAGRAM =====
IG_BUSINESS_ACCOUNT_ID=<ID Instagram Business аккаунта>
IG_ACCESS_TOKEN=<long-lived токен через Facebook Graph API>

# ===== THREADS =====
THREADS_USER_ID=<user_id>
THREADS_ACCESS_TOKEN=<long-lived token>

# ===== CLAUDE =====
ANTHROPIC_API_KEY=<ключ API>
EOF

chmod 600 ~/agents/travel-copywriter/config/.env
```

---

## 3. TELEGRAM — НАСТРОЙКА

Телеграм — самая простая часть. Бот уже есть, нужно только убедиться
что он имеет права администратора в канале.

### 3.1 Проверка прав бота

Бот должен быть добавлен в канал как администратор с правами:
- Отправка сообщений
- Отправка фото/медиа
- Редактирование сообщений

### 3.2 Публикация с фото

```python
# ~/agents/travel-copywriter/scripts/telegram_pub.py

import asyncio
import httpx
import os
from pathlib import Path

TG_BOT_TOKEN = os.environ["TG_BOT_TOKEN"]
TG_API = f"https://api.telegram.org/bot{TG_BOT_TOKEN}"


async def publish_telegram(
    chat_id: str,
    text: str,
    photos: list[str] | None = None,
) -> dict:
    """
    Публикует пост в Telegram-канал.
    photos — список путей к файлам изображений.
    """
    async with httpx.AsyncClient(timeout=60) as client:

        # Без фото — просто текст
        if not photos:
            resp = await client.post(
                f"{TG_API}/sendMessage",
                json={
                    "chat_id": chat_id,
                    "text": text,
                    "parse_mode": "HTML",
                    "disable_web_page_preview": True,
                },
            )
            return resp.json()

        # Одно фото
        if len(photos) == 1:
            # Если текст ≤ 1024 — в caption
            if len(text) <= 1024:
                with open(photos[0], "rb") as f:
                    resp = await client.post(
                        f"{TG_API}/sendPhoto",
                        data={
                            "chat_id": chat_id,
                            "caption": text,
                            "parse_mode": "HTML",
                        },
                        files={"photo": f},
                    )
                return resp.json()
            else:
                # Фото отдельно, текст следующим сообщением
                with open(photos[0], "rb") as f:
                    await client.post(
                        f"{TG_API}/sendPhoto",
                        data={"chat_id": chat_id},
                        files={"photo": f},
                    )
                resp = await client.post(
                    f"{TG_API}/sendMessage",
                    json={
                        "chat_id": chat_id,
                        "text": text,
                        "parse_mode": "HTML",
                    },
                )
                return resp.json()

        # Несколько фото — медиагруппа
        import json as json_mod

        media = []
        files = {}
        for i, photo_path in enumerate(photos):
            attach_name = f"photo{i}"
            media_item = {
                "type": "photo",
                "media": f"attach://{attach_name}",
            }
            if i == 0 and len(text) <= 1024:
                media_item["caption"] = text
                media_item["parse_mode"] = "HTML"
            files[attach_name] = open(photo_path, "rb")
            media.append(media_item)

        try:
            resp = await client.post(
                f"{TG_API}/sendMediaGroup",
                data={
                    "chat_id": chat_id,
                    "media": json_mod.dumps(media),
                },
                files=files,
            )
            result = resp.json()
        finally:
            for f in files.values():
                f.close()

        # Если текст длинный — отдельным сообщением
        if len(text) > 1024:
            await asyncio.sleep(1)
            await client.post(
                f"{TG_API}/sendMessage",
                json={
                    "chat_id": chat_id,
                    "text": text,
                    "parse_mode": "HTML",
                },
            )

        return result
```

---

## 4. VK — НАСТРОЙКА

### 4.1 Получение токена

**Вариант A — Через Standalone-приложение (рекомендуется):**

1. Создай приложение: https://vk.com/editapp?act=create
   - Тип: Standalone
   - Запомни `client_id` (ID приложения)

2. Получи токен через Implicit Flow:
   ```
   https://oauth.vk.com/authorize?
     client_id={APP_ID}
     &display=page
     &redirect_uri=https://oauth.vk.com/blank.html
     &scope=wall,photos,offline
     &response_type=token
     &v=5.199
   ```
   - Открой эту ссылку в браузере
   - Авторизуйся и разреши доступ
   - Из URL в адресной строке скопируй `access_token`

3. Scope `offline` даёт бессрочный токен (не нужно обновлять).

**Вариант B — Токен сообщества (если публикуешь от имени группы):**

1. Зайди в настройки группы → Работа с API
2. Создай ключ доступа с правами: Управление, Фотографии, Стена
3. Скопируй токен

### 4.2 Определение group_id

```bash
# Если публикуешь в группу:
curl -s "https://api.vk.com/method/groups.getById?access_token={TOKEN}&v=5.199" | python3 -m json.tool
# Возьми id из ответа

# Если на личную стену — group_id не нужен, owner_id = твой user_id
```

### 4.3 Публикация

```python
# ~/agents/travel-copywriter/scripts/vk_pub.py

import httpx
import os

VK_TOKEN = os.environ["VK_ACCESS_TOKEN"]
VK_GROUP_ID = os.environ["VK_GROUP_ID"]
VK_API = "https://api.vk.com/method"
VK_VERSION = os.environ.get("VK_API_VERSION", "5.199")


async def upload_vk_photos(photos: list[str]) -> list[str]:
    """Загружает фото на VK, возвращает список attachment-строк."""
    attachments = []
    async with httpx.AsyncClient(timeout=60) as client:
        for photo_path in photos:
            # 1. Получить URL для загрузки
            resp = await client.get(
                f"{VK_API}/photos.getWallUploadServer",
                params={
                    "group_id": VK_GROUP_ID,
                    "access_token": VK_TOKEN,
                    "v": VK_VERSION,
                },
            )
            upload_url = resp.json()["response"]["upload_url"]

            # 2. Загрузить фото
            with open(photo_path, "rb") as f:
                resp = await client.post(
                    upload_url,
                    files={"photo": f},
                )
            upload_result = resp.json()

            # 3. Сохранить на стену
            resp = await client.get(
                f"{VK_API}/photos.saveWallPhoto",
                params={
                    "group_id": VK_GROUP_ID,
                    "photo": upload_result["photo"],
                    "server": upload_result["server"],
                    "hash": upload_result["hash"],
                    "access_token": VK_TOKEN,
                    "v": VK_VERSION,
                },
            )
            saved = resp.json()["response"][0]
            attachments.append(f"photo{saved['owner_id']}_{saved['id']}")

    return attachments


async def publish_vk(
    text: str,
    photos: list[str] | None = None,
) -> dict:
    """Публикует пост на стену VK."""
    async with httpx.AsyncClient(timeout=60) as client:
        params = {
            "owner_id": f"-{VK_GROUP_ID}",  # минус = группа
            "from_group": 1,
            "message": text,
            "access_token": VK_TOKEN,
            "v": VK_VERSION,
        }

        if photos:
            attachments = await upload_vk_photos(photos)
            params["attachments"] = ",".join(attachments)

        resp = await client.post(f"{VK_API}/wall.post", data=params)
        return resp.json()
```

---

## 5. INSTAGRAM — НАСТРОЙКА

Instagram публикация работает через Facebook Graph API.
Требуется: Facebook Page, привязанный к Instagram Business/Creator аккаунту.

### 5.1 Подготовка

1. **Instagram аккаунт** должен быть Business или Creator
2. **Facebook Page** должна быть привязана к этому Instagram аккаунту
3. **Facebook App** (Meta for Developers): https://developers.facebook.com/apps/

### 5.2 Получение токена

1. Создай приложение на https://developers.facebook.com/
2. Добавь продукт «Instagram Graph API»
3. В App Dashboard → Tools → Graph API Explorer:
   - Выбери своё приложение
   - Permissions: `instagram_basic`, `instagram_content_publish`, `pages_show_list`, `pages_read_engagement`
   - Generate Access Token → авторизуйся
4. Получи short-lived token
5. Обменяй на long-lived (60 дней):
   ```bash
   curl "https://graph.facebook.com/v21.0/oauth/access_token\
     ?grant_type=fb_exchange_token\
     &client_id={APP_ID}\
     &client_secret={APP_SECRET}\
     &fb_exchange_token={SHORT_TOKEN}"
   ```

### 5.3 Получение Instagram Business Account ID

```bash
# Сначала найди Page ID
curl "https://graph.facebook.com/v21.0/me/accounts?access_token={TOKEN}"

# Затем найди IG Account ID через Page
curl "https://graph.facebook.com/v21.0/{PAGE_ID}?fields=instagram_business_account&access_token={TOKEN}"

# Запиши instagram_business_account.id в .env как IG_BUSINESS_ACCOUNT_ID
```

### 5.4 Публикация

**Важно:** Instagram API требует, чтобы фото были доступны по публичному URL.
Нужно сначала загрузить фото на доступный сервер (или использовать временный хостинг).

```python
# ~/agents/travel-copywriter/scripts/instagram_pub.py

import asyncio
import httpx
import os

IG_ACCOUNT_ID = os.environ["IG_BUSINESS_ACCOUNT_ID"]
IG_TOKEN = os.environ["IG_ACCESS_TOKEN"]
GRAPH_API = "https://graph.facebook.com/v21.0"


async def upload_photo_to_hosting(photo_path: str) -> str:
    """
    Загружает фото на публичный URL.
    Варианты: собственный nginx на VPS, imgbb.com API, Cloudinary.
    Вернуть нужно публичный URL.
    """
    # Вариант: отдать через nginx на VPS
    # Скопировать файл в /var/www/media/ и вернуть URL
    import shutil
    from pathlib import Path

    filename = Path(photo_path).name
    dest = f"/var/www/media/{filename}"
    shutil.copy2(photo_path, dest)
    # Замени на свой домен:
    return f"https://media.normpoisk.ru/{filename}"


async def publish_instagram_single(
    text: str,
    photo_path: str,
) -> str | None:
    """Публикует одно фото с подписью в Instagram."""
    async with httpx.AsyncClient(timeout=120) as client:
        image_url = await upload_photo_to_hosting(photo_path)

        # 1. Создать media container
        resp = await client.post(
            f"{GRAPH_API}/{IG_ACCOUNT_ID}/media",
            params={
                "image_url": image_url,
                "caption": text[:2200],  # лимит Instagram
                "access_token": IG_TOKEN,
            },
        )
        if resp.status_code != 200:
            print(f"IG container error: {resp.text}")
            return None
        creation_id = resp.json()["id"]

        # 2. Подождать обработки (Instagram обрабатывает асинхронно)
        await asyncio.sleep(5)

        # 3. Опубликовать
        resp = await client.post(
            f"{GRAPH_API}/{IG_ACCOUNT_ID}/media_publish",
            params={
                "creation_id": creation_id,
                "access_token": IG_TOKEN,
            },
        )
        if resp.status_code != 200:
            print(f"IG publish error: {resp.text}")
            return None
        return resp.json()["id"]


async def publish_instagram_carousel(
    text: str,
    photos: list[str],
) -> str | None:
    """Публикует карусель (2-10 фото) с подписью в Instagram."""
    async with httpx.AsyncClient(timeout=120) as client:
        # 1. Создать item containers для каждого фото
        item_ids = []
        for photo_path in photos[:10]:  # лимит 10 фото
            image_url = await upload_photo_to_hosting(photo_path)
            resp = await client.post(
                f"{GRAPH_API}/{IG_ACCOUNT_ID}/media",
                params={
                    "image_url": image_url,
                    "is_carousel_item": "true",
                    "access_token": IG_TOKEN,
                },
            )
            if resp.status_code != 200:
                print(f"IG carousel item error: {resp.text}")
                continue
            item_ids.append(resp.json()["id"])

        if not item_ids:
            return None

        await asyncio.sleep(5)

        # 2. Создать carousel container
        resp = await client.post(
            f"{GRAPH_API}/{IG_ACCOUNT_ID}/media",
            params={
                "media_type": "CAROUSEL",
                "children": ",".join(item_ids),
                "caption": text[:2200],
                "access_token": IG_TOKEN,
            },
        )
        if resp.status_code != 200:
            print(f"IG carousel error: {resp.text}")
            return None
        creation_id = resp.json()["id"]

        await asyncio.sleep(5)

        # 3. Опубликовать
        resp = await client.post(
            f"{GRAPH_API}/{IG_ACCOUNT_ID}/media_publish",
            params={
                "creation_id": creation_id,
                "access_token": IG_TOKEN,
            },
        )
        if resp.status_code != 200:
            print(f"IG carousel publish error: {resp.text}")
            return None
        return resp.json()["id"]
```

### 5.5 Хостинг фото для Instagram API

Instagram API не принимает файлы напрямую — нужен публичный URL.
Три варианта:

**Вариант A — Nginx на VPS (рекомендуется, уже есть сервер):**
```bash
# Создать директорию для медиа
sudo mkdir -p /var/www/media
sudo chown $USER:$USER /var/www/media

# Добавить в nginx.conf:
server {
    listen 80;
    server_name media.normpoisk.ru;  # или поддомен

    location / {
        root /var/www/media;
        expires 1h;
    }
}

# Перезапустить nginx
sudo nginx -t && sudo systemctl reload nginx
```

**Вариант B — imgbb.com API (без своего сервера):**
```python
async def upload_to_imgbb(photo_path: str) -> str:
    async with httpx.AsyncClient() as client:
        with open(photo_path, "rb") as f:
            import base64
            b64 = base64.b64encode(f.read()).decode()
        resp = await client.post(
            "https://api.imgbb.com/1/upload",
            data={"key": IMGBB_API_KEY, "image": b64, "expiration": 3600},
        )
        return resp.json()["data"]["url"]
```

**Вариант C — Cloudinary (бесплатный план):**
```bash
pip install cloudinary --break-system-packages
```

---

## 6. THREADS — НАСТРОЙКА

Настройка аналогична AI Trends агенту. Краткая версия:

### 6.1 Получение токена

1. В том же Facebook App добавь продукт «Threads API»
2. Scope: `threads_basic`, `threads_content_publish`
3. OAuth → short-lived → long-lived token (60 дней)

### 6.2 Публикация

```python
# ~/agents/travel-copywriter/scripts/threads_pub.py

import httpx
import os

THREADS_USER_ID = os.environ["THREADS_USER_ID"]
THREADS_TOKEN = os.environ["THREADS_ACCESS_TOKEN"]
BASE_URL = "https://graph.threads.net/v1.0"


async def publish_threads_text(text: str) -> str | None:
    """Публикует текстовый пост в Threads."""
    async with httpx.AsyncClient(timeout=30) as client:
        resp = await client.post(
            f"{BASE_URL}/{THREADS_USER_ID}/threads",
            params={
                "media_type": "TEXT",
                "text": text[:500],
                "access_token": THREADS_TOKEN,
            },
        )
        if resp.status_code != 200:
            return None
        creation_id = resp.json()["id"]

        resp = await client.post(
            f"{BASE_URL}/{THREADS_USER_ID}/threads_publish",
            params={
                "creation_id": creation_id,
                "access_token": THREADS_TOKEN,
            },
        )
        return resp.json().get("id") if resp.status_code == 200 else None


async def publish_threads_image(text: str, image_url: str) -> str | None:
    """Публикует пост с изображением в Threads."""
    async with httpx.AsyncClient(timeout=60) as client:
        resp = await client.post(
            f"{BASE_URL}/{THREADS_USER_ID}/threads",
            params={
                "media_type": "IMAGE",
                "image_url": image_url,
                "text": text[:500],
                "access_token": THREADS_TOKEN,
            },
        )
        if resp.status_code != 200:
            return None
        creation_id = resp.json()["id"]

        import asyncio
        await asyncio.sleep(3)

        resp = await client.post(
            f"{BASE_URL}/{THREADS_USER_ID}/threads_publish",
            params={
                "creation_id": creation_id,
                "access_token": THREADS_TOKEN,
            },
        )
        return resp.json().get("id") if resp.status_code == 200 else None
```

---

## 7. ОБЩИЙ МОДУЛЬ ПУБЛИКАЦИИ

Оркестратор, который управляет публикацией во все платформы:

```python
# ~/agents/travel-copywriter/scripts/publisher.py

import asyncio
import json
import os
from pathlib import Path
from datetime import datetime

from telegram_pub import publish_telegram
from vk_pub import publish_vk
from instagram_pub import publish_instagram_single, publish_instagram_carousel
from threads_pub import publish_threads_text, publish_threads_image
from photo_handler import upload_photo_to_hosting

TG_CHANNEL_ID = os.environ["TG_CHANNEL_ID"]
TG_ADMIN_CHAT_ID = os.environ["TG_ADMIN_CHAT_ID"]


async def publish_all(
    versions: dict,     # {"telegram": str, "vk": str, "instagram": str, "threads": str}
    photos: list[str],  # пути к файлам фото
    platforms: list[str] | None = None,  # None = все
) -> dict:
    """
    Публикует пост во все (или выбранные) платформы.
    Возвращает результат по каждой.
    """
    if platforms is None:
        platforms = ["telegram", "vk", "instagram", "threads"]

    results = {}
    errors = []

    # 1. Telegram (первым)
    if "telegram" in platforms:
        try:
            r = await publish_telegram(TG_CHANNEL_ID, versions["telegram"], photos)
            results["telegram"] = "✅"
        except Exception as e:
            results["telegram"] = f"❌ {e}"
            errors.append(f"Telegram: {e}")

    await asyncio.sleep(2)

    # 2. VK
    if "vk" in platforms:
        try:
            r = await publish_vk(versions["vk"], photos)
            results["vk"] = "✅"
        except Exception as e:
            results["vk"] = f"❌ {e}"
            errors.append(f"VK: {e}")

    await asyncio.sleep(2)

    # 3. Instagram
    if "instagram" in platforms:
        try:
            if len(photos) == 1:
                r = await publish_instagram_single(versions["instagram"], photos[0])
            elif len(photos) > 1:
                r = await publish_instagram_carousel(versions["instagram"], photos)
            else:
                results["instagram"] = "⚠️ Нет фото"
                r = None
            if r:
                results["instagram"] = "✅"
        except Exception as e:
            results["instagram"] = f"❌ {e}"
            errors.append(f"Instagram: {e}")

    await asyncio.sleep(2)

    # 4. Threads
    if "threads" in platforms:
        try:
            if photos:
                image_url = await upload_photo_to_hosting(photos[0])
                r = await publish_threads_image(versions["threads"], image_url)
            else:
                r = await publish_threads_text(versions["threads"])
            results["threads"] = "✅" if r else "❌ Ошибка публикации"
        except Exception as e:
            results["threads"] = f"❌ {e}"
            errors.append(f"Threads: {e}")

    # Сводка для Александра
    summary = "📤 Результат публикации:\n\n"
    emoji_map = {"telegram": "📱", "vk": "📘", "instagram": "📸", "threads": "🔗"}
    for p in platforms:
        summary += f"{emoji_map.get(p, '•')} {p.title()}: {results.get(p, '—')}\n"

    return {"results": results, "errors": errors, "summary": summary}
```

---

## 8. ОБНОВЛЕНИЕ ТОКЕНОВ META (Instagram + Threads)

Токены Meta живут 60 дней. Автообновление:

```bash
cat > ~/agents/travel-copywriter/scripts/token_refresh.sh << 'SHEOF'
#!/bin/bash
source ~/agents/travel-copywriter/config/.env
ENV_FILE=~/agents/travel-copywriter/config/.env
LOG_FILE=~/agents/travel-copywriter/data/token_refresh.log

# --- Instagram (Facebook Graph API) ---
NEW_IG_TOKEN=$(curl -s "https://graph.facebook.com/v21.0/oauth/access_token\
?grant_type=fb_exchange_token\
&client_id=${FB_APP_ID}\
&client_secret=${FB_APP_SECRET}\
&fb_exchange_token=${IG_ACCESS_TOKEN}" | jq -r '.access_token')

if [ "$NEW_IG_TOKEN" != "null" ] && [ -n "$NEW_IG_TOKEN" ]; then
    sed -i "s|IG_ACCESS_TOKEN=.*|IG_ACCESS_TOKEN=${NEW_IG_TOKEN}|" "$ENV_FILE"
    echo "$(date): IG token refreshed" >> "$LOG_FILE"
else
    echo "$(date): FAILED to refresh IG token" >> "$LOG_FILE"
fi

# --- Threads ---
NEW_TH_TOKEN=$(curl -s "https://graph.threads.net/refresh_access_token\
?grant_type=th_refresh_token\
&access_token=${THREADS_ACCESS_TOKEN}" | jq -r '.access_token')

if [ "$NEW_TH_TOKEN" != "null" ] && [ -n "$NEW_TH_TOKEN" ]; then
    sed -i "s|THREADS_ACCESS_TOKEN=.*|THREADS_ACCESS_TOKEN=${NEW_TH_TOKEN}|" "$ENV_FILE"
    echo "$(date): Threads token refreshed" >> "$LOG_FILE"
else
    echo "$(date): FAILED to refresh Threads token" >> "$LOG_FILE"
fi
SHEOF

chmod +x ~/agents/travel-copywriter/scripts/token_refresh.sh
```

Cron (каждые 50 дней):
```bash
0 3 */50 * * ~/agents/travel-copywriter/scripts/token_refresh.sh
```

> **VK**: токен с scope `offline` бессрочный, обновлять не нужно.

---

## 9. ЧЕКЛИСТ ЗАПУСКА

```
[ ] 1.  Создана структура папок ~/agents/travel-copywriter/
[ ] 2.  Заполнен .env (все токены и ID)
[ ] 3.  SKILL.md скопирован в папку агента

--- Telegram ---
[ ] 4.  Бот добавлен в канал как администратор
[ ] 5.  Тестовая отправка текста в канал — ✅
[ ] 6.  Тестовая отправка фото с caption — ✅
[ ] 7.  Тестовая отправка медиагруппы (2+ фото) — ✅

--- VK ---
[ ] 8.  Приложение создано, токен получен
[ ] 9.  group_id определён
[ ] 10. Тестовый пост на стену — ✅
[ ] 11. Тестовый пост с фото — ✅

--- Instagram ---
[ ] 12. Аккаунт переведён в Business/Creator
[ ] 13. Facebook Page привязана к IG
[ ] 14. Facebook App создано, токен получен
[ ] 15. IG Business Account ID определён
[ ] 16. Хостинг для фото настроен (nginx/imgbb/cloudinary)
[ ] 17. Тестовая публикация одного фото — ✅
[ ] 18. Тестовая публикация карусели — ✅

--- Threads ---
[ ] 19. Threads API подключён в Facebook App
[ ] 20. Токен получен
[ ] 21. Тестовая публикация текста — ✅
[ ] 22. Тестовая публикация с фото — ✅

--- Финал ---
[ ] 23. Автообновление токенов Meta в cron — ✅
[ ] 24. publisher.py: полный тест публикации во все 4 платформы — ✅
```

---

## 10. TROUBLESHOOTING

| Проблема | Решение |
|----------|---------|
| TG: `caption too long` | Текст > 1024 символов. Скрипт автоматически отправит текст отдельным сообщением |
| VK: `access denied` | Проверь scope токена (нужны `wall`, `photos`). Если токен сообщества — проверь права |
| VK: `wall.post` возвращает `captcha_needed` | VK подозревает бота. Увеличь паузу между постами, или реши капчу вручную один раз |
| IG: `media not ready` | Instagram не успел обработать фото. Увеличь `sleep` до 10–15 секунд |
| IG: `invalid image` | Фото должно быть JPEG/PNG, минимум 320px, максимум 30MB. Проверь формат |
| IG: URL не доступен | Instagram не может скачать фото с твоего сервера. Проверь: nginx запущен, файл существует, домен резолвится |
| Threads: 403 | Токен истёк. Запусти `token_refresh.sh` |
| Threads: `media not finished` | Увеличь паузу после создания container до 5 секунд |
| Все платформы: timeout | Сервер VPS за VPN? API Meta/VK могут блокировать некоторые IP. Проверь доступность через `curl` |
