
#define BLYNK_TEMPLATE_ID "Your Blynk ID"
#define BLYNK_TEMPLATE_NAME "Temperature and Humidity ESP32"
#define BLYNK_PRINT Serial

#include <WiFi.h>
#include <WiFiClient.h>
#include <BlynkSimpleEsp32.h>
#include <DHT.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// =====================================
// BLYNK AND WIFI CONFIGURATION
// =====================================

// Enter your NEW Blynk token and WiFi credentials
char auth[] = "Your Blynk Auth";
char ssid[] = "Your Wifi SSID";
char pass[] = "Your Wifi Password";

// =====================================
// DHT11 CONFIGURATION
// =====================================

#define DHTPIN 4
#define DHTTYPE DHT11

DHT dht(DHTPIN, DHTTYPE);

// =====================================
// COMPONENT PINS
// =====================================

#define BUZZER_PIN 2
#define RED_LED_PIN 19
#define GREEN_LED_PIN 17

// Motor Driver Pins
const int input2 = 12;
const int input1 = 27;
const int enablePin = 14;

// PWM Properties
const int freq = 5000;
const int resolution = 8;

// LCD Configuration
LiquidCrystal_I2C lcd(0x27, 16, 2);

BlynkTimer timer;

// =====================================
// TEMPERATURE THRESHOLD
// =====================================

#define TEMP_HIGH 35.0

// =====================================
// MAXIMUM AND MINIMUM VALUES
// =====================================

float maxTemp = -100.0;
float minTemp = 100.0;

float maxHumid = -1.0;
float minHumid = 101.0;

bool firstReading = true;

// Alarm State
bool buzzerState = false;
bool wasHighTemperature = false;

// =====================================
// SENSOR FUNCTION
// =====================================

void sendSensor() {

  float h = dht.readHumidity();
  float t = dht.readTemperature();

  // =====================================
  // CHECK SENSOR READINGS
  // =====================================

  if (isnan(h) || isnan(t)) {

    Serial.println("ERROR: Failed to read DHT11!");

    lcd.setCursor(0, 0);
    lcd.print("Sensor Error!   ");

    return;
  }

  // =====================================
  // UPDATE MAXIMUM AND MINIMUM VALUES
  // =====================================

  if (firstReading) {

    maxTemp = t;
    minTemp = t;

    maxHumid = h;
    minHumid = h;

    firstReading = false;
  }

  else {

    // Temperature tracking
    if (t > maxTemp) {
      maxTemp = t;
    }

    if (t < minTemp) {
      minTemp = t;
    }

    // Humidity tracking
    if (h > maxHumid) {
      maxHumid = h;
    }

    if (h < minHumid) {
      minHumid = h;
    }
  }

  // =====================================
  // SEND DATA TO BLYNK
  // =====================================

  if (Blynk.connected()) {

    Blynk.virtualWrite(V0, t);
    Blynk.virtualWrite(V1, h);

    Blynk.virtualWrite(V2, maxTemp);
    Blynk.virtualWrite(V3, minTemp);

    Blynk.virtualWrite(V4, maxHumid);
    Blynk.virtualWrite(V5, minHumid);

    // =====================================
    // DEBUG BLYNK CHART DATA
    // =====================================

    Serial.println();
    Serial.println("===== BLYNK CHART DATA =====");

    Serial.print("V0 Temperature: ");
    Serial.print(t, 1);
    Serial.println(" C");

    Serial.print("V1 Humidity: ");
    Serial.print(h, 1);
    Serial.println(" %");

    Serial.print("V2 Max Temp: ");
    Serial.print(maxTemp, 1);
    Serial.println(" C");

    Serial.print("V3 Min Temp: ");
    Serial.print(minTemp, 1);
    Serial.println(" C");

    Serial.print("V4 Max Humid: ");
    Serial.print(maxHumid, 1);
    Serial.println(" %");

    Serial.print("V5 Min Humid: ");
    Serial.print(minHumid, 1);
    Serial.println(" %");

    Serial.println("============================");
    Serial.println("Blynk data successfully sent!");

  }

  else {

    Serial.println("WARNING: Blynk is disconnected!");

  }

  // =====================================
  // TEMPERATURE AND FAN CONTROL
  // =====================================

  if (t >= TEMP_HIGH) {

    buzzerState = true;
    wasHighTemperature = true;

    digitalWrite(BUZZER_PIN, HIGH);
    digitalWrite(RED_LED_PIN, HIGH);
    digitalWrite(GREEN_LED_PIN, LOW);

    // Turn ON Exhaust Fan
    digitalWrite(input1, HIGH);
    digitalWrite(input2, LOW);

    // 100% Fan Speed
    ledcWrite(enablePin, 255);

    Serial.println("WARNING! HIGH TEMPERATURE!");
    Serial.println("EXHAUST FAN: ON");

  }

  else {

    buzzerState = false;

    digitalWrite(BUZZER_PIN, LOW);
    digitalWrite(RED_LED_PIN, LOW);
    digitalWrite(GREEN_LED_PIN, HIGH);

    // Turn OFF Exhaust Fan
    ledcWrite(enablePin, 0);

    digitalWrite(input1, LOW);
    digitalWrite(input2, LOW);

    // =====================================
    // NORMAL TEMPERATURE NOTIFICATION
    // =====================================

    if (wasHighTemperature) {

      if (Blynk.connected()) {

        Blynk.logEvent(
          "normal_temp",
          String("Temperature is back to normal! Current temperature: ")
          + String(t, 1) + " C"
        );

        Serial.println("NORMAL TEMPERATURE NOTIFICATION SENT!");

      }

      wasHighTemperature = false;
    }

    Serial.println("NORMAL TEMPERATURE!");
    Serial.println("EXHAUST FAN: OFF");

  }

  // =====================================
  // UPDATE LCD DISPLAY
  // =====================================

  lcd.setCursor(0, 0);
  lcd.print("                ");

  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(t, 1);
  lcd.print((char)223);
  lcd.print("C");

  lcd.setCursor(0, 1);
  lcd.print("Humidity: ");
  lcd.print(h, 1);
  lcd.print("%   ");

  // =====================================
  // SERIAL MONITOR STATUS
  // =====================================

  Serial.println();
  Serial.println("======= SENSOR STATUS =======");

  Serial.print("Temperature: ");
  Serial.print(t, 1);
  Serial.println(" C");

  Serial.print("Humidity: ");
  Serial.print(h, 1);
  Serial.println(" %");

  Serial.println("--- TEMPERATURE RECORDS ---");

  Serial.print("Maximum Temperature: ");
  Serial.println(maxTemp, 1);

  Serial.print("Minimum Temperature: ");
  Serial.println(minTemp, 1);

  Serial.println("--- HUMIDITY RECORDS ---");

  Serial.print("Maximum Humidity: ");
  Serial.println(maxHumid, 1);

  Serial.print("Minimum Humidity: ");
  Serial.println(minHumid, 1);

  Serial.println("--- COMPONENT STATUS ---");

  Serial.print("Green LED: ");
  Serial.println(t < TEMP_HIGH ? "ON" : "OFF");

  Serial.print("Red LED: ");
  Serial.println(t >= TEMP_HIGH ? "ON" : "OFF");

  Serial.print("Exhaust Fan: ");
  Serial.println(t >= TEMP_HIGH ? "ON" : "OFF");

  Serial.print("Buzzer: ");
  Serial.println(buzzerState ? "ON" : "OFF");

  Serial.println("=============================");
  Serial.println();

}

// =====================================
// SETUP
// =====================================

void setup() {

  Serial.begin(115200);
  delay(1000);

  Serial.println();
  Serial.println("================================");
  Serial.println("TEMPERATURE AND HUMIDITY SYSTEM");
  Serial.println("ESP32 INITIALIZATION");
  Serial.println("================================");

  // Initialize Buzzer
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);

  // Initialize LEDs
  pinMode(RED_LED_PIN, OUTPUT);
  pinMode(GREEN_LED_PIN, OUTPUT);

  digitalWrite(RED_LED_PIN, LOW);
  digitalWrite(GREEN_LED_PIN, HIGH);

  // Initialize Motor Pins
  pinMode(input1, OUTPUT);
  pinMode(input2, OUTPUT);
  pinMode(enablePin, OUTPUT);

  // Configure ESP32 PWM (Arduino Core 3.x)
  ledcAttach(enablePin, freq, resolution);

  // Initialize LCD
  Wire.begin(21, 23);

  lcd.init();
  lcd.backlight();

  lcd.setCursor(0, 0);
  lcd.print("Connecting WiFi");

  // Initialize DHT11
  dht.begin();

  Serial.println("DHT11 Initialized!");
  Serial.println("LCD Initialized!");
  Serial.println("Motor Driver Initialized!");

  // =====================================
  // CONNECT TO BLYNK AND WIFI
  // =====================================

  Serial.println("Connecting to Blynk...");

  Blynk.begin(auth, ssid, pass);

  Serial.println("Blynk Connected Successfully!");

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Connected!");

  delay(1500);
  lcd.clear();

  // =====================================
  // INITIAL SENSOR READING
  // =====================================

  sendSensor();

  // Read Sensor Every 2 Seconds
  timer.setInterval(2000L, sendSensor);

  Serial.println("System Ready!");
  Serial.println("Sensor Interval: 2 Seconds");

}

// =====================================
// LOOP
// =====================================

void loop() {

  Blynk.run();

  timer.run();

}
