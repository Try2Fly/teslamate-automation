# teslamate-automation

## Introducton

Following the example of @ngardiner I made some automations for charging my tesla.
My idea is to have 3 separate options:
- straight to 90%
- charge to 80%
- smart charge (charge to 80% and 1,5 hour before I need to leave start charging to 90%)
I haven't tested everything, as my charger at home still needs to be installed.

But it will look like this:
## Screenshot
![screenshot](https://user-images.githubusercontent.com/18633213/66951350-714b6480-f05a-11e9-9928-d9055440c03a.png)


## Configuration

for this I added the following components

First I added the following MQTT sensors which get their value from Teslamate:
## mqtt sensors
```
- platform: mqtt
  state_topic: "teslamate/cars/1/state"
  name: "tesla_state"
  icon: mdi:car-electric
  value_template: >
    {% if value == 'asleep' %}
      Asleep
    {% elif value == 'charging' %}
      Charging
    {% elif value == 'charging_complete' %}
      Charging Complete
    {% elif value == 'driving' %}
      Driving
    {% elif value == 'offline' %}
      Offline
    {% elif value == 'online' %}
      Online
    {% elif value == 'suspended' %}
      Trying to sleep
    {% elif value == 'updating' %}
      Installing Update
    {% else %}
      Unknown Status
    {% endif %}
- platform: mqtt
  state_topic: "teslamate/cars/1/locked"
  name: "tesla_locked"
  icon: mdi:lock-question
  value_template: >
    {% if value == 'true' %}
      Yes
    {% elif value == 'false' %}
      No
    {% else %}
      Unknown Status
    {% endif %}
- platform: mqtt
  state_topic: "teslamate/cars/1/plugged_in"
  name: "tesla_plugged_in"
  icon: mdi:power-plug
  value_template: >
    {% if value == 'true' %}
      Yes
    {% elif value == 'false' %}
      No
    {% else %}
      Unknown Status
    {% endif %}
- platform: mqtt
  state_topic: "teslamate/cars/1/sentry_mode"
  name: "tesla_sentry_mode"
  icon: mdi:alarm-light
  value_template: >
    {% if value == 'true' %}
      Yes
    {% elif value == 'false' %}
      No
    {% else %}
      Unknown Status
    {% endif %}
- platform: mqtt
  state_topic: "teslamate/cars/1/battery_level"
  name: "tesla_battery_level"
  icon: mdi:battery-80
  unit_of_measurement: "%"
  value_template: '{{ value }}'

- platform: mqtt
  state_topic: "teslamate/cars/1/charge_limit_soc"
  name: "tesla_charge_limit_soc"
  icon: mdi:battery-plus
  unit_of_measurement: "%"
  value_template: '{{ value }}'

- platform: mqtt
  state_topic: "teslamate/cars/1/ideal_battery_range_km"
  name: "tesla_ideal_battery_range_km"
  icon: mdi:map-marker-distance
  unit_of_measurement: "km"
  value_template: '{{ value | float }}'

- platform: mqtt
  state_topic: "teslamate/cars/1/est_battery_range_km"
  name: "tesla_est_battery_range_km"
  icon: mdi:map-marker-distance
  unit_of_measurement: "km"
  value_template: '{{ value | float }}'

- platform: mqtt
  name: tesla_latitude
  state_topic: "teslamate/cars/1/latitude"
  icon: mdi:crosshairs-gps

- platform: mqtt
  name: tesla_longitude
  state_topic: "teslamate/cars/1/longitude"
  icon: mdi:crosshairs-gps
````

##input datetime
Input datetime was added to set a time at which you plan to leave
```
input_datetime:
  tesla_leave:
    name: tesla leave time
    has_date: false
    has_time: true
```
## input boolean
Input boolean was added to control the choise of charging options
```
input_boolean:
  - charge_90_tesla:
      name: max charge
  - charge_80_tesla:
      name: charge to 80
  - charge_smart_tesla:
      name: smart charge
```

## input number
Input number was added to set a charge level at which you want charging to stop (at this moment it is not possible to set the charge level directly in the tesla, through the home assistant tesla component)
```
input_number:
  - tesla_charge_limit:
      min: 0
      max: 100
      unit_of_measurement: '%'
      icon: mdi:battery-charging-100
```

## template sensors and time_date sensor
I created sensors to calculate the time at which charging needs to start before you leave. In this case 1,5 hour (5600 seconds) to charge from 80% to 90%. And half hour before you need to leave the heating starts
```
sensor:
  - platform: template
    sensors:
    tesla_time_offset_charge:
      friendly_name: smart charge start
      value_template: "{{ (state_attr('input_datetime.tesla_leave', 'timestamp') - 5400) | timestamp_custom('%H:%M', false) }}"
      icon_template:  mdi:history
    tesla_time_offset_heat:
      friendly_name: smart heat start
      value_template: "{{ (state_attr('input_datetime.tesla_leave', 'timestamp') - 1800) | timestamp_custom('%H:%M', false) }}"
      icon_template: >-
          {% if is_state('climate.tesla_model_3_hvac_climate_system', 'off') %}
            mdi:car-seat
          {% else %}
            mdi:car-seat-heater
          {% endif %}
  - platform: time_date
    display_options:
      - 'time'
```
## template switch
Unfortunately it is not possible to use input_booleans in the picture-glance card of Lovelace. So I created template switches. Which also gave me te possibility to turn off an input boolean when another one is turned on.
```
switch:
  - platform: template
    switches:
      tesla_80:
        value_template: "{{ states('input_boolean.charge_80_tesla') }}"
        turn_on:
          - service: input_boolean.turn_on
            entity_id: input_boolean.charge_80_tesla
          - service: input_boolean.turn_off
            entity_id: input_boolean.charge_90_tesla
          - service: input_boolean.turn_off
            entity_id: input_boolean.charge_smart_tesla
        turn_off:
          - service: input_boolean.turn_on
            entity_id: input_boolean.charge_smart_tesla
      tesla_90:
        value_template: "{{ states('input_boolean.charge_90_tesla') }}"
        turn_on:
          - service: input_boolean.turn_on
            entity_id: input_boolean.charge_90_tesla
          - service: input_boolean.turn_off
            entity_id: input_boolean.charge_80_tesla
          - service: input_boolean.turn_off
            entity_id: input_boolean.charge_smart_tesla
        turn_off:
          - service: input_boolean.turn_on
            entity_id: input_boolean.charge_smart_tesla
      tesla_smart:
        value_template: "{{ states('input_boolean.charge_smart_tesla') }}"
        turn_on:
          - service: input_boolean.turn_on
            entity_id: input_boolean.charge_smart_tesla
          - service: input_boolean.turn_off
            entity_id: input_boolean.charge_90_tesla
          - service: input_boolean.turn_off
            entity_id: input_boolean.charge_80_tesla
        turn_off:
          - service: input_boolean.turn_on
            entity_id: input_boolean.charge_smart_tesla
```

## tracking car
to Track the car I used a system @ngardiner wrote down
```
automation:
- alias: Update Tesla location as MQTT location updates
  initial_state: on
  trigger:
    - platform: mqtt
      topic: teslamate/cars/1/latitude
    - platform: mqtt
      topic: teslamate/cars/1/longitude
  action:
    - service: device_tracker.see
      data_template:
        dev_id: tesla_location
        location_name: not_home
        gps: [ '{{ states.sensor.tesla_latitude.state }}', '{{ states.sensor.tesla_longitude.state }}' ]

proximity:
  home_tesla:
    zone: home
    devices:
      - device_tracker.tesla_location
    tolerance: 10
    unit_of_measurement: km

tesla:
  username: !secret tesla_username
  password: !secret tesla_password
  scan_interval: 3600

known_device:
tesla_location:
  hide_if_away: false
  icon: mdi:car
  mac:
  name: Tesla
  picture:
  track: true
```

### charging automations

Next are the automations for the 3 options: charge to 80%, charge to 90%, smart charge. NOTE, this is all with the assumption that the charge limit in the car itself is set to 90%.

first for charging to 80%

```
- alias: tesla max charge 80
  initial_state: on
  trigger:
    - platform: state
      entity_id: sensor.tesla_plugged_in
      to: "yes"
    - platform: state
      entity_id: input_boolean.charge_80_tesla
      to: 'on'
  condition:
    condition: and
    conditions:
      - condition: numeric_state
        entity_id: proximity.home_tesla
        below: 1
      - condition: state
        entity_id: sensor.tesla_plugged_in
        state: "yes"
      - condition: state
        entity_id: input_boolean.charge_80_tesla
        state: 'on'
  action:
    - service: input_number.set_value
      data:
        entity_id: input_number.tesla_charge_limit
        value: "80"
    - service: switch.turn_on
      entity_id: switch.tesla_model_3_charger_switch
``` 

Or charge straight to 90%
```
- alias: tesla max charge 90
  initial_state: on
  trigger:
    - platform: state
      entity_id: sensor.tesla_plugged_in
      to: "yes"
    - platform: state
      entity_id: input_boolean.charge_90_tesla
      to: 'on'
  condition:
    condition: and
    conditions:
      - condition: numeric_state
        entity_id: proximity.home_tesla
        below: 1
      - condition: state
        entity_id: sensor.tesla_plugged_in
        state: "yes"
      - condition: state
        entity_id: input_boolean.charge_90_tesla
        state: 'on'
  action:
    - service: input_number.set_value
      data:
        entity_id: input_number.tesla_charge_limit
        value: "90"
    - service: switch.turn_on
      entity_id: switch.tesla_model_3_charger_switch
```

or start smart charging. This is 2 automations. First one is to charge to 80%. The second one is to start charging to 90% when it is 1,5 hour before the set leaving time

```
- alias: tesla smart charge
  initial_state: on
  trigger:
    - platform: state
      entity_id: sensor.tesla_plugged_in
      to: "yes"
    - platform: state
      entity_id: input_boolean.charge_smart_tesla
      to: 'on'
  condition:
    condition: and
    conditions:
      - condition: numeric_state
        entity_id: proximity.home_tesla
        below: 1
      - condition: state
        entity_id: sensor.tesla_plugged_in
        state: "yes"
      - condition: state
        entity_id: input_boolean.charge_smart_tesla
        state: 'on'
  action:
    - service: input_number.set_value
      data:
        entity_id: input_number.tesla_charge_limit
        value: "80"
    - service: switch.turn_on
      entity_id: switch.tesla_model_3_charger_switch

- id: tesla smart charge start
  alias: 'Smart charge start'
  trigger:
    platform: template
    value_template: "{{ states('sensor.time') == states('sensor.tesla_time_offset_charge') }}"
  condition:
    condition: and
    conditions:
      - condition: numeric_state
        entity_id: proximity.home_tesla
        below: 1
      - condition: state
        entity_id: input_boolean.charge_smart_tesla
        state: 'on'
  action:
    - service: input_number.set_value
      data:
        entity_id: input_number.tesla_charge_limit
        value: "90"
    - service: switch.turn_on
      entity_id: switch.tesla_model_3_charger_switch
```

Then next automation is to stop the charging when the set charge limit is reached

```
- alias: tesla stop charge
  initial_state: on
  trigger:
    - platform: template
      value_template : "{{ states('sensor.tesla_battery_level') == states('input_number.tesla_charge_limit) " 
  condition:
    condition: numeric_state
    entity_id: proximity.home_tesla
    below: 1
  action:
    - service: switch.turn_off
      entity_id: switch.tesla_model_3_charger_switch
```
## heating automations
At last I also created an automation to start heating the car half a hour befor you leave. 

```
- id: tesla smart heating start
  alias: 'Smart heating start'
  trigger:
    platform: template
    value_template: "{{ states('sensor.time') == states('sensor.tesla_time_offset_heat') }}"
  condition:
    condition: and
    conditions:
      - condition: numeric_state
        entity_id: proximity.home_tesla
        below: 1
      - condition: state
        entity_id: input_boolean.charge_smart_tesla
        state: 'on'
      - condition: state
        entity_id: sensor.tesla_plugged_in
        state: "yes"
  action:
    - service: climate.turn_on
      entity_id: climate.tesla_model_3_hvac_climate_system
```

### ui-lovelace

Next I made 2 lovelace cards with a picture of the tesla in the www folder.

```
cards:
  - cards:
      - entity: sensor.tesla_battery_level
        name: charge level
        severity:
          green: 80
          red: 30
          yellow: 60
        type: gauge
      - entity: climate.tesla_model_3_hvac_climate_system
        name: temp
        type: thermostat
    type: horizontal-stack
  - entities:
      - switch.tesla_80
      - switch.tesla_90
      - switch.tesla_smart
    image: /local/red.jpg
    title: tesla
    type: picture-glance
  - entities:
      - entity: input_datetime.tesla_leave
      - entity: sensor.tesla_time_offset_charge
      - entity: sensor.tesla_time_offset_heat
      - entity: input_number.tesla_charge_limit
    show_header_toggle: false
    title: Smart charge
    type: entities
type: vertical-stack
```

```
cards:
  - type: map
    entities:
      - device_tracker.tesla_location
  - entities:
      - entity: sensor.tesla_state
      - entity: sensor.tesla_locked
      - entity: sensor.tesla_est_battery_range_km
      - entity: sensor.tesla_battery_level
      - entity: sensor.tesla_plugged_in
      - entity: sensor.tesla_sentry_mode
      - entity: sensor.tesla_charge_limit_soc
      - entity: sensor.tesla_ideal_battery_range_km
    type: entities
type: vertical-stack
```
