# Home Assistant Automations

Patterns for creating powerful automations in Home Assistant.

## When to Use

- Automating lights, climate, security
- Creating presence-based routines
- Integrating smart home devices
- Building notification workflows

## Automation Structure

Every automation has three parts:

```yaml
automation:
  - alias: "Turn on lights at sunset"
    trigger:
      # WHEN this happens...
    condition:
      # AND these are true...
    action:
      # DO this
```

## Triggers

### State Change

```yaml
trigger:
  - platform: state
    entity_id: binary_sensor.front_door
    from: "off"
    to: "on"
```

### Time-Based

```yaml
trigger:
  - platform: time
    at: "07:00:00"
    
  # Or relative to sun
  - platform: sun
    event: sunset
    offset: "-00:30:00"  # 30 min before sunset
```

### Device Trigger

```yaml
trigger:
  - platform: device
    device_id: abc123
    domain: zwave_js
    type: value_notification.entry_control
    command_class: 111
    property: "event"
    property_key: "001"
```

### Numeric State

```yaml
trigger:
  - platform: numeric_state
    entity_id: sensor.temperature
    above: 75
    for: "00:05:00"  # Must be above for 5 minutes
```

### Webhook

```yaml
trigger:
  - platform: webhook
    webhook_id: my_webhook_id
    allowed_methods:
      - POST
```

### Multiple Triggers

```yaml
trigger:
  - platform: state
    entity_id: binary_sensor.motion_living_room
    to: "on"
  - platform: state
    entity_id: binary_sensor.motion_kitchen
    to: "on"
```

## Conditions

### State Condition

```yaml
condition:
  - condition: state
    entity_id: sun.sun
    state: "below_horizon"
```

### Time Condition

```yaml
condition:
  - condition: time
    after: "22:00:00"
    before: "06:00:00"
    weekday:
      - mon
      - tue
      - wed
      - thu
      - fri
```

### Numeric Condition

```yaml
condition:
  - condition: numeric_state
    entity_id: sensor.living_room_temperature
    below: 70
```

### Template Condition

```yaml
condition:
  - condition: template
    value_template: "{{ states('sensor.guests') | int > 0 }}"
```

### AND/OR Logic

```yaml
condition:
  - condition: and
    conditions:
      - condition: state
        entity_id: sun.sun
        state: "below_horizon"
      - condition: or
        conditions:
          - condition: state
            entity_id: person.fred
            state: "home"
          - condition: state
            entity_id: person.alice
            state: "home"
```

## Actions

### Service Call

```yaml
action:
  - service: light.turn_on
    target:
      entity_id: light.living_room
    data:
      brightness_pct: 80
      color_temp: 350
```

### Delay

```yaml
action:
  - service: light.turn_on
    target:
      entity_id: light.porch
  - delay: "00:05:00"
  - service: light.turn_off
    target:
      entity_id: light.porch
```

### Wait for Trigger

```yaml
action:
  - wait_for_trigger:
      - platform: state
        entity_id: binary_sensor.door
        to: "off"
    timeout: "00:10:00"
    continue_on_timeout: true
```

### Choose (If/Else)

```yaml
action:
  - choose:
      - conditions:
          - condition: state
            entity_id: binary_sensor.motion
            state: "on"
        sequence:
          - service: light.turn_on
            target:
              entity_id: light.room
      - conditions:
          - condition: state
            entity_id: binary_sensor.motion
            state: "off"
        sequence:
          - service: light.turn_off
            target:
              entity_id: light.room
    default:
      - service: notify.mobile_app
        data:
          message: "Unknown motion state"
```

### Repeat

```yaml
action:
  - repeat:
      count: 3
      sequence:
        - service: light.toggle
          target:
            entity_id: light.warning
        - delay: "00:00:01"
```

### Notifications

```yaml
action:
  - service: notify.mobile_app_phone
    data:
      title: "Alert"
      message: "Motion detected at {{ now().strftime('%H:%M') }}"
      data:
        image: /local/camera_snapshot.jpg
        actions:
          - action: "DISARM"
            title: "Disarm"
          - action: "IGNORE"
            title: "Ignore"
```

## Complete Examples

### Motion-Activated Lights

```yaml
automation:
  - alias: "Motion lights - Living room"
    trigger:
      - platform: state
        entity_id: binary_sensor.motion_living_room
        to: "on"
    condition:
      - condition: state
        entity_id: sun.sun
        state: "below_horizon"
    action:
      - service: light.turn_on
        target:
          entity_id: light.living_room
        data:
          brightness_pct: 70

  - alias: "Motion lights off - Living room"
    trigger:
      - platform: state
        entity_id: binary_sensor.motion_living_room
        to: "off"
        for: "00:05:00"
    action:
      - service: light.turn_off
        target:
          entity_id: light.living_room
```

### Presence-Based Thermostat

```yaml
automation:
  - alias: "Away mode"
    trigger:
      - platform: state
        entity_id: group.family
        to: "not_home"
        for: "00:15:00"
    action:
      - service: climate.set_temperature
        target:
          entity_id: climate.thermostat
        data:
          temperature: 62

  - alias: "Home mode"
    trigger:
      - platform: state
        entity_id: group.family
        to: "home"
    action:
      - service: climate.set_temperature
        target:
          entity_id: climate.thermostat
        data:
          temperature: 72
```

### Doorbell Notification with Camera

```yaml
automation:
  - alias: "Doorbell pressed"
    trigger:
      - platform: state
        entity_id: binary_sensor.doorbell
        to: "on"
    action:
      - service: camera.snapshot
        target:
          entity_id: camera.front_door
        data:
          filename: /config/www/doorbell.jpg
      - delay: "00:00:02"
      - service: notify.mobile_app_phone
        data:
          title: "Doorbell"
          message: "Someone is at the door"
          data:
            image: /local/doorbell.jpg
            actions:
              - action: "UNLOCK"
                title: "Unlock Door"
```

### Adaptive Lighting

```yaml
automation:
  - alias: "Adaptive lighting"
    trigger:
      - platform: time_pattern
        minutes: "/15"
    condition:
      - condition: state
        entity_id: light.living_room
        state: "on"
    action:
      - service: light.turn_on
        target:
          entity_id: light.living_room
        data:
          color_temp: >
            {% set hour = now().hour %}
            {% if 6 <= hour < 10 %}
              300
            {% elif 10 <= hour < 17 %}
              250
            {% elif 17 <= hour < 21 %}
              350
            {% else %}
              450
            {% endif %}
```

## Templates

### Variables in Actions

```yaml
action:
  - variables:
      room: "{{ trigger.to_state.attributes.friendly_name | replace(' Motion', '') }}"
  - service: notify.mobile_app
    data:
      message: "Motion detected in {{ room }}"
```

### Dynamic Entity Selection

```yaml
action:
  - service: light.turn_on
    target:
      entity_id: "light.{{ trigger.entity_id.split('.')[1] | replace('_motion', '') }}"
```

## Debugging

```yaml
# Add trace logging
logger:
  default: info
  logs:
    homeassistant.components.automation: debug

# Use the trace feature in UI
# Settings > Automations > [Automation] > Traces
```

## Best Practices

1. **Use meaningful aliases** - "Motion lights - Living room" not "Automation 1"
2. **Add `for:` to triggers** - prevents rapid firing
3. **Use `mode: restart`** for retriggerable automations
4. **Test with trace** - visual debugging in UI
5. **Group related automations** - use folders in YAML
6. **Use blueprints** - reusable automation templates
