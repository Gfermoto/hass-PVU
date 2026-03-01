---
name: Bug report
about: Сообщить об ошибке в blueprint
title: "[bug] "
labels: bug
assignees: ''
---

## Что произошло

Кратко опишите проблему.

## Ожидаемое поведение

Что должно было произойти.

## Как воспроизвести

1.
2.
3.

## Окружение

- Home Assistant Core:
- Home Assistant OS/Supervisor:
- Blueprint: `pvu.yaml` / `pvu_min.yaml` (нужное оставить)
- Версия blueprint (из CHANGELOG): `v0.x.x`
- `debug_mode` включён: да / нет

## Trace / логи

Вставьте трейс автоматизации (скрин или текст) и ключевые состояния датчиков.

## Параметры blueprint

Укажите значения важных параметров:

- `away_behavior`:
- `outdoor_vent_min`:
- `outdoor_temp_policy`:
- `fan_mode_low/medium/high`:
- `eco_mode_enabled` (если `pvu.yaml`):
- `away_off_hvac_policy` / `away_off_fan_policy` (если `pvu.yaml`):
