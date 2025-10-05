# ESPHome Indoor Multi-Sensor Hardware

<p align="center">
  <a href="enclosure/README.md"><img src="enclosure/meta/ESPHome-Indoor-Multi-Sensor-Corner-Mount-Render.png" width="32%"></a>
  <a href="pcb/README.md"><img src="pcb/meta/ESPHome-Indoor-Multi-Sensor-Front-Render.png" width="32%"></a>
  <a href="pcb/README.md"><img src="pcb/meta/ESPHome-Indoor-Multi-Sensor-Back-Render.png" width="32%"></a>
</p>

There are a lot of presence sensors using a combination of PIR and High Frequency Radar Human Presence Detectors to get a combination of quick response and low movement detection like when sleeping. Good examples of these types of sensor are the [Everything Presence One](https://shop.everythingsmart.io/products/everything-presence-one-kit) and [Roomsense IQ](https://www.roomsenselabs.com/). The later has a sensor module that you can add to support all sorts readings like Particular Matter (PM) Carbon Monoxide. But since I like to do everything myself. I started thinking about how I would approach this and more specifically what sensors did I want.

I settled on the following characteristics:

* ESP32-S3
* ESPHome software development
* USB-C Power
* Multiple options for Radar presence detection modules
  - Hi-Link LD2410B
  - Hi-Link LD2410S
  - Hi-Link LD2420
  - Hi-Link LD2450
  - DFRobot C4001 (SEN609 and SEN610)
* PIR (Panasonic)
* Sensirion SEN66 All-in-One (Package A)
  - Temperature
  - Humidity
  - Particulate Matter (PM) 
  - CO₂
  - Volatile Organic Compound (VOC)
  - Nitrogen Oxide (NOₓ)
* Or individual Sensors (Package B)
  - Sensirion SHT4X Temperature and Humidity
  - Sensirion SCD4X CO₂
  - Sensirion SGP4X VOC and NOₓ
* Bosch BMP581 Pressure
* Figaro TGS5141 Carbon Monoxide (CO)
* AMS TSL2591 Light
* Microphone for sound levels
* A Speaker for alerts
* INA228 Power and Energy Monitor

## Status
* **Rev -:** Has been fabricated and tested. I had JLCPCB fabricate the boards and I self assembled using a Home-Brew reflow oven by [Whizoo Controleo3](https://whizoo.com/). All circuits have been tested and found to be operational. 
* **Rev A:** There was an issue with signal integrity on the I²C bus when communicating with the SEN66. To fix the problem I added stronger pullups and a new and dedicated I²C bus to the SEN66. Thus Rev A was born. This revision was never fabricated. But all my Rev - boards have been modified to Rev A.
* **Rev B:** Added support for the LD2410S/LD2420 Radars, CO temperature sensor and INA238 power monitors for 3.3V and 3.3VA. Removed Radar power control. Board has been fabricated and testing has begun.

## Design Decisions
### ESP32 and ESPHome
ESPHome is closely aligned with Home Assistant. In fact they are the same company. Home Assistant uses ESPHome for some of their hardware like the [Home Assistant Voice Preview Edition](https://www.home-assistant.io/voice-pe/) or [Bluetooth Proxy](https://esphome.io/components/bluetooth_proxy.html). I chose a newer ESP32-S3 for this design which also has Bluetooth support.

This project uses ESP32-S3-WROOM-2-N32R16V which has an enormous 32MB of flash and 16MB of octal PSRAM. These specs are way out of the normal range for an ESP32, so why? First I wanted to be able to store sounds directly on the unit itself mainly so it can sound out alerts even if Home Assistant is not connected. The PSRAM came along with the big Flash but it also has benefits. Sound pressure analysis likes more PSRAM and so does Bluetooth Proxying.

### Sensors
#### Radar based Human Presence Sensors
Three of the Hi-Link sensors (LS2410B, LD2420 and LD2450) are supported by ESPHome directly but the DFRobot C4001 and LD2410S are not directly supported. This project uses the LD2410 component without change. For the LD2450 I have created a config that allows you to specify an installation angle, handle coordinate transforms and all zone detection. I created (with help from the Internet) a nice Home Assistant Lovelace configuration to visualize the installation angle and zone [here](https://mikelawrence.github.io/esphome-indoor-multi-sensor-config/home-assistant.html) for the LD2450 Radar and the LD2410 Radar's Gate configuration. For the DFRobot C4001 Radar I wrote an ESPHome External Component and created a [Pull Request](https://github.com/esphome/esphome/pull/9810) to add it to ESPHome. However the DFRobot team has since started their own [Pull Request](https://github.com/esphome/esphome/pull/10675). I will add support for their Pull Request when it comes out of draft and works. There is a [Pull Request](https://github.com/esphome/esphome/pull/8486) for the LD2410S. I will add support soon for the LD2410S and LD2420 Radars.

#### Sensor choices
The [SEN66](https://sensirion.com/products/catalog/SEN66) may be an expensive sensor package but it does it all, Temperature, Humidity, PM, CO₂, VOC and NOₓ. A tiny fan is primarily used for measuring PM but it also moves air by the temperature and humidity sensors making them fairly responsive. I call this Package A. If you choose not use the SEN66 you can populate the Package B individual sensors ([SHT4X](https://sensirion.com/media/documents/33FD6951/67EB9032/HT_DS_Datasheet_SHT4x_5.pdf), [SCD4X](https://sensirion.com/media/documents/48C4B7FB/67FE0194/CD_DS_SCD4x_Datasheet_D1.pdf) and [SGP4X](https://sensirion.com/media/documents/A056FE9C/61E970C2/Sensirion_Flyer_Gas_Sensors_Web.pdf)) to get Temperature, Humidity, CO₂, VOC and NOₓ. The only thing missing is PM.

#### Carbon Monoxide
For some reason CO sensors are not as common as CO₂ sensors. Most are expensive and have short life spans. The  Figaro TGS5141 is inexpensive and has a 10-year life span but it isn't a nice all-in-one with I²C interface. You have to add a transimpedance amplifier and then digitize the analog signal. I chose to add an ADS1115 16-bit ADC right next to the transimpedance amplifier instead of running analog signal across the board to the questionable ESP32 built in ADC. The ADS1115 also a has true differential Programmable Gain Amplifier on the input. This reduces the CO sensor circuit complexity. This sensor is not going to be very accurate but detection of CO in even low levels is probably a reason for concern. So even if not a safety sensor it can help.

I verified the CO circuit using CO Bump Gas. It doesn't calibrate, but it will prove that the sensor responds to the presence of CO. I used a small upside down container on my outside bench with the sensor I wanted to test inside. Squirting some bump gas inside with the thin straw showed that 4 out of 5 sensors responded to the gas. The 5'th that didn't respond had a solder bridge between two pins on the transimpedance amplifier. A quick removal of the solder bridge fixed the non-functioning sensor.

Rev B adds a DS18S20 temperature sensor to the design for CO calibration. The intent is to glue it to the CO sensor and use that for temperature calibration instead of using the ambient temp or the SHT4X sensor on the other side of the board.

> [!WARNING]
> This CO sensor is not rated for safety nor is it particularly accurate. Use at your own risk! I still have UL listed (American) CO detectors as my main line of defense.

#### Pressure and Light
The [BMP581](https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bmp581-ds004.pdf) pressure sensor is primarily used as a barometric pressure sensor but it is sensitive enough for micro-pressure changes like when a door is closed. Pressure can also be used to compensate the CO₂ sensor in either the SEN6X or the SCD4X. Light is another measurement that may or may not be helpful but I included it. Keep in mind that this measurement is inside an enclosure thus has some significant limitations when it comes to accuracy.

#### Speaker and Microphone
The SEN66 takes up a lot of space so a small speaker is necessary. I choose a flat speaker from [Same Sky](https://www.sameskydevices.com/product/audio/speakers/miniature-(10-mm~40-mm)/cms-251437-24sp-x8) and started playing. My first attempts were pretty, flat, meaning these small speakers don't move a lot of air and in free space they have almost no volume. After doing some research it became clear, the speaker needs a enclosure. I was able to come up with a small 3D printed enclosure attached to the PCB itself. Man what a difference. This 2W speaker kicks some serious butt.

I threw in the microphone to measure sound levels. The intent is listen for noise as additional presence detection.

#### Power and Energy Monitor
The [INA228](https://www.ti.com/lit/ds/symlink/ina228.pdf) power and energy monitor can give instantaneous readings of voltage, current and power but it also measures energy over time (WHr).

### USB-C Power
The board does not use USB-C Power Delivery but I am expecting 5V @ 3A which is readily available for most USB-C Power Bricks. The PCB has the appropriate resistors on the CC pins which will request 5V @ 3A from a USB-C Power Brick. The board does not need anywhere close to 3A but peaks near 0.75A are possible so be careful using a USB 2.0 Power Brick with a 0.5A default current. Actual testing shows a 1W average for the Package A version and slightly less than 0.9W average for the Package B option as measured by the INA228. Quick calculations show about 8.1kWh/yr for Package A and 7.3kWh/yr for Package B. In Texas that is less than $1 per year. These number are for Rev -.

## Enclosure
<p align="center">
  <a href="enclosure/README.md"><img src="enclosure/meta/ESPHome-Indoor-Multi-Sensor-Angle-Wallplate-Mount-Render.png" width="70%"></a>
</p>

More information is in the enclosure [README](enclosure/README.md) file.

## Configuration
Click [here](https://mikelawrence.github.io/esphome-indoor-multi-sensor-config/) to go to the installation and configuration repository.
