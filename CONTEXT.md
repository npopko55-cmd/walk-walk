# walk-walk — контекст проекта

Минимальный мини-апп "Шагай Дома" для холодного трафика. Открывается в Telegram / VK / MAX через мини-аппы. Бэкенда нет — только статика на GitHub Pages.

## Стек и хостинг

- **Технология:** vanilla HTML + CSS + JS. Без сборщиков, без React/Vue.
- **Хостинг:** GitHub Pages, репо [npopko55-cmd/walk-walk](https://github.com/npopko55-cmd/walk-walk).
- **Прод URL:** `https://npopko55-cmd.github.io/walk-walk/`
- **Кастомный домен:** нет (в `offer.html` упоминается `walk-walk.ru`, но это юр-ссылка на главный сайт ШД, не на этот мини-апп).
- **Деплой:** push в `main` → GitHub Pages обновляется ~30 секунд.

## Структура файлов

```
walk-walk/
├── index.html          # TG мини-апп: главная (CTA + 1 экран оффера)
├── quiz.html           # TG мини-апп: квиз 7 вопросов
├── plan.html           # ОБЩИЙ для всех 3 мессенджеров: страница плана, тарифы
├── go.html             # Прокладка-редиректор для Я.Директа (UTM-хвост → мини-апп)
├── offer.html          # Юр-документ "Оферта"
├── privacy.html        # Политика конфиденциальности
├── cases.html          # Кейсы клиентов
├── ad.html             # Креатив для разрешения коммерции
├── approval.html       # Согласие на обработку ПДн
├── safe.html           # Гарантии возврата
├── vk/
│   ├── index.html      # VK мини-апп: главная (vk-bridge SDK)
│   └── quiz.html       # VK мини-апп: квиз
├── max/
│   ├── index.html      # MAX мини-апп: главная
│   └── quiz.html       # MAX мини-апп: квиз
├── images/             # Награды, кейсы, фоны
├── videos/             # Бесплатные тренировки (mp4)
├── output/             # mock-выгрузки
├── favicon.svg
└── legal.css           # стили юр-страниц
```

## Точки входа (мини-аппы)

| Мессенджер | Бот / app ID | Web App URL |
|---|---|---|
| Telegram | `@shagai_doma_bot` → `plan` | `https://npopko55-cmd.github.io/walk-walk/` |
| VK | `vk.com/app54574304` | `https://npopko55-cmd.github.io/walk-walk/vk/` |
| MAX | `max.ru/id222391942312_bot` | `https://npopko55-cmd.github.io/walk-walk/max/` |

После выбора программы в квизе все 3 мессенджера переходят в общий `plan.html`, где есть детектор мессенджера через `utm_medium`.

## Поток пользователя

```
[Trafic source]
   │
   ├── Я.Директ ─→ go.html ─→ редирект ─→ мини-апп (TG/VK/MAX)
   ├── Посевы TG ─→ t.me/shagai_doma_bot/plan?startapp=XXX
   ├── Посевы MAX ─→ max.ru/id222391942312_bot?startapp=XXX
   └── VK Ads ─→ vk.com/app54574304
       │
   index.html (или vk/index.html, max/index.html)
       │  событие: app_open
       ↓
   quiz.html (выбор пола/возраста/веса/целей)
       │  событие: quiz_start, quiz_complete
       ↓
   plan.html (персональный план + тарифы)
       │  событие: plan_view, tariff_click
       ↓
   my.walk-walk.ru/1_club (или /3_club) — оплата (внешний домен)
```

## Аналитика: Я.Метрика

**Счётчик:** `108598378` (название: walk-walk plan).
**Webvisor + Clickmap + Ecommerce(dataLayer) включены.**

### Цели — 61 шт

Делятся на 2 системы:

**1. "Какой мессенджер" (старая, не трогать) — 30 целей:**

| Префикс | Где живёт | Кол-во |
|---|---|---|
| `<event>` (без префикса) | TG мини-апп (index.html, quiz.html) | 12 целей |
| `vk_<event>` | VK мини-апп (vk/index.html, vk/quiz.html) | 9 целей |
| `max_<event>` | MAX мини-апп (max/index.html, max/quiz.html) | 9 целей |

Стандартные ивенты воронки: `app_open`, `cta_to_quiz`, `quiz_start`, `quiz_complete`, `plan_view`, `tariff_click`. Плюс служебные: `returning_user`, `preframe_shown/continue`, `loader_shown/complete`, `tg_write_granted/denied`, `free_training_play`.

**2. "Источник × мессенджер" (новая, для платных каналов) — 31 цель:**

Префикс двойной: `<источник>_<мессенджер>_<событие>`.

| Источник | Префикс | Мессенджер | Воронка |
|---|---|---|---|
| Я.Директ | `yad_tg_*` | TG | Я.Директ → TG |
| Я.Директ | `yad_vk_*` | VK | Я.Директ → VK (VK на стопе, ловим впрок) |
| Посевы TG | `seed_tg_*` | TG | Посевы TG |
| Посевы MAX | `seed_max_*` | MAX | Посевы MAX |
| VK таргет | `targ_vk_*` | VK | VK таргет (VK на стопе, ловим впрок) |
| — | `ad_click` | — | Отдельная цель: клик из рекламы на go.html (не в воронке) |

### Воронки (отчёты в Метрике)

**Существующие 3** (общие, любой источник):
- Воронка · Telegram Mini App
- Воронка · VK Mini App
- Воронка · MAX Mini App

**Новые 5** (нужно настроить руками в кабинете):
- Воронка · Я.Директ → TG (по 6 целям `yad_tg_*`)
- Воронка · Я.Директ → VK (по 6 целям `yad_vk_*`)
- Воронка · Посевы TG (по 6 целям `seed_tg_*`)
- Воронка · Посевы MAX (по 6 целям `seed_max_*`)
- Воронка · VK таргет (по 6 целям `targ_vk_*`)

## Детектор источника и мессенджера

Helper `window.__pairGoal(event, params)` вставлен во все 7 файлов с Метрикой. Шлёт **парную** цель с префиксом `<src>_<msg>_<event>` рядом с обычной.

### Логика определения источника (`srcKey`)

```
utm_source === 'yandex' ИЛИ startapp начинается с 'yad' → 'yad'
utm_source ∈ {'vk_ads','vk_target','vk_targ'} ИЛИ начинается с 'cpc_vk'/'cpm_vk' → 'targ'
startapp непустой и НЕ в чёрном списке и НЕ начинается с 'yad' → 'seed'
utm_source начинается с 'seed' → 'seed'
иначе → null (источник не определён, парная цель не шлётся)
```

**Чёрный список (технические маркеры, не считаются источниками):**
- `retake`, `reset` — пересдача квиза
- `test`, `debug`, `dev` — наши тесты

### Логика определения мессенджера (`msgKey`)

```
utm_medium === 'vk_miniapp'  → 'vk'
utm_medium === 'max_miniapp' → 'max'
utm_medium === 'tg_miniapp'  → 'tg'
иначе window.__msgPlatform   → 'vk'/'max'/'tg' (выставляется в начале каждого файла)
иначе по наличию SDK         → 'vk' (vkBridge), 'max' (max API), 'tg' (Telegram.WebApp)
```

### Где живёт helper

Helper встроен инлайном в `<script>` после инициализации Метрики в каждом из 7 файлов:
- `index.html`, `quiz.html`, `plan.html` (TG / общий)
- `vk/index.html`, `vk/quiz.html`
- `max/index.html`, `max/quiz.html`

В VK и MAX файлах перед helper-ом выставляется `window.__msgPlatform = 'vk'` / `'max'` соответственно — для надёжного определения мессенджера ещё до того как утилитарные SDK подгрузятся.

Helper не дублируется в отдельный файл, чтобы не плодить зависимости (один раз скопирован — везде работает). Если меняешь логику — меняй во всех 7 файлах.

## Прокладка go.html — для трафика с Я.Директа

**URL:** `https://npopko55-cmd.github.io/walk-walk/go.html`

**Параметры:**
- `dest=tg|vk|max` — куда редиректить (обязательный, по умолчанию tg)
- `startapp=...` — что положить в startapp мини-аппа (опционально, см. ниже)
- `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term`, `cm_id`, и весь хвост Я.Директа (auto-парсятся)

**Поведение:**
1. Принимает все UTM/cm_id Директа, шлёт в Метрику (`ad_click` + `ad_click_<dest>` + params)
2. Через 600 мс редиректит в `t.me/.../?startapp=<startapp>` (или vk.com / max.ru)
3. Если автоматический редирект не сработал — fallback CTA "Открыть вручную"

### Префиксование startapp

Любой кастомный `startapp` из URL Директа **всегда префиксуется `yad-`** (если ещё не префиксован). Это нужно чтобы детектор в мини-аппе корректно засчитал визит в `yad_<msg>_*`.

Примеры:
| `?startapp=` в URL Директа | Что прилетит в TG |
|---|---|
| `spring2026` | `yad-spring2026` |
| `yad-newyear` | `yad-newyear` (уже с префиксом — оставляется) |
| не указан, но есть cm_id | `yad-<campaign_id>` (берётся из cm_id) |
| ничего | `yad` (минимум) |

## Шаблоны ссылок для Я.Директа

В Я.Директе в "Ссылку на сайт" объявления вставляется:

```
https://npopko55-cmd.github.io/walk-walk/go.html
  ?dest=tg
  &utm_source=yandex
  &utm_medium={source_type}
  &utm_campaign={campaign_name_lat}|{campaign_id}
  &utm_term={keyword}
  &utm_content=gid:{gbid}|aid:{ad_id}|seg:{retargeting_id}|site:{source}|dev:{device_type}|pos:{position_type}{position}|reg:{region_id}
  &cm_id={campaign_id}_{gbid}_{ad_id}_{phrase_id}_{retargeting_id}_{source}_{source_type}_{device_type}_{position_type}_{region_id}
```

Для VK замени `dest=tg` на `dest=vk`, для MAX — `dest=max`.

Если хочешь свой маркер кампании — добавь `&startapp=spring2026` (любое слово). go.html автоматом префиксует `yad-`.

## Связки и доступы

**Доступы к счётчику 108598378:**
- `jenya.vyaznickov` (edit) — Женя Вязников, директолог
- `porg-kzjhbxtk` (edit) — служебный логин, проверить кто это

**Интеграция Метрика ↔ Я.Директ:**
- Через API не подтверждена. Проверить в кабинете Метрики: *Настройка → Загрузка данных → Связки с Директом*.
- Без этой связки `cm_id={campaign_id}_{gbid}_...` не расшифруется в имена кампаний.

## Известные ограничения

1. **VK мини-апп через хэш-фрагменты (`#ads_v1`) трекается криво.** Я.Метрика теряет launch_params от VK Bridge. Нужно отдельно решать через `VKWebAppGetLaunchParams`. Пока VK на стопе.
2. **Сквозные визиты между go.html и plan.html не сшиваются.** Разные ClientID Метрики (браузер vs Telegram WebView). Воронка `ad_click → app_open` всегда покажет 0% — это нормально. Поэтому `ad_click` оставлен как отдельная цель, не первый шаг воронки.
3. **Telegram игнорирует все query-параметры кроме `startapp`.** Поэтому UTM-хвост напрямую на `t.me/.../` не повесить — нужна прокладка go.html.
4. **`my.walk-walk.ru/1_club` и `/3_club` — внешние страницы оплаты.** Они на другом домене, у них свой счётчик/трекинг. Цели `tariff_click` ловятся только при клике, а конверсия в оплату трекается уже у Димы / на стороне my.walk-walk.ru.

## Метрика API — для отладки в треде

OAuth-токен Метрики (если есть) → можно дёргать API напрямую через curl:

```bash
TOKEN="..."
# Список целей
curl -s "https://api-metrika.yandex.net/management/v1/counter/108598378/goals" \
  -H "Authorization: OAuth $TOKEN" | python3 -m json.tool

# Создать цель типа action (JS-событие)
curl -s -X POST "https://api-metrika.yandex.net/management/v1/counter/108598378/goals" \
  -H "Authorization: OAuth $TOKEN" -H "Content-Type: application/json" \
  -d '{"goal":{"name":"...","type":"action","conditions":[{"type":"exact","url":"event_id"}]}}'

# Свежий трафик за сегодня по источникам
curl -s "https://api-metrika.yandex.net/stat/v1/data?ids=108598378&metrics=ym:s:visits&dimensions=ym:s:UTMSource,ym:s:UTMCampaign&date1=today&date2=today" \
  -H "Authorization: OAuth $TOKEN" | python3 -m json.tool
```

**Безопасность:** OAuth-токен — пароль на ~1 год. Не пиши в чат и не коммить в репо. Если случайно засветил — отозвать на [oauth.yandex.ru/client](https://oauth.yandex.ru/client).

## Чек-лист при новых правках

Когда добавляешь новый источник или мессенджер:

1. **API: создать N×6 целей** через POST `/counter/108598378/goals` (по 6 шагов воронки)
2. **Код: расширить детектор `srcKey()` в helper-е** во всех 7 файлах (одинаковая копия)
3. **Документировать** в этом CONTEXT.md
4. **Воронку настроить руками** в кабинете Метрики (по 6 шагам)
