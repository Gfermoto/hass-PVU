# Blueprint для автоматизации ПВУ Turkov в Home Assistant

Долго искал подходящую ПВУ, остановился на Turkov из-за наличия интеграции — но оказалось, что для грамотного управления нужна качественная автоматизация. Дописал свою, делюсь в виде Blueprint.

В автоматике: температурный контур по day/night/away, управление вентилятором по качеству воздуха (CO2, PM2.5 и др.), антизаморозка, anti-flap и явные политики в режиме отсутствия. У меня по воздуху завязано на AirGradient, но датчики можно любые или обойтись без них. Blueprint завязан на climate-сущность — теоретически может подойти и к другим установкам с climate в HA, но проверял только на Turkov.

Blueprint: https://github.com/Gfermoto/hass-PVU/blob/main/pvu.yaml

Нужна интеграция Turkov (climate): https://github.com/alryaz/hass-turkov

Если попробуете — буду благодарен за фидбек. Нашёл баг или есть идея улучшения — welcome в Issues: https://github.com/Gfermoto/hass-PVU/issues
