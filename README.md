# DSMR 2.2/3.0 component for ESPHome

The [SlimmeLezer](https://www.zuidwijk.com/product/slimmelezer/) is a compact build easy to use module to read data via
the P1 port on a Smart Meter. Based on an ESP8266 (Wemos D1), the SlimmeLezer is perfect to use with
[ESPHome](https://esphome.io) and integrates seamless into [Home Assistant](https://www.home-assistant.io).

### Status of this fork

To quickly get DSMR2.2/3.0 support for the SlimmeLezer without impacting working version 4/5 users, this fork was
created after some discussion in [issue #12](https://github.com/zuidwijk/dsmr/issues/12) of the original 
[zuidwijk/dsmr](https://github.com/zuidwijk/dsmr) repository.

DSMR2.2/3.0 is now supported out of the box in ESPHome and custom firmware for the SlimmeLezer has been released.

 - [ESPHome DSMR documentation](https://esphome.io/components/sensor/dsmr.html#older-dsmr-meters-support)
 - [SlimmeLezer firmware for DSMR 2.x with state_fix](https://www.zuidwijk.com/download/slimmelezer-firmware-for-dsmr-2-x/)
   - [How to flash the SlimmeLezer by usb](https://www.zuidwijk.com/how-to-flash-the-slimmelezer-by-usb/)

This repository is now out-of-date, so please use the alternatives mentioned above. The actual content has been moved
to the `old-do-not-use` branch.
