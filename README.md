## What does this template sensor do?
The sensor's entities attribute is a list object of entity id's of entities that have a state of unknown, unavailable, or null (none)?

The state of the sensor is a count of entity id's in the entities attribute.
## How do I install this sensor?
### Install Package
The easiest way to use this sensor is to install it as a [package](https://www.home-assistant.io/docs/configuration/packages/).

If you already have packages enabled in your configuration, simply download package_unavailable_entities.yaml to your packages directory.

To enable packages in your configuation, create a folder in your config directory named packages and add the following line to your configuration.yaml file.

    homeassistant:
      packages: /config/packages
### Install Without Pacakges
To create this sensor without installing as a package simply copy the relevant code and paste in an appropriate place in your configuration.yaml file.

**The template sensor AND the ignored_entities group ARE BOTH REQURED for the template to render.**

If do not want to use the group.ignored_entities group you must also delete the following filter from the template sensor.

    |rejectattr('entity_id','in',state_attr('group.ignored_entities','entity_id'))

The logger filter and example automation are both optional.
## Customizing The Sensor
There are several things you can do to customize the results of this sensor.
### Ignore Seconds
To prevent the sensor from reporting entities that are unavailable for short intervals you can adjust the `ignore_seconds` value.

The default value is 60, meaning a sensor is not reported as unavailable until it's state has been unknown, unavailable, or null(none) for at least 60 seconds.
The miniumun recommended value for 'ignore_seconds' is 5 to prevent template warnings.
### Ignore Domains
If you wish to ignore additional domains besides the default ignored group domain replace the following filter.

    |rejectattr('domain','eq','group')
For example, to ignore the button and number domains replace it with the following filter. **The group domain should always be ignored.**

    |rejectattr('domain','in',['group','button','number'])
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

This is an [example of the sensor entities attribute template from my configuration](https://github.com/jazzyisj/home-assistant-config/blob/master/packages/hass/package_unavailable_entities.yaml) that contains several additional filters.

    {% set ignore_sec = 60 %}
    {% set ignore_ts = (now().timestamp() - ignore_sec)|as_datetime %}
    {{ states
      |rejectattr('domain','in',['group','camera'])
      |rejectattr('entity_id','in',state_attr('group.ignored_entities','entity_id'))
      |rejectattr('entity_id','search','_alarm_volume|_next_alarm|_alarms')
      |rejectattr('entity_id','search','_memory_percent|_cpu_percent')
      |rejectattr('entity_id','search','_timers|_device|_do_not_disturb')
      |rejectattr('entity_id','search','browser_')
      |rejectattr('last_changed','ge',ignore_ts)
      |selectattr('state','in',['unavailable','unknown','none'])|map(attribute='entity_id')|list }}

## Include Domains Instead Of Exclude
To monitor only one domain, you can limit the initial states object to that domain.  Note the group domain rejectattr filter is not required in this case because we are only monitoring the light domain.

    {% set ignore_ts = (now().timestamp() - 60)|as_datetime %}
    {{ states.light
      |rejectattr('entity_id','in',state_attr('group.ignored_entities','entity_id'))
      |rejectattr('last_changed','ge',ignore_ts)
      |selectattr('state','in',['unavailable','unknown','none'])|map(attribute='entity_id')|list }}


If you wish to monitor more than one specified domain you can use a selectattr filter to select a list of domains.  The group domain rejectattr filter is also not required here.

    {% set ignore_ts = (now().timestamp() - 60)|as_datetime %}
    {{ states|selectattr('domain','in',['sensor','binary_sensor'])
      |rejectattr('entity_id','in',state_attr('group.ignored_entities','entity_id'))
      |rejectattr('last_changed','ge',ignore_ts)
      |selectattr('state','in',['unavailable','unknown','none'])|map(attribute='entity_id')|list }}

## What is the log filter for?
Some users have reported occasional template warnings in their Home Assistant log, especially when reloading templates.

This warning is inconsequential and does not affect the sensor operation.  The logger filter suppresses these warnings.

**NOTE: Enabling this filter will suppress template loop warnings for ALL template sensors**

Delete or comment out the logger filter code if you do not want template loop warnings supressed.
