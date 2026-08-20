# Управление сценами OBS Studio в Home Assistant

Этот гайд позволит вам настроить переключение между сценами в OBS Studio с помощью сервера умного дома Home Assistant. Вы сможете использовать сцены в любых ваших автоматизациях и скриптах Home Assistant, а если у вас к Home Assistant прикручена Яндекс Станция, то вы сможете настроить переключение между сценами в OBS Studio с помощью голосовых команд, что будет полезно стримерам с инвалидностью.

---
## Требования для работы

* Установленный **Home Assistant** на локальном сервере.
* Установленный **MQTT-брокер Eclipse Mosquitto** в локальном Docker.
* Установленный **Python 3.x** в системе, где будет запускаться OBS Studio.

---
## 1. Установка MQTT-брокера Eclipse Mosquitto в локальном Docker

В качестве примера будет описана установка Eclipse Mosquitto в Synology NAS в Container Manager с помощью docker-compose. Для установки в обычный Docker возможно потребуется модифицировать docker-compose.

1. Создайте в Synology File Station следующую структуру папок:

```
/docker/mosquitto/config/
/docker/mosquitto/data/
/docker/mosquitto/log/
```

2. Перейдите в /docker/mosquitto/config/ и создайте файл mosquitto.conf со следующим содержанием:

```
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log
#password_file /mosquitto/config/pwfile
allow_anonymous true 
listener 1883
```

Так как у нас локальный сервер и скрипт OBS2HA.py будет работать только во время записи или стрима в OBS Studio, то доступ к MQTT можно сделать анонимным без пароля, если вам нужна максимальная безопасность, то можете использовать MQTT с паролем.

3. Создайте в Container Manager новый проект с названием mosquitto, выберите путь /docker/mosquitto/, выберите в источнике **«Создать docker-compose.yml»**, вставьте в окно ниже следующий код:

```yaml
services:
  mosquitto:
    image: eclipse-mosquitto:latest
    container_name: mosquitto
    ports:
      - 1883:1883
      - 9001:9001
    volumes:
      - ./config/mosquitto.conf:/mosquitto/config/mosquitto.conf
      - ./data:/mosquitto/data
      - ./log:/mosquitto/log
    restart: always
```

---

## 2. Добавление интеграции MQTT в Home Assistant

  1. Откройте веб-интерфейс Home Assistant и перейдите в **«Настройки» → «Устройства и службы»**. Нажмите **«Добавить интеграцию»** и введите в поиск **MQTT**.
  2. Выберите **MQTT**, укажите IP-адрес вашего MQTT-брокера (IP-адрес сервера, на котором запущен Home Assistant и Eclipse Mosquitto) и порт **1883**.
  3. Нажмите **«Отправить»**. Если всё введено правильно, Home Assistant подключится к MQTT-брокеру, запущенному в Docker, и покажет сообщение об успешном подключении.

---

## 3. Создание пользователя в Home Assistant для OBS Studio

1. Откройте веб-интерфейс Home Assistant, перейдите в **«Настройки»** → **«Люди»** и нажмите **«Добавить человека»**.
2. Напишите имя **obs** и отметьте **«Разрешите вход в систему»**.
3. В открывшемся окне напишите имя пользователя **obs** и придумайте пароль.

---

## 4. Включение WebSocket-сервера в OBS Studio и подготовка сцен

1. Запустите OBS Studio. перейдите в **«Сервис»** → **«Настройки сервера WebSocket»**.
2. Отметьте **«Включить сервер WebSocket»**.
3. Оставьте порт сервера по умолчанию **4455** и отметьте **«Включить вход в аккаунт»**.
4. Придумайте пароль сервера и нажмите **«Создать пароль»**.
5. Нажмите **«Показать сведения о подключении»** и запишите IP-сервер, порт, пароль.

**Подготовка сцен**

Убедитесь, что все названия сцен в OBS Studio написаны на английском языке без пробелов. Это нужно, чтобы скрипт OBS2HA.py корректно обрабатывал имена сцен и передавал их в Home Assistant.

---

## 5. Установка Python 3.x и настройка скрипта OBS2HA.py

На компьютере с OBS Studio необходимо установить Python 3.x и необходимые библиотеки для работы скрипта OBS2HA.py.

1. Скачайте инсталлятор Python 3.x с официального сайта [python.org](https://www.python.org/downloads/) (Get the standalone installer for Python 3.x) и запустите его. При установке отметьте опции «Add Python to PATH» и «Install for all users (Admin)». Это позволит запускать Python-скрипты из командной строки.

2. Нажмите в Windows **«Пуск»**, напишите в поиске **cmd** и нажмите Enter.

3. Введите следующую команду:

```python
pip3 install paho-mqtt
pip3 install obs-websocket-py
```
   
   Это установит Python-библиотеку **paho-mqtt** для работы с MQTT и **obs-websocket-py** для взаимодействия с OBS Studio через WebSocket.
   
4. Скачайте скрипт OBS2HA.py в удобное вам место и откройте файл в текстовом редакторе. Найдите и измените в нем следующие строчки с настройками подключения:
```python
# OBS WEBSOCKET SERVER
host = "192.168.1.10" # IP-адрес вашего WebSocket-сервера в OBS
port = 4455 # Порт вашего WebSocket-сервера в OBS
password = "Пароль" # Пароль вашего WebSocket-сервера в OBS

# MQTT SERVER
MQTT_HOST = "192.168.1.11" # IP-адрес вашего MQTT-сервера 
MQTT_USER = "obs" # Имя пользователя obs в Home Assistant
MQTT_PW = "Пароль" # Пароль пользователя obs в Home Assistant
MQTT_PORT = 1883 # Порт вашего MQTT-сервера по умолчанию
```

5. Помимо управлениями сценами, можно управлять включением и отключением звука и микрофона в OBS Studio и их уровнями громкости, по умолчанию работает с Windows. Для MacOS нужно найти в OBS2HA.py следующие строчки:

```python
setup_mute_switches_in_homeassistant("wasapi_output_capture")
setup_mute_switches_in_homeassistant("wasapi_input_capture")
setup_mute_switches_in_homeassistant("wasapi_process_output_capture")
setup_volume_control_in_homeassistant("wasapi_output_capture")
setup_volume_control_in_homeassistant("wasapi_input_capture")
setup_volume_control_in_homeassistant("wasapi_process_output_capture")
```

И заменить их следующими строчками:

```python
setup_mute_switches_in_homeassistant("sck_audio_capture")
setup_mute_switches_in_homeassistant("coreaudio_input_capture")
setup_volume_control_in_homeassistant("sck_audio_capture")
setup_volume_control_in_homeassistant("coreaudio_input_capture")
```

---

## 6. Проверка работы скрипта OBS2HA.py

1. Запустите OBS Studio и запустите скрипт OBS2HA.py двойным щелчком мыши или через команду:

```python
python OBS2HA.py
```

Появится консольное окно скрипта:

```python
OBS Connection Successful
MQTT Connection Successful
```

Не закрывайте это окно — скрипт должен постоянно работать в фоновом режиме. Он будет слушать события OBS Studio через WebSocket и пересылать их в Home Assistant по MQTT. После каждого запуска OBS Studio запускайте скрипт заново, чтобы связь поддерживалась.

2. Откройте веб-интерфейс Home Assistant. Перейдите в **«Настройки» → «Устройства и службы»** и выберите **MQTT**. Если все работает нормально, то должна отображаться работа службы OBSHA. Открыв ее вы должны будете увидеть все ваши сцены в OBS с именем типа Scene Game. Можете проверить работу переключения сцен с помощью переключателей.
---

Теперь вы можете использовать переключение сцен в OBS Studio в любых ваших автоматизациях и скриптах в Home Assistant. Если у вас прикручена Яндекс Станция через интеграцию Yandex Smart Home, то вы можете создать в Home Assistant скрипт с названием на русском языке, выбрать объект сцены в OBS, выбрать комнату. После чего добавить скрипт в Yandex Smart Home и он автоматически появится в вашем Умном доме Яндекса.
