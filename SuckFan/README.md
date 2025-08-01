# Suck Fan Controller

Using [ESP Home](https://esphome.io) and [Home Assistant](https://www.home-assistant.io) this allows local and remote control of the relay and 3PH contactor.

The YAML file contains the code necesary for ESPHome to compile the code to run the interface.

[Code lives here](/SuckFan/esp-suck-fan.yaml)

<img src="/SuckFan/Suck-Fan-1.jpeg" alt="Picture of the control box." style="max-width: 50%; width: 500px;">

This uses the following hardware to turn on a three phase contactor.

# Controller
ESP32 Board Relay Board: https://a.co/d/eZbQkHe

<img src="/SuckFan/ESP32-Relay-Board.jpeg" alt="Picture of the esp32 board." style="max-width: 50%; width: 500px;">

This board has 4 realys, and zero documentation.
  relay_1: GPIO32
  relay_2: GPIO33
  relay_3: GPIO25
  relay_4: GPIO26

# Display
* Adafruit 1.9" 320x170 Color IPS TFT Display - ST7789
* https://www.adafruit.com/product/5394

# Buttons
Two buttons that I just had in my stock of stuff. Adafruit as some reliable ones [here](https://www.adafruit.com/product/915)

# Enclosure
[Waterproof Outdoor Electrical Box Junction Box Weatherproof IP67 ABS Plastic Enclosure Switch Router Project Box with Mounting Plate with Wall Bracket 2 Cable Glands Clear 11.4"x7.5"x5.5"](https://a.co/d/iZUPQZJ)

# 3-Phase 18Amp Contactor
LC1D18G7 AC Contactor 120V coil 3NO
Schneider Contactor LC1D18G7 3P 18A
https://www.electricalengineeringtoolbox.com/2022/04/how-to-wire-photocell-switch-to.html?m=1
