# ATTENTION! THE STRUCTURE FOR THIS SENSOR HAS CHANGED!
# DOCS HAVE NOT BEEN UPDATED YET

The old version is here [`package_unavailable_entities_old.yaml`](https://github.com/jazzyisj/unavailable-entities-sensor/blob/main/old/package_unavailable_entities_old.yaml).

## What does this template sensor do?
This sensor iterates the state object and returns entities that have a state of unknown or unavailable.

The sensor state is a count of unavailable entities and the entity_id attribute is a list of those entity id's.

## Requirements ##
To use this template as is you must be running at least Home Assistant v2022.5.

## How do I use this sensor?
### Install As Package
The easiest way to use this sensor is to install it as a [package](https://www.home-assistant.io/docs/configuration/packages/).

If you already have packages enabled in your configuration, download [`package_unavailable_entities.yaml`](https://github.com/jazzyisj/unavailable-entities-sensor/blob/main/package_unavailable_entities.yaml) to your packages directory.

To enable packages in your configuation, create a folder in your config directory named `packages` and add the following line to your `configuration.yaml` file.

    homeassistant:
      packages: !include_dir_named packages
### Install Without Packages
To create this sensor without using packages simply copy the relevant template code to an appropriate place in your configuration.yaml file. The ignored entities group and example automation are optional.

**NOTE!  You must reload templates, group entities (if you are using the ignored entities group), and automations (if you are utilizing the example automation) after adding the package or code to your configuration.**
## Customizing The Sensor
### Ignore Seconds
To change the time the sensor will ignore newly available entities that become unavailable, adjust the `ignore_seconds` value.  The default value is 60, meaning a sensor is not reported as unavailable until it's state has been unknown or unavailable for at least 60 seconds.

**A value for 'ignore_seconds' less than 5 seconds may cause template loop warnings in your home assistant log, particularly when template sensors are reloaded.**
### Ignore Domains
Stateless domains (button, scene etc.) are excluded by default.  The group domain is also excluded as many groups will always have a state of `unknown`.

To track these domains remove them from the ignored domains filter.

Example - remove the `input_button` domain.

    |rejectattr('domain','in',['button','event','group','input_text','scene'])

If you wish to ignore additional domains you can add them to the ignored domains filter.

Example - add the `switch` domain.

    |rejectattr('domain','in',['button','event','group','input_button','input_text','scene','switch'])

If you want to monitor the `group` domain, you must add the following filter to the template if you are using the ignored entities group option as this group will usually have a state of `unknown`.

    |rejectattr('entity_id','eq','group.ignored_unavailable_entities')
### Ignore Specific Entities
Uncomment the `group.ignored_unavailable_entities` declaration and add the entity_id's to ignore to the group entities list.
### Ignore Matching Entities
**Search Test**

If you have multiple entities to ignore that share a common uniquly identifiable portion of their entity_id name you can exclude them without having to add each individual sensor to the ingore_entities group adding filters with a search test to the template.

    |rejectattr('entity_id','search','wifi_')

Be as specific as possible in your filters so you don't exclude unintended entities!  For example, You have these sensors in your configuration and want to exclude just the wifi signal strengh sensors. Rejecting a search of 'wifi_' will also exclude the binary connected sensor.

    - binary_sensor.wifi_connected
    - sensor.wifi_downstairs
    - sensor.wifi_upstairs

You can also use [regex pattern matching](https://regex101.com/) in a seach rejectattr (selectattr) filter.

    |rejectattr('entity_id','search','(_alarm_volume|_next_alarm|_alarms)')

The filter above effectively combines these three filters.

    |rejectattr('entity_id','search','_alarm_volume')
    |rejectattr('entity_id','search','_next_alarm')
    |rejectattr('entity_id','search','_alarms')

**Contains Test**

The [contains](https://www.home-assistant.io/docs/configuration/templating/#contains) test can also be used if required. Regex matching is not available for the contains test.

    |rejectattr('entity_id','contains','wifi_')

### Excluding Specific Integrations
You can exclude entities from a specific integration by using an `in` test for the entity_id and the [integration_entities() function](https://www.home-assistant.io/docs/configuration/templating/#integrations).

    |rejectattr('entity_id','in',integration_entities('hassio'))

### Excluding Specific Devices
You can exclude entities from a specific integration by using an `in` test for the entity_id and the [device_entities() function](https://www.home-assistant.io/docs/configuration/templating/#devices).

    |rejectattr('entity_id','in',device_entities('fffe8e4c87c68ee60e0ae84c295676ce'))
## Full Example
    template:
        - sensor:
            - name: "Unavailable Entities"
                unique_id: unavailable_entities
                icon: "{{ iif(states(this.entity_id)|int(-1) > 0,'mdi:alert-circle','mdi:check-circle') }}"
                state_class: measurement
                unit_of_measurement: entities
                state: >
                {% set entities = state_attr(this.entity_id,'entity_id') %}
                {{ entities|count if entities != none else none }}
                attributes:
                entity_id: >
                    {% set ignore_seconds = 60 %}
                    {% set ignored = state_attr('group.ignored_unavailable_entities','entity_id') %}
                    {% set ignore_ts = (now().timestamp() - ignore_seconds)|as_datetime %}
                    {% set entities = states
                        |rejectattr('domain','in',['button','event','group','input_button','input_text','scene'])
                        |rejectattr('entity_id','search','browser_')
                        |rejectattr('entity_id','search','_alarm_volume|_next_alarm|_alarms')
                        |rejectattr('entity_id','contains','_memory_percent')
                        |rejectattr('entity_id','in',integration_entities('hassio'))
                        |rejectattr('entity_id','in',device_entities('fffe8e4c87c68ee60e0ae84c295676ce'))
                        |rejectattr('last_changed','ge',ignore_ts) %}
                    {% set entities =  entities|rejectattr('entity_id','in',ignored) if ignored != none else entities %}
                    {{ entities|map(attribute='entity_id')|reject('has_value')|list|sort }}

See [Home Assistant Templating](https://www.home-assistant.io/docs/configuration/templating/) additional options.
### Specifing Entities to Monitor
You can configure the sensor to only monitor entities you specify instead of monitoring all entities and specifing the entities to ignore by using select or selectattr filters instead of reject and rejectattr filters. Remember, filters are cumlative and entities may be already excluded by previous filters.

This example monitors only the `sensor` domain from the Shelly integration that contain the string "_power" in their entity_id.
## Example
    template:
        - sensor:
            - name: "Unavailable Entities"
                unique_id: unavailable_entities
                icon: "{{ iif(states(this.entity_id)|int(-1) > 0,'mdi:alert-circle','mdi:check-circle') }}"
                state_class: measurement
                unit_of_measurement: entities
                state: >
                    {% set ignore_seconds = 60 %}
                    {% set ignored = state_attr('group.ignored_unavailable_entities','entity_id') %}
                    {% set ignore_ts = (now().timestamp() - ignore_seconds)|as_datetime %}
                    {% set entities = states.sensor
                        |selectattr('entity_id','in',integration_entities('shelly'))
                        |selectattr('entity_id','contains','_power')
                        |rejectattr('last_changed','ge',ignore_ts) %}
                    {% set entities =  entities|rejectattr('entity_id','in',ignored) if ignored != none else entities %}
                    {{ entities|map(attribute='entity_id')|reject('has_value')|list|sort }}

## Multiple Sensors
This sensor is simply a template sensor.  You can create as many of them as you need in your configuration.  I have many similar sensors in [my configuration](https://github.com/jazzyisj/home-assistant-config) such as an [offline zwave devices binary sensor](https://github.com/jazzyisj/home-assistant-config/blob/master/templates/template_zwave.yaml#L43) and an [unavailble media players sensor](https://github.com/jazzyisj/home-assistant-config/blob/master/templates/media/template_media_players.yaml#L28) in addition to the default sensor defined in this package.
## Using in Automations
There is an example automation provided in the package that will display unavailable entities as a persistent notification.  You can change this automation to meet your requirements.  This automation is enabled by default.  You can comment it out or delete it if not required.

## Display in the UI
By default the `entity_id` attribute of an entity is not displayed in a more_info dialogue to increase the effeciency of the HA databae.  This template follows that standard.  If you wish to display the entity id's in the more info dialogue, change the name of the `entity_id` attribute to something else in the template configuration.  Don't forget to change it in the state template too!

    state: >
        {% set entities = state_attr(this.entity_id,'entities') %}
        {{ entities|count if entities != none else none }}
    attributes:
        entities: >

### Using Auto Entities and Fold Entity Row
Using the [auto-entities](https://github.com/thomasloven/lovelace-auto-entities) and [fold-entity-row](https://github.com/thomasloven/lovelace-fold-entity-row) plugins is an excellent way to display your unavailable entities sensor in your UI.

[Example Entities Card](https://github.com/jazzyisj/unavailable-entities-sensor/blob/master/examples/auto_entities_card.yaml)

*Closed Fold Entity Row*

![Example](https://github.com/jazzyisj/unavailable-entities-sensor/blob/main/images/entities_card_closed_example.png)

*Open Fold Entity Row*

![Example](https://github.com/jazzyisj/unavailable-entities-sensor/blob/main/images/entities_card_open_example.png)