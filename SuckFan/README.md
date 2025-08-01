# Suck Fan Controller

I have a shop with a paint booth. The old way to control the fan was a 3phase knife switch. 3Phase fan timers with a contactor seem to be hundreds of dollars. I wanted to be 
able to also use home assistant to control the fan remotely or with timers/actions/automations from home assistant. 

This is seutp using [ESP Home](https://esphome.io) and [Home Assistant](https://www.home-assistant.io) which allows for both local and remote control of the relay and 3PH contactor.

The YAML file contains the code necesary for ESPHome to compile the code to run the ESP32 device, along with the LCD interface.

ESP Home YAML File Config [lives here](/SuckFan/esp-suck-fan.yaml)

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
