#define BLYNK_TEMPLATE_ID    "TMPL6ENOplWmX"
#define BLYNK_TEMPLATE_NAME  "blynk"
#define BLYNK_AUTH_TOKEN     "UE_u1do3NV627a7D7gmhEnOWg1b2ls0I"

#define BLYNK_PRINT Serial
#include <WiFi.h>
#include <WiFiClient.h>
#include <BlynkSimpleEsp32.h>
#include <ESP32Servo.h>
#include <DHT.h>

// WiFi & Blynk
char auth[] = BLYNK_AUTH_TOKEN;
char ssid[] = "Poco M6 Pro";
char pass[] = "bantalguling";

// Pin Setup
#define SERVO_PIN    13
#define LED_PIN      4
#define BUZZER_PIN   5
#define FAN_PIN      15     // <--- Tambahan untuk kipas
#define DHTPIN       21
#define DHTTYPE      DHT22
#define TRIG_PIN     12
#define ECHO_PIN     14

DHT dht(DHTPIN, DHTTYPE);
Servo doorServo;
BlynkTimer timer;

// Pengaturan
#define HAND_NEAR_DISTANCE_CM 5
#define SERVO_OPEN_ANGLE      45
#define SERVO_CLOSED_ANGLE    0
#define AUTO_CLOSE_DELAY_MS   3000
#define BUZZER_TEMP_THRESHOLD 40.0

bool doorOpen = false;
bool autoMode = true;
bool buzzerOn = false;
unsigned long lastDetected = 0;

void setup() {
  Serial.begin(115200);
  Blynk.begin(auth, ssid, pass);

  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(FAN_PIN, OUTPUT);           // <--- Inisialisasi kipas
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  digitalWrite(LED_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(FAN_PIN, LOW);         // <--- Pastikan kipas mati di awal

  doorServo.attach(SERVO_PIN);
  doorServo.write(SERVO_CLOSED_ANGLE);
  doorOpen = false;

  dht.begin();

  timer.setInterval(3000L, sendSensor);
  timer.setInterval(3000L, checkUltrasonic);

  Serial.println("🔧 Setup selesai. Servo awal tertutup. Memulai sistem...");
}

BLYNK_CONNECTED() {
  Blynk.syncVirtual(V0, V1, V3, V4); // <--- Sinkronisasi V4 juga
}

// 🌡️ Baca sensor suhu & kontrol buzzer otomatis
void sendSensor() {
  float h = dht.readHumidity();
  float t = dht.readTemperature();

  if (isnan(h) || isnan(t)) {
    Serial.println("[DHT] ❌ Gagal membaca sensor!");
    return;
  }

  Serial.printf("[DHT] Suhu: %.2f °C | Kelembaban: %.2f %%\n", t, h);
  Blynk.virtualWrite(V2, t);

  if (t >= BUZZER_TEMP_THRESHOLD && !buzzerOn) {
    digitalWrite(BUZZER_PIN, HIGH);
    buzzerOn = true;
    Serial.println("[BUZZER] 🔔 ON (Suhu tinggi!)");
  } else if (t < BUZZER_TEMP_THRESHOLD && buzzerOn) {
    digitalWrite(BUZZER_PIN, LOW);
    buzzerOn = false;
    Serial.println("[BUZZER] 🔕 OFF (Suhu normal)");
  }
}

// 📏 Baca jarak tangan & kontrol servo otomatis
void checkUltrasonic() {
  float distance = readDistanceCM();
  if (distance > 0) {
    Serial.printf("[Ultrasonik] Jarak: %.2f cm\n", distance);
  } else {
    Serial.println("[Ultrasonik] ❌ Gagal membaca jarak!");
  }

  if (autoMode) {
    if (distance > 0 && distance < HAND_NEAR_DISTANCE_CM) {
      lastDetected = millis();
      if (!doorOpen) {
        doorServo.write(SERVO_OPEN_ANGLE);
        doorOpen = true;
        Serial.println("[SERVO] 🚪 Buka (Tangan < 5cm)");
        Blynk.virtualWrite(V1, 1);
      }
    } else if (doorOpen && millis() - lastDetected > AUTO_CLOSE_DELAY_MS) {
      doorServo.write(SERVO_CLOSED_ANGLE);
      doorOpen = false;
      Serial.println("[SERVO] 🔒 Tutup (Tangan menjauh)");
      Blynk.virtualWrite(V1, 0);
    }
  }
}

float readDistanceCM() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000);
  if (duration == 0) return -1;

  float distance = duration * 0.0343 / 2;
  if (distance <= 0 || distance > 200) return -1;

  return distance;
}

// 🚪 Manual buka/tutup servo dari Blynk (V1)
BLYNK_WRITE(V1) {
  int val = param.asInt();
  autoMode = false;
  doorServo.write(val ? SERVO_OPEN_ANGLE : SERVO_CLOSED_ANGLE);
  doorOpen = val;
  Serial.printf("[MANUAL] Servo %s\n", val ? "BUKA" : "TUTUP");

  timer.setTimeout(10000L, []() {
    autoMode = true;
    Serial.println("[AUTO] Mode otomatis aktif kembali");
  });
}

// 💡 Kontrol LED dari Blynk (V0)
BLYNK_WRITE(V0) {
  int val = param.asInt();
  digitalWrite(LED_PIN, val);
  Serial.printf("[LED] %s\n", val ? "ON" : "OFF");
}

// 🔊 Manual kontrol buzzer dari Blynk (V3)
BLYNK_WRITE(V3) {
  int val = param.asInt();
  digitalWrite(BUZZER_PIN, val);
  buzzerOn = val;
  Serial.printf("[BUZZER - MANUAL] %s\n", val ? "ON" : "OFF");
}

// 🌬️ Kontrol kipas dari Blynk (V4)
BLYNK_WRITE(V4) {
  int val = param.asInt();
  digitalWrite(FAN_PIN, val);
  Serial.printf("[FAN] %s\n", val ? "ON" : "OFF");
}

void loop() {
  Blynk.run();
  timer.run();
}
