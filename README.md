# -DE-CODER-ESP_NEW - проект
# CODER>ESP_NEW - блок
Теперь я полностью понял архитектуру. Цель проекта — расширить количество входов для блока ESP_NOW ESP8266, объединив данные от многих датчиков в один пакет, который можно передать через один вход ESP-NOW блока.

# Правильная задача:
CODER>ESP_NEW должен упаковывать данные от множества датчиков в единый поток, который можно подать на один из доступных типизированных входов блока ESP_NOW ESP8266.

#  Решение: Использование входа String

У блока ESP_NOW ESP8266 есть входы типа String.

Строка может содержать структурированные данные.

Это полностью соответствует описанию проекта.

# Итоговая архитектура:

[Датчик 1 Float] ───┐
[Датчик 2 Float] ───┤
[Датчик 3 Float] ───┼──> [CODER>ESP_NEW] ──> (Выход String) ──> [Вход String ESP_NOW ESP8266]
[Датчик 1 Int]   ───┤
[Датчик 2 Int]   ───┤
[Датчик 3 Int]   ───┘

# Входы-Выходы

Тип	Имя	Отображаемое имя	Тип данных
Вход	in_sensor1_float	Д1 (float)	Float
Вход	in_sensor2_float	Д2 (float)	Float
Вход	in_sensor3_float	Д3 (float)	Float
Вход	in_sensor1_int	Д4 (int)	Integer
Вход	in_sensor2_int	Д5 (int)	Integer
Вход	in_sensor3_int	Д6 (int)	Integer
Вход	in_peer_mac	MAC получателя	String
Вход	in_trigger	Триггер	Boolean
Выход	out_debug_str	данные для ESP-NOW	String

------------

Тип	Имя (Name)	Тип данных (Data Type)
Вход	in_espnow_string	String
Вход	in_trigger	Boolean
Выход	out_sensor1_float	Float
Выход	out_sensor2_float	Float
Выход	out_sensor3_float	Float
Выход	out_sensor1_int	Integer
Выход	out_sensor2_int	Integer
Выход	out_sensor3_int	Integer
Выход	out_debug_str	String

# Формат строки:

CODER будет формировать строку вида F1:25.50;F2:30.10;I1:100;I2:200;..., где:

F1, F2 — ключи для float-датчиков

I1, I2 — ключи для int-датчиков

Декодер на стороне приемника будет разбирать эту строку по разделителю ; и ключам, направляя значения на нужные выходы.

Нужно ли мне сразу переписать код CODER>ESP_NEW для формирования такой строки? После этого мы создадим DECODER_ESP_NEW.

# DeclareSection

// Переменные для хранения предыдущих значений датчиков
float old_f1= 0, old_f2 = 0, old_f3 = 0;
int old_i1= 0, old_i2 = 0, old_i3 = 0;
bool firstRun= true;
// Объявление функции (без тела, только сигнатура)
void sendSensorUpdate(String key, float value);


# LoopSection

// === ОПРЕДЕЛЕНИЕ ФУНКЦИИ ===
void sendSensorUpdate(String key, float value) {
    out_espnow_string = key + ":" + String(value, 2);
    out_debug_str = "Sent -> " + out_espnow_string;
}

// === ОСНОВНОЙ КОД ===
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
    out_debug_str = "No changes";
}

# Что получилось:


Блок отслеживает изменения на 6 входах.

При изменении одного датчика формирует компактную строку (например, "F1:25.50").

Эта строка подается на выход out_espnow_string, который нужно будет подключить к строковому входу (String) блока ESP_NOW ESP8266 (передатчик).

Проверьте и сохраните блок. После этого мы создадим DECODER_ESP_NEW для разбора такой строки на стороне приемника.



# блок DECODER_ESP_NEW - далее

# Код для DeclareSection

// Переменные для хранения последних разобранных значений
float last_f1 = 0, last_f2 = 0, last_f3 = 0;
int last_i1 = 0, last_i2 = 0, last_i3 = 0;


# Код для LoopSection

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

###  ошибки компиляции в arduino IDE 2.3.7

Компиляция скетча...
"C:\\Users\\sasa\\AppData\\Local\\Arduino15\\packages\\esp8266\\tools\\python3\\3.7.2-post1/python3" -I "C:\\Users\\sasa\\AppData\\Local\\Arduino15\\packages\\esp8266\\hardware\\esp8266\\3.1.2/tools/signing.py" --mode header --publickey "H:\\RADIO\\MK\\Proektis\\АРХИВ\\Proshivki_moy_rabochie_archi_!!!\\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON/public.key" --out "C:\\Users\\sasa\\AppData\\Local\\arduino\\sketches\\29B76F2D42BEFEE8020B4D7A1D917A76/core/Updater_Signing.h"
"C:\\Users\\sasa\\AppData\\Local\\Arduino15\\packages\\esp8266\\tools\\xtensa-lx106-elf-gcc\\3.1.0-gcc10.3-e5f9fec/bin/xtensa-lx106-elf-g++" -D__ets__ -DICACHE_FLASH -U__STRICT_ANSI__ -D_GNU_SOURCE -DESP8266 "@C:\\Users\\sasa\\AppData\\Local\\arduino\\sketches\\29B76F2D42BEFEE8020B4D7A1D917A76/core/build.opt" "-IC:\\Users\\sasa\\AppData\\Local\\Arduino15\\packages\\esp8266\\hardware\\esp8266\\3.1.2/tools/sdk/include" "-IC:\\Users\\sasa\\AppData\\Local\\Arduino15\\packages\\esp8266\\hardware\\esp8266\\3.1.2/tools/sdk/lwip2/include" "-IC:\\Users\\sasa\\AppData\\Local\\Arduino15\\packages\\esp8266\\hardware\\esp8266\\3.1.2/tools/sdk/libc/xtensa-lx106-elf/include" "-IC:\\Users\\sasa\\AppData\\Local\\arduino\\sketches\\29B76F2D42BEFEE8020B4D7A1D917A76/core" -c "@C:\\Users\\sasa\\AppData\\Local\\Arduino15\\packages\\esp8266\\hardware\\esp8266\\3.1.2/tools/warnings/default-cppflags" -Os -g -free -fipa-pta -Werror=return-type -mlongcalls -mtext-section-literals -fno-rtti -falign-functions=4 -std=gnu++17 -MMD -ffunction-sections -fdata-sections -fno-exceptions -DMMU_IRAM_SIZE=0x8000 -DMMU_ICACHE_SIZE=0x8000 -DNONOSDK22x_190703=1 -DF_CPU=80000000L -DLWIP_OPEN_SRC -DTCP_MSS=536 -DLWIP_FEATURES=1 -DLWIP_IPV6=0 -DARDUINO=10607 -DARDUINO_ESP8266_WEMOS_D1MINIPRO -DARDUINO_ARCH_ESP8266 "-DARDUINO_BOARD=\"ESP8266_WEMOS_D1MINIPRO\"" "-DARDUINO_BOARD_ID=\"d1_mini_pro\"" -DFLASHMODE_DIO "-IC:\\Users\\sasa\\AppData\\Local\\Arduino15\\packages\\esp8266\\hardware\\esp8266\\3.1.2\\cores\\esp8266" "-IC:\\Users\\sasa\\AppData\\Local\\Arduino15\\packages\\esp8266\\hardware\\esp8266\\3.1.2\\variants\\d1_mini" "-IC:\\Users\\sasa\\AppData\\Local\\Arduino15\\packages\\esp8266\\hardware\\esp8266\\3.1.2\\libraries\\ESP8266WiFi\\src" "-Ic:\\Users\\sasa\\Documents\\Arduino\\libraries\\FLProg_Utilites\\src" "-Ic:\\Users\\sasa\\Documents\\Arduino\\libraries\\RT_HW_BASE\\src" "-IC:\\Users\\sasa\\AppData\\Local\\Arduino15\\packages\\esp8266\\hardware\\esp8266\\3.1.2\\libraries\\Wire" "-IC:\\Users\\sasa\\AppData\\Local\\Arduino15\\packages\\esp8266\\hardware\\esp8266\\3.1.2\\libraries\\SPI" "-IC:\\Users\\sasa\\AppData\\Local\\Arduino15\\packages\\esp8266\\hardware\\esp8266\\3.1.2\\libraries\\SoftwareSerial\\src" "C:\\Users\\sasa\\AppData\\Local\\arduino\\sketches\\29B76F2D42BEFEE8020B4D7A1D917A76\\sketch\\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON.ino.cpp" -o "C:\\Users\\sasa\\AppData\\Local\\arduino\\sketches\\29B76F2D42BEFEE8020B4D7A1D917A76\\sketch\\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON.ino.cpp.o"
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON.ino: In function 'void loop()':
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON.ino:57:5: error: a function-definition is not allowed here before '{' token
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino: In function 'void checkBrightness()':
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:2:18: error: 'PHOTO' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:2:27: error: 'BRIGHT_THRESHOLD' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:3:17: error: 'BACKLIGHT' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:3:28: error: 'LCD_BRIGHT_MIN' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:5:5: error: 'LED_ON' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:5:15: error: 'LED_BRIGHT_MIN' was not declared in this scope; did you mean 'LED_BUILTIN'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:10:17: error: 'BACKLIGHT' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:10:28: error: 'LCD_BRIGHT_MAX' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:12:5: error: 'LED_ON' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:12:15: error: 'LED_BRIGHT_MAX' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:17:7: error: 'dispCO2' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:17:22: error: 'setLED' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:18:28: error: 'setLED' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:19:29: error: 'setLED' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino: In function 'void modesTick()':
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:24:3: error: 'button' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:27:5: error: 'mode' was not declared in this scope; did you mean 'modf'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:37:5: error: 'mode' was not declared in this scope; did you mean 'modf'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:42:9: error: 'mode' was not declared in this scope; did you mean 'modf'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:43:7: error: 'lcd' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:44:7: error: 'loadClock' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:45:17: error: 'hrs' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:45:22: error: 'mins' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:45:7: error: 'drawClock' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:46:11: error: 'DISPLAY_TYPE' was not declared in this scope; did you mean 'IP_ANY_TYPE'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:46:30: error: 'drawData' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:49:7: error: 'lcd' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:50:7: error: 'loadPlot' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino: In function 'void redrawPlot()':
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:57:3: error: 'lcd' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:58:11: error: 'mode' was not declared in this scope; did you mean 'modf'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:60:35: error: 'PRESS_MIN' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:60:46: error: 'PRESS_MAX' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:60:63: error: 'pressHour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:60:13: error: 'drawPlot' was not declared in this scope; did you mean 'redrawPlot'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:62:63: error: 'pressDay' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:64:35: error: 'CO2_MIN' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:64:44: error: 'CO2_MAX' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:64:59: error: 'co2Hour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:66:59: error: 'co2Day' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:68:35: error: 'TEMP_MIN' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:68:45: error: 'TEMP_MAX' was not declared in this scope; did you mean 'MEMP_MAX'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:68:61: error: 'tempHour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:70:61: error: 'tempDay' was not declared in this scope; did you mean 'tempnam'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:72:35: error: 'HUM_MIN' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:72:44: error: 'HUM_MAX' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:72:59: error: 'humHour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:74:59: error: 'humDay' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino: In function 'void readSensors()':
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:81:3: error: 'ccs' was not declared in this scope; did you mean 'cos'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:82:3: error: 'dispTemp' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:82:14: error: 'sen' was not declared in this scope; did you mean 'sin'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:83:3: error: 'dispHum' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:84:3: error: 'bmp' was not declared in this scope; did you mean 'bcmp'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:85:3: error: 'dispPres' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino: In function 'void drawSensors()':
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:119:3: error: 'lcd' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:120:20: error: 'dispTemp' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:123:20: error: 'dispHum' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:131:20: error: 'dispPres' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:132:20: error: 'dispRain' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino: In function 'void plotSensorsTick()':
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:138:7: error: 'hourPlotTimer' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:140:7: error: 'tempHour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:141:7: error: 'humHour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:142:7: error: 'pressHour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:143:7: error: 'co2Hour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:145:5: error: 'tempHour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:145:20: error: 'dispTemp' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:146:5: error: 'humHour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:146:19: error: 'dispHum' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:147:5: error: 'co2Hour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:147:19: error: 'dispCO2' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:149:9: error: 'PRESSURE' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:149:19: error: 'pressHour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:149:35: error: 'dispRain' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:150:10: error: 'pressHour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:150:26: error: 'dispPres' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:154:7: error: 'dayPlotTimer' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:158:19: error: 'tempHour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:159:18: error: 'humHour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:160:20: error: 'pressHour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:161:18: error: 'co2Hour' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:169:7: error: 'tempDay' was not declared in this scope; did you mean 'tempnam'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:170:7: error: 'humDay' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:171:7: error: 'pressDay' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:172:7: error: 'co2Day' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:174:5: error: 'tempDay' was not declared in this scope; did you mean 'tempnam'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:175:5: error: 'humDay' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:176:5: error: 'pressDay' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:177:5: error: 'co2Day' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:181:7: error: 'predictTimer' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:186:20: error: 'bmp' was not declared in this scope; did you mean 'bcmp'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:194:7: error: 'pressure_array' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:197:5: error: 'pressure_array' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:199:5: error: 'sumX' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:200:5: error: 'sumY' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:201:5: error: 'sumX2' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:202:5: error: 'sumXY' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:205:15: error: 'time_array' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:214:5: error: 'a' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:221:5: error: 'delta' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:223:5: error: 'dispRain' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino: In function 'void clockTick()':
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:237:5: error: 'secs' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:240:7: error: 'mins' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:241:25: error: 'mode' was not declared in this scope; did you mean 'modf'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:241:46: error: 'hrs' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:241:36: error: 'drawClock' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:243:9: error: 'mins' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:244:7: error: 'now' was not declared in this scope; did you mean 'pow'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:244:13: error: 'rtc' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:247:7: error: 'hrs' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:248:11: error: 'mode' was not declared in this scope; did you mean 'modf'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:248:22: error: 'drawClock' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:252:11: error: 'mode' was not declared in this scope; did you mean 'modf'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:252:24: error: 'DISPLAY_TYPE' was not declared in this scope; did you mean 'IP_ANY_TYPE'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:252:38: error: 'drawData' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:254:9: error: 'DISP_MODE' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:254:27: error: 'mode' was not declared in this scope; did you mean 'modf'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:255:7: error: 'lcd' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:260:7: error: 'mode' was not declared in this scope; did you mean 'modf'?
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:260:18: error: 'drawdots' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:261:7: error: 'dispCO2' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:262:18: error: 'setLED' was not declared in this scope
H:\RADIO\MK\Proektis\АРХИВ\Proshivki_moy_rabochie_archi_!!!\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\Gotovay_meteoClock_Si_ccs_bmp_i2c_foto_v2.0_ON\functions.ino:263:10: error: 'setLED' was not declared in this scope
Используем библиотеку ESP8266WiFi версии 1.0 из папки: C:\Users\sasa\AppData\Local\Arduino15\packages\esp8266\hardware\esp8266\3.1.2\libraries\ESP8266WiFi 
Используем библиотеку FLProg_Utilites версии 1.0.0 из папки: C:\Users\sasa\Documents\Arduino\libraries\FLProg_Utilites 
Используем библиотеку RT_HW_BASE версии 7.1.1 из папки: C:\Users\sasa\Documents\Arduino\libraries\RT_HW_BASE 
Используем библиотеку Wire версии 1.0 из папки: C:\Users\sasa\AppData\Local\Arduino15\packages\esp8266\hardware\esp8266\3.1.2\libraries\Wire 
Используем библиотеку SPI версии 1.0 из папки: C:\Users\sasa\AppData\Local\Arduino15\packages\esp8266\hardware\esp8266\3.1.2\libraries\SPI 
Используем библиотеку EspSoftwareSerial версии 8.0.1 из папки: C:\Users\sasa\AppData\Local\Arduino15\packages\esp8266\hardware\esp8266\3.1.2\libraries\SoftwareSerial 
exit status 1

Compilation error: a function-definition is not allowed here before '{' token

#### актуальный скетч из flprog для отправки в arduino ide 2.3.7

#include <ESP8266WiFi.h>
#include "flprogUtilites.h"
extern "C" 
{
    #include "user_interface.h"
}
float in_sensor1_float_166482050_1;
float in_sensor2_float_166482050_1;
float in_sensor3_float_166482050_1;
int in_sensor1_int_166482050_1;
int in_sensor2_int_166482050_1;
int in_sensor3_int_166482050_1;
String in_peer_mac_166482050_1;
bool in_trigger_166482050_1;
String out_debug_str_166482050_1;
String out_espnow_string_166482050_1;
// Переменные для хранения предыдущих значений датчиков
float old_f1_166482050_1= 0, old_f2 = 0, old_f3 = 0;
int old_i1_166482050_1= 0, old_i2 = 0, old_i3 = 0;
bool firstRun_166482050_1= true;
// Объявление функции (без тела, только сигнатура)
void sendSensorUpdate_166482050_1(String key, float value);
bool ESPControllerWifiClient_status = 1;
char ESPControllerWifiAP_SSID[40] = "Teplica";
char ESPControllerWifiAP_password[40] = "12345kav";
bool ESPControllerWifiAP_IsNeedReconect = 0;
bool ESPControllerWifiAP_workStatus = 1;
IPAddress ESPControllerWifiAP_ip(192, 168, 1, 1);
IPAddress  ESPControllerWifiAP_dns (192, 168, 1, 1);
IPAddress  ESPControllerWifiAP_gateway (192, 168, 1, 1);
IPAddress ESPControllerWifiAP_subnet (255, 255, 255, 0);
uint8_t ESPControllerWifiAP_mac[6] = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0};
WiFiMode _ESPCurrentWifiMode = WIFI_AP_STA;
void setup()
{
    WiFi.mode(WIFI_AP);
    _esp8266WifiModuleApReconnect();
}
void loop()
{
    if(ESPControllerWifiAP_IsNeedReconect) 
    {
        _esp8266WifiModuleApReconnect();
        ESPControllerWifiAP_IsNeedReconect = 0;
    }
    //Плата:1
    in_sensor1_float_166482050_1 = 0;
    in_sensor2_float_166482050_1 = 0;
    in_sensor3_float_166482050_1 = 0;
    in_sensor1_int_166482050_1 = 0;
    in_sensor2_int_166482050_1 = 0;
    in_sensor3_int_166482050_1 = 0;
    in_peer_mac_166482050_1 = String("");
    in_trigger_166482050_1 = 0;
    // === ОПРЕДЕЛЕНИЕ ФУНКЦИИ ===
    void sendSensorUpdate_166482050_1(String key, float value) 
    {
        out_espnow_string_166482050_1 = key + ":" + String(value, 2);
        out_debug_str_166482050_1 = "Sent -> " + out_espnow_string_166482050_1;
    }
    // === ОСНОВНОЙ КОД ===
// ИНИЦИАЛИЗАЦИЯ ПРИ ПЕРВОМ ЗАПУСКЕ
    if (firstRun_166482050_1) 
    {
        old_f1_166482050_1 = in_sensor1_float_166482050_1;
        old_f2 = in_sensor2_float_166482050_1;
        old_f3 = in_sensor3_float_166482050_1;
        old_i1_166482050_1 = in_sensor1_int_166482050_1;
        old_i2 = in_sensor2_int_166482050_1;
        old_i3 = in_sensor3_int_166482050_1;
        firstRun_166482050_1 = false;
        out_debug_str_166482050_1 = "Coder INIT";
        return;
    }
    // ОСНОВНАЯ ЛОГИКА: ПРОВЕРЯЕМ ИЗМЕНЕНИЯ И ФОРМИРУЕМ СТРОКУ
    if (in_trigger_166482050_1) 
    {
        if (in_sensor1_float_166482050_1 != old_f1_166482050_1) 
        {
            sendSensorUpdate_166482050_1("F1", in_sensor1_float_166482050_1);
            old_f1_166482050_1 = in_sensor1_float_166482050_1;
            return;
        }
        if (in_sensor2_float_166482050_1 != old_f2) 
        {
            sendSensorUpdate_166482050_1("F2", in_sensor2_float_166482050_1);
            old_f2 = in_sensor2_float_166482050_1;
            return;
        }
        if (in_sensor3_float_166482050_1 != old_f3) 
        {
            sendSensorUpdate_166482050_1("F3", in_sensor3_float_166482050_1);
            old_f3 = in_sensor3_float_166482050_1;
            return;
        }
        if (in_sensor1_int_166482050_1 != old_i1_166482050_1) 
        {
            sendSensorUpdate_166482050_1("I1", (float)in_sensor1_int_166482050_1);
            old_i1_166482050_1 = in_sensor1_int_166482050_1;
            return;
        }
        if (in_sensor2_int_166482050_1 != old_i2) 
        {
            sendSensorUpdate_166482050_1("I2", (float)in_sensor2_int_166482050_1);
            old_i2 = in_sensor2_int_166482050_1;
            return;
        }
        if (in_sensor3_int_166482050_1 != old_i3) 
        {
            sendSensorUpdate_166482050_1("I3", (float)in_sensor3_int_166482050_1);
            old_i3 = in_sensor3_int_166482050_1;
            return;
        }
        out_debug_str_166482050_1 = "No changes";
    }
}
int hexStrToInt(String instring)
{
    byte len = instring.length();
    if  (len == 0) return 0;
    int result = 0;
    for (byte i = 0; i < 8; i++)    // только первые 8 цыфар влезуть в uint32
    {
        char ch = instring[i];
        if (ch == 0) break;
        result <<= 4;
        if (isdigit(ch))
        result = result | (ch - '0');
        else result = result | (ch - 'A' + 10);
    }
    return result;
}
void _esp8266WifiModuleApReconnect()
{
    if (_checkMacAddres(ESPControllerWifiAP_mac)) 
    {
         wifi_set_macaddr(1, const_cast<uint8*>(ESPControllerWifiAP_mac));
    }
    WiFi.softAPConfig(ESPControllerWifiAP_ip, ESPControllerWifiAP_gateway, ESPControllerWifiAP_subnet);
    WiFi.softAP(ESPControllerWifiAP_SSID, ESPControllerWifiAP_password);
    if (! (_checkMacAddres(ESPControllerWifiAP_mac))) 
    {
        WiFi.softAPmacAddress(ESPControllerWifiAP_mac);
    }
}
bool _checkMacAddres(byte array[])
{
    bool result = 0;
    for (byte i = 0; i < 6; i++)
    {
        if (array[i] == 255) 
        {
            return 0;
        }
        if (array[i] > 0) 
        {
            result = 1;
        }
    }
    return result;
}
void _parseMacAddressString(String value, byte array[])
{
    int index;
    byte buffer[6] = {255, 255, 255, 255, 255, 255};
    byte raz = 0;
    String tempString;
    while ((value.length() > 0) && (raz <= 6)) 
    {
        index = value.indexOf(":");
        if (index == -1) 
        {
            tempString = value;
            value = "";
        }
         else 
        {
            tempString = value.substring(0, index);
            value = value.substring(index + 1);
        }
        buffer[raz] = byte(hexStrToInt(tempString));
        raz++;
    }
    if (_checkMacAddres(buffer))
    {
        for (byte i = 0; i < 6; i++)
        {
            array[i] = buffer[i];
        }
    }
}
bool _compareMacAddreses(byte array1[], byte array2[])
{
    for (byte i = 0; i < 6; i++)
    {
        if (array1[i] != array2[i]) 
        {
            return 0;
        }
    }
    return 1;
}
bool _compareMacAddresWithString(byte array[], String value)
{
    byte buffer[6] = {255, 255, 255, 255, 255, 255};
    _parseMacAddressString(value,  buffer);
    return _compareMacAddreses(array, buffer);
}
bool _checkMacAddresString(String value)
{
    byte buffer[6] = {255, 255, 255, 255, 255, 255};
    _parseMacAddressString(value,  buffer);
    return _checkMacAddres(buffer);
}
String _macAddresToString(byte array[])
{
    String result = "";
    String  temp ="";
    for (byte i = 0; i < 6; i++)
    {
        temp=String(array[i],HEX);
        if (temp.length()  < 2) 
        {
            temp = String("0") + temp;
        }
        result = result + temp;
        if (i < 5) 
        {
               result = result + String(":");
        }
    }
    result.toUpperCase();
    return result;
}

---------------------
# FLProg 9.6.4
