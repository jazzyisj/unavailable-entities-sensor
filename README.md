## What does this template sensor do?
This sensor iterates the state object and returns entities that have a state of unknown, unavailable, or null (none).

The sensor state is a count of unavailable entities and the entities attribute is a list of those entity id's.

## How do I install this sensor?
### Install Package
The easiest way to use this sensor is to install it as a [package](https://www.home-assistant.io/docs/configuration/packages/).

If you already have packages enabled in your configuration, simply download package_unavailable_entities.yaml to your packages directory.

To enable packages in your configuation, create a folder in your config directory named packages and add the following line to your configuration.yaml file.

    homeassistant:
      packages: /config/packages
### Install Without Pacakges
To create this sensor without installing as a package simply copy the relevant code and paste in an appropriate place in your configuration.yaml file. The logger filter and example automation are optional. **The template sensor AND the ignored_entities group ARE BOTH REQURED for the default template to render.**
## Customizing The Sensor
There are several things you can do to customize the results of this sensor to meet your requirments.

[Monitoring Different States](#monitoring-different-states#)]

### Remove Ignore Group
If do not want to use the group.ignored_unavailable_entities group you must also delete the following filter from the template sensor.

    |rejectattr('entity_id','in',state_attr('group.ignored_entities','entity_id'))

### Ignore Seconds
To change the time the sensor will ignore newly available entities that become unavailable, adjust the `ignore_seconds` value.  The default value is 60, meaning a sensor is not reported as unavailable until it's state has been unknown, unavailable, or null(none) for at least 60 seconds.

**Note: A value for 'ignore_seconds' less than 5 seconds may cause template loop warnings in your home assistant log, particularly when template sensors are reloaded.**
### Ignore Domains
If you wish to ignore additional domains besides the default ignored group domain you can replace the filter

    |rejectattr('domain','eq','group')
With this to ignore button and number domains.

    |rejectattr('domain','in',['group','button','number'])
### Monitoring Groups
 **The group domain should usually be ignored.**   Any group that has entities that has entities that aren't utilized by the group domain will have a state of unknown which more or less results in a permanent items listed in your sensor.  However if you do wish to monitor group states for some reason this can be accomplished by changing the group domain rejectattr filter `|rejectattr('domain',eq,'group')` to reject just the entity group.ignored_unavailable_entities `|rejectattr('entity_id','eq','group.ignored_unavailable_entities')`.
### Ignore Matching Entities
If you have several entities to ignore that share a common uniquly identifiable portion of their entity_id name you can exclude them without adding each individual sensor
to the ingore_entities group by adding a rejectattr filter using a search test.  You can add as many of these filters as you need. Be as specific as possible in your filters so you don't exclude unintended entities!  If the entities you want don't have a specific enough string to use and they have a unique_id in HA you can rename them in the UI using a more specific string in the entity_id if necessary.

Eg You have these sensors in your configuration. And want to exclude just the wifi signal strengh sensors. Rejecting a search of 'wifi_' will also exclude the binary connected sensor.
    - binary_sensor.wifi_connected
    - sensor.wifi_downstairs
    - sensor.wifi_upstairs

Rename the signal strengh sensors with amore specific name and update the filter with the new search.

    - sensor.wifi_strength_downstairs
    - sensor.wifi_strength_upstairs

    |rejectattr('entity_id','search','wifi_strength_')

Now the signal strength WIFI sensors will be rejected but the WIFI connected sensor will still be monitored.

**RegEx Filters**

You can also use [regex pattern matching](https://regex101.com/) with search in a rejectattr (selectattr) filter.

    |rejectattr('entity_id', 'search', '(_alarm_volume|_next_alarm|_alarms)')

Would combine these three filters

    |rejectattr('entity_id','search','_alarm_volume')
    |rejectattr('entity_id','search','_next_alarm')
    |rejectattr('entity_id','search','_alarms')

This is an [example of this sensor](https://github.com/jazzyisj/) that contains several filters.

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

## Monitor Specified Entities
### Single Domain
To only monitor one domain, you can limit the states object search to that domain.  Note the group domain rejectattr filter is not required in this case because we are only monitoring the light domain.

    {{ states.light
      |rejectattr('entity_id','in',state_attr('group.ignored_unavailable_entities','entity_id'))
      |selectattr('state','in',['unavailable','unknown','none'])|map(attribute='entity_id')|list }}

### Multiple Domains
If you wish to monitor more than one specified domain you can use a selectattr filter to select a list of domains.  The group domain rejectattr filter is also not required here.

    {{ states|selectattr('domain','in',['sensor','binary_sensor'])
      |rejectattr('entity_id','in',state_attr('group.ignored_unavailable_entities','entity_id'))
      |selectattr('state','in',['unavailable','unknown','none'])|map(attribute='entity_id')|list }}

### Group of Entities
    {{ states
      |rejectattr('domain == 'group')
      |selectattr('entity_id','in',state_attr('group.monitored_unavailable_entities','entity_id'))
      |selectattr('state','in',['unavailable','unknown','none'])|map(attribute='entity_id')|list }}

**Note the group name change to something that makes more sense.**
    group:
      monitored_unavailable_entities:
        entities:
          - sensor.some_entity
          - binary_sensor.some_other_entity
## Monitoring Different States
Although out of the scope of the purposes of the unavailable entities sensor, if you'd like to monitor a different state other than unavailable you can do that too!

The following template would return all lights that have been on for more than 15 minutes.

    {% set ignore_seconds = 900 %}
    {% set ignore_ts = (now().timestamp() - ignore_seconds)|as_datetime %}
    {{ states.light
      |rejectattr('last_changed','ge',ignore_ts)
      |selectattr('state','in','on')|map(attribute='entity_id')|list }}

## What is the log filter for?
Some users have reported occasional template warnings in their Home Assistant log, especially when reloading templates.

This warning is inconsequential and does not affect the sensor operation.  The logger filter suppresses these warnings.

**NOTE: Enabling this filter will suppress template loop warnings for ALL template sensors**

Delete or comment out the logger filter code if you do not want template loop warnings supressed.

## Multiple Sensors
You can add as many version of these sensors as you need.  See the [customized_unavailable_sensor_examples_package.yaml[(https://github.com/jazzyisj/unavailable-entities-sensor/blob/main/examples/example_customized_sensor_package.yaml) for examples how to do add
additional sensors to the package.