# ha-learning

Учебный репозиторий: углублённое изучение Home Assistant до уровня продвинутого пользователя.
Практика — на реальной системе (только чтение) и на тестовом инстансе `test-ha/` (можно ломать).

## Структура

```
ha-learning/
├── README.md          ← этот файл
├── PROGRESS.md        ← статус модулей, текущий урок, слабые места, открытые вопросы
├── CHEATSHEET.md      ← сквозная шпаргалка: самое полезное из всех модулей
├── notes/             ← один файл на модуль: теория, примеры, задания, мои ответы, «запомнить»
│   ├── _template.md
│   └── 00-architecture.md
├── snippets/          ← проверенные куски для переиспользования
│   ├── templates/     ← Jinja (*.jinja / *.md с примерами)
│   └── automations/   ← YAML automations/scripts
└── test-ha/           ← тестовый Home Assistant (Docker) для экспериментов
    ├── docker-compose.yml
    ├── README.md
    └── config/
```

## План (модули)

| # | Модуль | Файл |
|---|--------|------|
| 0 | Как HA устроен внутри | `notes/00-architecture.md` |
| 1 | Сущности и состояния | `notes/01-entities-states.md` |
| 2 | Event bus, actions, context | `notes/02-events-actions-context.md` |
| 3 | Templates (Jinja2) | `notes/03-templates.md` |
| 4 | Helpers и производные сущности | `notes/04-helpers.md` |
| 5 | Automations в глубину | `notes/05-automations.md` |
| 6 | Scripts, scenes, blueprints | `notes/06-scripts-scenes-blueprints.md` |
| 7 | Recorder, история, статистика | `notes/07-recorder-statistics.md` |
| 8 | Конфигурация как код (packages, includes, secrets) | `notes/08-config-as-code.md` |
| 9 | Интеграции изнутри: ZHA, MQTT, REST | `notes/09-integrations.md` |
| 10 | Dashboards (Lovelace) | `notes/10-dashboards.md` |
| 11 | API и расширение (REST/WS, AppDaemon, custom integration) | `notes/11-api-extending.md` |
| 12 | Эксплуатация и надёжность | `notes/12-operations.md` |

Файлы модулей создаются по мере прохождения на основе `notes/_template.md`.

## Как проходит одна сессия

1. Новый чат в проекте claude.ai → «продолжаем обучение».
2. Claude читает `PROGRESS.md` и нужный конспект, продолжает с текущего урока.
3. Теория → примеры на моих сущностях → задания.
4. Я делаю задания (на проде — только чтение, всё остальное — в `test-ha`) и присылаю ответы.
5. Разбор → обновление конспекта, `PROGRESS.md`, при необходимости `CHEATSHEET.md` → commit.

Одна сессия = один урок. Знания копятся в файлах, а не в истории чата.

## Правила

- На реальном HA: Developer Tools → Template / States / Events — можно; создавать и менять automations, helpers — только в `test-ha`, пока не разобрались.
- Файлы в `.storage/` на проде только читать. Никогда не редактировать при работающем HA.
- В репозиторий не коммитим секреты, токены, базу, `.storage/` тестового инстанса.
