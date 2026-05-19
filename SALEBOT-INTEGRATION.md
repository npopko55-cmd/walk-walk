# Salebot — интеграция и техническая логика

Документ для техспеца, который будет настраивать декодирование UTM-меток на стороне Salebot для записи в карточку юзера.

---

## TL;DR

1. Юзер кликает по рекламе → попадает на нашу прокладку `go.html`
2. Прокладка отправляет UTM-данные в Я.Метрику и редиректит в TG с **закодированным startapp**
3. Юзер открывает мини-апп ИЛИ пишет `/start` боту → **Salebot получает строку из startapp**
4. **Задача техспеца:** декодировать эту строку и сохранить UTM в карточку юзера, чтобы потом можно было сегментировать рассылки

---

## 1. Архитектура (зачем нужна прокладка)

### Проблема
В Telegram-ссылке `https://t.me/shagai_doma_bot/plan?startapp=XXX` Telegram отбрасывает ВСЕ query-параметры кроме `startapp`. Поэтому полный UTM-хвост Я.Директа (utm_source, utm_campaign, cm_id, erid и т.д.) **физически невозможно** довести до бота напрямую.

### Решение
Промежуточная страница `go.html` на нашем сайте, которая:
- Принимает полный URL с UTM-метками от Я.Директа (или посевов)
- Логирует визит в Я.Метрику со всеми параметрами
- Упаковывает критичные UTM-поля в короткий base64-код
- Редиректит юзера в Telegram, передавая упакованный код в `startapp`

---

## 2. URL прокладки и параметры

### URL
```
https://npopko55-cmd.github.io/walk-walk/go.html
```

### Поддерживаемые параметры
| Параметр | Назначение | Пример |
|---|---|---|
| `dest` | Куда редиректить: `tg` / `vk` / `max`. По умолчанию `tg` | `tg` |
| `from` | Источник: `yad` (Я.Директ, по умолчанию) или `seed` (посевы) | `seed` |
| `startapp` | Кастомный идентификатор кампании/канала | `spring2026`, `hacking` |
| `utm_*` | Все стандартные UTM-метки | `yandex`, `cpc`, ... |
| `cm_id` | cm_id Я.Директа | `709930302_5750934_...` |
| `campaign_id` | ID кампании | `709930302` |
| `erid` | ОРД-маркировка (для посевов с РФ-маркировкой) | `2Vtzqx4xNSU` |

### Что делает прокладка
1. Парсит все UTM-параметры из URL
2. Отправляет в Я.Метрику (счётчик `108598378`):
   - `ym('params', { ad_click: {все_utm} })`
   - `ym('reachGoal', 'ad_click')`
   - `ym('reachGoal', 'ad_click_tg')` (или `_vk`, `_max`)
3. Упаковывает в base64-startapp 3 поля:
   - `f` = откуда (y=яндекс, s=посевы)
   - `c` = идентификатор кампании (campaign_id или название канала)
   - `e` = erid (если есть)
4. Редиректит:
   - `tg://resolve?domain=shagai_doma_bot&appname=plan&startapp=u<base64>` (нативное открытие TG-приложения)
   - Через 1.2 сек fallback на `https://t.me/shagai_doma_bot/plan?startapp=u<base64>`

---

## 3. Формат закодированного startapp

### Структура
```
u<base64url>
```

- Префикс `u` — маркер что дальше идёт base64 (без префикса = старый формат `yad-XXX` / `seed-XXX`)
- Base64url — стандартный base64 с заменой:
  - `+` → `-`
  - `/` → `_`
  - убран padding `=`

### Что внутри base64
После раскодирования получается querystring:
```
f=y&c=999&e=2Vtzqx4xNSU
```

### Ключи (короткие чтобы влезть в TG-лимит 64 симв)
| Ключ | Значение | Маппинг |
|---|---|---|
| `f` | from | `y` → utm_source=`yandex`, source=яндекс_директ. `s` → utm_source=`tg_seed`, source=посев |
| `c` | campaign | utm_campaign / идентификатор кампании |
| `e` | erid | ОРД-маркировка |

### Лимиты
- TG startapp = максимум **64 символа**
- Допустимые символы: `[A-Za-z0-9_-]`
- Наш формат всегда укладывается (тесты прошли на 5/5 кейсов, макс 37 симв)

---

## 4. Тестовые векторы

Готовые примеры для тестирования декодера:

| Входной startapp | Должно декодироваться в |
|---|---|
| `uZj15JmM9OTk5` | `{utm_source:"yandex", utm_campaign:"999", _from:"yad"}` |
| `uZj15JmM9MTEyMzQ1Njc4` | `{utm_source:"yandex", utm_campaign:"112345678", _from:"yad"}` |
| `uZj15JmM9c3ByaW5nMjAyNg` | `{utm_source:"yandex", utm_campaign:"spring2026", _from:"yad"}` |
| `uZj1zJmM9aGFja2luZyZlPTJWdHpxeDR4TlNV` | `{utm_source:"tg_seed", utm_campaign:"hacking", erid:"2Vtzqx4xNSU", _from:"seed"}` |
| `uZj1zJmM9bWFya2V0aW5nMjQ` | `{utm_source:"tg_seed", utm_campaign:"marketing24", _from:"seed"}` |

### Особые случаи
- Если startapp **не начинается с `u`** → это старый/прямой формат, не декодировать. Возможные значения:
  - `yad-999`, `yad-spring2026` (старый формат Я.Директа)
  - `seed-hacking`, `marketing24`, `fitnesstop` (посевы — могут быть как с префиксом `seed-`, так и без)
  - `retake`, `reset` — служебные флаги (пересдача квиза), НЕ источник трафика

---

## 5. Готовый код декодера

### Python
```python
import base64
from urllib.parse import unquote

def decode_startapp(sp: str) -> dict | None:
    """Декодирует startapp из формата u<base64> в dict с UTM-метками.
    Возвращает None если формат не u<base64> (старый формат yad-XXX / seed-XXX)."""
    if not sp or not sp.startswith('u') or len(sp) < 4:
        return None
    try:
        b64 = sp[1:].replace('-', '+').replace('_', '/')
        # Восстановить padding
        b64 += '=' * (-len(b64) % 4)
        decoded = base64.b64decode(b64).decode('utf-8')
        result = {}
        for kv in decoded.split('&'):
            if '=' not in kv:
                continue
            k, v = kv.split('=', 1)
            v = unquote(v)
            if k == 'f':
                result['utm_source'] = 'yandex' if v == 'y' else 'tg_seed'
                result['_from'] = 'yad' if v == 'y' else 'seed'
            elif k == 'c':
                result['utm_campaign'] = v
            elif k == 'e':
                result['erid'] = v
        return result if '_from' in result else None
    except Exception:
        return None
```

### JavaScript (для Salebot.pro блока JS-код или Cloudflare Worker)
```javascript
function decodeStartapp(sp) {
  if (!sp || sp.charAt(0) !== 'u' || sp.length < 4) return null;
  try {
    var b64 = sp.slice(1).replace(/-/g, '+').replace(/_/g, '/');
    while (b64.length % 4) b64 += '=';
    var decoded = atob(b64);
    var result = {};
    decoded.split('&').forEach(function(kv) {
      var ix = kv.indexOf('=');
      if (ix < 0) return;
      var k = kv.slice(0, ix);
      var v = decodeURIComponent(kv.slice(ix + 1));
      if (k === 'f') {
        result.utm_source = (v === 'y') ? 'yandex' : 'tg_seed';
        result._from = (v === 'y') ? 'yad' : 'seed';
      } else if (k === 'c') result.utm_campaign = v;
      else if (k === 'e') result.erid = v;
    });
    return result._from ? result : null;
  } catch (e) { return null; }
}
```

### PHP
```php
function decodeStartapp(string $sp): ?array {
    if (!$sp || $sp[0] !== 'u' || strlen($sp) < 4) return null;
    $b64 = strtr(substr($sp, 1), '-_', '+/');
    $b64 .= str_repeat('=', (4 - strlen($b64) % 4) % 4);
    $decoded = base64_decode($b64);
    if ($decoded === false) return null;
    $result = [];
    foreach (explode('&', $decoded) as $kv) {
        if (strpos($kv, '=') === false) continue;
        [$k, $v] = explode('=', $kv, 2);
        $v = urldecode($v);
        if ($k === 'f') {
            $result['utm_source'] = $v === 'y' ? 'yandex' : 'tg_seed';
            $result['_from'] = $v === 'y' ? 'yad' : 'seed';
        } elseif ($k === 'c') $result['utm_campaign'] = $v;
        elseif ($k === 'e') $result['erid'] = $v;
    }
    return isset($result['_from']) ? $result : null;
}
```

---

## 6. Интеграция в Salebot

### Где Salebot получает startapp
Когда юзер кликает по deeplink `t.me/shagai_doma_bot?start=XXX` или открывает мини-апп через стартовую команду — параметр `XXX` приходит в Salebot как:
- В Salebot.pro: переменная `{first_message_arg}` или `{start_message_arg}` (зависит от версии)
- В кастомных ботах: payload команды `/start`

### Алгоритм для Salebot
1. **На входе в бот** (триггер «Новый пользователь» или «Команда /start»):
   - Прочитать `{first_message_arg}` (или эквивалент)
   - Вызвать декодер (через JS-блок, либо через HTTP-запрос на webhook)
   - Сохранить результат в **поля карточки юзера**:
     - `utm_source` (yandex / tg_seed)
     - `utm_campaign` (например 709930302 или hacking)
     - `_from` (yad / seed)
     - `erid` (если есть, для отчётности по ОРД)

2. **Что добавить в Salebot заранее:**
   - **Создать поля юзера:** `utm_source`, `utm_campaign`, `utm_from`, `erid`
   - **Создать сегменты** для рассылок:
     - «Пришли из Я.Директа» (фильтр: `utm_source = yandex`)
     - «Пришли с посевов» (фильтр: `utm_source = tg_seed`)
     - «Кампания такая-то» (фильтр: `utm_campaign = X`)

### Псевдокод сценария Salebot
```
ТРИГГЕР: новое сообщение от юзера, текст начинается с /start
  ↓
1. Прочитать аргумент команды → переменная {sp}
  ↓
2. Если {sp} начинается с "u":
      JS-блок (или HTTP-вызов webhook) → декодировать {sp}
      → сохранить в карточку: utm_source, utm_campaign, _from, erid
   Иначе:
      Старый формат (yad-XXX / seed-XXX / просто слово):
      → сохранить как utm_campaign = {sp}
      → определить utm_source по префиксу (yad → yandex, seed/прочее → tg_seed)
  ↓
3. Отправить welcome-сообщение с кнопкой запуска мини-аппа:
   t.me/shagai_doma_bot/plan?startapp={sp}
```

---

## 7. Варианты реализации декодера в Salebot

### A. JS-блок прямо в Salebot.pro (если у вас Salebot.pro)
Salebot.pro поддерживает блок «Выполнить JS». Туда вставить JavaScript-код выше + код сохранения в переменные:

```javascript
var sp = '{first_message_arg}';
var result = decodeStartapp(sp);
if (result) {
  saveVariable('utm_source', result.utm_source);
  saveVariable('utm_campaign', result.utm_campaign);
  saveVariable('utm_from', result._from);
  if (result.erid) saveVariable('erid', result.erid);
}
```
(точный синтаксис `saveVariable` посмотри в доке Salebot)

### B. HTTP webhook на Cloudflare Worker (универсальный путь)
1. Создать аккаунт на cloudflare.com (бесплатно)
2. Создать Worker с кодом-декодером (JS из примера выше) который принимает `?sp=...` и возвращает JSON
3. В Salebot блок «HTTP-запрос» → URL `https://walkwalk-decoder.workers.dev/?sp={first_message_arg}` → распарсить ответ → сохранить в переменные

Можем поднять Worker за 10 минут, если решите идти этим путём.

### C. PHP-скрипт на любом хостинге
Если есть свой хостинг — поднять PHP-файл с кодом выше + Salebot вызывает его как webhook.

---

## 8. Что уже работает на нашей стороне (готово)

| Компонент | Статус |
|---|---|
| `go.html` — прокладка с UTM-трекингом | ✅ |
| Кодирование UTM в startapp (base64) | ✅ |
| Я.Метрика — счётчик `108598378`, цели `ad_click`, `ad_click_tg/vk/max` | ✅ |
| Я.Метрика — парные цели `yad_tg_*` / `seed_tg_*` (детектор по источнику) | ✅ |
| tg:// deeplink для нативного открытия TG-приложения | ✅ |
| Callback в `ym('reachGoal', ..., cb)` чтобы не терять хиты при редиректе | ✅ |
| `from=seed` параметр для прокладки посевов (с ERID) | ✅ |
| Декодер base64 на стороне мини-аппа (для Метрики) | ✅ |

## 9. Что нужно сделать со стороны Salebot (не готово)

| Задача | Кто |
|---|---|
| Создать поля юзера: `utm_source`, `utm_campaign`, `utm_from`, `erid` | техспец Salebot |
| Добавить блок декодирования startapp на старте сценария | техспец Salebot |
| Создать сегменты для рассылок по UTM-меткам | техспец Salebot |
| (опционально) Welcome-сообщение с inline-кнопкой запуска мини-аппа | техспец Salebot |

---

## 10. Контакты и ссылки

- **Репозиторий сайта:** https://github.com/npopko55-cmd/walk-walk
- **Прокладка:** https://npopko55-cmd.github.io/walk-walk/go.html
- **Документ с архитектурой:** [CONTEXT.md](./CONTEXT.md)
- **Я.Метрика счётчик:** 108598378 (npopko55-cmd.github.io/walk-walk/)
- **Бот:** [@shagai_doma_bot](https://t.me/shagai_doma_bot)

## 11. Тестовые ссылки для проверки

После настройки на стороне Salebot — протестировать что декодирование работает:

**Тест 1 — Я.Директ:**
```
https://npopko55-cmd.github.io/walk-walk/go.html?dest=tg&utm_source=yandex&campaign_id=999
```
Открыть → попадёт в TG-бот. Salebot должен сохранить: `utm_source=yandex, utm_campaign=999, utm_from=yad`

**Тест 2 — посев с ERID:**
```
https://npopko55-cmd.github.io/walk-walk/go.html?dest=tg&from=seed&startapp=hacking&erid=2Vtzqx4xNSU
```
Salebot должен сохранить: `utm_source=tg_seed, utm_campaign=hacking, utm_from=seed, erid=2Vtzqx4xNSU`

**Тест 3 — старый формат (без base64):**
```
https://t.me/shagai_doma_bot/plan?startapp=marketing24
```
Прокладку обходит. Salebot получает startapp = `marketing24` → декодер вернёт null → fallback на старую логику: `utm_campaign=marketing24, utm_source=tg_seed (по умолчанию)`
