# Replacement Control Board for the Keg Master Series 3 (Model: BC168)

I acquired a Keg Master Series 3 kegerator a while ago. While it's a nice fridge, it was not without it's issues.

1. If power was cut, it would revert back to a default temperature (0.4°C degrees if I remember correctly), and displaying fahrenheit (why!). Beyond the obvious issue of it forgetting the set temperature, this was more an issue because:
2. It was running cold. Any lower than 0.4°C and the kegerator is dangerously close to below freezing. I've managed to freeze a keg and numerous bottles.

So, the obvious thing to do was to make a new controller!

This controller has a ESP32, as well as a 128x32 i²c LCD. 

I've replaced the thermistor with a Dallas ds18b20 temperature sensor (in parasitic mode so I can reuse the two-wires), and added a line for a heater relay so the board can be reused for a fermenter.

The front buttons are wired up as well. The esphome firmware I use switches out the "unit" button with a on/off button.

## Example esphome configuration

```yaml
esphome:
  name: kegerator
  
esp32:
  board: esp32-s2-saola-1

i2c:
  sda: GPIO16
  scl: GPIO15

font:
  - file: "gfonts://Roboto"
    id: font_large
    size: 18
  - file: "gfonts://Roboto"
    id: font_small
    size: 8

display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x32"
    lambda: |-
      float current_temperature = id(kegerator_pid).current_temperature;
      float target_temperature = id(kegerator_pid).target_temperature;
      
      if(id(kegerator_pid).mode == CLIMATE_MODE_COOL) {
        if(!std::isnan(current_temperature)) {
          it.printf(64, 0, id(font_large), TextAlign::TOP_CENTER, "%.1f°C", current_temperature);
        } else {
          it.printf(64, 0, id(font_large), TextAlign::TOP_CENTER, "--°C");
        }
      } else {
          it.printf(64, 0, id(font_large), TextAlign::TOP_CENTER, "Off");
      } 

      if(!std::isnan(target_temperature)) {
        it.printf(64, 32, id(font_small), TextAlign::BOTTOM_CENTER, "%.1f°C", target_temperature);
      } else {
        it.printf(64, 32, id(font_small), TextAlign::BOTTOM_CENTER, "--°C");
      }
      
one_wire:
  - platform: gpio
    pin: GPIO14

sensor:
  - platform: dallas_temp
    id: temperature
    name: "Temperature"
    update_interval: 1s
    unit_of_measurement: "°C"
    device_class: "temperature"
    state_class: "measurement"
    accuracy_decimals: 1
    filters:
      - exponential_moving_average:
          alpha: 0.8
          send_every: 30

binary_sensor:
  - platform: gpio
    id: btn_temp_up
    pin:
      number: GPIO21
      inverted: true
      mode:
          input: true
          pullup: true
    filters:
       autorepeat:
        - delay: 1s
          time_off: 100ms
          time_on: 100ms

    on_click:
      lambda: |-
        float target_temperature = id(kegerator_pid).target_temperature;
        auto call = id(kegerator_pid).make_call();
        call.set_target_temperature(target_temperature + 0.1);
        call.perform();
 
  - platform: gpio
    id: btn_temp_down
    pin: 
      number: GPIO20
      inverted: true
      mode:
          input: true
          pullup: true
    filters:
       autorepeat:
        - delay: 1s
          time_off: 100ms
          time_on: 100ms
    on_click:
      lambda: |-
        float target_temperature = id(kegerator_pid).target_temperature;
        auto call = id(kegerator_pid).make_call();
        call.set_target_temperature(target_temperature - 0.1);
        call.perform();

  - platform: gpio
    id: btn_units
    pin: 
      number: GPIO19     
      inverted: true
      mode:
          input: true
          pullup: true
    on_click:
      then:
        lambda: |-
          auto call = id(kegerator_pid).make_call();
          if(id(kegerator_pid).mode == CLIMATE_MODE_COOL) {
            call.set_mode("OFF");
          } else {
            call.set_mode("COOL");
          }
          call.perform();


button:
  - platform: template
    name: "Kegerator PID Climate Autotune"
    on_press:
      - climate.pid.autotune: kegerator_pid

output:
  id: compressor
  platform: slow_pwm
  pin: GPIO18
  period: 1200s

climate:
  - platform: pid
    id: kegerator_pid
    name: "Kegerator"
    sensor: temperature
    default_target_temperature: 7°C
    cool_output: compressor
    visual:
      min_temperature: -7
      max_temperature: 32
      temperature_step: 0.1
    control_parameters:
      kp: 0.19714
      ki: 0.00028
      kd: 34.89431

logger:

api:
  password: ""

ota:
  - platform: esphome
    password: ""

wifi:
  ssid: WIFI_SSID
  password: WIFI_PASSWORD

  ap:
    ssid: "Kegerator"
    password: ""

captive_portal:
```
