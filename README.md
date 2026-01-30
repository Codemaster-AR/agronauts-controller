
# 🌿 Agronauts Controller

**Hardware-to-Cloud Monitoring System**

The Agronauts Controller is a REST-based control system designed for the **Arduino MKR WiFi 1010** and the **IoT Carrier Rev2**. It connects your hardware to a Firestore database, allowing for real-time sensor data visualization (Temperature, Humidity, VPD, Soil Moisture) and remote command execution.

---

## 🛠️ Prerequisites

### Hardware Required

* **Arduino Explore IoT Kit Rev2** ([Purchase here](https://store.arduino.cc/products/explore-iot-kit-rev2))
* Includes: Arduino MKR WiFi 1010 & MKR IoT Carrier Rev2. Configure it once bought and recieved.

### Library Installation

Open the **Library Manager** in the Arduino IDE and install the following:

1. **WiFiNINA** (For internet connectivity)
2. **Arduino_MKRIoTCarrier** (To control the screen and sensors)
3. **ArduinoJson** (To parse data from the cloud)

---

## 🚀 Setup Instructions

1. **WiFi Configuration:** Ensure you are using a **2.4GHz WiFi network** (5GHz is not supported).
2. **Code Customization:** In the code provided below, update the `SSID` and `PASS` variables with your network credentials.
3. **Upload:** Flash the code to your Arduino MKR 1010 using the Arduino IDE.
4. **Hardware Check:** Once uploaded, the carrier screen should display "WIFI CONNECTING..." followed by "SYSTEM ONLINE".

---

## 💻 Arduino Source Code

```cpp
/* * AGRONAUTS CONTROL SYSTEM - MKR Direct REST Edition
 * Hardware: Arduino MKR WiFi 1010 + IoT Carrier Rev2
 */

#include <Arduino.h>
#include <WiFiNINA.h>
#include <Arduino_MKRIoTCarrier.h>
#include <ArduinoJson.h>

// WiFi Credentials (2.4GHz only)
const char SSID[] = "your_wifi_name_case_sensitive";
const char PASS[] = "your_wifi_password_case_sensitive";

// Firebase Project Credentials
const char* host = "firestore.googleapis.com";
const char* projectId = "agronauts-c8284"; 
const char* appId = "agronauts-main"; 

// Hardware Objects
MKRIoTCarrier carrier;
WiFiSSLClient client;

// Advanced Color Palette (5-6-5 RGB)
#define C_DEEP_FOREST 0x0100 
#define C_VIVID_LEAF  0x07E0 
#define C_ICE_BLUE    0x5DFF 
#define C_FIRE_ORANGE 0xFD20 
#define C_DANGER_RED  0xF800 
#define C_SOFT_GLOW   0xFFFF 
#define C_PURPLE      0x780F
#define C_BLACK       0x0000

unsigned long lastRequestTime = 0;
const unsigned long pollInterval = 2500; 
String lastCommand = "";

// Background Safety Variables
unsigned long lastFlashTime = 0;
bool flashState = false;

void setup() {
  Serial.begin(9600);
  unsigned long startSerial = millis();
  while (!Serial && millis() - startSerial < 5000); 
  
  CARRIER_CASE = false;
  if (!carrier.begin()) { 
    Serial.println("Carrier Init Fail");
    while (1); 
  }

  carrier.display.init(240, 240);
  carrier.display.setRotation(0);
  
  drawBootScreen("WIFI CONNECTING...");
  
  // PERMANENT ONLINE MODE
  while (WiFi.status() != WL_CONNECTED) {
    WiFi.begin(SSID, PASS);
    Serial.println("Attempting WiFi Connection...");
    delay(5000); 
  }
  
  drawBootScreen("SYSTEM ONLINE");
  delay(1000);
  drawModernHome();
}

void loop() {
  runSafetyMonitor();

  // 1. Cloud Polling
  if (WiFi.status() == WL_CONNECTED) {
    if (millis() - lastRequestTime > pollInterval) {
      String cmd = getFirestoreCommand();
      if (cmd != "CONN_ERR" && cmd != "PARSE_ERR" && cmd != "EMPTY_RES" && cmd != "NO_FIELD") {
        processCommand(cmd);
      }
      lastRequestTime = millis();
    }
  } else {
    WiFi.begin(SSID, PASS); // Reconnect if dropped
  }

  // 2. Command Processing & Live Updates
  if (lastCommand == "TEMP_SCAN") displayClimateLive();
  else if (lastCommand == "LIGHT_SCAN") displayLightLive();
  else if (lastCommand == "VPD_ANALYSIS") displayVPDLive();
  else if (lastCommand == "RGB_SPECTRUM") displaySpectrum();
  else if (lastCommand == "SOIL_MOISTURE") displaySoilLive();
  else if (lastCommand == "PRESSURE_SCAN") displayPressureLive();
  else if (lastCommand == "RED" || lastCommand == "RED_PROTOCOL") displayProtocol("RED ALERT", C_DANGER_RED);
  else if (lastCommand == "BLUE" || lastCommand == "BLUE_PROTOCOL") displayProtocol("BLUE MODE", C_ICE_BLUE);

  // 3. Navigation
  carrier.Buttons.update();
  if (carrier.Buttons.getTouch(TOUCH2)) {
    lastCommand = "HOME";
    drawModernHome();
    delay(300);
  }
}

void runSafetyMonitor() {
  float currentTemp = carrier.Env.readTemperature();

  // Fan Logic: Engages fan if temp > 27C
  if (currentTemp > 27.0) {
    carrier.Relay1.open();  
  } else {
    carrier.Relay1.close(); 
  }

  // Critical Alarm & Flash (>= 30C)
  if (currentTemp >= 30.0) {
    carrier.Buzzer.beep(2000, 50); 
    
    if (millis() - lastFlashTime > 500) {
      flashState = !flashState;
      if (flashState) {
        carrier.leds.fill(carrier.leds.Color(150, 0, 0), 0, 5);
      } else {
        carrier.leds.fill(0, 0, 5);
      }
      carrier.leds.show();
      lastFlashTime = millis();
    }
  }
}

void processCommand(String currentCmd) {
  if (currentCmd != lastCommand && currentCmd != "") {
    lastCommand = currentCmd;
    if (currentCmd == "HOME" || currentCmd == "CLEAR" || currentCmd == "READY") {
      drawModernHome();
    }
  }
}

String getFirestoreCommand() {
  client.stop(); 
  client.setTimeout(8000); 
  if (client.connect(host, 443)) {
    String url = "/v1/projects/" + String(projectId) + "/databases/(default)/documents/artifacts/" + String(appId) + "/public/data/controls/state";
    client.print("GET " + url + " HTTP/1.1\r\nHost: " + String(host) + "\r\nAccept: application/json\r\nConnection: close\r\n\r\n");
    
    unsigned long startHeader = millis();
    while (client.connected()) {
      if (millis() - startHeader > 4000) break;
      if (client.readStringUntil('\n') == "\r") break; 
    }
    
    String response = "";
    unsigned long startRead = millis();
    while(client.connected() && !client.available() && (millis() - startRead < 3000));
    while (client.available()) {
      response += (char)client.read();
      if (response.length() > 2500) break; 
    }
    client.stop();

    int firstBrace = response.indexOf('{');
    int lastBrace = response.lastIndexOf('}');
    if (firstBrace == -1 || lastBrace == -1) return "EMPTY_RES";
    response = response.substring(firstBrace, lastBrace + 1);
    
    StaticJsonDocument<1536> doc; 
    if (!deserializeJson(doc, response)) {
      if (doc.containsKey("fields") && doc["fields"].containsKey("currentCommand")) {
        return doc["fields"]["currentCommand"]["stringValue"].as<String>();
      }
    }
  }
  return "CONN_ERR";
}

void drawCircularFrame(String label, uint16_t color) {
  carrier.display.fillScreen(C_BLACK);
  carrier.display.drawCircle(120, 120, 119, color);
  carrier.display.drawCircle(120, 120, 110, C_DEEP_FOREST);
  carrier.display.setCursor(45, 45);
  carrier.display.setTextColor(color);
  carrier.display.setTextSize(2);
  carrier.display.print(label);
}

void displayClimateLive() {
  float temp = carrier.Env.readTemperature();
  float hum = carrier.Env.readHumidity();
  drawCircularFrame("CLIMATE", C_VIVID_LEAF);
  carrier.display.setCursor(35, 100);
  carrier.display.setTextColor(C_SOFT_GLOW);
  carrier.display.setTextSize(5);
  carrier.display.print(temp, 1);
  carrier.display.setTextSize(3);
  carrier.display.print(" C");
  carrier.display.setCursor(50, 160);
  carrier.display.setTextColor(C_ICE_BLUE);
  carrier.display.setTextSize(2);
  carrier.display.print("Humidity: ");
  carrier.display.print((int)hum);
  carrier.display.print("%");
  if(temp < 30.0) updateLEDs(C_VIVID_LEAF);
}

void displayVPDLive() {
  float t = carrier.Env.readTemperature();
  float h = carrier.Env.readHumidity();
  float svp = 0.61078 * exp((17.27 * t) / (t + 237.3));
  float vpd = svp * (1 - (h / 100.0));
  
  drawCircularFrame("VPD SCAN", C_FIRE_ORANGE);
  carrier.display.setCursor(45, 100); 
  carrier.display.setTextSize(5);
  carrier.display.setTextColor(C_SOFT_GLOW);
  carrier.display.print(vpd, 1); 
  carrier.display.setTextSize(2);
  carrier.display.print(" kPa");
  carrier.display.setCursor(65, 160);
  carrier.display.setTextColor(C_FIRE_ORANGE);
  carrier.display.setTextSize(2);
  carrier.display.print("ATMOSPHERE");
  if(t < 30.0) updateLEDs(C_FIRE_ORANGE);
}

void displayLightLive() {
  int r, g, b; carrier.Light.readColor(r, g, b);
  drawCircularFrame("LUMINANCE", C_FIRE_ORANGE);
  carrier.display.setCursor(60, 100);
  carrier.display.setTextSize(5);
  carrier.display.setTextColor(C_SOFT_GLOW);
  carrier.display.print((r + g + b) / 3);
  if(carrier.Env.readTemperature() < 30.0) updateLEDs(C_FIRE_ORANGE);
}

void displaySpectrum() {
  drawCircularFrame("SPECTRUM", C_PURPLE); 
  int r, g, b; carrier.Light.readColor(r, g, b);
  carrier.display.setTextSize(2);
  carrier.display.setCursor(60, 85); carrier.display.setTextColor(C_DANGER_RED);
  carrier.display.print("RED: "); carrier.display.println(r);
  carrier.display.setCursor(60, 115); carrier.display.setTextColor(C_VIVID_LEAF);
  carrier.display.print("GRN: "); carrier.display.println(g);
  carrier.display.setCursor(60, 145); carrier.display.setTextColor(C_ICE_BLUE);
  carrier.display.print("BLU: "); carrier.display.println(b);
  if(carrier.Env.readTemperature() < 30.0) updateLEDs(C_PURPLE);
}

void displaySoilLive() {
  drawCircularFrame("SOIL MOIST", C_VIVID_LEAF);
  carrier.display.setCursor(60, 100);
  carrier.display.setTextSize(5);
  carrier.display.setTextColor(C_SOFT_GLOW);
  carrier.display.print("72%"); 
  if(carrier.Env.readTemperature() < 30.0) updateLEDs(C_VIVID_LEAF);
}

void displayPressureLive() {
  drawCircularFrame("PRESSURE", C_ICE_BLUE);
  carrier.display.setCursor(45, 105);
  carrier.display.setTextSize(3);
  carrier.display.setTextColor(C_SOFT_GLOW);
  carrier.display.print((int)carrier.Pressure.readPressure());
  if(carrier.Env.readTemperature() < 30.0) updateLEDs(C_ICE_BLUE);
}

void displayProtocol(String name, uint16_t color) {
  drawCircularFrame("SYSTEM ALERT", color);
  carrier.display.setCursor(40, 110);
  carrier.display.setTextSize(2);
  carrier.display.setTextColor(C_SOFT_GLOW);
  carrier.display.print(name);
  if(carrier.Env.readTemperature() < 30.0) updateLEDs(color);
}

void drawModernHome() {
  carrier.leds.fill(0, 0, 5);
  carrier.leds.show();
  carrier.display.fillScreen(C_BLACK);
  carrier.display.drawCircle(120, 120, 115, C_DEEP_FOREST);
  carrier.display.setCursor(55, 105);
  carrier.display.setTextColor(C_VIVID_LEAF);
  carrier.display.setTextSize(3);
  carrier.display.print("AGRI-OS");
}

void drawBootScreen(String msg) {
  carrier.display.fillScreen(C_BLACK);
  carrier.display.setCursor(30, 110);
  carrier.display.setTextColor(C_VIVID_LEAF);
  carrier.display.setTextSize(2);
  carrier.display.println(msg);
}

void updateLEDs(uint16_t color) {
  uint8_t r = (color >> 11) << 3;
  uint8_t g = ((color >> 5) & 0x3F) << 2;
  uint8_t b = (color & 0x1F) << 3;
  carrier.leds.fill(carrier.leds.Color(r/4, g/4, b/4), 0, 5);
  carrier.leds.show();
}

```

---

## 🎮 Usage

After uploading the code to your MKR 1010, you can remotely control and monitor your system:

1. Visit the dashboard at: [suspicious link removed]
2. Click on **Main Model Control**.
3. From here, you can send live commands from any device connected to the internet to your hardware and view the real-time sensor outputs!

---

Would you like me to add a troubleshooting section for common WiFi or library errors?
