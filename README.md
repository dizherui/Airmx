# 将Airmx加湿器集成到 Home Assistant
Компонент позволяет управлять увлажнителями AIRMX и Tion Iris через Home Assistant полностью локально. Одновременное управление через HA и оригинальное китайское приложение AIRMX невозможно.

Список шагов:
* Установите аддон
* Установите компонент
* Выполните настройки на роутере или запустите выделенную точку доступа на ESP8266
* Сбросьте увлажнитель через зажатие на 3 секунды кнопок Включение + Долив
* Добавьте увлажнитель в Home Assistant
* Опционально: подключите увлажнитель к "домашней" точке доступа

Чат интеграции: [@homeassistant_airmx](https://t.me/homeassistant_airmx)

**Компонент в стадии тестирования, возможны любые проблемы!**

## Поддерживаемые устройства
* AirWater A2
* AirWater A3
* AirWater A3S
* AirWater A3S V2 (он же Tion Iris)
* AirWater A5

## Установка аддона
Для работы компонента требуется установить дополнительный аддон. На него будет перенаправлены запросы, которые устройство пытается отправить к китайским серверам `i.airmx.cn` и `awm.airmx.cn`.

Для установки аддона:
* Настройки > Дополнения > Магазин дополнений
* 3 точки (правый верхний угол) > Репозитории
* Введите `https://github.com/dext0r/airmx` и нажмите Добавить
* 3 точки (правый верхний угол) > Проверить наличие обновлений > Обновите страницу
* В поиске введите `airmx` и установите аддон

Не забудьте запустить аддон и обязательно включите автозагрузку.

## Установка компонента
**Рекомендованный способ:** [HACS](https://hacs.xyz/)
* Установите и настройте [HACS](https://hacs.xyz/docs/use/#getting-started-with-hacs)
* Откройте HACS > Три точки в верхнем правом углу > Пользовательские репозитории
* Добавьте репозиторий `dext0r/airmx` (тип `Интеграция`)
* В поиске найдите и откройте `AIRMX` > Скачать
* Перезагрузите Home Assistant

**Ручной способ**
* Скачайте архив `airmx.zip` из [последнего релиза](https://github.com/dext0r/airmx/releases/latest)
* Создайте подкаталог `custom_components/airmx` в каталоге где расположен файл `configuration.yaml`
* Распакуйте содержимое архива в `custom_components/airmx`
* Перезагрузите Home Assistant

## Перенаправление запросов на аддон
### Роутер Keenetic
1. Создайте выделенный [сегмент сети](https://help.keenetic.com/hc/ru/articles/360005236300-Сегменты-сети):
   * Имя сегмента: любое (например `airmx`)
   * Беспроводная сеть: включить
   * Имя сети SSID: `miaoxin`
   * Защита: `WPA2-PSK`
   * Пароль: `miaoxin666`
   * Использовать NAT: включить
   * Политика доступа: Без доступа в интернет
2. Откройте [командную строку](https://help.keenetic.com/hc/ru/articles/213965889-Интерфейс-командной-строки-CLI-интернет-центра) через веб-интерфейс по адресу `my.keenetic.ru/a` (или `192.168.1.1/a`)
3. Выполните последовательно команды (вместо `X.X.X.X` - IP сервера HA):
   * `ip static tcp 82.157.56.105 255.255.255.255 80 X.X.X.X 25880 !i.airmx.cn`
   * `ip static tcp 140.143.130.176 255.255.255.255 1883 X.X.X.X 25883 !awm.airmx.cn`
   * `ip host i.airmx.cn 82.157.56.105`
   * `ip host awm.airmx.cn 140.143.130.176`

### Роутер Mikrotik
1. Создайте WiFi интерфейс с SSID `miaoxin` и паролем `miaoxin666`
2. Создайте правила DNAT и статические записи в DNS:
```
# вместо X.X.X.X - IP сервера HA
# (по-желанию можно использовать один IP для i и awm)
/ip dns static add address=82.157.56.105 name=i.airmx.cn ttl=30s
/ip dns static add address=140.143.130.176 name=awm.airmx.cn ttl=30s
/ip firewall address-list add address=82.157.56.105 comment=i.airmx.cn list=airmx
/ip firewall address-list add address=140.143.130.176 comment=awm.airmx.cn list=airmx
/ip firewall mangle add action=mark-connection chain=prerouting comment=airmx dst-address-list=airmx new-connection-mark=airmx passthrough=no
/ip firewall nat add action=masquerade chain=srcnat comment=airmx-masquarade connection-mark=airmx
/ip firewall nat add action=dst-nat chain=dstnat comment=airmx-http dst-address-list=airmx dst-port=80 protocol=tcp to-addresses=X.X.X.X to-ports=25880
/ip firewall nat add action=dst-nat chain=dstnat comment=airmx-mqtt dst-address-list=airmx dst-port=1883 protocol=tcp to-addresses=X.X.X.X to-ports=25883
```

### Другие роутеры
Требуется:
1. Создать сеть с SSID `miaoxin` и паролем `miaoxin666`
2. Настроить DNATы (проброс портов, вместо `X.X.X.X` - IP сервера HA):
  * `i.airmx.cn 82.157.56.105 80/tcp` на `X.X.X.X:25880`
  * `awm.airmx.cn 140.143.130.176 1883/tcp` на `X.X.X.X:25883`

Получилось настроить? - Напишите в чат [@homeassistant_airmx](https://t.me/homeassistant_airmx)

### Мини-роутер на ESP8266
Если у вас нет ни Кинетика, ни Микротика, а у вашего роутера лапки - остаётся последний вариант: создать точку доступа на ESP8266.

Для этого:
1. Прошейте в ESP8266 любым прошивальщиком эту прошивку: [airmx-esp-gate.ino.bin](https://github.com/dext0r/airmx/raw/main/airmx-esp-gate/build/esp8266.esp8266.nodemcu/airmx-esp-gate.ino.bin)
2. После загрузки подключитесь к точке доступа `miaoxin` с паролем `miaoxin666`
3. Откройте страницу конфигуратора: `http://192.168.4.1:8888` -> WIFI
4. Выберите домашнюю точку доступа и обязательно укажите IP сервера Home Assistant

## Добавление увлажнителя в Home Assistant
Перед подключением убедитесь, что все сетевые настройки сделаны верно. Для этого подключитесь к точке доступа `miaoxin` и перейдите на страницу `http://i.airmx.cn` - вы должны увидеть сообщение `AIRMX addon`

Для добавления увлажнителя:
1. Сбросьте увлажнитель через зажатие на 3 секунды кнопок Включение + Долив (должен пропищать)
2. После сброса увлажнитель автоматически подключится к точке доступа `miaoxin/miaoxin666` и сделает запрос на `http://i.airmx.cn`, этот запрос будет переадресован в аддон.
3. В Home Assistant откройте Настройки > Устройства и службы > Добавить интеграцию > AIRMX (если нет в списке - обновите страницу)
4. Выберите "Автоматическая настройка"

## Подключение увлажнителя к домашней точке доступа
Необязательный, но рекомендованный шаг.

Через Home Assistant (нужен настроенный Bluetooth адаптер):
* Откройте Настройки > Устройства и службы > Добавить интеграцию > AIRMX
* Выберите "Привязать устройство к точке доступа"

Через родное приложение AIRMX:
* Требуется вход через Apple ID или китайский номер телефона (получить смску :))
* Следуйте [скриншотам](./images/ios) (обязательно дать права на геолокацию)
* После ввода SSID и пароля подождать 20-30 секунд и можно закрывать приложение

## Прочее
* Изменение привязки к точке доступа может быть сделано в любой момент, удалять устройство из HA не требуется
