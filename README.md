# -DE-CODER-ESP_NEW

Теперь я полностью понял архитектуру. Цель проекта — расширить количество входов для блока ESP_NOW ESP8266, объединив данные от многих датчиков в один пакет, который можно передать через один вход ESP-NOW блока.

Правильная задача:
CODER>ESP_NEW должен упаковывать данные от множества датчиков в единый поток, который можно подать на один из доступных типизированных входов блока ESP_NOW ESP8266.
#  Решение: Использование входа String
У блока ESP_NOW ESP8266 есть входы типа String.

Строка может содержать структурированные данные.

Это полностью соответствует описанию проекта.
Итоговая архитектура:
[Датчик 1 Float] ───┐
[Датчик 2 Float] ───┤
[Датчик 3 Float] ───┼──> [CODER>ESP_NEW] ──> (Выход String) ──> [Вход String ESP_NOW ESP8266]
[Датчик 1 Int]   ───┤
[Датчик 2 Int]   ───┤
[Датчик 3 Int]   ───┘
# Формат строки:
CODER будет формировать строку вида F1:25.50;F2:30.10;I1:100;I2:200;..., где:

F1, F2 — ключи для float-датчиков

I1, I2 — ключи для int-датчиков

Декодер на стороне приемника будет разбирать эту строку по разделителю ; и ключам, направляя значения на нужные выходы.

Нужно ли мне сразу переписать код CODER>ESP_NEW для формирования такой строки? После этого мы создадим DECODER_ESP_NEW.

# DeclareSection

// Переменные для хранения предыдущих значений датчиков
float old_f1 = 0, old_f2 = 0, old_f3 = 0;
int old_i1 = 0, old_i2 = 0, old_i3 = 0;
bool firstRun = true;

# LoopSection

// ИНИЦИАЛИЗАЦИЯ ПРИ ПЕРВОМ ЗАПУСКЕ
if (firstRun) {
    old_f1 = in_sensor1_float;
    old_f2 = in_sensor2_float;
    old_f3 = in_sensor3_float;
    old_i1 = in_sensor1_int;
    old_i2 = in_sensor2_int;
    old_i3 = in_sensor3_int;
    firstRun = false;
    out_debug_str = "Coder INIT";
    return;
}

// ОСНОВНАЯ ЛОГИКА: ПРОВЕРЯЕМ ИЗМЕНЕНИЯ И ФОРМИРУЕМ СТРОКУ
if (in_trigger) {
    // --- Проверка FLOAT датчиков ---
    if (in_sensor1_float != old_f1) {
        sendSensorUpdate("F1", in_sensor1_float);
        old_f1 = in_sensor1_float;
        return;
    }
    if (in_sensor2_float != old_f2) {
        sendSensorUpdate("F2", in_sensor2_float);
        old_f2 = in_sensor2_float;
        return;
    }
    if (in_sensor3_float != old_f3) {
        sendSensorUpdate("F3", in_sensor3_float);
        old_f3 = in_sensor3_float;
        return;
    }

    // --- Проверка INT датчиков (преобразуем int во float) ---
    if (in_sensor1_int != old_i1) {
        sendSensorUpdate("I1", (float)in_sensor1_int);
        old_i1 = in_sensor1_int;
        return;
    }
    if (in_sensor2_int != old_i2) {
        sendSensorUpdate("I2", (float)in_sensor2_int);
        old_i2 = in_sensor2_int;
        return;
    }
    if (in_sensor3_int != old_i3) {
        sendSensorUpdate("I3", (float)in_sensor3_int);
        old_i3 = in_sensor3_int;
        return;
    }
    out_debug_str = "No changes"; // Сообщение, если триггер был, но изменений нет
}

// === ВСПОМОГАТЕЛЬНАЯ ФУНКЦИЯ ДЛЯ ФОРМИРОВАНИЯ СТРОКИ ===
void sendSensorUpdate(String key, float value) {
    // Формируем строку в формате "КЛЮЧ:ЗНАЧЕНИЕ" (например, "F1:25.50")
    out_espnow_string = key + ":" + String(value, 2);
    // Для отладки выводим более подробную информацию в монитор порта
    out_debug_str = "Sent -> " + out_espnow_string;
}
# Что получилось:
Блок отслеживает изменения на 6 входах.

При изменении одного датчика формирует компактную строку (например, "F1:25.50").

Эта строка подается на выход out_espnow_string, который нужно будет подключить к строковому входу (String) блока ESP_NOW ESP8266 (передатчик).

Проверьте и сохраните блок. После этого мы создадим DECODER_ESP_NEW для разбора такой строки на стороне приемника.
#
# блока DECODER_ESP_NEW
#
## Код для DeclareSection
// Переменные для хранения последних разобранных значений
float last_f1 = 0, last_f2 = 0, last_f3 = 0;
int last_i1 = 0, last_i2 = 0, last_i3 = 0;
## Код для LoopSection
// ОСНОВНАЯ ЛОГИКА: Активируется по переднему фронту in_trigger
if (in_trigger && in_espnow_string.length() > 0) {
    // Ищем разделитель ':'
    int colonIndex = in_espnow_string.indexOf(':');
    
    if (colonIndex != -1) {
        // Разделяем строку на ключ и значение
        String key = in_espnow_string.substring(0, colonIndex);
        String valStr = in_espnow_string.substring(colonIndex + 1);
        
        // Преобразуем строку значения в число (float)
        float value = valStr.toFloat();
        
        // Распределяем значение по ключу
        if (key == "F1") {
            last_f1 = value;
            out_sensor1_float = last_f1;
            out_debug_str = "D: F1=" + String(last_f1, 2);
        }
        else if (key == "F2") {
            last_f2 = value;
            out_sensor2_float = last_f2;
            out_debug_str = "D: F2=" + String(last_f2, 2);
        }
        else if (key == "F3") {
            last_f3 = value;
            out_sensor3_float = last_f3;
            out_debug_str = "D: F3=" + String(last_f3, 2);
        }
        else if (key == "I1") {
            last_i1 = (int)value; // Преобразуем float обратно в int
            out_sensor1_int = last_i1;
            out_debug_str = "D: I1=" + String(last_i1);
        }
        else if (key == "I2") {
            last_i2 = (int)value;
            out_sensor2_int = last_i2;
            out_debug_str = "D: I2=" + String(last_i2);
        }
        else if (key == "I3") {
            last_i3 = (int)value;
            out_sensor3_int = last_i3;
            out_debug_str = "D: I3=" + String(last_i3);
        }
        else {
            out_debug_str = "D: Unknown key -> " + in_espnow_string;
        }
    } else {
        out_debug_str = "D: Invalid format -> " + in_espnow_string;
    }
}
#  Итоговая архитектура на приёмной стороне:
[ESP_NOW ESP8266 (Режим "Прием")] --> (Выход String) --> [DECODER_ESP_NEW.in_espnow_string]
                                                               |
                                            [DECODER_ESP_NEW.out_sensor1_float] --> [Nextion]
                                            [DECODER_ESP_NEW.out_sensor2_float] --> [Nextion]
                                                               ... и т.д.
## Архитектура отладки на приёмной стороне
[ESP_NOW ESP8266 (Receiver)] --- (debug String) ---> [Software UART 1] ---> Монитор порта (Строка 1)
         ↓
(espnow_string) ---> [DECODER_ESP_NEW] --- (out_debug_str) ---> [Software UART 2] ---> Монитор порта (Строка 2)

# Настройка блока ESP_NOW ESP8266 (приёмник)
В параметрах блока активируйте флажок для создания выхода debug (см. пункт 2.3 описания: "debug - выход для вывода информации при отладке...").

В проекте FLProg:

Добавьте блок Software UART.

Подключите выход debug блока ESP_NOW ESP8266 ко входу блока Software UART.

В настройках Software UART выберите, например, UART1 и скорость (115200).

Теперь в Мониторе порта вы будете видеть системные сообщения от ESP-NOW.
# Настройка DECODER_ESP_NEW для отладки
Мы уже предусмотрели выход out_debug_str. Нужно вывести его на второй виртуальный порт.

В проекте FLProg:

Добавьте второй блок Software UART (или используйте другой канал того же блока, если он поддерживает несколько).

Подключите выход out_debug_str блока DECODER_ESP_NEW ко входу этого Software UART.

В настройках выберите, например, UART2.
# Пример кода для LoopSection DECODER_ESP_NEW (улучшенная отладка)
Чтобы в Мониторе порта было понятнее, можно немного изменить строку отладки в коде LoopSection DECODER_ESP_NEW. Найдите блоки out_debug_str = "D: F1=..." и замените формирование строки на более информативное:
# LoopSection блока DECODER_ESP_NEW:
// ОСНОВНАЯ ЛОГИКА: Активируется по переднему фронту in_trigger
if (in_trigger && in_espnow_string.length() > 0) {
    // Ищем разделитель ':'
    int colonIndex = in_espnow_string.indexOf(':');
    
    if (colonIndex != -1) {
        // Разделяем строку на ключ и значение
        String key = in_espnow_string.substring(0, colonIndex);
        String valStr = in_espnow_string.substring(colonIndex + 1);
        
        // Преобразуем строку значения в число (float)
        float value = valStr.toFloat();
        
        // Распределяем значение по ключу и формируем ОДНУ отладочную строку
        if (key == "F1") {
            last_f1 = value;
            out_sensor1_float = last_f1;
            out_debug_str = "[DECODER] Key:F1 Val:" + String(last_f1, 2);
        }
        else if (key == "F2") {
            last_f2 = value;
            out_sensor2_float = last_f2;
            out_debug_str = "[DECODER] Key:F2 Val:" + String(last_f2, 2);
        }
        else if (key == "F3") {
            last_f3 = value;
            out_sensor3_float = last_f3;
            out_debug_str = "[DECODER] Key:F3 Val:" + String(last_f3, 2);
        }
        else if (key == "I1") {
            last_i1 = (int)value;
            out_sensor1_int = last_i1;
            out_debug_str = "[DECODER] Key:I1 Val:" + String(last_i1);
        }
        else if (key == "I2") {
            last_i2 = (int)value;
            out_sensor2_int = last_i2;
            out_debug_str = "[DECODER] Key:I2 Val:" + String(last_i2);
        }
        else if (key == "I3") {
            last_i3 = (int)value;
            out_sensor3_int = last_i3;
            out_debug_str = "[DECODER] Key:I3 Val:" + String(last_i3);
        }
        else {
            out_debug_str = "[DECODER] ERR: Unknown key -> " + in_espnow_string;
        }
    } else {
        out_debug_str = "[DECODER] ERR: Invalid format -> " + in_espnow_string;
    }
}
                                                                           
#  Порядок тестирования
Соберите схему передатчика (CODER>ESP_NEW → ESP_NOW (Transmitter)).

Соберите схему приёмника как указано выше.

Откройте Монитор порта (два окна для двух UART или один с фильтрацией).

Подайте изменение на датчик передатчика. Должны увидеть:
                    
Сообщение о отправке на стороне передатчика.

Сообщение о приёме от ESP_NOW (Receiver).

Сообщение "[DECODER] Key:..." о успешном разборе.
 # Ключевое изменение:
 Все 6 строк out_debug_str = "D: F1=... заменены на единый формат "[DECODER] Key:XX Val:...".

Добавлены префиксы [DECODER] для фильтрации и ERR: для ошибок.

### ошибки в arduino IDE
C:\Users\sasa\AppData\Local\Temp\.arduinoIDE-unsaved202606-8336-12vz7e7.m828\sketch_jan6a\sketch_jan6a.ino:14:1: error: 'FLProgUartParametrs' does not name a type
   14 | FLProgUartParametrs Uart_256272636_data;
      | ^~~~~~~~~~~~~~~~~~~
C:\Users\sasa\AppData\Local\Temp\.arduinoIDE-unsaved202606-8336-12vz7e7.m828\sketch_jan6a\sketch_jan6a.ino: In function 'void setup()':
C:\Users\sasa\AppData\Local\Temp\.arduinoIDE-unsaved202606-8336-12vz7e7.m828\sketch_jan6a\sketch_jan6a.ino:56:3: error: 'Uart_256272636_data' was not declared in this scope
   56 |   Uart_256272636_data.speed = 115200;
      |   ^~~~~~~~~~~~~~~~~~~
C:\Users\sasa\AppData\Local\Temp\.arduinoIDE-unsaved202606-8336-12vz7e7.m828\sketch_jan6a\sketch_jan6a.ino: In function 'void loop()':
C:\Users\sasa\AppData\Local\Temp\.arduinoIDE-unsaved202606-8336-12vz7e7.m828\sketch_jan6a\sketch_jan6a.ino:143:6: error: a function-definition is not allowed here before '{' token
  143 |      {
      |      ^
C:\Users\sasa\AppData\Local\Temp\.arduinoIDE-unsaved202606-8336-12vz7e7.m828\sketch_jan6a\sketch_jan6a.ino:166:9: error: 'sendSensorUpdate' was not declared in this scope
  166 |         sendSensorUpdate("F1", in_sensor1_float_41199591_1);
      |         ^~~~~~~~~~~~~~~~
C:\Users\sasa\AppData\Local\Temp\.arduinoIDE-unsaved202606-8336-12vz7e7.m828\sketch_jan6a\sketch_jan6a.ino:171:9: error: 'sendSensorUpdate' was not declared in this scope
  171 |         sendSensorUpdate("F2", in_sensor2_float_41199591_1);
      |         ^~~~~~~~~~~~~~~~
C:\Users\sasa\AppData\Local\Temp\.arduinoIDE-unsaved202606-8336-12vz7e7.m828\sketch_jan6a\sketch_jan6a.ino:176:9: error: 'sendSensorUpdate' was not declared in this scope
  176 |         sendSensorUpdate("F3", in_sensor3_float_41199591_1);
      |         ^~~~~~~~~~~~~~~~
C:\Users\sasa\AppData\Local\Temp\.arduinoIDE-unsaved202606-8336-12vz7e7.m828\sketch_jan6a\sketch_jan6a.ino:182:9: error: 'sendSensorUpdate' was not declared in this scope
  182 |         sendSensorUpdate("I1", (float)in_sensor1_int_41199591_1);
      |         ^~~~~~~~~~~~~~~~
C:\Users\sasa\AppData\Local\Temp\.arduinoIDE-unsaved202606-8336-12vz7e7.m828\sketch_jan6a\sketch_jan6a.ino:187:9: error: 'sendSensorUpdate' was not declared in this scope
  187 |         sendSensorUpdate("I2", (float)in_sensor2_int_41199591_1);
      |         ^~~~~~~~~~~~~~~~
C:\Users\sasa\AppData\Local\Temp\.arduinoIDE-unsaved202606-8336-12vz7e7.m828\sketch_jan6a\sketch_jan6a.ino:192:9: error: 'sendSensorUpdate' was not declared in this scope
  192 |         sendSensorUpdate("I3", (float)in_sensor3_int_41199591_1);
      |         ^~~~~~~~~~~~~~~~
C:\Users\sasa\AppData\Local\Temp\.arduinoIDE-unsaved202606-8336-12vz7e7.m828\sketch_jan6a\sketch_jan6a.ino:198:121: error: 'Uart_256272636_data' was not declared in this scope
  198 |     if( ! (0)) {if( ! ((out_debug_str_41199591_1) == (_stou1))) {FLProgUart.printUart(String(out_debug_str_41199591_1), Uart_256272636_data.number);}}
