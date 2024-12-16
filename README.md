### ATTENTION! THE STRUCTURE FOR THIS SENSOR HAS CHANGED!

[Read about the changes here.](https://github.com/jazzyisj/unavailable-entities-sensor/discussions/57)

## What does this package do?

This package creates a group of entities that have no value (a state of unknown or unavailable) and a sensor that provides a count of the entities in this group.

## How do I install it?

### Install As Package

The easiest way to use this sensor is to install it as a [package](https://www.home-assistant.io/docs/configuration/packages/).

If you already have packages enabled in your configuration, download [`package_unavailable_entities.yaml`](https://github.com/jazzyisj/unavailable-entities-sensor/blob/main/package_unavailable_entities.yaml) to your packages directory.

To enable packages in your configuation, create a folder in your config directory named `packages` and add the following line to your `configuration.yaml` file.

    homeassistant:
      packages: !include_dir_named packages

### Install Without Packages

To create this sensor without using packages simply copy the relevant code to an appropriate place in your configuration.yaml file. The automation is optional.

**NOTE!  You must reload automations, templates, and group entities after adding the package or code to your configuration.**

## Customizing The Template

### Ignore Domains

Domains that are "stateless" (button, scene, event etc) or that don't make sense to monitor are excluded by default. To monitor these domains remove them from the ignored domains filter.

Example - remove the `input_button` domain.

    |rejectattr('domain', 'in', ['button', 'event', 'group', 'input_text', 'scene'])

If you wish to ignore additional domains you can also add them to the ignored domains filter.

Example - add the `switch` domain.

    |rejectattr('domain', 'in', ['button', 'event', 'group', 'input_button', 'input_text', 'scene', 'switch'])

### Ignore Specific Entities

Uncomment the `group.ignored_unavailable_entities` declaration and add the entity_id's to ignore to the group entities list.

### Ignore Matching Entities

**Search Test**

If you have multiple entities to ignore that share a common uniquly identifiable portion of their entity_id name you can exclude them without having to add each individual sensor to the ingore_entities group by adding filters using a search test to the template.

    |rejectattr('entity_id', 'search', 'wifi_')

Be as specific as possible in your filters so you don't exclude unintended entities!  For example, if you have the following sensors in your configuration and just want to exclude just the wifi signal strengh sensors, rejecting a search of 'wifi_' will also exclude `binary_sensor.wifi_connected`.

    - binary_sensor.wifi_connected
    - sensor.wifi_downstairs
    - sensor.wifi_upstairs

You can also use [regex pattern matching](https://regex101.com/) in a seach rejectattr (selectattr) filter.

    |rejectattr('entity_id', 'search', '(_alarm_volume|_next_alarm|_alarms)')

The filter above effectively combines these three filters.

    |rejectattr('entity_id', 'search', '_alarm_volume')
    |rejectattr('entity_id', 'search', '_next_alarm')
    |rejectattr('entity_id', 'search', '_alarms')

**Contains Test**

The [contains](https://www.home-assistant.io/docs/configuration/templating/#contains) test can also be used if required. Regex matching is not available for the contains test.

    |rejectattr('entity_id', 'contains', 'wifi_')

### Excluding Specific Integrations

You can exclude entities from a specific integration by using an `in` test for the entity_id and the [integration_entities() function](https://www.home-assistant.io/docs/configuration/templating/#integrations).

    |rejectattr('entity_id', 'in',integration_entities('hassio'))

### Excluding Specific Devices

You can exclude entities from a specific integration by using an `in` test for the entity_id and the [device_entities() function](https://www.home-assistant.io/docs/configuration/templating/#devices).

    |rejectattr('entity_id', 'in',device_entities('fffe8e4c87c68ee60e0ae84c295676ce'))

## Full Example

    automation:
      ~~~
          action:
            - action: group.set
              data:
                object_id: unavailable_entities
                entities: >
                    {{ states
                        | rejectattr('domain', 'in', ['button', 'conversation', 'event', 'group', 'image',
                          'input_button', 'input_text', 'remote', 'tts', 'scene', 'stt'])
                        | rejectattr('entity_id', 'in', state_attr('group.ignored_entities', 'entity_id'))
                        | rejectattr('entity_id', 'eq', 'group.unavailable_entities')
                        | rejectattr('domain', 'in', ['button', 'event', 'group', 'input_button', 'input_text', 'scene'])
                        | rejectattr('entity_id', 'search', 'browser_')
                        | rejectattr('entity_id', 'search', '_alarm_volume|_next_alarm|_alarms')
                        | rejectattr('entity_id', 'contains', '_memory_percent')
                        | rejectattr('entity_id', 'in',integration_entities('hassio'))
                        | rejectattr('entity_id', 'in',device_entities('fffe8e4c87c68ee60e0ae84c295676ce'))
                        | selectattr('state', 'in', ['unknown', 'unavailable'])
                        | map(attribute='entity_id') | list | sort }}

See [Home Assistant Templating](https://www.home-assistant.io/docs/configuration/templating/) additional options.

### Specifing Entities to Monitor - Custom Sensors

You can configure the sensor to only monitor entities you specify instead of monitoring all entities and specifing the entities to ignore by using select or selectattr filters instead of reject and rejectattr filters. Remember, filters are cumlative and entities may be already excluded by previous filters.

This is useful to create sensors that monitor specific domains, integrations etc. You can create as many groups and related sensors as you need.

This example monitors only the `sensor` domain from the Shelly integration that contain the string `_power` in the entity_id.

## Example

    template:
      - sensor:
          - name: "Unavailable Shelly Power Entities"
            unique_id: unavailable_shelly_power_entities
            icon: "{{ iif(states(this.entity_id)|int(-1) > 0, 'mdi:alert-circle', 'mdi:check-circle') }}"
            state_class: measurement
            state: >
              {% set entities = state_attr('group.unavailable_shelly_power_entities', 'entity_id') %}
              {{ entities | count if entities != none else -1 }}

    automation:
      ~~~
          action:
            - action: group.set
              data:
                object_id: unavailable_shelly_power_entities
                entities: >
                    {{ states.sensor
                        | selectattr('entity_id', 'in', integration_entities('shelly'))
                        | selectattr('entity_id', 'contains', '_power')
                        | selectattr('state', 'in', ['unknown', 'unavailable'])
                        | map(attribute='entity_id') | list | sort }}

## Using With Automations
There is an example automation provided in the package that will display unavailable entities as a persistent notification.  You can change this automation to meet your requirements.  This automation is enabled by default.  You can comment it out or delete it if not required.

## Display in the UI
To display a list of unavailable entities open the more-info dialogue of group.unavailable_entities.

![Example](https://github.com/jazzyisj/unavailable-entities-sensor/blob/main/images/group_more_info.png)

### Using Auto Entities and Fold Entity Row
Using the [auto-entities](https://github.com/thomasloven/lovelace-auto-entities) and [fold-entity-row](https://github.com/thomasloven/lovelace-fold-entity-row) plugins is an excellent way to display your unavailable entities sensor in your UI.

[Example Entities Card](https://github.com/jazzyisj/unavailable-entities-sensor/blob/master/examples/auto_entities_card.yaml)

*Closed Fold Entity Row*

![Example](https://github.com/jazzyisj/unavailable-entities-sensor/blob/main/images/entities_card_closed_example.png)

*Open Fold Entity Row*

![Example](https://github.com/jazzyisj/unavailable-entities-sensor/blob/main/images/entities_card_open_example.png)