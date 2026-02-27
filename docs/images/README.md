# Иллюстрации для статьи (ARTICLE_RU.md)

Папки:

| Папка | Назначение |
|-------|------------|
| `diagrams/` | Схемы и блок-схемы (часть можно сгенерировать автоматически) |
| `screenshots/` | Скриншоты Home Assistant (мастер automation, уведомления, дашборд, KPI) |
| `photos/` | Фото реальной установки ПВУ / окружения |

## Сгенерированные диаграммы (уже в репо)

- **Схема трёх контуров** — `diagrams/three-contours.png`
- **Блок-схема одного цикла** — `diagrams/cycle-flowchart.png`
- **Сценарий away** — `diagrams/away-scenario.png`

## Что нужно добавить вручную

- Скриншот диагностического уведомления (persistent_notification) → `screenshots/debug-notification.png`
- Скриншот мастера настройки automation (секции blueprint) → `screenshots/automation-wizard.png`
- Скриншот дашборда или KPI-карточки → `screenshots/dashboard.png` или `screenshots/kpi.png`
- График «антизаморозка: температура за окном vs режим» (если нужен) — лучше из HA или другого инструмента
- **Фото установки** — в `photos/` (имя на ваше усмотрение)

В статье вставлены пометки `[Здесь будет: ...]` — при добавлении файлов замените их на относительные ссылки на изображения, например:  
`![Описание](images/diagrams/three-contours.png)`.
