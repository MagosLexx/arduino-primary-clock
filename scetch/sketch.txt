/*
 * Скетч для управления часовым механизмом через RTC DS3231
 * Лицензия: MIT
 * Автор: MagosLexx, 2025
 * GitHub: https://github.com/MagosLexx/arduino-primary-clock
 */

// 1. Подключение необходимых библиотек
#include <Wire.h>      // Для работы с I2C-интерфейсом
#include <RTClib.h>    // Для работы с модулем реального времени

// 2. Конфигурация аппаратных пинов (тип uint8_t - оптимальный для номеров пинов)
const uint8_t IN1 = 5;      // Пин управления первой катушкой (MOSFET/драйвер)
const uint8_t IN2 = 6;      // Пин управления второй катушкой (MOSFET/драйвер)
const uint8_t LED_PIN = 4;  // Пин для светодиодной индикации (D4)

// 3. Параметры временных интервалов (тип unsigned long для точного учёта времени)
const unsigned long PULSE_DURATION = 1000UL;    // Длительность импульса в миллисекундах
const unsigned long DEBOUNCE_DELAY = 100UL;    // Защитный интервал после импульса
const unsigned long TIME_CHECK_INTERVAL = 500UL;// Проверка времени каждые 500 мс
const unsigned long LED_DURATION = 1000UL;     // Длительность свечения светодиода (1 сек)

// 4. Глобальные переменные состояния
RTC_DS3231 rtc;           // Объект для работы с RTC-модулем
bool pulsePolarity = true; // Флаг текущей полярности импульса
uint8_t lastMinute = 255;  // Хранит последнюю обработанную минуту (255 - начальное невалидное значение)
unsigned long lastTimeCheck = 0; // Время последней проверки
unsigned long ledStartTime = 0;  // Время включения светодиода
bool ledActive = false;          // Флаг активности светодиода

void setup() {
  // 5. Инициализация цифровых выходов
  pinMode(IN1, OUTPUT);    // Настройка IN1 как выхода
  pinMode(IN2, OUTPUT);    // Настройка IN2 как выхода
  pinMode(LED_PIN, OUTPUT);// Настройка светодиода как выхода
  digitalWrite(IN1, LOW);  // Гарантированное выключение при старте
  digitalWrite(IN2, LOW);  // Гарантированное выключение при старте
  digitalWrite(LED_PIN, LOW); // Выключение светодиода

  // 6. Инициализация I2C-шины с дополнительными параметрами
  Wire.begin();            // Старт I2C-интерфейса
  Wire.setTimeout(100);    // Установка таймаута 100 мс для операций

  // 7. Инициализация модуля реального времени
  if (!rtc.begin()) {      // Проверка подключения DS3231
    // Критическая ошибка - модуль не обнаружен
    while (true) {         // Бесконечный цикл
      // Аварийная индикация светодиодом
      digitalWrite(LED_PIN, HIGH);
      delay(200);
      digitalWrite(LED_PIN, LOW);
      delay(200);
    }
  }

  // 8. Коррекция времени при необходимости
  if (rtc.lostPower() || rtc.now().year() < 2020) {
    rtc.adjust(DateTime(2000, 1, 1, 0, 0, 0)); // Установка "эталонного" времени
  }
}

void loop() {
  // 9. Управление светодиодом (выключение по истечении времени)
  if (ledActive && (millis() - ledStartTime >= LED_DURATION)) {
    digitalWrite(LED_PIN, LOW);
    ledActive = false;
  }

  // 10. Проверяем время только если прошло более 500 мс с последней проверки
  if (millis() - lastTimeCheck >= TIME_CHECK_INTERVAL) {
    lastTimeCheck = millis();
    
    // 11. Получение текущего времени от RTC
    DateTime now = rtc.now(); // Чтение временной метки
    
    // 12. Условие срабатывания (с двойной проверкой):
    // - Текущая минута ≠ последней обработанной
    // - Секунды = 0 (начало минуты)
    if (now.minute() != lastMinute && now.second() == 0) {
      lastMinute = now.minute(); // Фиксация обработанной минуты
      
      // 13. Активация светодиода
      digitalWrite(LED_PIN, HIGH);
      ledStartTime = millis();
      ledActive = true;
      
      // 14. Подача управляющего импульса
      digitalWrite(IN1, pulsePolarity ? HIGH : LOW);
      digitalWrite(IN2, pulsePolarity ? LOW : HIGH);
      delay(PULSE_DURATION);
      
      // 15. Гарантированное выключение катушек
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, LOW);
      
      // 16. Смена полярности с защитой от прерываний
      noInterrupts();
      pulsePolarity = !pulsePolarity;
      interrupts();
      
      // 17. Точная выдержка защитного интервала
      unsigned long start = millis();
      while (millis() - start < DEBOUNCE_DELAY) {}
    }
  }
  
  // 18. Фоновая пауза для снижения нагрузки
  delay(10);
}
