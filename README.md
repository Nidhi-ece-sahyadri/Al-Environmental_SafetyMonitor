


# Environmental Safety Monitoring System using ESP32, Raspberry Pi & Firebase

This project presents a real-time Industrial Safety Monitoring System that uses multiple sensors to detect hazardous environmental conditions. It utilizes an ESP32 microcontroller for sensor interfacing, Firebase for cloud storage, a Raspberry Pi for local ML-based decision-making, and alert systems like SMS, voice feedback, and visual indicators.

---

## 🔧 Features

- Monitors Temperature, Humidity, Gas, and Flame
- Real-time data logging to Firebase
- ML model running on Raspberry Pi classifies environment as SAFE or DANGER
- OLED Display for live status
- SMS alert via CircuitDigest API
- Voice alert for danger
- Automatic fan activation for ventilation
- Power/WiFi status indicators

  ---

## 🔧 Components Used

| Component       | Description                                             |
|----------------|---------------------------------------------------------|
| ESP32           | Microcontroller for interfacing with sensors and Firebase |
| Raspberry Pi    | Runs ML model, handles local logging and decision logic |
| DHT11 Sensor    | Measures temperature and humidity                       |
| MQ2 Sensor      | Detects gas/smoke levels                                |
| Flame Sensor    | Detects flame or fire                                   |
| Green LED       | Indicates safe environment                              |
| Red LED         | Indicates dangerous environment                         |
| Relay Module    | Controls fan (acts as exhaust system)                   |
| Fan (Exhaust)   | Activated during high gas levels                        |
| OLED Display    | Displays sensor values and system status                |

---

## ⚙️ How to Run

### 🧠 ESP32 Setup

1. Open `ESP32_Code/esp32_monitoring.ino` in Arduino IDE or PlatformIO.
2. Configure your:
   - Wi-Fi credentials
   - Firebase credentials
3. Connect:
   - DHT11 to GPIO 4
   - MQ2 to GPIO 34 (Analog)
   - Flame sensor to GPIO 14
   - Buzzer, LEDs, Fan, Relay as per diagram
4. Upload the code and open Serial Monitor for logs.

<details>
<summary>📄 <strong>ESP32 Code</strong> (click to expand)</summary>

```cpp
#include <WiFi.h>
#include <FirebaseESP32.h>
#include <DHT.h>

// Firebase Credentials
#define API_KEY "YOUR_FIREBASE_API_KEY"
#define DATABASE_URL "https://your-project-id.firebaseio.com"
#define USER_EMAIL "your-email@example.com"
#define USER_PASSWORD "your-password"

// WiFi Credentials
#define WIFI_SSID "YOUR_WIFI_SSID"
#define WIFI_PASSWORD "YOUR_WIFI_PASSWORD"

// Sensor Pins
#define DHTPIN 4
#define DHTTYPE DHT11
#define FLAME_SENSOR_PIN 14
#define MQ2_SENSOR_PIN 34

// Output Pins
#define BUZZER_PIN 12
#define RED_PIN 25
#define GREEN_PIN 26
#define BLUE_PIN 27
#define RELAY_FAN_PIN 33

DHT dht(DHTPIN, DHTTYPE);
FirebaseData fbdo;
FirebaseAuth firebaseAuth;
FirebaseConfig firebaseConfig;
bool firebaseInitialized = false;

void setup() {
  Serial.begin(115200);
  dht.begin();
  pinMode(FLAME_SENSOR_PIN, INPUT);
  pinMode(MQ2_SENSOR_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RED_PIN, OUTPUT);
  pinMode(GREEN_PIN, OUTPUT);
  pinMode(BLUE_PIN, OUTPUT);
  pinMode(RELAY_FAN_PIN, OUTPUT);

  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(RED_PIN, LOW);
  digitalWrite(GREEN_PIN, LOW);
  digitalWrite(BLUE_PIN, LOW);
  digitalWrite(RELAY_FAN_PIN, HIGH);

  Serial.print("🔌 Connecting to WiFi");
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 20) {
    digitalWrite(BLUE_PIN, HIGH);
    delay(300);
    digitalWrite(BLUE_PIN, LOW);
    delay(300);
    Serial.print(".");
    attempts++;
  }

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\n✅ WiFi Connected");
    digitalWrite(BLUE_PIN, HIGH);
    firebaseConfig.api_key = API_KEY;
    firebaseConfig.database_url = DATABASE_URL;
    firebaseAuth.user.email = USER_EMAIL;
    firebaseAuth.user.password = USER_PASSWORD;
    Firebase.begin(&firebaseConfig, &firebaseAuth);
    Firebase.reconnectWiFi(true);
    firebaseInitialized = true;
  } else {
    Serial.println("\n❌ WiFi Connection Failed");
    digitalWrite(BLUE_PIN, LOW);
    firebaseInitialized = false;
  }
}

void checkWiFiAndFirebase() {
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("🔄 Reconnecting WiFi...");
    WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
    delay(5000);
    if (WiFi.status() == WL_CONNECTED) {
      Serial.println("✅ WiFi Reconnected");
      digitalWrite(BLUE_PIN, HIGH);
      if (!firebaseInitialized) {
        firebaseConfig.api_key = API_KEY;
        firebaseConfig.database_url = DATABASE_URL;
        firebaseAuth.user.email = USER_EMAIL;
        firebaseAuth.user.password = USER_PASSWORD;
        Firebase.begin(&firebaseConfig, &firebaseAuth);
        Firebase.reconnectWiFi(true);
        firebaseInitialized = true;
        Serial.println("✅ Firebase Initialized");
      }
    } else {
      Serial.println("❌ WiFi still not connected");
      digitalWrite(BLUE_PIN, LOW);
      firebaseInitialized = false;
    }
  }
}

void sendToFirebase(float temp, float hum, bool flame, int gas) {
  if (WiFi.status() == WL_CONNECTED) {
    Firebase.setFloat(fbdo, "/SensorData/temperature", temp);
    Firebase.setFloat(fbdo, "/SensorData/humidity", hum);
    Firebase.setBool(fbdo, "/SensorData/flame", flame);
    Firebase.setInt(fbdo, "/SensorData/gas", gas);
    Serial.println("📡 Data sent to Firebase");
  } else {
    Serial.println("❌ WiFi not connected");
  }
}

void alertBlink(int durationMillis) {
  unsigned long start = millis();
  while (millis() - start < durationMillis) {
    digitalWrite(RED_PIN, HIGH);
    digitalWrite(BUZZER_PIN, HIGH);
    delay(250);
    digitalWrite(RED_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
    delay(250);
  }
}

void loop() {
  checkWiFiAndFirebase();
  float temp = dht.readTemperature();
  float hum = dht.readHumidity();
  bool flameDetected = digitalRead(FLAME_SENSOR_PIN) == LOW;
  int gasValue = analogRead(MQ2_SENSOR_PIN);

  if (isnan(temp) || isnan(hum)) {
    Serial.println("❌ DHT Read Failed!");
    delay(5000);
    return;
  }

  Serial.printf("🌡 Temp: %.2f °C\n", temp);
  Serial.printf("💧 Hum: %.2f %%\n", hum);
  Serial.printf("🔥 Flame: %s\n", flameDetected ? "YES" : "NO");
  Serial.printf("🌫 Gas: %d\n", gasValue);

  sendToFirebase(temp, hum, flameDetected, gasValue);

  if (flameDetected || gasValue > 1500) {
    Serial.println("⚠ ALERT: Flame or Gas Detected");
    digitalWrite(RELAY_FAN_PIN, LOW);
    digitalWrite(GREEN_PIN, LOW);
    alertBlink(5000);
  } else {
    Serial.println("✅ Safe");
    digitalWrite(RELAY_FAN_PIN, HIGH);
    digitalWrite(GREEN_PIN, HIGH);
    digitalWrite(BUZZER_PIN, LOW);
    digitalWrite(RED_PIN, LOW);
  }

  delay(2000);
}
```
---
## 📦 Dependencies (Install These on Raspberry Pi)

Run the following commands on your Raspberry Pi terminal:

```bash
sudo apt update
sudo apt install python3-pip espeak -y
```


```bash
pip3 install firebase-admin pandas joblib requests pillow adafruit-circuitpython-ssd1306
```

## 🧠 Methodology — AI-Enabled Danger Detection (Raspberry Pi)

---

### 2.1 Data Acquisition from Firebase

The Raspberry Pi continuously polls the Firebase Realtime Database at regular intervals to retrieve the latest sensor readings uploaded by the ESP32 sender node.

**Retrieved parameters:**

| Parameter | Sensor |
|---|---|
| Temperature (°C) | DHT11 |
| Humidity (%) | DHT11 |
| Gas Level (analog value) | MQ2 Sensor |
| Flame Status (0 / 1) | Flame Sensor |

A Python script handles the Firebase connection using the `firebase-admin` SDK, authenticating via a service account credentials file.

---

### 2.2 Data Preprocessing and Noise Reduction

Raw sensor data from industrial environments often contains noise and transient spikes. A **sliding window averaging technique** is applied to address this:

1. The system maintains a rolling buffer of the **last 5 readings** for each parameter.
2. The **mean value** of each buffer is computed.
3. These averaged values are passed as input to the ML model.

This smoothens momentary false spikes (e.g., a brief gas sensor fluctuation) and ensures decisions are based on stable, representative data.

---

### 2.3 Machine Learning Model — Random Forest Classifier

#### Model Selection

A Random Forest Classifier was chosen for the following reasons:

- Handles non-linear relationships between sensor parameters
- Robust to outliers and noise
- Provides high accuracy with a small training dataset
- Fast inference time, suitable for real-time edge deployment

#### Training

The model was trained on a labelled dataset with the following structure:

| Feature | Description |
|---|---|
| Temperature | Ambient temperature reading (°C) |
| Humidity | Relative humidity (%) |
| Gas Level | Analog sensor output (0–4095) |
| Flame Status | Binary (0 = No flame, 1 = Flame detected) |
| **Label** | **SAFE (0) or DANGER (1)** |

Training data was collected by simulating both safe and hazardous conditions and manually labelling each reading. The trained model was serialized using `pickle` (`.pkl` file) and loaded at runtime on the Raspberry Pi.

#### Prediction

At each inference cycle:

1. Averaged sensor values are formatted as a feature vector.
2. The feature vector is passed to the loaded Random Forest model.
3. The model outputs a binary classification: `SAFE` or `DANGER`.

---

### 2.4 Decision Engine and Alert Triggering

Based on the model's output, the decision engine determines the appropriate response:

| Prediction | Action |
|---|---|
| `SAFE` | Update OLED with current values; no alert triggered |
| `DANGER` | Trigger voice alert + OLED warning + SMS notification |

> A **cooldown timer** is implemented to prevent repeated alerts within a short window, avoiding alert fatigue for workers.

---

### 2.5 Alert Subsystems

#### 🔊 Voice Alert (eSpeak TTS)

The `eSpeak` engine is invoked via a Python subprocess call. The spoken message is dynamically generated based on which parameter crossed the threshold.

```
"Warning! High gas level detected."
"Warning! Flame detected. Evacuate immediately."
```

#### 🖥️ OLED Display (SSD1306)

Driven using the `Adafruit_SSD1306` Python library over I2C. Displays:

- Live temperature, humidity, gas level, and flame status
- Current system state (`SAFE` / `DANGER`) in large text

#### 📱 SMS Notification (CircuitDigest Cloud API)

An HTTP POST request is sent to the CircuitDigest API endpoint with the alert message and recipient number. If internet is unavailable, the alert is queued and retried.

---

### 2.6 Local Data Logging (Offline Reliability)

Every reading — along with the model prediction and timestamp — is appended to a local JSON log file on the Raspberry Pi.

```json
{
  "timestamp": "2025-06-10T14:32:05",
  "temperature": 38.2,
  "humidity": 65.4,
  "gas_level": 1820,
  "flame": 0,
  "prediction": "DANGER"
}
```

This log serves as an audit trail and enables post-incident analysis even when cloud connectivity was unavailable.

---

### 2.7 System Loop

```
START
  |
  ├─ Poll Firebase for latest sensor reading
  ├─ Add to rolling buffer (last 5 readings)
  ├─ Compute average values
  ├─ Pass to Random Forest model
  ├─ Get prediction (SAFE / DANGER)
  |
  ├─ [SAFE]   → Update OLED → Log to JSON → Wait (interval)
  |
  └─ [DANGER] → Voice Alert
               → OLED Warning
               → SMS via CircuitDigest API
               → Log to JSON
               → Cooldown timer
               → Wait (interval)
```
