## What does this template sensor do?
This sensor iterates the state object and returns entities that have a state of unknown or unavailable.

The sensor state is a count of unavailable entities and the entity_id attribute is a list of those entity id's.

## Requirements ##
To use this template as is you must be running at least Home Assistant v2022.5.

## How do I use this sensor?
### Install Package
The easiest way to use this sensor is to install it as a [package](https://www.home-assistant.io/docs/configuration/packages/).

If you already have packages enabled in your configuration, simply download package_unavailable_entities.yaml to your packages directory.

To enable packages in your configuation, create a folder in your config directory named packages and add the following line to your configuration.yaml file.

    homeassistant:
      packages: !include_dir_named packages
### Install Without Pacakges
To create this sensor without using packages simply copy the relevant template code to an appropriate place in your configuration.yaml file. The ignored entities group and example automation are optional. 

## Customizing The Sensor
There are several things you can do to customize the results of this sensor to meet your requirments.

### Ignore Seconds
To change the time the sensor will ignore newly available entities that become unavailable, adjust the `ignore_seconds` value.  The default value is 60, meaning a sensor is not reported as unavailable until it's state has been unknown or unavailable for at least 60 seconds.

**A value for 'ignore_seconds' less than 5 seconds may cause template loop warnings in your home assistant log, particularly when template sensors are reloaded.**
### Ignore Domains
If you wish to ignore additional domains besides the default ignored group domain you can replace the filter

    |rejectattr('domain','eq','group')
With the following to ignore, for example, the button and number domains.

    |rejectattr('domain','in',['group','button','number'])

**Some domains will report a state of unknown until they have been given a state for the first time.  Buttons and scenes are examples of this.  Once a button has been pressed at least once, or a scene has been activated at least once they will no longer have a state of unknown.**
### Ignore Specific Entities
Uncomment the `group.ignored_unavailable_entities definition` and add the entity to the group entities list.
### Ignore Matching Entities
If you have several entities to ignore that share a common uniquly identifiable portion of their entity_id name you can exclude them without adding each individual sensor
to the ingore_entities group by adding a rejectattr filter using a search test.  You can add as many of these filters as you need. Be as specific as possible in your filters so you don't exclude unintended entities!  

Eg You have these sensors in your configuration. And want to exclude just the wifi signal strengh sensors. Rejecting a search of 'wifi_' will also exclude the binary connected sensor.

    - binary_sensor.wifi_connected
    - sensor.wifi_downstairs
    - sensor.wifi_upstairs

**RegEx Filters**

You can also use [regex pattern matching](https://regex101.com/) with search in a rejectattr (selectattr) filter.

    |rejectattr('entity_id', 'search', '(_alarm_volume|_next_alarm|_alarms)')

Would combine these three filters

    |rejectattr('entity_id','search','_alarm_volume')
    |rejectattr('entity_id','search','_next_alarm')
    |rejectattr('entity_id','search','_alarms')

### Monitoring Groups
Group entities either have a [calculated state](https://www.home-assistant.io/integrations/group/#group-state-calculation) or a state of **unknown** meaning there usually is no point to monitoring group entities for a state of unknown.  For this reason by default group entities are not reported by this sensor.  This will not exclude specific domain group entities such as light groups or switch groups because those entities become part of the light or switch domains, not the group domain. 

If you do wish to monitor group states this can be accomplished deleting the group domain rejectattr filter `|rejectattr('domain',eq,'group')` or replacing it with `|rejectattr('entity_id','eq','group.ignored_unavailable_entities')` if you are using the ignored entities group option.

## Examples
    {% set ignore_sec = 60 %}
    {% set ignore_ts = (now().timestamp() - ignore_sec)|as_datetime %}
    {{ states.sensor
      |rejectattr('entity_id','in',state_attr('group.ignored_unavailable_entities','entity_id'))
      |rejectattr('entity_id','search','_alarm_volume|_next_alarm|_alarms')
      |rejectattr('entity_id','search','_memory_percent|_cpu_percent')
      |rejectattr('entity_id','search','_timers|_device|_do_not_disturb')
      |rejectattr('entity_id','search','browser_')
      |rejectattr('last_changed','ge',ignore_ts)
      |selectattr('state','in',['unavailable','unknown','none'])|map(attribute='entity_id')|list }}
