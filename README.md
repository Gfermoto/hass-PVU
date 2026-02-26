# hass-PVU
## Система автоматизации приточно-вытяжной вентиляции (ПВУ)

### Описание
Данный проект представляет собой готовое решение для автоматизации приточно-вытяжной вентиляции на базе Home Assistant с использованием оборудования Turkov. Система обеспечивает комфортный микроклимат в помещении, оптимизируя энергопотребление и качество воздуха.

### Changelog

История изменений перенесена в отдельный файл: [`CHANGELOG.md`](./CHANGELOG.md).

### Варианты blueprint

- `pvu_min.yaml` — стабильная базовая версия (минимум изменений, приоритет совместимости).
- `pvu.yaml` — основная развиваемая версия (новые улучшения и UX-доработки).

### Компоненты системы
1. **Контроллер**: Home Assistant
   - Версия: 2023.12.0 или новее
   - Рекомендуется установка на выделенное устройство

2. **Вентиляционная установка**: 
   - Производитель: Turkov
   - Модель: с рекуперацией тепла и влаги (3 ступени рекуперации)
   - Рабочий диапазон температур: до -35°C
   - Подключение: **ModBus RTU по IP** и/или **MQTT** (Wi‑Fi модуль, прошивка 2.0.0+)

   **Референс конфигурации для текущего blueprint**:
   - Установка: **Zenit Heco V 350 1,5E (EPP)**.
   - Дополнительно: внешний охладитель + электрический нагреватель.
   - Рекомендуемые режимы HVAC в автоматизации: `off`, `fan_only`, `heat`, `cool` (без `dry`).

3. **Интеграция Turkov** (один из вариантов):
   - **[hass-turkov](https://github.com/alryaz/hass-turkov)** — по IP (ModBus RTU), climate-сущность в HA
   - **MQTT** — топики вида `turkov/<серийный_номер>/...`, см. раздел «Подключение Turkov по MQTT» ниже

4. **Датчик качества воздуха**: [AirGradient](https://www.airgradient.com/)
   - Измеряемые параметры:
     * CO₂ (0-40000 ppm)
     * Температура (-10 до +60°C)
     * Относительная влажность (0-100%)
     * PM2.5, VOC Index, NOx Index (опционально, в любой комбинации)
     * Для расширенной логики можно использовать как внутренние, так и наружные сенсоры
   - Возможности:
     * WiFi подключение
     * MQTT интеграция
     * Открытый исходный код
     * DIY сборка или готовое решение

### Быстрый старт (3 шага)

1. Установите интеграцию Turkov и получите рабочую `climate`-сущность (через `hass-turkov` или `climate.mqtt`).
2. Убедитесь, что в HA есть два обязательных датчика:
   - температура в помещении (`sensor` с `device_class: temperature`);
   - влажность в помещении (`sensor` с `device_class: humidity`).
3. Импортируйте `pvu.yaml`, выберите `climate` + 2 обязательных датчика, сохраните автоматизацию.

Минимальная рабочая конфигурация уже на этом этапе начнет управлять HVAC и вентилятором. Остальные датчики можно добавлять постепенно.

### Функциональные возможности (актуально для `pvu.yaml`)

#### 1. Управление температурой и режимами HVAC
- Целевая температура выбирается по сценарию:
  - `temp_day` днем;
  - `temp_night` ночью;
  - `temp_away` при отсутствии (если presence определен как away).
- Используется гистерезис:
  - `temp_hysteresis_on` для входа в `heat/cool`;
  - `temp_hysteresis_off` для зоны возврата.
- Команды применяются в два шага:
  1) `climate.set_temperature` (установка уставки, если изменилась);
  2) `climate.set_hvac_mode` (если целевой режим отличается от текущего).
- Для HVAC добавлен anti-flap контроль: `hvac_min_switch_minutes` задаёт минимальный интервал между переключениями режимов.

#### 2. Антизаморозка и ограничения по наружной температуре
- Охлаждение разрешается только если `temp_outdoor > outdoor_cool_min`.
- HVAC-вентиляция (`fan_only`) разрешается только если `temp_outdoor > outdoor_vent_min`.
- Если система в "зоне нормы", но `fan_only` запрещен по холоду снаружи, выбирается `heat` как безопасный fallback.
- Новая настройка `outdoor_temp_policy`:
  - `conservative` — если датчик улицы недоступен, `cooling/fan_only` не разрешаются;
  - `permissive` — при недоступном датчике улицы ограничения по `temp_outdoor` не применяются.

#### 3. Управление скоростью вентилятора
- Логика 3 ступеней: `low` -> `medium` -> `high`.
- Источники разгона: влажность и/или загрязнение воздуха внутри (CO2/PM2.5/VOC/NOx).
- Наружные датчики качества воздуха ограничивают агрессивный разгон (при плохом наружном воздухе максимум `medium`).
- В develop-ветке доступен плавный спад скорости (`fan_stepdown_enabled`): при очистке воздуха переход идёт ступенчато `high -> medium -> low`.

#### 4. Профили климата и жилья (Этап 3)
- `climate_profile`:
  - `continental` — базовый сбалансированный профиль;
  - `mild` — мягче пороги anti-freeze/охлаждения и чуть более ранняя реакция;
  - `cold` — более консервативные anti-freeze/охлаждение и повышенный гистерезис.
- `home_profile`:
  - `house` — базовый профиль по умолчанию (менее агрессивная реакция на IAQ, пороги выше);
  - `apartment` — более чувствительная реакция IAQ относительно `house`;
  - `office` — более строгая реакция на IAQ (пороги ниже).
- Профили влияют на эффективные пороги и отражаются в `debug_mode`.
- Дефолтный стартовый набор (под загородный дом, климат МО):
  - `climate_profile=continental`, `home_profile=house`;
  - `temp_hysteresis_on=1.2`, `temp_hysteresis_off=0.6`;
  - `outdoor_cool_min=20`, `outdoor_vent_min=3`.

#### 5. Presence и режим отсутствия
- Поддерживаются источники: `binary_sensor`, `person/group`, `alarm_control_panel`, `custom`.
- Важное поведение:
  - если пользователь выбрал `presence_mode: alarm`, но указал не alarm-сущность (например `group.*`), применяется безопасный fallback по `not_home/off`.
- Параметр `away_behavior`:
  - `maintain`: поддерживать уставку `temp_away`;
  - `off`: выключать установку.

#### 6. Fail-safe логика применения команд
- Перед вызовами `climate.set_hvac_mode` / `climate.set_fan_mode` проверяется:
  - целевой режим не пустой;
  - целевой режим отличается от текущего;
  - целевой режим присутствует в `hvac_modes` / `fan_modes` устройства.
- Для HVAC дополнительно проверяется минимальный интервал переключений (`hvac_min_switch_minutes`), чтобы избежать частых дерганий режима.
- Это защищает от ошибок на устройствах с неполным набором режимов.
- Дополнительно проверяется доступность `climate`-сущности (`unavailable/unknown` не допускаются к сервис-вызовам).

#### 7. Диагностика (develop-ветка)
- Параметр `debug_mode` включает диагностические `persistent_notification`.
- Уведомления показывают:
  - текущие/целевые режимы;
  - причины, почему режим не применён (`desired_empty`, `already_set`, `unsupported_mode`, `min_interval_hold`, `climate_unavailable`);
  - активные профили `climate_profile` и `home_profile`;
  - причины ограничений (например, неизвестная `temp_outdoor` при `conservative`);
  - расчётные флаги (`cool_allowed`, `vent_allowed`, `air_boost_count`).

### Установка и настройка

1. **Установка Home Assistant**:
   ```bash
   # Следуйте официальной инструкции на
   # https://www.home-assistant.io/installation/
   ```

2. **Установка интеграции Turkov** (по IP):
   - Добавьте репозиторий в HACS
   - Установите интеграцию
   - Перезагрузите Home Assistant

   **Подключение Turkov по MQTT** (альтернатива или дополнение к IP):
   - Документация: [Turkov — подключение по MQTT](https://turkov.ru/blog/articles/podklyuchenie_oborudovaniya_k_sistemam_umnyy_dom_po_protokolu_mqtt/)
   - Нужен Wi‑Fi модуль с прошивкой **2.0.0** и выше.
   - Базовый путь топиков: **`turkov/<серийный_номер>/...`** (серийный номер вида `X-XXXXXX-XX`, указан на устройстве и в веб-интерфейсе `http://turkov-X-XXXXXX-XX.local/`).

   **Топики состояний** (устройство публикует):
   | Топик | Описание |
   |------|----------|
   | `turkov/<serial>/status` | Подключение: `online` / `offline` |
   | `turkov/<serial>/state/on` | Вкл/выкл |
   | `turkov/<serial>/state/fan_speed` | Скорость вентиляторов |
   | `turkov/<serial>/state/fan_mode` | Режим вентиляторов |
   | `turkov/<serial>/state/temp_sp` | Уставка температуры |
   | `turkov/<serial>/state/indoor_temperature` | Температура в помещении |
   | `turkov/<serial>/state/outdoor_temperature` | Температура с улицы |
   | `turkov/<serial>/state/supply_temperature` | Температура притока |
   | `turkov/<serial>/state/indoor_humidity` | Влажность в помещении |
   | `turkov/<serial>/state/filter` | Загрязнение фильтров |

   **Топики команд** (HA публикует для управления):
   - `turkov/<serial>/command/...` — подтопики обычно совпадают с именами из state (например `on`, `fan_speed`, `fan_mode`, `temp_sp`). Точный формат значений смотрите в документации Turkov или в исходящих сообщениях при управлении с веб-интерфейса/приложения.

   **Встроенный CO2-датчик Turkov (пример для вашей установки):**
   - Для чтения встроенного датчика CO2 нужен настроенный MQTT-брокер (Mosquitto/EMQX и т.п.), к которому подключен Wi-Fi модуль установки.
   - Для установки с примерным серийным номером `2-400187-01` используйте топик:
     - `turkov/2-400187-01/state/CO2_level`
   - Если у вас разнесенная конфигурация через `!include`, это нормально: ниже пример именно блока сенсора, который можно положить в отдельный файл и подключить через include.
   - Минимальный рабочий пример сенсора:
   ```yaml
   mqtt:
     sensor:
       - name: "Turkov_CO2"
         state_topic: "turkov/2-400187-01/state/CO2_level"
         device_class: carbon_dioxide
         state_class: measurement
         unit_of_measurement: "ppm"
   ```

   После настройки MQTT в HA можно завести сенсоры по state-топикам и при необходимости управление через сервис `mqtt.publish` в топики `command/...`. Blueprint `pvu.yaml` рассчитан на climate-сущность (из hass-turkov); при использовании только MQTT нужно либо поднять climate через [MQTT Climate](https://www.home-assistant.io/integrations/climate.mqtt/) с этими топиками, либо доработать автоматизацию под прямые вызовы `mqtt.publish`.

3. **Настройка AirGradient**:
   - Подключите датчик к сети WiFi
   - Настройте MQTT брокер
   - Добавьте сенсоры в Home Assistant

4. **Импорт Blueprint**:
   - Скопируйте выбранный файл (`pvu_min.yaml` или `pvu.yaml`) в папку `blueprints/automation/`
   - Импортируйте через интерфейс Home Assistant

   **Требования к датчикам в `pvu.yaml`**:
   - **Обязательные**: температура в помещении и влажность в помещении.
   - **Опциональные**: все остальные (внутренние и наружные CO₂ / PM2.5 / VOC / NOx, наружная температура, presence).
   - Система работает с любой комбинацией опциональных сенсоров.

   **Поведение blueprint** (`pvu.yaml`):
   - Приоритет выше у **внутренних** датчиков качества воздуха.
   - **Наружные** датчики используются как ограничитель разгона.
   - Для температуры и качества воздуха используется гистерезис (включение/возврат по разным порогам).
   - Есть антизаморозочная логика для HVAC-вентиляции через `outdoor_vent_min`.
   - Отдельным шагом применяется уставка `climate.set_temperature` для режима поддержания.
   - Реакция на изменение датчиков (включая опциональные) и presence — по их изменению, плюс периодический пересчет каждые 2 минуты.

   **Edge-cases и безопасное поведение**:
   - при `climate: unavailable` сервисные вызовы не отправляются;
   - при `unknown/unavailable` у датчиков используется fail-safe поведение без падения логики;
   - нереалистичные выбросы датчиков фильтруются и не участвуют в принятии решений;
   - если рассчитанный режим не поддерживается устройством, автоматика сохраняет текущий режим.

   **Важно про VOC/NOx в AirGradient**:
   - Это индексы (обычно шкала 1–500), а не абсолютные концентрации газов.
   - NOx Index часто близок к 1 в «чистом» фоне; рост выше 1 означает ухудшение относительно базовой линии.
   - Значения NOx 2–3 обычно не считаются тревожными; для автоматических триггеров практичный стартовый порог — около 20.
   - В проекте пороги можно адаптировать под ваш реальный профиль помещения.

### Конфигурация

1. **Основные параметры** (`configuration.yaml`):
   ```yaml
   # Пример базовой конфигурации (Turkov по IP через hass-turkov)
   climate:
     - platform: turkov
       name: "PVU"
       modbus_host: 192.168.1.x
       modbus_port: 502

   sensor:
     - platform: mqtt
       name: "CO2 Level"
       state_topic: "airgradient/co2"
       unit_of_measurement: "ppm"
     - platform: mqtt
       name: "Humidity"
       state_topic: "airgradient/humidity"
       unit_of_measurement: "%"
     - platform: mqtt
       name: "PM2.5"
       state_topic: "airgradient/pm25"
       unit_of_measurement: "µg/m³"

   notify:
     - platform: telegram
       name: "Telegram Notifications"
       chat_id: "YOUR_CHAT_ID"
       default_message: "Уведомление от системы вентиляции"
   ```

   **Пример сенсоров Turkov по MQTT** (подставьте свой серийный номер вместо `X-XXXXXX-XX`):
   ```yaml
   mqtt:
     sensor:
       - name: "Turkov температура в помещении"
         state_topic: "turkov/X-XXXXXX-XX/state/indoor_temperature"
         unit_of_measurement: "°C"
         device_class: temperature
       - name: "Turkov влажность в помещении"
         state_topic: "turkov/X-XXXXXX-XX/state/indoor_humidity"
         unit_of_measurement: "%"
         device_class: humidity
       - name: "Turkov температура с улицы"
         state_topic: "turkov/X-XXXXXX-XX/state/outdoor_temperature"
         unit_of_measurement: "°C"
         device_class: temperature
       - name: "Turkov статус"
         state_topic: "turkov/X-XXXXXX-XX/status"
   ```

2. **Автоматизации** (`automations.yaml`):
   ```yaml
   - alias: "PVU Turkov Auto"
     use_blueprint:
       path: Gfermoto/pvu.yaml
       input:
         turkov_climate: climate.pvu
         temp_indoor: sensor.indoor_temperature
         humidity_indoor: sensor.indoor_humidity
         temp_outdoor: sensor.outdoor_temperature
         co2_sensor: sensor.indoor_co2
         presence_home: person.user
   ```

### Требования

1. **Аппаратные**:
   - Home Assistant с процессором ARM/x86
   - 2GB RAM минимум
   - Стабильное подключение к сети
   
2. **Программные**:
   - Home Assistant OS/Core
   - MQTT брокер
   - HACS

### Разработка и качество

- Руководство для контрибьюторов: [`CONTRIBUTING.md`](./CONTRIBUTING.md)
- Релизный чеклист: [`docs/RELEASE_CHECKLIST.md`](./docs/RELEASE_CHECKLIST.md)
- Дорожная карта: [`docs/ROADMAP.md`](./docs/ROADMAP.md)
- Калибровка порогов по реальным данным: [`docs/TUNING.md`](./docs/TUNING.md)
- Шаблон постоянного Lovelace-дашборда: [`airflow_dashboard.yaml`](./airflow_dashboard.yaml)
- Облегченный постоянный дашборд: [`airflow_dashboard_min.yaml`](./airflow_dashboard_min.yaml)
- Шаблон для режима `Manual card` (полный): [`airflow_card.yaml`](./airflow_card.yaml)
- Шаблон для режима `Manual card` (облегченный): [`airflow_card_min.yaml`](./airflow_card_min.yaml)
- KPI-пакет для верификации порогов на реальных данных: [`airflow_kpi_package.yaml`](./airflow_kpi_package.yaml)
- FAQ: [`docs/FAQ.md`](./docs/FAQ.md)
- Шаблоны issues: [`.github/ISSUE_TEMPLATE`](./.github/ISSUE_TEMPLATE)
- Базовая CI-валидация (Markdown/YAML): [`.github/workflows/validate.yml`](./.github/workflows/validate.yml)
- Материал для статьи: [`docs/ARTICLE_RU.md`](./docs/ARTICLE_RU.md)
- Короткий анонс для Telegram: [`docs/TG_POST_RU.md`](./docs/TG_POST_RU.md)

### Импорт Lovelace без ошибок

- Для **целого дашборда** используйте файлы `airflow_dashboard*.yaml` и вставляйте их через **Панели управления -> Редактировать в YAML**.
- Для **ручной карточки** используйте файлы `airflow_dashboard*_card.yaml` и вставляйте их в **Manual card**.
- Ошибка `No card type configured` обычно означает, что YAML целого дашборда вставлен в редактор одной карточки.

### Быстрый запуск KPI-верификации

- Подключите `airflow_kpi_package.yaml` как package в Home Assistant.
- Замените `entity_id` в пакете под свои сенсоры и climate-сущность.
- После перезапуска используйте новые сущности:
  - `sensor.airflow_*_ok_percent_24h` — доля времени в целевой зоне;
  - `sensor.airflow_boost_fan_percent_24h` — доля времени fan в `medium/high`.

### Поддержка и обратная связь
- GitHub Issues: [создать обращение](https://github.com/Gfermoto/hass-PVU/issues)
- Telegram каналы:
  * [DIYIoT_Lab](https://t.me/DIYIoT_Lab) - Новости и обновления проекта
  * [DIYIoT_Zone](https://t.me/DIYIoT_Zone) - Сообщество разработчиков и энтузиастов

### Лицензия
[MIT License](./LICENSE)
