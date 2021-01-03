---
title: "Esp8266 Adventures"
date: 2020-01-02T23:00:49-03:00
lastmod: 2020-01-02T23:00:49-03:00
draft: false
screenshot: /esp8266.jpg
tags: [esp8266, arduino, mqtt, k3s, prometheus, grafana]
---

![ESP8266 photo](/esp8266.png)

# ESP8266 Adventures

How could technology help me in future impacted by global warming? This is a question that has been following me the past decade, specially where can my current area of study help me in this future. I imagine that all our accumulated knowledge from tradition and science will be needed in order  to navigate through a new era of scarcity. My main area of study is in Computer Science and I've mostly had contact with intangible stuff. This is were I set sail in my vacation: exploring were I can touch the tangible with the intangible.

Giving the scarcity scenario I thought that the most basic gadget I could build to learn for the future is a simple weather station. You need light, healthy soil and the right amount of water to grow food.

My idea was just collecting the temperature, air humidity, soil humidity and sending it to my cluster for processing. For that I needed some type of wireless connection. So I went online and bought the most simple looking and cheep module I cloud find to couple it to and Arduino I've already owned. I got the ESP-12e and that is were a hole new world opened beneath my feet and thing got a little more complicated.

## Programming the ESP8266

What I didn't know is that the ESP8266 is not just a Wifi module, it can be programmed and even run a web server inside. Of course there was a community leveraging this for IOT, and they led me to [OpenMQTTGateway](https://docs.openmqttgateway.com/). This project collects various signals and sensor from IOT devices and sends it to a queue to be used by HomeAutomation services such as [Home Assistant](https://www.home-assistant.io/). Given my astonishment of the capacity of this module of course I didn't have any tools that would make my life easier in programming it.

Since I had an Arduino I thought it would be easy to use it to program the ESP. My first blocker is that I saw on a tutorial that I needed an external 3,3V power source because Arduino's 3,3V output didn't have enough current for the module. I found 3 used 1,5V batteries that connected in series gave me 3,6V. The datasheet says that 3,6V is within the operational voltage, so achievement unlocked for recycling used batteries. (I do have to notice that I found out later that used batteries are really bad at keeping the voltage I measured without any usage.)

Now I just had to flash the OpenMQTTGateway project into the module. Well, to put it into flash mode you have to reboot the module with the GPIO0. I found this nice schematics in instructables:

![Image showing how to connect Arduino to flash an ESP8266 module](/arduino_schematics.png)
[https://www.instructables.com/ESP-12E-ESP8266-With-Arduino-Uno-Getting-Connected/](https://www.instructables.com/ESP-12E-ESP8266-With-Arduino-Uno-Getting-Connected/)

My execution of it was a disaster because I bought the wrong pin bar (the ESP-12E has a smaller distance between the roles in comparison to the default protoboard) so I just attached the wires directly to the module. Of course that doing so made it very unstable and I couldn't breath near it without triggering a reset. But it I got it be be stable enough to flash it. The next issue I had was not flowing correctly the instructions: I was trying to use Arduino IDE to program the module and it wasn't able to connect. First of all it is very important to connect the ground to reset in the Arduino board. Apparently when you open the usb-to-serial converter Arduino automatically resets and keeping reset pin to ground avoids that.

# Pin info

![Image showing the ESP8266 pin and their usage.](/esp8266_pins.png)

⚠️ To flash software to the module no other sensor can be attached.

![Image showing where the pins are located in the ESP 8266.](https://circuits4you.com/wp-content/uploads/2016/12/ESP-12-PinOut.png)

I ended buying a fiber glass protoboard to weld everything together and make the setup more stable (I admit I do have practice welding to get more motor coordination). Finally I also added a USB cable and a tension regulator to 3,3V for constant power supply.

![Image of the welded final setup of my ESP with a tension regulator and the DHT sensor.](/final_esp_setup.jpg)

You might ask me why didn't you just buy the [nodemcu](https://www.nodemcu.com/)? Well, I basically didn't know of its existence and I never thought that "wifi module" could do so much. It was fun to learn, but it is certainly better to buy the nodemcu because it is pretty cheap (even in Brazil) and flashing it and connecting stuff to it is a lot simpler.

## OpenMQTTGateway Configuration

To manually configure the network and MQTT I added the following to [User_config.h](https://github.com/1technophile/OpenMQTTGateway/blob/development/main/User_config.h):

```c
#define ESPWifiManualSetup true
#define MQTT_SERVER "your.server"
#define MQTT_USER "user"
#define MQTT_PASS "pass"
#define wifi_ssid "ssid"
#define wifi_password "pass"
#define SECURE_CONNECTION
#define MQTT_PORT "9443"
const char* certificate CERT_ATTRIBUTE = R"EOF("
-----BEGIN CERTIFICATE-----
... // your root certificate: see here for let`s encypt https://letsencrypt.org/certificates/
-----END CERTIFICATE-----
")EOF";
```

## Collecting the metrics

To collect and view the sensor data I'm using Mosquitto, Prometheus and Grafana. For Prometheus to be able to collect the data I'm running [an exporter](https://github.com/kpetremann/mqtt-exporter) that reads the data from the queue and provides a http endpoint in the format Prometheus expects.

![Diagram showing the the flow where the data is collected, sent do MQTT. Moquitto exported read the queue and exposes the data to Prometheus and Grafana queries Prometheus and renders the dasboard.](/esp_home.png)

![Dashboard showing the metrics temperature and humidity, collected by the sensor conected to the ESP8266.](/grafana.png)

## References

[🔗 OpenMQTTGateway](https://docs.openmqttgateway.com/)

[🔗 AT instruction set](https://www.espressif.com/sites/default/files/documentation/4a-esp8266_at_instruction_set_en.pdf)
