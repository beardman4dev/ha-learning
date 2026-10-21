# Шпаргалка Home Assistant

Пополняется после каждого модуля. Только то, что реально пригождается.

---

## М0 · Архитектура

**Три сущности ядра:** event bus (что произошло) · state machine (что сейчас, в памяти) · action registry (что можно сделать).

**Registry ≠ state.** Registry (`.storage/`) — что *существует*, переживает рестарт. State machine — что *сейчас*, живёт в памяти и заполняется интеграциями после старта.

**Где что лежит в `/config`:**

| Путь | Что |
|------|-----|
| `configuration.yaml` | YAML-конфиг, точка входа |
| `automations.yaml` / `scripts.yaml` / `scenes.yaml` | то, что создано в UI-редакторах |
| `secrets.yaml` | значения для `!secret` |
| `.storage/core.config_entries` | интеграции, настроенные через UI |
| `.storage/core.entity_registry` | `unique_id` ↔ `entity_id`, имена, area, disabled/hidden |
| `.storage/core.device_registry` | устройства |
| `.storage/core.restore_state` | последние состояния для RestoreEntity (helpers и др.) |
| `.storage/lovelace*` | дашборды в storage mode |
| `home-assistant_v2.db` | recorder (история + статистика) |
| `home-assistant.log` | лог |
| `custom_components/` | кастомные интеграции (HACS) |

**Ключевые события:** `state_changed`, `call_service`, `automation_triggered`, `script_started`, `homeassistant_start`, `homeassistant_started`, `homeassistant_stop`.

**`restored: true` + `unavailable`** = сущность есть в registry, но её интеграция ещё не загрузилась (или не загрузится вообще).

```jinja
{# Сущности-«сироты»: в registry есть, интеграция не поднялась #}
{{ states | selectattr('attributes.restored', 'defined') | map(attribute='entity_id') | list }}
```

**Порядок отладки:** hardware → power → network → integration → device → entity → state → automation → template → UI.
