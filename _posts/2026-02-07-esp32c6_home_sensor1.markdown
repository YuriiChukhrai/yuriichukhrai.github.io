---
layout: post
title: "DIY. Building a HomeKit Environmental Sensor with ESP32-C6, DHT22, SGP40, and BMP180"
subtitle: Build a Smart Home Environmental Monitor with ESP32-C6 and HomeKit Integration
date:   2026-02-08 23:59:59 -0700
categories: diy
tags: [ESP32-C6, HomeKit, IoT, Environmental Sensors, DIY Smart Home, Air Quality Monitoring, PlatformIO]
author: Yurii Chukhrai
---

## Introduction

The air we breathe indoors can often be more polluted than the air outside.
Volatile Organic Compounds (VOCs) from cleaning products, cooking, and even furniture can impact our health and well-being.
Wouldn't it be great to have a clear picture of your indoor environment, seamlessly integrated into your Apple Home app?
This guide will walk you through building your own HomeKit-compatible environmental sensor using the powerful ESP32-C6 microcontroller and a suite of sensors: the DHT22 for temperature and humidity, the SGP40 for air quality (VOCs), and the BMP180 for atmospheric pressure.

Apple HomeKit offers a robust, secure, and user-friendly platform for managing smart home devices. 
Instead of juggling multiple apps for different brands, HomeKit unifies your devices into a single, intuitive interface.
By building a HomeKit-native sensor, we transform a DIY project into a polished, Apple-ecosystem-friendly device.


## Microcontroller: ESP32-C6

The ESP32-C6 is a fantastic choice for IoT projects, especially for a multi-sensor node like this. It's an upgrade in many ways:
* **Wi-Fi 6 & Bluetooth 5.0 LE:** Superior wireless connectivity for robust HomeKit communication.
* **Low Power Consumption:** Ideal for battery-powered applications, though for this project, we'll assume constant power.
* **Dual-Core Processing:** Efficiently handles Wi-Fi, sensor polling, and HomeKit communication simultaneously.
* **Rich Peripheral Set:** Plenty of GPIOs for connecting multiple sensors using various communication protocols (I2C, 1-Wire).
* **GPIO Matrix:** A key feature that allows flexible pin remapping, meaning you can assign I2C (SDA/SCL) and other functions to almost any digital pin. This proved crucial in our troubleshooting!
* **Arduino IDE & PlatformIO Support:** A wide ecosystem for development. We'll be using PlatformIO for this guide.

We'll be specifically using the **[Seeed Studio XIAO ESP32-C6](https://files.seeedstudio.com/wiki/SeeedStudio-XIAO-ESP32C6/res/esp32-c6_datasheet_en.pdf)**, a tiny but powerful board perfect for compact sensor nodes.

![esp32-c6.jpg](../resources/diy/2026-02-07_esp32c6_home-sensor1/images/esp32-c6.jpg){:.mx-auto.d-block :}
Studio XIAO ESP32-C6.
{: style="text-align:center;"}

## Sensors: The Senses: DHT22, SGP40, and BMP180

Our environmental sensor will gather three crucial types of data: temperature/humidity, air quality, and atmospheric pressure.

* **[DHT22 (Temperature & Humidity Sensor)](https://cdn.sparkfun.com/assets/f/7/d/9/c/DHT22.pdf):**
    * **What it does:** Measures ambient air temperature (in Celsius) and relative humidity (in percentage).
    * **Contribution:** Provides essential environmental context. Humidity, in particular, is vital for the SGP40's accuracy.
    * **Communication:** 1-Wire digital protocol.

* **[SGP40 (VOC Air Quality Sensor)](https://sensirion.com/media/documents/296373BB/6203C5DF/Sensirion_Gas_Sensors_Datasheet_SGP40.pdf):**
    * **What it does:** Detects a wide range of Volatile Organic Compounds (VOCs) and provides an "Air Quality Index" (0-500).
    * **Contribution:** This is our primary air quality monitor. A rising index indicates increasing levels of indoor pollutants.
    * **Communication:** I2C.
    * **Key Insight:** The SGP40 requires temperature and humidity data for **compensation**. Without it, its readings would be inaccurate due to the effect of water vapor on its sensing film. The `sgp.measureVocIndex(t, h)` function does this crucial calculation.

* **[BMP180 (Barometric Pressure Sensor)](https://cdn-shop.adafruit.com/datasheets/BST-BMP180-DS000-09.pdf):**
    * **What it does:** Measures atmospheric pressure (in Pascals/hPa) and also provides temperature.
    * **Contribution:** Useful for weather monitoring, predicting local weather changes, or even rudimentary altitude detection.
    * **Communication:** I2C.
    * **Note:** While it also measures temperature, the DHT22 is usually more accurate for ambient room temperature. We'll use the BMP180's temperature for its own compensation.

> **NOTE:** I2C (Inter-Integrated Circuit) is a widely used, two-wire serial communication protocol designed by 
> [Philips Semiconductor](https://www.geeksforgeeks.org/computer-organization-architecture/i2c-communication-protocol/) for connecting lower-speed peripheral integrated circuits (ICs) to processors and microcontrollers.
> It uses only two bidirectional lines—SDA (Serial Data) and SCL (Serial Clock)—allowing a master device to control multiple slave devices on the same bus, making it ideal for short-distance, on-board communication.

    
## Wiring and Connections

The XIAO ESP32-C6 is tiny, so precise wiring is crucial. We'll use a breadboard for simplicity.

**Required Components:**

* Seeed Studio XIAO ESP32-C6
* DHT22 Temperature & Humidity Sensor
* Adafruit SGP40 Air Quality Sensor
* BMP180 Barometric Pressure Sensor
* Breadboard & Jumper Wires
* USB-C Cable for power and programming

**Wiring Diagram:**

Remember the XIAO ESP32-C6 has flexible GPIOs. We found that the default I2C pins (D4/D5) were problematic, so we'll use D0 and D2 for I2C and D3 for the DHT22. D1 is reserved for the internal RGB LED.

| Component | XIAO Pin                     | GPIO # | Function |
| :-------- |:-----------------------------|:-------| :------- |
| **DHT22** | VCC                          | 3V3    | Power    |
| **DHT22** | GND                          | GND    | Ground   |
| **DHT22** | Data                         | **D1** | Signal   |
| **SGP40** | VCC                          | 3V3    | Power    |
| **SGP40** | GND                          | GND    | Ground   |
| **SGP40** | SDA(Serial Data)             | **D0** | I2C Data |
| **SGP40** | SCL(Serial Clock)            | **D2** | I2C Clock |
| **BMP180** | VCC                          | 3V3    | Power    |
| **BMP180** | GND                          | GND    | Ground   |
| **BMP180** | SDA                          | **D0** | I2C Data |
| **BMP180** | SCL                          | **D2** | I2C Clock |


> **Important Note for I2C:** Both the SGP40 and BMP180 share the same SDA (D0) and SCL (D2) lines.
> This is perfectly normal for I2C, as each device has a unique address.

## IDE

We'll use **PlatformIO** for development, as it simplifies library management and ESP32 project structure.

### PlatformIO Setup

1. Install VS Code: If you haven't already, install Visual Studio Code.
1. Install PlatformIO Extension:** Search for "PlatformIO IDE" in the VS Code Extensions tab and install it.
1. Create New Project:
   1. Click the PlatformIO icon on the VS Code sidebar.
   1. Click "New Project".
   1. Name your project (e.g., `HomeKit_Air_Sensor`).
   1. Board: Search for `Seeed XIAO ESP32C6` or `esp32-c6-devkitc-1`.
   1. Framework: `Arduino`.
   1. Click "Finish".

###  Configuration `platformio.ini`

```ini
[env:seeed_xiao_esp32c6]
platform = https://github.com/pioarduino/platform-espressif32/releases/download/stable/platform-espressif32.zip
board = seeed_xiao_esp32c6
framework = arduino
monitor_speed = 115200
board_build.partitions = partitions.csv
lib_deps =
    HomeSpan
    adafruit/Adafruit SGP40 Sensor  ; For SGP41/40
    adafruit/DHT sensor library
    adafruit/Adafruit Unified Sensor
    adafruit/Adafruit BMP085 Library

; Force the uploader to use the correct speed and reset method
upload_speed = 115200
upload_protocol = esptool

; These are crucial for the C6's internal USB
build_flags = 
    -D ARDUINO_USB_MODE=1
    -D ARDUINO_USB_CDC_ON_BOOT=1
```

### Code

Now, let's build our src/main.cpp file. This is the heart of our project.

<details markdown="1">
<summary>Full C code</summary>

```C
#include "HomeSpan.h"
#include "DHT.h"
#include <Adafruit_SGP40.h>
#include <Adafruit_BMP085.h>

// 1. CUSTOM MACROS (Must be at the very top, outside any functions)
// These create Service::Barometer and Characteristic::AtmosphericPressure
CUSTOM_CHAR(AtmosphericPressure, E863F10F-079E-48FF-8F27-9C2605A29F52, PR+EV, FLOAT, 1013.25, 700, 1200, false);
CUSTOM_SERV(Barometer, E863F00A-079E-48FF-8F27-9C2605A29F52);

// Pin and Sensor Definitions
#define DHTPIN 1       
#define SDA_PIN 0      
#define SCL_PIN 2      
#define ORANGE_LED 15     
#define DHTTYPE DHT22

DHT dht(DHTPIN, DHTTYPE);
Adafruit_SGP40 sgp;
Adafruit_BMP085 bmp;

bool sgpFound = false;
bool bmpFound = false;

// 2. HOMESPAN DEVICE STRUCTURES
struct DEV_Sensors : Service::TemperatureSensor {
  SpanCharacteristic *temp;
  SpanCharacteristic *hum;
  SpanCharacteristic *voc;
  SpanCharacteristic *vocIdx;
  SpanCharacteristic *pres;
  SpanCharacteristic *hic;

  DEV_Sensors() : Service::TemperatureSensor() {
    
    // -- Temperature --
    new Characteristic::Name("Temperature sensor");
    temp = new Characteristic::CurrentTemperature(20);
    temp->setRange(-40, 80);

    // -- Humidity --
    new Service::HumiditySensor();
    hum = new Characteristic::CurrentRelativeHumidity(50);

    // -- Air Quality (SGP40) --
    new Service::AirQualitySensor();
    voc = new Characteristic::AirQuality(1);         // 1-5 scale for Home App
    vocIdx = new Characteristic::VOCDensity(100);    // Raw VOC Index
    vocIdx->setRange(0, 500);

    // -- Barometer (Eve App Support) --
    new Service::Barometer(); 
    new Characteristic::Name("Barometer");
    pres = new Characteristic::AtmosphericPressure();
    pres->setUnit("mbar"); // Eve prefers mbar/hPa

    // -- Heat Index (Treated as a second Temp sensor) --
    new Service::TemperatureSensor();
    new Characteristic::Name("Heat Index");
    new Characteristic::ConfiguredName("Heat Index");
    hic = new Characteristic::CurrentTemperature(20);
    hic->setRange(-50, 100);

    Serial.println("All HomeKit Services Initialized");
  }

  void loop() override {
    // Update every 5 seconds
    if (temp->timeVal() > 5000) {
      digitalWrite(ORANGE_LED, LOW); // LED ON

      float t = dht.readTemperature();
      float h = dht.readHumidity();
      
      if (!isnan(t) && !isnan(h)) {
        temp->setVal(t);
        hum->setVal(h);
        hic->setVal(dht.computeHeatIndex(t, h, false));

        if (bmpFound) {
          float p_hPa = bmp.readPressure() / 100.0F; // BMP180 returns Pascals
          pres->setVal(p_hPa);
        }

        if (sgpFound) {
          int32_t vocIndex = sgp.measureVocIndex(t, h);
          vocIdx->setVal(vocIndex);

          // Map VOC Index to 1-5 AirQuality
          int aqLevel = 1;
          if(vocIndex > 350) aqLevel = 5;
          else if(vocIndex > 250) aqLevel = 4;
          else if(vocIndex > 150) aqLevel = 3;
          else if(vocIndex > 50) aqLevel = 2;
          voc->setVal(aqLevel);
          Serial.printf("T: %.1fC, H: %.1f%%, P: %.1f hPa, HIC: %.1fC, VOC: %d\n", t, h, pres->getVal<float>(), hic->getVal<float>(), vocIndex);
        }
      }
      digitalWrite(ORANGE_LED, HIGH); // LED OFF
    }
  }
};

// 3. SETUP & MAIN LOOP
void setup() {
  Serial.begin(115200);
  delay(2000);
  
  pinMode(ORANGE_LED, OUTPUT);
  digitalWrite(ORANGE_LED, LOW); 

  Wire.begin(SDA_PIN, SCL_PIN);
  dht.begin();
  bmpFound = bmp.begin();
  sgpFound = sgp.begin();

  homeSpan.setPairingCode("{pair code here}");
  homeSpan.setWifiCredentials("{name of your 2.4G WiFi}", "{password}");
  homeSpan.begin(Category::Sensors, "IoT#1 Monitor");

  new SpanAccessory();
    new Service::AccessoryInformation();
      new Characteristic::Name("XIAO Air Station#1");
      new Characteristic::Manufacturer("YC");
      new Characteristic::Model("XIAO-ESP32C6");
    
    // Instantiate our unified sensor class
    new DEV_Sensors();

  digitalWrite(ORANGE_LED, HIGH); 
  Serial.println("System Ready.");
}

void loop() {
  homeSpan.poll();
  
  // Blink if sensors are disconnected
  if (!sgpFound || !bmpFound) {
    if (millis() % 1000 < 200) digitalWrite(ORANGE_LED, LOW);
    else digitalWrite(ORANGE_LED, HIGH);
  }
}
```
</details>

## Testing and Troubleshooting Tips
![ready.jpg](../resources/diy/2026-02-07_esp32c6_home-sensor1/images/ready.jpg){:.mx-auto.d-block :}
Ready for debugging.
{: style="text-align:center;"}

### Initial Upload and Pairing

* Replace Placeholders: Make sure you've updated _YOUR_WIFI_SSID_ and _YOUR_WIFI_PASSWORD_ in the code.
* Upload: Connect your XIAO ESP32-C6 via USB-C and upload the code using PlatformIO.
* Serial Monitor: Open the Serial Monitor (PlatformIO -> Monitor) at 115200 baud. Watch for:
  * SGP40 Detected! and BMP180 Detected!
  * Initial sensor readings.

#### Home App Pairing

* Open the Apple Home app on your iPhone.
* Tap the + icon -> "Add Accessory" -> "More options..."
* Select your "XIAO Env. Monitor" (it might appear as "ESP32-C6 Env. Monitor" initially).
* When prompted, enter the pairing code `{your code here}`.
* Assign it to a room.

![ihome_balcony.jpg](../resources/diy/2026-02-07_esp32c6_home-sensor1/images/ihome_balcony.jpg){:.mx-auto.d-block :}
Apple Home
{: style="text-align:center;"}

![eve_app.png](../resources/diy/2026-02-07_esp32c6_home-sensor1/images/eve_app.png){:.mx-auto.d-block :}
[Eve HomeKit app](https://apps.apple.com/us/app/eve-for-matter-homekit/id917695792)
{: style="text-align:center;"}

### Common Issues and Solutions
1. "SGP40 NOT FOUND" / "BMP180 NOT FOUND":
    * Check Wiring: Double-check SDA (**D0**) and SCL (**D2**) wires. 
    * Ensure power (**3V3**) and ground are solid.
    * I2C Scanner: Re-run the I2C scanner code we used previously to confirm I2C devices are visible on **D0/D2**. 
    * Loose Connections: Tiny breadboards can have flaky connections. Wiggle wires gently.
    * Sensor Fault: Rarely, a sensor might be faulty.

1. "DHT22 Sensor Read Failed!":
   * Wiring: Check DHT22 Data to **D1**, and power/ground.
   * Delay: DHT22 needs at least 2 seconds between readings. Our 5-second interval is fine. 
   * Library: Ensure adafruit/DHT sensor library and adafruit/Adafruit Unified Sensor are installed.

1. "No Response" in Home App:
   * Wi-Fi: Check _WIFI_SSID_ and _WIFI_PASSWORD_ are correct.
   * Power: Ensure the ESP32-C6 has stable power.
   * HomeSpan Reset: If you've been experimenting, you might need to factory reset HomeSpan. Type `H` in the Serial Monitor, then `R` to reset the database. You'll need to re-pair afterwards.


### Understanding the Pressure

Since HomeKit doesn't have a native "Atmospheric Pressure" service the HomePan library allows us to create custom services ([CustomService](https://github.com/HomeSpan/HomeSpan/blob/master/examples/Other%20Examples/CustomService)) and characteristics in HomeSpan to implement an Atmospheric Pressure Sensor recognized by the [Eve for HomeKit app](https://apps.apple.com/us/app/eve-for-matter-homekit/id917695792). 
See [Custom Characteristics and Custom Services Macros](https://github.com/HomeSpan/HomeSpan/blob/master/docs/Reference.md#custom-characteristics-and-custom-services-macros) for full details.

## Summary

This project ([GitHub link](https://github.com/YuriiChukhrai/electronics/tree/main/IoT-3)) demonstrates how to build a HomeKit-compatible environmental sensor using the ESP32-C6 microcontroller and sensors like DHT22, SGP40, and BMP180. 
By integrating these components, you can monitor temperature, humidity, air quality, and atmospheric pressure directly in the Apple Home app, offering a seamless and secure smart home experience.