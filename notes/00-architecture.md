# Модуль 0 · Как Home Assistant устроен внутри

> Статус: 🟡 · Начат: 2026-10-21

## Зачем это

Почти каждая «странность» HA объясняется одним из трёх механизмов ниже. Пример из моей системы: после restore ZHA-датчик был в списке устройств, но `unavailable`. Это не баг, а разница между **registry** (что существует) и **state machine** (что сейчас).

## Теория

### 1. Один процесс, один event loop

Home Assistant Core — Python-приложение на `asyncio`. Один процесс, один event loop. Интеграции — Python-пакеты, которые ядро загружает в этот же процесс. Медленная или зависшая интеграция может тормозить всё ядро (отсюда предупреждения «Detected blocking call…» в логах).

Внутри есть объект `hass`, у которого три главные подсистемы:

```
                ┌───────────────────────────────────────────────┐
  интеграции ──▶│  hass.states   STATE MACHINE (в памяти)       │──┐
  (ZHA,         │  entity_id → State(state, attributes, ...)    │  │ каждое изменение
   Solarman,    └───────────────────────────────────────────────┘  │ = событие
   Tasmota…)    ┌───────────────────────────────────────────────┐  ▼
                │  hass.bus      EVENT BUS                       │◀─ state_changed,
                │  все события системы                           │   call_service, ...
                └───────────────────────────────────────────────┘
                ┌───────────────────────────────────────────────┐
 UI/automation ▶│  hass.services ACTION REGISTRY                 │──▶ интеграция
 /API           │  switch.turn_on, number.set_value, ...         │    шлёт команду
                └───────────────────────────────────────────────┘    устройству
```

- **State machine** — текущее состояние каждой сущности. Живёт **только в памяти**. После рестарта пустая и заполняется заново.
- **Event bus** — шина событий. Всё, что происходит, публикуется как событие. Automations, recorder, frontend — подписчики этой шины.
- **Action registry** — каталог действий (`switch.turn_on`, `number.set_value`, `script.xxx`). В UI и YAML с 2024.8 они называются *actions*, раньше *services*. Событие при вызове до сих пор называется `call_service`.

### 2. Интеграции, платформы, config entries

- **Integration** — пакет с `manifest.json`: встроенный (`homeassistant/components/<domain>`) или кастомный (`custom_components/<domain>`, например Solarman из HACS).
- **Config entry** — экземпляр настроенной через UI интеграции. Хранится в `.storage/core.config_entries`. У одной интеграции может быть несколько entries (два инвертора — два entry).
- **Platform** — тип сущностей, которые интеграция создаёт. Solarman создаёт сущности платформ `sensor`, `binary_sensor`, `number`, `select`, `switch`, `time`, `button`. Поэтому у меня есть и `sensor.inverterdeye_pv_power`, и `number.inverterdeye_grid_max_export_power`: одна интеграция, разные платформы. **Домен в entity_id = платформа.**
- Часть интеграций настраивается YAML-ом (`template`, `recorder`, `input_*` из YAML, `utility_meter`), часть только через UI.

### 3. Registries: что существует

Лежат в `/config/.storage/` как JSON. Переживают рестарт.

| Файл | Что хранит |
|---|---|
| `core.config_entries` | настроенные интеграции |
| `core.device_registry` | устройства: производитель, модель, прошивка, identifiers, area |
| `core.entity_registry` | связь **`unique_id` ↔ `entity_id`**, имя, area, `disabled_by`, `hidden_by`, labels, оверрайды единиц |
| `core.area_registry`, `core.floor_registry`, `core.label_registry` | зоны, этажи, метки |
| `core.restore_state` | последние состояния сущностей, которые умеют восстанавливаться |

**Ключевая идея:** у сущности есть стабильный `unique_id` (его даёт интеграция, например серийник инвертора + имя регистра), а `entity_id` — просто «человеческий» адрес, который можно переименовать. Registry связывает одно с другим. Сущности без `unique_id` (часть старых YAML-сущностей) в registry не попадают, и в UI их нельзя переименовать или переместить в area.

### 4. Жизненный цикл старта

1. Загружается core config и все registries.
2. Для **каждой** сущности из entity registry, чья интеграция ещё не загрузилась, HA сразу кладёт в state machine заглушку: `state: unavailable` + атрибут **`restored: true`**. Так automations и UI видят, что сущность существует, хотя данных пока нет.
3. Интеграции поднимаются (в несколько стадий, параллельно).
4. Интеграция загрузилась → создаёт свои сущности → реальное состояние заменяет заглушку.
   - Helpers (`input_*`, `timer`, `counter`), trigger-based template sensors, `utility_meter` и т.п. восстанавливают последнее значение из `core.restore_state` (механизм **RestoreEntity**).
   - Интеграции устройств (ZHA, Solarman) выставляют состояние, когда получат данные от устройства. До этого сущность `unavailable` или `unknown`.
5. События `homeassistant_start` → `homeassistant_started`.

Если интеграция не загрузилась вообще (удалена, ошибка, кастомный компонент сломался после обновления), её сущности так и остаются `unavailable` с `restored: true`. Это «сироты».

**Моя ситуация после restore:** ZHA-устройство есть в device registry, сущность есть в entity registry → в UI всё «на месте». Но ZHA ещё не получила от датчика ни одного сообщения (sleepy-устройство, mesh перестраивается) → state остаётся `unavailable`. Уровень проблемы: **device / network**, а не entity и не automation.

### 5. События: структура

```yaml
event_type: state_changed
data:
  entity_id: sensor.inverterdeye_load_power
  old_state: { state: "376", attributes: {...}, last_changed: ..., last_updated: ..., last_reported: ... }
  new_state: { state: "402", ... }
origin: LOCAL          # LOCAL — внутри HA, REMOTE — пришло через API
time_fired: "2026-10-21T20:15:03.123+00:00"
context:
  id: 01J...           # уникальный id цепочки
  parent_id: null      # кто породил (automation/script)
  user_id: null        # какой пользователь (если из UI/приложения)
```

Важные события: `state_changed`, `call_service`, `automation_triggered`, `script_started`, `homeassistant_start` / `homeassistant_started` / `homeassistant_stop`, `zha_event` (кнопки Zigbee), `component_loaded`. Context подробно разберём в Модуле 2.

### 6. Recorder

Подписан на event bus и пишет в SQLite `home-assistant_v2.db`:
- `states` + `states_meta` + `state_attributes` — история состояний (по умолчанию хранится 10 дней, `purge_keep_days`);
- `events` — прочие события;
- `statistics_short_term` (5 мин) и `statistics` (1 час) — агрегаты, **хранятся бессрочно**, но только для сущностей со `state_class`.

Отсюда: история (graph за неделю) и долгосрочная статистика (Energy dashboard за год) — разные таблицы с разными правилами. Модуль 7.

### 7. Frontend и API

Браузер и мобильное приложение — просто клиенты. Они ходят в ядро по **WebSocket API** (`/api/websocket`): подписка на `state_changed`, вызов actions, чтение registries. Есть ещё REST API (`/api/...`). Всё, что умеет UI, можно сделать своим кодом с long-lived token (Модуль 11).

### 8. HAOS: слои на моём Pi 4

```
Raspberry Pi 4
└── Home Assistant OS (минимальный Linux + Docker)
    └── Supervisor (контейнер, управляет остальными)
        ├── homeassistant   ← Core: всё, о чём выше
        ├── add-ons         ← File editor, Samba, Mosquitto, ... (отдельные контейнеры)
        ├── dns, audio, multicast (mDNS → homeassistant.local), observer
```

Типы установки: **OS** (прод) и **Container** (тестовый `test-ha`). В Container нет Supervisor и add-ons, но ядро полностью то же. Все темы Модулей 1–11 работают одинаково.

### 9. Что лежит в `/config`

| Путь | Что |
|---|---|
| `configuration.yaml` | точка входа YAML-конфига |
| `automations.yaml`, `scripts.yaml`, `scenes.yaml` | сюда пишут UI-редакторы |
| `secrets.yaml` | значения для `!secret` |
| `.storage/` | registries, config entries, helpers из UI, дашборды (storage mode) |
| `custom_components/` | кастомные интеграции (HACS) |
| `www/` | статика, доступная как `/local/...` |
| `blueprints/` | blueprints |
| `home-assistant_v2.db` | recorder |
| `home-assistant.log` | лог текущего запуска |

## Примеры на моей системе

```jinja
{# Developer Tools → Template. Сущности-«сироты»: в registry есть, интеграция не поднялась #}
{{ states | selectattr('attributes.restored', 'defined') | map(attribute='entity_id') | list }}

{# Сколько сущностей создала каждая платформа Solarman #}
{% set ids = integration_entities('solarman') %}
{% for d in ids | map('regex_replace', '\\..*$', '') | unique | sort %}
{{ d }}: {{ ids | select('match', d ~ '\\.') | list | count }}
{%- endfor %}
```

Если второй шаблон вернёт пусто, значит домен интеграции называется иначе. Его видно в URL страницы интеграции: Settings → Devices & services → Solarman → `/config/integrations/integration/<домен>`.

## Задания

Всё на проде в режиме **только чтение**.

**1. Entity registry.** Открой `.storage/core.entity_registry` (через File editor или Studio Code Server add-on, или по Samba; **ничего не меняй**). Найди запись `sensor.inverterdeye_pv_power`. Выпиши значения полей `platform`, `unique_id`, `device_id`, `config_entry_id`, `original_name`, `has_entity_name`.
Вопрос: что из этого стабильно, а что можно поменять из UI? Что будет с automation, которая ссылается на entity_id, если переименовать сущность?

**2. Device registry.** По `device_id` из задания 1 найди устройство в `.storage/core.device_registry`. Что там записано про модель и прошивку? Сравни с данными о прошивке, которые ты сам видел на инверторе.

**3. Event bus вживую.** Developer Tools → Events → «Listen to events» → `state_changed` → Start listening на 20–30 секунд. Найди событие для `sensor.inverterdeye_load_power`. Какие из полей `last_changed`, `last_updated`, `last_reported` отличаются в `old_state` и `new_state`? Как часто приходят события по этой сущности и почему (подсказка: `sensor.inverterdeye_update_interval`)?

**4. Context.** Там же слушай `call_service` и переключи любой свет из UI. Что в `context.user_id`? Потом нажми физический выключатель того же света (если есть) и посмотри `state_changed`: есть ли `user_id` теперь?

**5. Сироты.** Выполни первый шаблон из «Примеров». Есть ли у тебя сущности с `restored: true` прямо сейчас? Если есть — от какой они интеграции и почему она не поднялась (Settings → System → Logs, Settings → Repairs)?

**6. Объясни своими словами** (3–5 предложений): почему после restore из backup ZHA-датчик был виден в Devices, но `unavailable`, и почему в такой ситуации бессмысленно сразу чинить automation или удалять устройство.

**Бонус:** Settings → System → Repairs → ⋮ → System information. Выпиши Installation Type, версию Core, размер/движок recorder (если показан). Это пригодится, чтобы закрепить ту же версию в `test-ha`.

## Мои ответы

<!-- ответы сюда или в чат; разбор добавится ниже -->

## Разбор

## Запомнить

- Ядро = state machine (в памяти) + event bus + action registry.
- Registry (`.storage/`) — что **существует**. State machine — что **сейчас**. Существует ≠ доступно.
- `unique_id` стабилен, `entity_id` — переименовываемый адрес.
- `unavailable` + `restored: true` = интеграция сущности ещё (или вообще) не загрузилась.
- Домен в entity_id = платформа; одна интеграция создаёт сущности многих платформ.
- История (дни) и статистика (бессрочно, нужен `state_class`) — разные вещи.
- `.storage/` на проде — только читать, и никогда при работающем HA не править.

## Ссылки

- Архитектура ядра: https://developers.home-assistant.io/docs/architecture/core
- Entity registry: https://developers.home-assistant.io/docs/entity_registry_index
- Events: https://www.home-assistant.io/docs/configuration/events/
- Типы установки: https://www.home-assistant.io/installation/
