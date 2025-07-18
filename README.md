# Qt_Interface_STM
Создание интерфейса в Qt для STM32. Простой интерфейс, позволяющий взаимодействовать с подключенными к STM32 модулями.

Просто буду записывать всё, что я буду делать. Составив потом из этого некую документацию.

# Проект: Система мониторинга и управления устройствами на STM32 с Qt-интерфейсом
**Цель**: Создание системы для удалённого управления периферийными устройствами STM32 через графический интерфейс Qt с визуализацией данных в реальном времени.
## Аппаратная часть (STM32)
1. **Периферия**:
   - Датчик температуры/влажности (DHT22)
   - Светодиоды (3 шт: красный, зелёный, синий)
   - Кнопка пользователя
   - Потенциометр (аналоговый вход)
   - Реле для управления "тяжёлой" нагрузкой (имитация)
2. **Функционал STM32**:
   - Опрос датчиков каждые 2 секунды
   - Управление светодиодами и реле
   - АЦП для чтения потенциометра
   - Обработка прерываний от кнопки
   - UART-коммуникация с протоколом:
     ```json
     {"temp":25.5, "hum":45, "adc":0.75, "btn":0, "leds":[1,0,1]}
     ```
## Программная часть (Qt)
1. **Интерфейс**:
   - Панель показаний датчиков (температура, влажность, АЦП)
   - График изменения температуры (QCustomPlot)
   - Кнопки управления светодиодами
   - Ползунок для управления реле
   - Индикатор состояния кнопки на STM32
   - Лог событий
2. **Функционал Qt**:
   - Подключение к COM-порту
   - Парсинг JSON-данных
   - Визуализация показаний в реальном времени
   - Отправка команд:
     ```json
     {"led_r":1, "led_g":0, "led_b":0, "relay":0.5}
     ```
   - Экспорт данных в CSV
   - Аварийное оповещение при выходе за пороги температуры
## Протокол связи
| Направление | Данные | Формат |
|-------------|--------|--------|
| **STM32 → Qt** | Показания датчиков | JSON с полями: `temp`, `hum`, `adc`, `btn`, `leds` (массив состояний светодиодов [R, G, B]) |
| **Qt → STM32** | Управляющие команды | JSON с полями: `led_r`, `led_g`, `led_b` (0/1), `relay` (0.0 - выключено, 1.0 - включено) |
## Требования
1. **Для STM32**:
   - Микроконтроллер: STM32F4xx (например, STM32F411RE)
   - Библиотеки: HAL, библиотека для DHT22
   - Среда разработки: STM32CubeIDE
   - Скорость UART: 115200 бод
2. **Для Qt**:
   - Qt версия 5.15 или выше
   - Модули: SerialPort, Widgets (и, возможно, Charts, если не используется QCustomPlot)
   - Компилятор: MinGW или MSVC
## Структура репозитория
```
├── STM32_Firmware/          # Проект CubeIDE
│   ├── Core/
│   │   ├── Src/
│   │   │   ├── main.c       # Основной цикл
│   │   │   ├── stm32f4xx_it.c
│   │   │   └── ...          # Другие исходные файлы
│   │   └── Inc/             # Заголовочные файлы
│   │       ├── main.h
│   │       ├── uart_protocol.h # Парсинг команд и отправка данных
│   │       └── ...
│   └── ...                  # Другие папки проекта CubeIDE (Drivers, etc)
├── Qt_Interface/            # Проект Qt
│   ├── include/             
│   ├── src/
│   │   ├── MainWindow.cpp   # Основная логика
│   │   ├── SerialHandler.cpp # Работа с COM-портом
│   │   └── ... 
│   ├── ui/                  # Файлы интерфейса
│   └── ...                  # Другие файлы проекта
├── Docs/                    # Схемы подключения (если есть)
├── README.md                # Этот файл
└── LICENSE                  # Лицензия (например, MIT)
```
## Этапы разработки
1. Реализация базовой прошивки STM32 с UART
2. Создание JSON-протокола обмена
3. Разработка Qt-интерфейса:
   - Настройка SerialPort
   - Визуализация данных
   - Управляющие элементы
4. Тестирование связки:
   - Эмуляция данных через терминал
   - Реальная работа с платой
## Примеры кода
**STM32 (отправка данных)**:
```c
void send_sensor_data(float temp, float hum, float adc_val, uint8_t btn_state, uint8_t *leds) {
    char buffer[128];
    sprintf(buffer, "{\"temp\":%.1f,\"hum\":%.1f,\"adc\":%.2f,\"btn\":%d,\"leds\":[%d,%d,%d]}\r\n",
            temp, hum, adc_val, btn_state, leds[0], leds[1], leds[2]);
    HAL_UART_Transmit(&huart2, (uint8_t*)buffer, strlen(buffer), HAL_MAX_DELAY);
}
```
**Qt (приём данных)**:
```cpp
void SerialHandler::readData() {
    static QByteArray buffer;
    buffer += serialPort->readAll();
    int endIndex;
    while ((endIndex = buffer.indexOf("\r\n")) != -1) {
        QByteArray packet = buffer.left(endIndex);
        buffer = buffer.mid(endIndex + 2); // +2 для пропуска "\r\n"
        QJsonParseError error;
        QJsonDocument doc = QJsonDocument::fromJson(packet, &error);
        if (error.error != QJsonParseError::NoError) {
            qDebug() << "JSON error:" << error.errorString();
            continue;
        }
        if (doc.isObject()) {
            QJsonObject obj = doc.object();
            double temp = obj["temp"].toDouble();
            double hum = obj["hum"].toDouble();
            double adc = obj["adc"].toDouble();
            int btn = obj["btn"].toInt();
            QJsonArray leds = obj["leds"].toArray();
            // ... обработка данных, испускаем сигнал
            emit newSensorData(temp, hum, adc, btn, leds);
        }
    }
}
```
