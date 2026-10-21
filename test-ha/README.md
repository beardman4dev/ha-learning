# HA Lab (test-ha)

Тестовый Home Assistant Container на Orange Pi 5 Pro. Нужен с Модуля 4 (helpers) и обязателен для Модулей 5–8 (automations, scripts, config as code). Модули 0–3 можно проходить на проде в режиме «только чтение».

## Запуск

```bash
# на Orange Pi 5 Pro
git clone <URL_РЕПО> ha-learning
cd ha-learning/test-ha
docker compose up -d
docker compose logs -f ha-lab   # дождаться "Home Assistant initialized"
```

Открыть `http://<IP_ORANGE_PI>:8124` → пройти onboarding (создать локального пользователя, часовой пояс Europe/Kyiv).

## Чем отличается от прода

| | Прод (Pi 4) | Lab (Orange Pi) |
|---|---|---|
| Тип установки | Home Assistant OS | Home Assistant Container |
| Supervisor, add-ons | есть | нет (для обучения не нужны) |
| Ядро, шаблоны, automations, helpers | — | то же самое |
| Реальные устройства | есть | нет — вместо них `packages/lab_basics.yaml` |

Позже (Модуль 5) подключим живые данные DEYE с прода в режиме только чтения, чтобы тренировать automations на настоящих `sensor.inverterdeye_*` без риска управлять реальным железом.

## Полезное

```bash
docker compose restart ha-lab       # рестарт ядра
docker compose pull && docker compose up -d   # обновление
docker exec -it ha-lab bash         # шелл внутри контейнера
```

Проверка конфига без рестарта: UI → Developer Tools → YAML → Check configuration.

## Что в git, а что нет

В git: `configuration.yaml`, `packages/`, `automations.yaml`, `scripts.yaml`, `scenes.yaml`.
Не в git (см. `.gitignore`): `.storage/`, база, логи, `secrets.yaml`. То, что создано в UI как helper, живёт в `.storage/` и в репозиторий не попадёт — поэтому всё важное пишем в `packages/`.
