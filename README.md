# DSMR 2.2/3.0 component for ESPHome

The [SlimmeLezer](https://www.zuidwijk.com/product/slimmelezer/) is a compact build easy to use module to read data via
the P1 port on a Smart Meter. Based on an ESP8266 (Wemos D1), the SlimmeLezer is perfect to use with
[ESPHome](https://esphome.io) and integrates seamless into [Home Assistant](https://www.home-assistant.io).

## :warning: DSMR version 2.2/3.0 fork

This repository is a fork from the original [zuidwijk/dsmr](https://github.com/zuidwijk/dsmr) but adapted to support
DSMR versions 2.2 and 3.0.

### Status of this fork

To quickly get DSMR2.2/3.0 support for the SlimmeLezer without impacting working version 4/5 users, this fork was
created after some discussion in [issue #12](https://github.com/zuidwijk/dsmr/issues/12).

There has now been work on the upstream ESPHome DSMR code to get the changes below incorporated as well and have all
versions supported in the same codebase. See: [esphome#2157](https://github.com/esphome/esphome/pull/2157)

You can use the [example configuration YAML in that PR](https://github.com/esphome/esphome/pull/2157#issue-712837636)
to test if the proposed changes work for you. **Please test that configuration first before using the one from this
repo!** The more we test the faster we can get the required changes merged into the main ESPHome codebase.

### Incorporated fixes

The changes needed for DSMR v2.2/3 and included in this repo are the following:

#### 1. UART Settings

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

#### 2. Disable CRC Check

Since CRCs are not present in DSMR 2.2, I have removed all the references to the CRC check methods in the `P1Parser`
of the `parser.h` file:

```diff
     // Look for ! that terminates the data
     const char *data_end = data_start;
-    uint16_t crc = _crc16_update(0, *str);  // Include the / in CRC
     while (data_end < str + n && *data_end != '!') {
-      crc = _crc16_update(crc, *data_end);
       ++data_end;
     }

-    if (data_end >= str + n)
-      return res.fail(F("No checksum found"), data_end);
-
-    crc = _crc16_update(crc, *data_end);  // Include the ! in CRC
-
-    ParseResult<uint16_t> check_res = CrcParser::parse(data_end + 1, str + n);
-    if (check_res.err)
-      return check_res;
-
-    // Check CRC
-    if (check_res.result != crc) {
-      Serial.println(check_res.result);
-      Serial.println(crc);
-
-      return res.fail(F("Checksum mismatch"), data_end + 1);
-    }
-
     res = parse_data(data, data_start, data_end, unknown_error);
-    res.next = check_res.next;
+    res.next = data_end;
     return res;
   }
```

A better solution would of course be to make the CRC check optional by setting a boolean value in the YAML file, which
is now being considered in [esphome#2157](https://github.com/esphome/esphome/pull/2157).

#### 3. Support for MBus (gas) values

Based on [an older comment](https://github.com/matthijskooijman/arduino-dsmr/issues/22#issuecomment-489328956) on an
upstream discussion a hack has been applied that looks past the carriage return/line feed whether there is a value
there and includes it to be parsed.

```cpp
 // Parse data lines
    while (line_end < end) {
      if (*line_end == '\r' || *line_end == '\n') {
        if (!(*(line_end+1) =='(' || *(line_end + 2) =='(')){
        ParseResult<void> tmp = parse_line(data, line_start, line_end, unknown_error);
        if (tmp.err)
          return tmp;
        line_start = line_end + 1;
      }
      }
      line_end++;
    }
```

A similar approach is being investigated upstream as well. To make the value parsed in the code itself there is more
refactoring needed, as explained [in the upstream issue](https://github.com/matthijskooijman/arduino-dsmr/issues/22#issuecomment-489360436)

This non-parsed field is then added as a value to be delivered to Home Assistant (in `slimmelezer.yaml`):

```yaml
text_sensor:
  - platform: dsmr
    gas_delivered_hack:
      name: "Gas Delivered"
```

What you will end up with in Home Assistant is a bit of an ugly looking text value like this:

```
(210816110000)(08)(60)(1)(0-1:24.2.0)(m3)
 (09833.800)
```

But with a bit of regex magic you can turn it into a numeric value by adding a template sensor in your Home Assistant
configuration:

```yaml
template:
  sensor:
    - name: "Gas Delivered Numeric"
      unique_id: gas_delivered_numeric
      state: '{{states("sensor.gas_delivered_hack")| regex_findall_index(find="\d*\.\d*", index=2)|float}}'
      unit_of_measurement: "m³"
      state_class: measurement
```

If this does not result in the right value, you can paste the state template in the Template Editor in the Home Assistant
Developer Tools tab and play with the number behind `index=`.

Below is the original README from [zuidwijk/dsmr](https://github.com/zuidwijk/dsmr).

---

## DSMR component
The main goal is to create one universal component, which can be used in every country. Though the DSMR (Dutch Smart Meter Requirements) is [specified](https://www.netbeheernederland.nl/_upload/Files/Slimme_meter_15_a727fce1f1.pdf) with pre specified OBIS code, not every country has exact the same code. Some examples:

**Version information for P1 output**
- Default OBIS: 1-3:0.2.8.255
- Belgium OBIS: 0-0:96.1.4.255

**Meter Reading electricity delivered to client (Tariff 1) in 0,001 kWh**
- Default OBIS:	1-0:1.8.1.255
- Luxembourg OBIS:	1-0:1.8.0.255

**Meter Reading electricity delivered by dient (Tariff 1) in 0,001 kWh**
- Default	OBIS: 1-0:2.8.1.255
- Luxembourg	OBIS: 1-0:2.8.0.255

Some countries like Luxembourg, Sweden and Hungary, uses kvar next to kW. Therefor all deviant OBIS code is added as extra fields. This gives more sensors than needed, yet it can be used in every country where DSMR based Smart Meters is being used.

### Decryption data for Luxembourg
Smart Meters used in Luxembourg are using encryption. Decryption for Luxembourg is build in the code. This can be defined in the code:
```YAML
dsmr:
  id: dsmr_instance
  decryption_key: '00112233445566778899AABBCCDDEEFF'
 ```
 
 When the key is not set in the code, or when the key changes, it can be set/changes via a Service within Home Assistant, created via below api:
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

In Home Assistant go to Services and select the service ESPHome: {name}_set_dsmr_key. There fill in the code received from the provider:
![SlimmeLezer_set_key](https://user-images.githubusercontent.com/10123063/127783141-52d3ae77-e02b-4296-a1fb-78ab3bbe5ff3.jpg)

 
### Different uart
The SlimmeLezer is built with a logic inverter on the pcb. Connecting that directly to the Rx of the Wemos, causes that it can't be flashed via USB as it constanly pulls the Rx either high or low. Therefor I'm using the 2nd uart, on pin D7. That's why the uart is specified on pin D7 in the code:
```YAML
uart:
  baud_rate: 115200
  rx_pin: D7
```

