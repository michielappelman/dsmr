# DSMR 2.2/3.0 component for ESPHome

The [SlimmeLezer](https://www.zuidwijk.com/product/slimmelezer/) is a compact build easy to use module to read data via
the P1 port on a Smart Meter. Based on an ESP8266 (Wemos D1), the SlimmeLezer is perfect to use with
[ESPHome](https://esphome.io) and integrates seamless into [Home Assistant](https://www.home-assistant.io).

## :warning: DSMR version 2.2/3.0 fork :warning:

This repository is a fork from the original [zuidwijk/dsmr](https://github.com/zuidwijk/dsmr) but adapted to support
DSMR versions 2.2 and 3.0.

## DSMR component
The main goal is to create one universal component, which can be used in every country. Though the DSMR (Dutch Smart
Meter Requirements) is [specified](https://www.netbeheernederland.nl/_upload/Files/Slimme_meter_15_a727fce1f1.pdf) with
pre specified OBIS code, not every country has exact the same code. Some examples:

**Version information for P1 output**
- Default OBIS: 1-3:0.2.8.255
- Belgium OBIS: 0-0:96.1.4.255

**Meter Reading electricity delivered to client (Tariff 1) in 0,001 kWh**
- Default OBIS: 1-0:1.8.1.255
- Luxembourg OBIS: 1-0:1.8.0.255

**Meter Reading electricity delivered by dient (Tariff 1) in 0,001 kWh**
- Default OBIS: 1-0:2.8.1.255
- Luxembourg OBIS: 1-0:2.8.0.255

Some countries like Luxembourg, Sweden and Hungary, uses kvar next to kW. Therefor all deviant OBIS code is added as
extra fields. This gives more sensors than needed, yet it can be used in every country where DSMR based Smart Meters is
being used.

### Decryption data for Luxembourg
Smart Meters used in Luxembourg are using encryption. Decryption for Luxembourg is build in the code. This can be
defined in the code:

```YAML
dsmr:
  id: dsmr_instance
  decryption_key: '00112233445566778899AABBCCDDEEFF'
```

When the key is not set in the code, or when the key changes, it can be set/changes via a Service within Home Assistant,
created via below api:

```YAML
# Enable Home Assistant API
api:
  services:
    service: set_dsmr_key
    variables:
      private_key: string
    then:
      - logger.log:
          format: Setting private key %s. Set to empty string to disable
          args: [private_key.c_str()]
      - globals.set:
          id: has_key
          value: !lambda "return private_key.length() == 32;"
      - lambda: |-
          if (private_key.length() == 32)
            private_key.copy(id(stored_decryption_key), 32);
          id(dsmr_instance).set_decryption_key(private_key);
```

In Home Assistant go to Services and select the service `ESPHome: {name}_set_dsmr_key`. There fill in the code received
from the provider: ![SlimmeLezer_set_key](https://user-images.githubusercontent.com/10123063/127783141-52d3ae77-e02b-4296-a1fb-78ab3bbe5ff3.jpg)

### Different UART
The SlimmeLezer is built with a logic inverter on the pcb. Connecting that directly to the Rx of the Wemos, causes that
it can't be flashed via USB as it constanly pulls the Rx either high or low. Therefor I'm using the 2nd uart, on pin D7.
That's why the uart is specified on pin D7 in the code:

```YAML
uart:
  baud_rate: 115200
  rx_pin: D7
```

#### UART Settings for different DSMR versions

DSMR versions 2.2 and 3.0 use different settings for the serial interface:

```yaml
uart:
  baud_rate: 9600
  data_bits: 7
  parity: EVEN
  stop_bits: 1
  rx_pin: D7
```

This can mess with the logging over the same channel, so it is disabled. You can still see your logs through the API.

```yaml
logger:
  baud_rate: 0
```
