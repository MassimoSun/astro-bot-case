# 🔮 Астра — Telegram-бот: астрология + таро + ИИ

Персонализированный астрологический бот: расчёты на швейцарских эфемеридах
(swiss ephemeris), ИИ-астролог с памятью диалога, расклады таро, мини-приложение
(Telegram Mini App) и монетизация через Telegram Stars.

**Исходный код закрыт** — это коммерческий продукт. Этот репозиторий —
демонстрационный кейс: архитектура, метрики, технические решения и фрагменты
кода, которые показывают уровень владения стеком.

🔗 **Живой бот:** [@astra_tar0t_bot](https://t.me/astra_tar0t_bot) (продакшен)

---

## 🧩 Что умеет

| Функция | Описание |
|---|---|
| 🌍 Натальная карта | Дата/время/город рождения → полная карта: 12 планет, дома (Плацид), аспекты. Свое ядро расчёта поверх swisseph |
| 🌙 Гороскоп дня | Не «по знаку» — считается по натальной карте + текущие транзиты |
| 🃏 Таро | Колода Уэйта 78 карт, 5 раскладов (от «Карты дня» до «Кельтского креста»), 1 бесплатный расклад в день |
| 💬 ИИ-астролог | Диалог с памятью контекста; в системный промпт подставляются реальные данные карты пользователя |
| ❤️ Синастрия | Совместимость пары по двум картам (Pro) |
| 📅 Прогноз на месяц | Почтенно-недельный срез транзитов (Pro) |
| ⭐ Telegram Stars | Платежи: invoice, pre-checkout, активация Pro на 30 дней |
| 🌐 Mini App | Ввод профиля, гороскоп, тяга карт, оплата — всё внутри Telegram |
| 🔒 GDPR / 152-ФЗ | `/delete` удаляет все данные пользователя полностью |

## 🏗 Архитектура

```
┌─────────────┐   ┌──────────────┐   ┌─────────────┐
│  Telegram   │   │  Mini App    │   │   Бот-админ │
│ (aiogram 3) │   │  (WebApp SDK)│   │ /stats, /bc │
└──────┬──────┘   └──────┬───────┘   └─────────────┘
       │ aiogram          │ aiohttp, HMAC-SHA256 initData
       ▼                  ▼
┌──────────────────────────────────────┐
│ handlers/ — роутеры по фичам         │
│ services.py — лимиты, подписки,      │
│   GDPR, память ИИ (бизнес-логика)    │
├──────────────────────────────────────┤
│ astro.py    — ядро астрорасчётов     │
│ tarot.py    — колода + расклады      │
│ ai.py       — провайдеры ИИ + ретраи │
├──────────────────────────────────────┤
│ db.py — SQLAlchemy 2 (async)         │
│   users, natal_charts, subscriptions,│
│   daily_usage, ai_conversations,     │
│   payments                           │
└──────────────────────────────────────┘
       │
   ┌───┴────┐
   ▼        ▼
SQLite    PostgreSQL
(тесты)   (продакшен)
```

**Стек:** Python 3.11 · aiogram 3.31 · aiohttp · SQLAlchemy 2 (async) ·
swiss ephemeris (pyswisseph 2.10) · OpenAI-совместимый клиент
(Gemini как основной провайдер, OpenRouter как резервный) · Docker · GitHub Actions

---

## 🧪 Качество

- **202 теста**, покрытие **96.7%**, жёсткий порог 92% в CI
- Линтеры: **ruff** + **mypy** (strict-режим)
- CI на каждый push: тесты на Python 3.11/3.12, `pytest --cov`, артефакт `coverage.xml`
- Покрытие по модулям: `db`, `geo`, `ai`, `services`, `states`, `main` — 100%
- Тесты изолированы: Telegram API мокается, сеть не нужна.

---

## 🔐 Безопасность (выдержка — что реализовано в коде)

### Валидация Telegram initData

Mini App API не доверяет заголовкам. Подпись `initData` проверяется
HMAC-SHA256 по токену бота:

```python
init_data = request.headers.get("X-Telegram-Init-Data", "")
if init_data:
    token = settings.bot_token
    if not token or not check_webapp_signature(token, init_data):
        return None, sm          # 401
    parsed = parse_webapp_init_data(init_data)
    raw = parsed.auth_date
    ...
    if auth_ts <= 0 or time.time() - auth_ts > _INITDATA_MAX_AGE:
        return None, sm          # защита от replay
```

Защита от повтора: `auth_date` проверяется на свежесть (24 часа). До этого
юзер определялся по заголовку `X-Tg-User-Id`, которому мог доверять любой.

### Деградация астроядра

```python
try:
    import swisseph as swe
    HAS_SWISSEPH = True
except ImportError:  # pragma: no cover
    swe = None
    HAS_SWISSEPH = False
```

На Python 3.12 нет wheel pyswisseph. Бот не падает — таро и ИИ-чат
продолжают работать, астро-функции возвращают осмысленную ошибку 503.

### Ретраи ИИ-провайдеров

```python
def _is_retryable(err: Exception) -> bool:
    """Временные сбои, которые стоит повторить (перегрузка 503, лимит запросов)."""
    msg = str(err)
    return bool(
        "503" in msg or "high demand" in msg or "UNAVAILABLE" in msg
        or "429" in msg or "rate" in msg.lower()
        or "APIConnectionError" in type(err).__name__
        or "APITimeoutError" in type(err).__name__
    )
```

При отказе основного провайдера запрос уходит на резервный.

### Контекст ИИ из натальной карты

ИИ-астролог работает не с абстрактным знаком, а с реальными данными:

```python
def build_context_for_user(natal, transits) -> str:
    """Текстовая выжимка данных карты для системного промпта."""
    parts = [
        "ДАННЫЕ ПОЛЬЗОВАТЕЛЯ (натальная карта):",
        f"- Солнце: {natal.get('sun_sign', '?')}",
        f"- Луна: {natal.get('moon_sign', '?')}",
        f"- Асцендент: {natal.get('ascendant', '?')}",
    ]
    ...
    if transits:
        parts.append("ТЕКУЩИЕ ТРАНЗИТЫ (сегодня):")
        ...
```

## 📈 Монетизация

| | Free | Pro (149 Stars ≈ 299 ₽/мес) |
|---|---|---|
| Гороскоп дня | ✅ | ✅ |
| Таро | 1 / день | ♾ |
| ИИ-астролог | 1 вопрос / день | ♾ |
| Синастрия | ❌ | ✅ |
| Прогноз на месяц | ❌ | ✅ |

## 💼 Автор

ИИ-автоматизация, Telegram-боты под ключ: от ТЗ до продакшена
(сервер, CI/CD, мониторинг).

**Контакты:** Telegram @CleptoJuke
