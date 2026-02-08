

# IoT Environment Monitor: MKR 1010 + IoT Carrier (Rev2) through Thingspeak

This project uses the **Arduino MKR WiFi 1010** mounted on the **MKR IoT Carrier (Rev2)** to stream live environmental data (Temperature, Humidity, and Pressure) to **ThingSpeak**.

## Link to thingspeak:
[https://thingspeak.mathworks.com/](https://thingspeak.mathworks.com/)

## 📦 Required Hardware

* **Arduino MKR WiFi 1010**
* **Arduino MKR IoT Carrier (Rev2)** (from the Oplà IoT Kit)
* Micro-USB cable

## 🔧 Library Installation

Open the Arduino IDE Library Manager (**Ctrl+Shift+I**) and install:

1. `WiFiNINA` – For WiFi connectivity.
2. `Arduino_MKRIoTCarrier` – For accessing the sensors and display.
3. `ThingSpeak` – For sending data to the cloud.

## 📡 ThingSpeak Configuration

1. Log in to [ThingSpeak](https://thingspeak.com/).
2. Create a **New Channel**.
3. Enable 3 Fields:
* **Field 1:** Temperature (°C)
* **Field 2:** Humidity (%)
* **Field 3:** Pressure (kPa)


4. Go to the **API Keys** tab and copy your **Write API Key**.

## 💻 The Code

### `secrets.h`

Keep your credentials in a separate tab to prevent accidental sharing:

```cpp
#define SECRET_SSID "Your_WiFi_Name"
#define SECRET_PASS "Your_WiFi_Password"
#define SECRET_CH_ID 0000000               // Your Channel ID
#define SECRET_WRITE_APIKEY "XYZ123..."     // Your Write API Key

```

### `Main_Sketch.ino`

This script reads the Rev2 carrier sensors and updates ThingSpeak every 20 seconds.

```cpp
#include <Arduino_MKRIoTCarrier.h>
#include <WiFiNINA.h>
#include <ThingSpeak.h>
#include "secrets.h"

MKRIoTCarrier carrier;
WiFiClient client;

void setup() {
  Serial.begin(9600);
  
  // Initialize Carrier (Rev2)
  CARRIER_CASE = false; // Set to true if using the plastic enclosure
  if (!carrier.begin()) {
    Serial.println("Carrier not detected. Check mounting!");
    while (1);
  }

  ThingSpeak.begin(client);
  carrier.display.fillScreen(ST77XX_BLACK);
  carrier.display.println("Connecting WiFi...");
}

void loop() {
  // Check WiFi Connection
  if (WiFi.status() != WL_CONNECTED) {
    WiFi.begin(SECRET_SSID, SECRET_PASS);
    delay(5000);
    return;
  }

  // Read Carrier Rev2 Sensors
  float temp = carrier.Env.readTemperature();
  float hum  = carrier.Env.readHumidity();
  float pres = carrier.Env.readPressure();

  // Update On-Board Display
  carrier.display.fillScreen(ST77XX_BLACK);
  carrier.display.setCursor(20, 40);
  carrier.display.print("Temp: "); carrier.display.print(temp);
  carrier.display.setCursor(20, 60);
  carrier.display.print("Sending to Cloud...");

  // Send Multiple Fields to ThingSpeak
  ThingSpeak.setField(1, temp);
  ThingSpeak.setField(2, hum);
  ThingSpeak.setField(3, pres);

  int response = ThingSpeak.writeFields(SECRET_CH_ID, SECRET_WRITE_APIKEY);

  if (response == 200) {
    Serial.println("ThingSpeak Update Successful");
  } else {
    Serial.println("Error: " + String(response));
  }

  // ThingSpeak update limit is 15s; we use 20s for stability
  delay(20000); 
}

```

## 🚀 Key Features of Rev2 Carrier

* **BME688 Sensor:** The Rev2 uses the BME688, which is more accurate for air quality and pressure than the original Rev1.
* **Capacitive Touch:** You can modify the code to change what is shown on the OLED display by touching the buttons `0-4`.
* **Relays:** Use ThingSpeak **TalkBack** to remotely trigger the two mechanical relays on the back of the carrier!

---


