# Алгоритм работы логики pvu.yaml

![Алгоритм работы ПВУ](./images/diagrams/logic-flowchart.png)

Документ описывает **все ветки принятия решений** в blueprint `pvu.yaml`.
Для минимального варианта (`pvu_min.yaml`) логика идентична, за исключением отсутствия
ECO-режима, профилей климата/жилья, anti-flap и debug.

---

## 1. Общий цикл

```mermaid
flowchart TD
    TRIGGER(["⏱ Триггер\nтаймер каждые 2 мин\nили изменение датчика"])
    TRIGGER --> VARS["🧮 Вычисление переменных\nвремя / присутствие / t_in / t_out\nIAQ / humidity / ECO флаги"]
    VARS --> HVAC_DECIDE["🔀 desired_hvac_mode\n(см. схему 2)"]
    HVAC_DECIDE --> FAN_DECIDE["🌀 desired_fan_mode\n(см. схему 3)"]
    FAN_DECIDE --> STEP0{"📐 Шаг 0\nclimate_available\nAND desired ≠ off\nAND |current_target − target| > 0.2?"}
    STEP0 -->|Да| SET_TEMP["climate.set_temperature\n= target_temp"]
    STEP0 -->|Нет| STEP1
    SET_TEMP --> STEP1
    STEP1{"⚙️ Шаг 1\nhvac_apply_reason\n= apply?"}
    STEP1 -->|Да| SET_HVAC["climate.set_hvac_mode\n= desired_hvac_mode"]
    STEP1 -->|Нет| STEP2
    SET_HVAC --> POST_HVAC{"debug_mode AND\nverify_delay > 0?"}
    POST_HVAC -->|Да| VERIFY_HVAC["⏳ ждать N сек\nпроверить state\n≠ desired → уведомление"]
    POST_HVAC -->|Нет| STEP2
    VERIFY_HVAC --> STEP2
    STEP2{"⚙️ Шаг 2\nfan_apply_reason\n= apply?"}
    STEP2 -->|Да| SET_FAN["climate.set_fan_mode\n= desired_fan_mode"]
    STEP2 -->|Нет| STEP3
    SET_FAN --> POST_FAN{"debug_mode AND\nverify_delay > 0?"}
    POST_FAN -->|Да| VERIFY_FAN["⏳ ждать N сек\nпроверить fan_mode\n≠ desired → уведомление"]
    POST_FAN -->|Нет| STEP3
    VERIFY_FAN --> STEP3
    STEP3{"🪵 Шаг 3\ndebug_need_notify?"}
    STEP3 -->|Да| NOTIFY["persistent_notification\ndebug_message"]
    STEP3 -->|Нет| END
    NOTIFY --> END(["✅ Конец цикла"])
```

---

## 2. Выбор режима HVAC (`desired_hvac_mode`)

```mermaid
flowchart TD
    START([" "]) --> A{"is_away AND\naway_behavior = off?"}

    A -->|Да| OFF(["desired = **off**\n↓ установка выключена"])
    A -->|Нет| B{"eco_active?\n(только pvu.yaml)"}

    B -->|Да| OFF
    B -->|Нет| C{"vent_allowed?\noutdoor_temp > outdoor_vent_min_eff"}

    C -->|Нет — мороз| HEAT(["desired = **heat**\nподогрев приточного воздуха\n+ рекуперация"])
    C -->|Да| D{"cool_allowed?\noutdoor_temp > outdoor_cool_min_eff\nAND temp_in_valid\nAND indoor > target + hyst_on_eff?"}

    D -->|Да — жарко| COOL(["desired = **cool**\nохлаждение приточного воздуха"])
    D -->|Нет| FANONLY(["desired = **fan_only**\nвентиляция с рекуперацией\n← режим по умолчанию"])

    style OFF fill:#f28b82
    style HEAT fill:#fdd663
    style COOL fill:#a8d8ea
    style FANONLY fill:#b5ead7
```

### Примечания к ECO-блоку

```mermaid
flowchart TD
    ECO_EN{"eco_mode_enabled = true?"} -->|Нет| INACTIVE(["eco_active = false"])
    ECO_EN -->|Да| SENSORS{"eco_has_sensors?\n(хотя бы один датчик воздуха задан)"}
    SENSORS -->|Нет — нет датчиков| INACTIVE
    SENSORS -->|Да| ALL_OK{"eco_all_ok?\nВСЕ активные датчики\nниже своих ECO-порогов"}
    ALL_OK -->|Нет — воздух грязный| INACTIVE
    ALL_OK -->|Да — воздух чист| ACTIVE(["eco_active = true\n→ desired_hvac = off"])

    style ACTIVE fill:#b5ead7
    style INACTIVE fill:#e0e0e0
```

**ECO-пороги** (участвуют только если соответствующий датчик задан):

| Датчик | Порог |
|---|---|
| CO₂ | `eco_co2_max` (по умолч. 700 ppm) |
| PM2.5 | `eco_pm25_max` (по умолч. 12 µg/m³) |
| VOC | `eco_voc_max` (по умолч. 200) |
| NOx | `eco_no_max` (по умолч. 8) |
| Влажность | `eco_humidity_max` (по умолч. 60 %) |

---

## 3. Выбор скорости вентилятора (`desired_fan_mode`)

```mermaid
flowchart TD
    START([" "]) --> A{"is_away AND\naway_behavior = off?"}

    A -->|Да, follow_fan_mode| F_OFF(["desired_fan = **off**"])
    A -->|Да, hvac_only| F_SKIP(["desired_fan = '' \nне менять\n(защита от coupling)"])
    A -->|Нет| B{"outdoor_air_bad?\n(наружный CO₂/PM2.5/VOC/NOx\nвыше порогов блокировки)"}

    B -->|Да + boost_count ≥ 1| F_MED1(["desired_fan = **medium**\nограничение по улице"])
    B -->|Да, boost = 0| F_LOW1(["desired_fan = **low**\nплохой воздух снаружи\nнет смысла разгонять"])
    B -->|Нет| C{"air_boost_count ≥ 2?\n(несколько загрязнителей\nвыше порогов)"}

    C -->|Да| F_HIGH(["desired_fan = **high**\nинтенсивная вентиляция"])
    C -->|Нет| D{"air_boost_count = 1?\n(один загрязнитель\nвыше порога)"}

    D -->|Да| F_MED2(["desired_fan = **medium**\nумеренная вентиляция"])
    D -->|Нет| E{"all_air_clear AND humidity_clear?\n(все чисто)"}

    E -->|Нет — что-то в переходной зоне| F_HOLD(["desired_fan = ''\nне менять текущую скорость"])
    E -->|Да — всё чисто| F{"fan_stepdown_enabled\nAND current_fan = high?"}

    F -->|Да| F_DOWN1(["desired_fan = **medium**\nплавный спад: high → medium"])
    F -->|Нет| G{"fan_stepdown_enabled\nAND current_fan = medium?"}

    G -->|Да| F_DOWN2(["desired_fan = **low**\nплавный спад: medium → low"])
    G -->|Нет| F_LOW2(["desired_fan = **low**\nбазовая скорость"])

    style F_OFF fill:#f28b82
    style F_HIGH fill:#fdd663
    style F_MED1 fill:#fdcfa4
    style F_MED2 fill:#fdcfa4
    style F_LOW1 fill:#b5ead7
    style F_LOW2 fill:#b5ead7
    style F_DOWN1 fill:#b5ead7
    style F_DOWN2 fill:#b5ead7
    style F_SKIP fill:#e0e0e0
    style F_HOLD fill:#e0e0e0
```

**`air_boost_count`** = сумма активных триггеров: CO₂ > порог + PM2.5 > порог + VOC > порог + NOx > порог + влажность > порога.

---

## 4. Принятие решения о применении (`hvac_apply_reason`)

```mermaid
flowchart TD
    START([" "]) --> A{"climate_available?\nstate не unavailable/unknown"}
    A -->|Нет| R_UNAVAIL(["❌ climate_unavailable\nкоманды не отправляются"])
    A -->|Да| B{"desired_hvac_mode\nне пустой?"}
    B -->|Нет — пустая строка| R_EMPTY(["⏭ desired_empty\nне менять режим"])
    B -->|Да| C{"desired == current\n(уже в нужном режиме)?"}
    C -->|Да| R_ALREADY(["⏭ already_set\nкоманда не нужна"])
    C -->|Нет| D{"hvac_switch_allowed?\nпрошло ≥ hvac_min_switch_minutes\nИЛИ skip_for_off = true?"}
    D -->|Нет — anti-flap| R_HOLD(["⏳ min_interval_hold\nждём окончания интервала"])
    D -->|Да| E{"desired в списке\nhvac_modes устройства?"}
    E -->|Нет| R_UNSUPPORTED(["⚠️ unsupported_mode\nустройство не поддерживает"])
    E -->|Да| R_APPLY(["✅ apply\nотправляем команду"])

    style R_UNAVAIL fill:#f28b82
    style R_UNSUPPORTED fill:#fdd663
    style R_HOLD fill:#fdcfa4
    style R_EMPTY fill:#e0e0e0
    style R_ALREADY fill:#e0e0e0
    style R_APPLY fill:#b5ead7
```

Аналогичная логика для `fan_apply_reason` (без anti-flap интервала).

---

## 5. Матрица режимов: условие → результат

### HVAC (итоговая таблица)

| Условие (приоритет сверху вниз) | `desired_hvac` | Физика |
|---|---|---|
| `is_away AND away=off` | `off` | Установка выключена |
| `eco_active` (только pvu.yaml) | `off` | Воздух чист, экономия |
| `outdoor < outdoor_vent_min` | `heat` | Подогрев приточного воздуха |
| `outdoor > outdoor_cool_min` AND `indoor > target + hyst` | `cool` | Охлаждение притока |
| Всё остальное | `fan_only` | Вентиляция с рекуперацией |

### Fan (итоговая таблица)

| Условие (приоритет сверху вниз) | `desired_fan` |
|---|---|
| `away=off` + `follow_fan_mode` | `off` |
| `away=off` + `hvac_only` | `''` (не менять) |
| `outdoor_air_bad` + `boost ≥ 1` | `medium` |
| `outdoor_air_bad` + `boost = 0` | `low` |
| `boost_count ≥ 2` | `high` |
| `boost_count = 1` | `medium` |
| Всё чисто + stepdown + текущий `high` | `medium` |
| Всё чисто + stepdown + текущий `medium` | `low` |
| Всё чисто | `low` |
| Переходная зона (не clear, но и не boost) | `''` (не менять) |

---

## 6. Граф состояний ПВУ

```mermaid
stateDiagram-v2
    [*] --> fan_only : старт при наличии людей\n(нормальные условия)

    fan_only --> off : away=off\nили eco_active
    off --> fan_only : вернулись домой\nили воздух загрязнился

    fan_only --> heat : outdoor опустился\nниже outdoor_vent_min
    heat --> fan_only : outdoor поднялся\nвыше outdoor_vent_min

    fan_only --> cool : жарко внутри\n+ outdoor > outdoor_cool_min
    cool --> fan_only : indoor вернулся в норму

    heat --> off : away=off (немедленно\nили через anti-flap)
    cool --> off : away=off

    note right of off
        anti-flap действует
        для off и всех переходов
        (кроме away=immediate)
    end note
```

---

## 7. Условные обозначения переменных

| Переменная | Описание |
|---|---|
| `vent_allowed` | `outdoor_temp > outdoor_vent_min_eff` (или permissive) |
| `cool_allowed` | `outdoor_temp > outdoor_cool_min_eff` (или permissive) |
| `outdoor_air_bad` | любой наружный датчик IAQ выше порога блокировки |
| `air_boost_count` | количество внутренних IAQ-датчиков, превысивших порог |
| `all_air_clear` | CO₂, PM2.5, VOC, NOx — все ниже нижнего порога |
| `humidity_clear` | влажность ниже нижнего порога (или датчик не задан) |
| `eco_active` | ECO включён + есть датчики + все ниже ECO-порогов |
| `hvac_switch_allowed` | с последнего переключения прошло ≥ `hvac_min_switch_minutes` |
| `skip_hvac_min_interval_for_off` | `away=off` + `away_off_hvac_policy=immediate` |
