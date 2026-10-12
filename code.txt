#include <Wire.h>
#include <RTClib.h>
#include <TM1637Display.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// -------- PIN DEFINITIONS --------
#define CLK 2
#define DIO 3
#define LED_PIN 13   // PM indicator LED

// -------- DISPLAY SETUP --------
TM1637Display tmDisplay(CLK, DIO);

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 oled(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// -------- RTC --------
RTC_DS3231 rtc;

// -------- DAY & MONTH NAMES --------
const char* days[] = {
  "Sunday", "Monday", "Tuesday", "Wednesday",
  "Thursday", "Friday", "Saturday"
};

const char* months[] = {
  "Jan", "Feb", "Mar", "Apr", "May", "Jun",
  "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"
};

// -------- BRIGHTNESS SETTINGS --------
#define DAY_TMD_BRIGHTNESS     4
#define NIGHT_TMD_BRIGHTNESS   1

#define DAY_OLED_CONTRAST      0xAF
#define NIGHT_OLED_CONTRAST    0x3F

void setup() {
  pinMode(LED_PIN, OUTPUT);

  Wire.begin();
  rtc.begin();

  // // ⚠️ REMOVE OR COMMENT THIS LINE AFTER ADDING RTC BATTERY
  // rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));

  // TM1637 initial brightness
  tmDisplay.setBrightness(DAY_TMD_BRIGHTNESS);

  // OLED init
  oled.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  oled.clearDisplay();
  oled.display();
}

void loop() {
  DateTime now = rtc.now();

  // -------- NIGHT MODE CHECK --------
  bool nightMode = (now.hour() >= 22 || now.hour() < 6);

  // -------- APPLY BRIGHTNESS --------
  tmDisplay.setBrightness(
    nightMode ? NIGHT_TMD_BRIGHTNESS : DAY_TMD_BRIGHTNESS
  );

  oled.ssd1306_command(SSD1306_SETCONTRAST);
  oled.ssd1306_command(
    nightMode ? NIGHT_OLED_CONTRAST : DAY_OLED_CONTRAST
  );

  // -------- 12-HOUR TIME --------
  int hour24 = now.hour();
  bool isPM = (hour24 >= 12);

  int hour12 = hour24 % 12;
  if (hour12 == 0) hour12 = 12;

  // -------- TM1637 TIME (HH:MM) --------
  int timeValue = hour12 * 100 + now.minute();
  bool colonOn = (now.second() % 2 == 0);

  tmDisplay.showNumberDecEx(
    timeValue,
    colonOn ? 0x40 : 0x00,
    true
  );

  // -------- PM LED --------
  digitalWrite(LED_PIN, isPM ? HIGH : LOW);

  // -------- OLED DISPLAY --------
  oled.clearDisplay();
  oled.setTextColor(SSD1306_WHITE);

  // DAY (BIG & CENTERED)
  oled.setTextSize(2);
  oled.setCursor(10, 0);
  oled.println(days[now.dayOfTheWeek()]);

  // DATE (BIG)
  oled.setTextSize(2);
  oled.setCursor(4, 32);

  if (now.day() < 10) oled.print("0");
  oled.print(now.day());
  oled.print(" ");
  oled.print(months[now.month() - 1]);
  oled.print(" ");
  oled.print(now.year() % 100);

  oled.display();

  delay(200);
}
