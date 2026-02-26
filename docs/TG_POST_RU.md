# Turkov AirFlow: проект завершен (v0.5.3)

Финализировал проект и зафиксировал стабильную рабочую версию: `v0.5.3`  
<https://github.com/Gfermoto/hass-PVU>

Что в итоге:

- разделение на `pvu_min.yaml` (стабильная база) и `pvu.yaml` (полный режим);
- явные политики для `away=off`:
  - `away_off_hvac_policy` (`respect_min_interval` / `immediate`);
  - `away_off_fan_policy` (`hvac_only` / `follow_fan_mode`);
- post-check диагностика через `command_verify_delay_sec` (если команда ушла, но state не изменился — это видно сразу);
- полевой beta-pass по критичным сценариям: `home/away`, anti-flap, day/night.

Важно: в зимнем контуре `cool/fan_only` могут ограничиваться самим устройством/интеграцией — это отдельно зафиксировано в документации.

Релизы: <https://github.com/Gfermoto/hass-PVU/releases>  
README: <https://github.com/Gfermoto/hass-PVU/blob/main/README.md>  
Changelog: <https://github.com/Gfermoto/hass-PVU/blob/main/CHANGELOG.md>
