﻿
# openHASP APDS9930 proximity sensor - Custom Code documentation v1.00

![APDS9930 board module](https://github.com/htvekov/openHASP-APDS9930-proximity-sensor-Custom-Code/blob/main/apds9930.PNG) ![Sunton connnectors](https://github.com/htvekov/openHASP-APDS9930-proximity-sensor-Custom-Code/blob/main/sunton.PNG)

### Revision:
- **1.00** (2023-04-29)
Initial Custom Code release

# openHASP Custom Code for APDS9930 proximity sensor

- Custom code for handling `idle_off` command internally in openHASP code upon proximity detection
- Setting of custom brightness level (low or high) depending on APDS9930 sensors ambient light lux value. This will make it possible to turn on the display upon proximity detection at night at low brightness level
- Proximity detection, value and ambient light lux level are written to log and published via MQTT custom topic. Typical time consumption for entire detection, ambient light lux reading, issuing commands / MQTT messages and log writes are some 115 milliseconds

### Custom MQTT message example:

```yaml
topic: hasp/sunton_02/state/custom
payload: {"proximity":620,"lux":5}
```

### Log example with proximity detection on openHASP plate from `idle_long` state:
```yaml
[19:53:46.848][61428/73772 16][22028/24228 10] MSGR: idle=off
[19:53:46.863][61428/70340 12][22028/24228 10] MQTT PUB: idle => off
[19:53:46.878][59380/68408 13][22028/24228 10] CUST: Proximity detected. Level: 817
[19:53:46.894][57332/66468 13][22028/24228 10] CUST: Ambient light lux: 20
[19:53:46.909][55284/66132 16][22028/24228 10] MSGR: backlight={'state':1,'brightness':191}
[19:53:46.924][61428/72176 14][22028/24228 10] HASP: First touch Disabled
[19:53:46.942][59380/68648 13][22028/24228 10] MQTT PUB: backlight => {"state":"on","brightness":191}
[19:53:46.959][59380/70592 15][22028/24228 10] MQTT PUB: custom => {"proximity":817,"lux":20}
[19:53:48.683][61428/73772 16][22028/24228 10] CUST: Clear proximity throttle flag
```

- Proximity detection throttle flag is set upon detection and is reset in the `void custom_every_5seconds()` loop. This effectively reduces time consumption in code for continuous proximity detection to max. once every five seconds
- Ambient light lux value is read every 60 seconds and written to log. Sensor value is also added to- and exposed with openHASPs generic `sensor` MQTT message (`TelePeriod` settings define the interval)

### openHASP sensors MQTT publish example:

```yaml
topic: hasp/sunton_02/state/sensors
payload: {"time":"2023-04-28T21:23:10","uptimeSec":2700,"uptime":"0T00:45:00","lux":5}
```

#### *`brightness_low`, `brightness_high` and `ambient_light_threshold` variables can all be set via custom MQTT command and hence controlled dynamically at runtime 🚀🥳*

### Usage scheme upon proximity detection:

| Ambient light level <= `ambient_light_threshold`   | Ambient light level > `ambient_light_threshold`     |
| -------------------------------------------------- | --------------------------------------------------- |
| `brightness_low` value used for brightness setting | `brightness_high` value used for brightness setting |
 
### Example setting `brightness_low` via HA Developer tools:
```yaml
service: mqtt.publish
data:
  topic: hasp/sunton_02/command/custom/brightness_low
  payload: 25
```

When proximity is registered above defined threshold value, an `idle_off` command is instantly fired internally in the openHASP code (wakeup device). This is followed by a `backlight={'state':1,'brightness':xx}` command, where brightness value is set to either `brightness_low` or `brightness_high`,  depending on what ambient light lux level has been read upon proximity detection

> ***Note***
> When using openHASP Home Assistant Custom Component, the `idle_off` command will also trigger a state and brightness commmand from the CC. As the brightness level from CC is fixed, this will unfortunately instantly owerwrite any brightness setting done in custom code. So until this gets fixed in the CC, a hack is needed in openHASP Custom Components `lights.py` file in order to ignore actions on `idle_off` commands for specific openHASP devices

### Hack example - openHASP Custom Component `lights.py` file:

```python
   async def async_listen_idleness(self):
        """Listen to messages on MQTT for HASP idleness."""

        @callback
        async def idle_message_received(msg):
            """Process MQTT message from plate."""
            message = HASP_IDLE_SCHEMA(msg.payload)

            if message == HASP_IDLE_OFF:
                brightness = self._awake_brightness
                backlight = 1
            elif message == HASP_IDLE_SHORT:
                brightness = self._idle_brightness
                backlight = 1
            elif message == HASP_IDLE_LONG:
                brightness = self._awake_brightness
                backlight = 0
            else:
                return
            #
            # Skip IDLE_OFF commands for specific openHASP devices
            # skip device = device MQTT nodename
            #
            skip_device ="sunton_02"     
            if self.name == skip_device + " backlight" and message == HASP_IDLE_OFF:
                return
            else:
                new_state = {"state": backlight, "brightness": brightness}

            _LOGGER.debug(
                "Idle state for %s is %s - Dimming to %s; Backlight to %s",
                self.name,
                message,
                brightness,
                backlight,
            )
```

### Example pin connection between APDS9930 module board and a 7" Sunton 8048S070C device

| openHASP device | APDS-9930 Board | Function  |
| --------------- | --------------- | --------- |
| 3.3V            | VCC             | Power     |
| 3.3V            | VL              | Power     |
| GND             | GND             | Ground    |
| GPIO19          | SDA             | I²C Data  |
| GPIO20          | SCL             | I²C Clock |
| NC              | INT             | Interrupt |

> ***Note***
> The APDS9930 library shares Wire pin usage with openHASP device touch controller connections. On the 5- and 7" Sunton devices all needed pins are easily accessible via the P3 and P4 JST connectors. Interrupt pin (INT) is not connnected nor used in this Custom Code


### Config keywords:

- Add git+https://github.com/depau/APDS9930.git to lib-deps in `platformio.ini` file
- Copy `my_custom.h` and `my_custom.ccp` files to openhasp/src/custom folder
- Revise `platformio_override.ini` and `user_config_override.h` files
- Patch openHASP Custom Config `lights.py` file in Home Assistant or copy file from this repo instead

Suggestions, improvements, error reporting etc. are very welcome ! 🙂

April, 2023 @htvekov
