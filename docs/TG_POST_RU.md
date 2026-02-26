# Turkov AirFlow v0.5.0: Этап 3 завершен

Выкатил `v0.5.0` и закрыл Этап 3 roadmap:
<https://github.com/Gfermoto/hass-PVU>

Что нового в develop (`pvu.yaml`):

- `climate_profile`: `continental` / `mild` / `cold`;
- `home_profile`: `apartment` / `house` / `office`;
- профили влияют на эффективные пороги температуры, anti-freeze и IAQ;
- сезонный smoke-test оформлен как регулярная практика в чеклисте релиза.
- дефолтный старт для МО/загородного дома: `climate_profile=continental`, `home_profile=house`.

Плюс по UX и сопровождению:

- постоянные дашборды: `airflow_dashboard.yaml`, `airflow_dashboard_min.yaml`;
- карточки для `Manual card`: `airflow_card.yaml`, `airflow_card_min.yaml`;
- KPI-пакет для верификации порогов по реальным данным: `airflow_kpi_package.yaml`.

Если у вас Turkov + HA, можно брать как есть и адаптировать под свой климат/тип жилья без переписывания логики.

Релизы: <https://github.com/Gfermoto/hass-PVU/releases>  
Документация: `README.md`  
Что изменилось: `CHANGELOG.md`
