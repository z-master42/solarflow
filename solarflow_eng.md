## Integrating SolarFlow into Home Assistant
### Introduction
Zendure operates an MQTT broker for retrieving information for the products SuperBase V, Satellite Battery and SolarFlow. This is currently the only official way to access information outside of the app provided by Zendure. Since Zendure plays out the data via its own broker, the connection is forced to run through the internet. Purely local control is currently and officially not yet possible.
[MQTT](https://en.wikipedia.org/wiki/MQTT) is an open network protocol for machine-to-machine communication. As a rule, several clients are connected to a broker. The exchanged messages are defined hierarchically by topics. In order to access corresponding information or send commands, the corresponding topic must be subscribed to.
### Preparation
The prerequisite for use is an account with Zendure (which everyone should have by installing the app for controlling the SolarFlow).
To retrieve the MQTT data of your own SolarFlow, you also need an 'appKey' and an 'appSecret'.
To get these two values, you need the email address you registered with at Zendure and the serial number of your PV hub.
I used the command line tool _curl_ to retrieve the serial number.

**Procedure on a Microsoft operating system**
+ Open the command prompt with `Windows key + R`.
+ Enter `cmd`.
+ Enter the following command in the command line:

  Region setting in the Zendure app to _"Global"_
  ```
  curl -i -v --json "{'snNumber': 'YourHubSerialNumber', 'account': 'YourEmailaddress'}" https://app.zendure.tech/v2/developer/api/apply
  ```
  Region setting in the Zendure app on a _"European country"_
  ```
  curl -i -v --json "{'snNumber': 'YourHubSerialNumber', 'account': 'YourEmailaddress'}" https://app.zendure.tech/eu/developer/api/apply
  ```
+ Beforehand, of course, you have entered your serial number and the email address you use instead of the placeholders.

**Proceeding on a Linux operating system**
+ Open a terminal window with `Ctrl+Alt+T`.
+ Enter the following command in the command line:

  Region setting in the Zendure app to _"Global"_
  ```
  curl -i -X POST -H 'Content-Type: application/json' -d '{"snNumber": "YourHubSerialNumber", "account": "YourEmailaddress"}' https://app.zendure.tech/v2/developer/api/apply
  ```
  Region setting in the Zendure app on a _"European country"_
  ```
  curl -i -X POST -H 'Content-Type: application/json' -d '{"snNumber": "YourHubSerialNumber", "account": "YourEmailaddress"}' https://app.zendure.tech/v2/developer/api/apply
  ```
+ Beforehand, of course, you have entered your serial number and the email address you use instead of the placeholders.

**Reply**

If you have not made any mistakes, an answer should appear as follows:
```
{"code":200,"success":true,"data":{"appKey":"YourAppKey","secret":"YourAppSecret","mqttUrl":"mqtt.zen-iot.com","port":1883},"msg":"Successful operation"}
```

Instead of the placeholders, you will find your 'appKey' and your 'appSecret'. Both are letter-number combinations.
### Setting up MQTT in Home Assistant
Depending on how far you have gone in Home Assistant, there are now several ways to reach your goal.

1. **Option: No MQTT device connected to Home Assistant yet**
   
   If this is your first contact with MQTT, things will go quite quickly.
   + Add a new integration via _Settings - Devices & Services_. Look for MQTT there and select that without any other designations.
   + The username is your `appKey` and the password your `appSecret`. The URL of the broker and the port were also delivered to you with the above answer: `mqtt.zen-iot.com` with port `1883`.
   + In order for data to come in, you have to subscribe to a topic, as mentioned at the beginning. This is done here by activating _Enable Discovery_ on the configuration page and entering `appKey' as _Discovery prefix_..
     
     ![grafik](https://github.com/z-master42/solarflow/assets/66380371/769a98f3-8786-42c3-8cc5-0d761df5aee7)
2. **Option: You already use an MQTT broker in Home Assistant**
   
   If you have already gained experience with MQTT, e.g. because you have connected sensors or sockets with Home Assistant via it, it is very likely that you have installed the Mosquitto broker add-on and configured the MQTT integration for the connection with it.
   
   Problem: Only one MQTT integration can be installed in Home Assistant.
   
   Solution: You have to build a bridge to the MQTT broker from Zendure.
   There are two ways to proceed. Either you let Home Assistant create all available sensors automatically or you add them manually. First of all: I added mine manually, so I had the option of adapting them at the same time.
   1. **Check beforehand**
      + In order to check whether the Zendure broker is transmitting data from your SolarFlow at all, you can use the programme [MQTT-Explorer](http://mqtt-explorer.com/), which is available for the most common operating systems.
      + Create a new connection with your access data (`appKey`, `appSecret`) as shown in the screenshot.

        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/f4555568-a65c-43a1-8c8a-e6fdbb46bb96)
      + Under Advanced (button) you must then subscribe to your `appKey` as a topic, i.e. add it as a new subscription --> `appKey/#`.

        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/59e3ea65-2927-4e6f-9bd9-815e761ed3d9)
      + The values should then come in at a fairly early stage. When you open everything up, it looks something like this:

        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/cb2d9c67-40ce-4bcb-bd18-054219adf43d)
        Under the broker address, an entry appears that reads like your `appKey` (all in 🔴). Below that there are three more entries; `switch`, `sensor`, and one that identifies your SolarFlow. We will therefore call it `deviceID` from now on (all in 🔵).
        + `switch` contains as entries the building instructions for the switches available so far.
        + `sensor` contains as entries the building instructions for the sensors available so far.
        + `deviceID` contains the status of the sensors as entries, but only those whose value has changed.
          
        The following sensors and switches are currently available:

          | Field | Description | device_class | Automatic discovery (2. iii.)
          | -------- | -------- | -------- | -------- |
          | electricLevel | Device battery percentage | sensor | yes |
          | remainOutTime | Remaining discharge time | sensor | yes |
          | remainInputTime | Remaining charging time | sensor | yes |
          | socSet | Charge capacity limitation | sensor | yes |
          | inputLimit | input limit | sensor | no |
          | outputLimit | output limit | sensor | yes |
          | solarInputPower | solar input power | sensor | yes |
          | packInputPower | pack input power | sensor | yes |
          | outputPackPower | output pack power | sensor | yes |
          | outputHomePower | output home power | sensor | yes |
          | packNum | pack num | sensor | yes |
          | packState | pack state (0: standby 1: input 2: output) | sensor | yes |
          | buzzerSwitch | buzzer switch | switch | yes |
          | masterSwitch | master switch | switch | yes |
          | wifiState | wifi state | sensor | no |
          | hubState | Hub output status (0: stop output standby 1: stop output and shut down) | sensor | no |
          | inverseMaxPower | inverter max power | sensor | no |
          | packData | pack Data (maxVol, minVol, maxTemp, socLevel, sn) | sensor | yes |
          | solarPower1 | Solar1 Input Power | sensor | yes |
          | solarPower2 | Solar2 Input Power | sensor | yes |
          | passMode | Bypass Mode (0: auto 1: always off 2: always on) | sensor | yes |
          | autoRecover | Automatic recovery of bypass mode settings (0: off 1: on) | switch | yes |
          | sn | Hub serial number | sensor | no |
        
        ***Note**: The sensors that are specified as switches are only used to display information. The function cannot be switched on or off via the switch.


   2. **Same start**
      + The beginning is the same for both options.
      + First, the bridge to the Zendure broker must be established. To do this, you must access the `share` directory of your Home Assistant. This is not possible via the File Editor Addon, for example. You only have access to the `config` directory. You can do this via the Samba addon. At this point, however, we will use the Terminal & SSH Addon. Of course, you have to install it for this. First create a file with the following content. It doesn't matter what the file is called and which editor or text processing programme you use for it.
      + Create a new text file.
      + Insert the following [content](zendure.conf):
        
        ```
        connection external-bridge
        address mqtt.zen-iot.com:1883
        remote_username <appKey>
        remote_password <appSecret>
        remote_clientid <appKey>
        topic <appKey>/# both 0
        ```
      + Replace everything between <> **including the <>** with your own data, of course. There must be no <> left.
      + Install the Terminal & SSH Addon in the Addon area, then start it and open the user interface.
      + Change to the `share` directory with the command `cd share`.
      + Check with `ls` if there is already a folder `mosquitto`. If a new input line simply appears after the command, the folder does not yet exist, otherwise an entry `mosquitto` appears.
      + If the folder does not exist, create it with `mkdir mosquitto`.
      + Change to the directory with `cd mosquitto`.
      + Now create a new file and edit it with the editor _nano_. Enter `nano zendure.conf`.
      + Now copy the content from your previously created text file and paste it into the terminal window. **To do this**, hold down the `Shift` key on your keyboard and click with the `right mouse button` in the terminal window and select `Paste`. Now the contents of your text file should appear in the terminal window.
      + If you want Home Assistant to create the sensor entities for you automatically, skip the next steps and continue with iii.
      + Close the _nano_ editor with `Ctrl + X`.
      + Confirm the question that appears at the bottom of the terminal with `y`.
      + Confirms the next question with `Enter`.
      + In the configuration of the Mosquitto add-on, check whether `active` is set to `true` under _Customize_.
      + Finally, restart the Mosquitto-Addon.
      + An entry similar to `Connecting bridge external-bridge (mqtt.zen-iot.com:1883)` should then appear in the log. You may have to refresh the log several times (patience). If something comes up with a timeout or something like that, simply restart the addon.
   3. **Automatic insertion**
      
      In order for Home Assistant to automatically create the sensors and switches, it must know how they are structured. Home Assistant also has specifications as to how the topics must look for this to work.
      Simply add another line to your `zendure.conf` with:
      
      ```
      topic # in 0 homeassistant/sensor/<appKey>/ <appKey>/sensor/device/
      ```
      Now continue upstairs from where you just jumped off.
   4. **Manual insertion**

      The manual creation of entities is done in Home Assistant via `configuration.yaml`. This is located in the `config` directory. You can also access this via Samba. However, the way via the File Editor Addon is just as good. In this addon, the inserted code is marked in colour for better readability and if there are formatting or syntax errors, these are displayed directly. In general, you can write everything in the `configuration.yaml`. Over time, however, it becomes a bit confusing, as everything is written one below the other in a text file. It is better to store the corresponding configurations in new files.
      + Open your `configuration.yaml`.
      + Add a new line under the following block, which is located at the beginning of the file: `mqtt: !include mqtt.yaml`.
        
        ```yaml
        group: !include groups.yaml
        automation: !include automations.yaml
        script: !include scripts.yaml
        scene: !include scenes.yaml
        ```
        **Note**: The block does not have to look exactly like this. Here, too, it depends on how far you have already gone in Home Assistant. If I remember correctly, however, at least one `!include` line should already be present.
      + Create a new file `mqtt.yaml` and insert [content](mqtt.yaml) below.
        
        ```yaml
           sensor:
             - name: "Hub State"
               unique_id: "<deviceID>hubState"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.hubState | int }}"
               device: 
                 name: "SolarFlow"
                 identifiers: "<YourPVHubSerialNumber>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"

             - name: "Solar Input Power"
               unique_id: "<deviceID>solarInputPower"
               state_topic: "<appKey>/<deviceID>/state"
               unit_of_measurement: "W"
               device_class: "power"
               value_template: "{{ value_json.solarInputPower | int(0) }}"
               state_class: "measurement"
               device: 
                 name: "SolarFlow"
                 identifiers: "<YourPVHubSerialNumber>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
      
             - name: "Pack Input Power"
               unique_id: "<deviceID>packInputPower"
               state_topic: "<appKey>/<deviceID>/state"
               unit_of_measurement: "W"
               device_class: "power"
               value_template: "{{ value_json.packInputPower | int(0) }}"
               state_class: "measurement"
               device: 
                 name: "SolarFlow"
                 identifiers: "<YourPVHubSerialNumber>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
      
             - name: "Output Pack Power"
               unique_id: "<deviceID>outputPackPower"
               state_topic: "<appKey>/<deviceID>/state"
               unit_of_measurement: "W"
               device_class: "power"
               value_template: "{{ value_json.outputPackPower | int(0) }}"
               state_class: "measurement"
               device: 
                 name: "SolarFlow"
                 identifiers: "<YourPVHubSerialNumber>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
      
             - name: "Output Home Power"
               unique_id: "<deviceID>outputHomePower"
               state_topic: "<appKey>/<deviceID>/state"
               unit_of_measurement: "W"
               device_class: "power"
               value_template: "{{ value_json.outputHomePower | int(0) }}"
               state_class: "measurement"
               device: 
                 name: "SolarFlow"
                 identifiers: "<YourPVHubSerialNumber>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
      
             - name: "Output Limit"
               unique_id: "<deviceID>outputLimit"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.outputLimit | int }}"
               unit_of_measurement: "W"
               device: 
                 name: "SolarFlow"
                 identifiers: "<YourPVHubSerialNumber>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "Input Limit"
               unique_id: "<deviceID>inputLimit"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.inputLimit | int }}"
               unit_of_measurement: "W"
               device: 
                 name: "SolarFlow"
                 identifiers: "<YourPVHubSerialNumber>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "Remain Out Time"
               unique_id: "<deviceID>remainOutTime"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.remainOutTime | int }}"
               device_class: "duration"
               unit_of_measurement: "min"
               device: 
                 name: "SolarFlow"
                 identifiers: "<YourPVHubSerialNumber>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "Remain Input Time"
               unique_id: "<deviceID>remainInputTime"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.remainInputTime | int }}"
               device_class: "duration"
               unit_of_measurement: "min"
               device: 
                 name: "SolarFlow"
                 identifiers: "<YourPVHubSerialNumber>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "Pack State"
               unique_id: "<deviceID>packState"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.packState | int }}"
               device: 
                 name: "SolarFlow"
                 identifiers: "<YourPVHubSerialNumber>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "Pack Num"
               unique_id: "<deviceID>packNum"
               state_topic: "<appKey>/<deviceID>/state"
               value_template: "{{ value_json.packNum | int }}"
               device: 
                 name: "SolarFlow"
                 identifiers: "<YourPVHubSerialNumber>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "Electric Level"
               unique_id: "<deviceID>electricLevel"
               state_topic: "<appKey>/<deviceID>/state"
               unit_of_measurement: "%"
               device_class: "battery"
               value_template: "{{ value_json.electricLevel | int }}"
               device: 
                 name: "SolarFlow"
                 identifiers: "<YourPVHubSerialNumber>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
    
             - name: "SOC Set"
               unique_id: "<deviceID>socSet"
               state_topic: "<appKey>/<deviceID>/state"
               unit_of_measurement: "%"
               value_template: "{{ value_json.socSet | int / 10 }}"
               device: 
                 name: "SolarFlow"
                 identifiers: "<YourPVHubSerialNumber>"
                 manufacturer: "Zendure"
                 model: "SmartPV Hub 1200 Controller"
        
            - name: "Inverse Max Power"
              unique_id: "<deviceID>inverseMaxPower"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: "{{ value_json.inverseMaxPower | int }}"
              unit_of_measurement: "W"
              device: 
                name: "SolarFlow"
                identifiers: "<YourPVHubSerialNumber>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"
      
            - name: "WiFi State"
              unique_id: "<deviceID>wifiState"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: "{{ value_json.wifiState | bool('') }}"
              device: 
                name: "SolarFlow"
                identifiers: "<YourPVHubSerialNumber>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Solar Power 1"
              unique_id: "<deviceID>solarPower1"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: "{{ value_json.solarPower1 | int(0) }}"
              unit_of_measurement: "W"
              device_class: "power"
              state_class: "measurement"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Solar Power 2"
              unique_id: "<deviceID>solarPower2"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: "{{ value_json.solarPower2 | int(0) }}"
              unit_of_measurement: "W"
              device_class: "power"
              state_class: "measurement"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Pass Mode"
              unique_id: "<deviceID>passMode"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: "{{ value_json.passMode | int }}"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Batterie <Nr> maxTemp"
              unique_id: "<deviceID>Batterie<Nr>maxTemp"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: >
                {% for i in value_json.packData %}
                  {% if i.sn == "<EureBatterieSeriennummer>" %}
                    {{ (i.maxTemp | float - 273.15) | round(2) }}
                  {% endif %}
                {% endfor %}
              unit_of_measurement: "°C"
              device_class: "temperature"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Batterie <Nr> maxVol"
              unique_id: "<deviceID>Batterie<Nr>maxVol"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: >
                {% for i in value_json.packData %}
                  {% if i.sn == "<EureBatterieSeriennummer>" %}
                    {{ i.maxVol | float / 100 }}
                  {% endif %}
                {% endfor %}
              unit_of_measurement: "V"
              device_class: "voltage"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Batterie <Nr> minVol"
              unique_id: "<deviceID>Batterie<Nr>minVol"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: >
                {% for i in value_json.packData %}
                  {% if i.sn == "<EureBatterieSeriennummer>" %}
                    {{ i.minVol | float / 100 }}
                  {% endif %}
                {% endfor %}
              unit_of_measurement: "V"
              device_class: "voltage"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - name: "Batterie <Nr> socLevel"
              unique_id: "<deviceID>Batterie<Nr>socLevel"
              state_topic: "<appKey>/<deviceID>/state"
              value_template: >
                {% for i in value_json.packData %}
                  {% if i.sn == "<EureBatterieSeriennummer>" %}
                    {{ i.socLevel | int }}
                  {% endif %}
                {% endfor %}
              unit_of_measurement: "%"
              device_class: "battery"
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"
      
          switch:
            - unique_id: "<deviceID>masterSwitch"
              state_topic: "<appKey>/<deviceID>/state"
              state_off: false
              command_topic: "<appKey>/<deviceID>/masterSwitch/set"
              name: "Master Switch"
              device_class: "switch"
              value_template: "{{ value_json.masterSwitch | default('') }}"
              payload_on: true
              payload_off: false
              state_on: true
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - unique_id: "<deviceID>buzzerSwitch"
              state_topic: "<appKey>/<deviceID>/state"
              state_off: false
              command_topic: "<appKey>/<deviceID>/buzzerSwitch/set"
              name: "Buzzer Switch"
              device_class: "switch"
              value_template: "{{ value_json.buzzerSwitch | default('') }}"
              payload_on: true
              payload_off: false
              state_on: true
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"

            - unique_id: "<deviceID>autoRevover"
              state_topic: "<appKey>/<deviceID>/state"
              state_off: false
              command_topic: "<appKey>/<deviceID>/autoRevover/set"
              name: "Auto Recover"
              device_class: "switch"
              value_template: "{{ value_json.autoRevover | default('') }}"
              payload_on: true
              payload_off: false
              state_on: true
              device: 
                name: "SolarFlow"
                identifiers: "<EurePVHubSeriennummer>"
                manufacturer: "Zendure"
                model: "SmartPV Hub 1200 Controller"
        ```
      + Save the file.
      + Open the developer tools and check whether your configuration is error-free, then restart Home Assistant.
      + If you make further changes to this sensor and switch configuration, add or remove something, it is sufficient to click on _Manually configured MQTT entities_ in the _Reload YAML configuration_ area.
        
        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/ac61106b-200e-44ab-bce8-e9c7fc3ff0ef)
      + Now you should find a new device called 'SolarFlow' under _Devices & Services_ in your MQTT integration, which contains the corresponding sensors and switches under the above names. Very few of them will already have a value from the beginning, because as mentioned at the beginning, the Zendure broker only plays out value changes, so the respective value must have changed compared to its previous state. To force an update of pretty much all sensors, you can simply change the house feed or the charge limit in the app.
      + Finally, a few comments on the adjustments I have made, in addition to the Zendure sensor building instructions:
        + I added a default value to all sensors in the `value_template` line. Due to the fact that only value changes are transmitted, Home Assistant writes a warning (`Template variable warning: 'dict object' has no attribute 'blablabla' when rendering '{{ value_json.blablabla }}'`) in your log for each sensor for which no value was included in the last data exchange, as Home Assistant expects to receive a value for each sensor. I have provided all power sensors with the default value `int(0)`, the rest only with `int`. This ensures that Home Assistant continues to display the last value until a new one is transmitted or that a sensor is also set to 0 when it is no longer updated, e.g. when the battery changes from charging to discharging.
        + I have added `state_class: measurement` to _Solar Input Power_, _Pack Input Power_, _Output Pack Power_ and _Output Home Power_ so that Home Assistant can also calculate with these. I don't know if this is really necessary at the moment. However, in order to be able to use the values in the energy dashboard, they still have to be integrated into a consumption value (power times time). For this purpose, there is the _Riemann Sum Integral Sensor_ in the Help section of Home Assistant. I have set the method to `Left` and the metric prefix to `Kilo`. Once you have run through a few values, you can use them in the Energy Dashboard.
        + _Output Limit_ and _Input Limit_ have been given the `unit_of_measurement: "W"`.
        + _Remain Input Time_ and _Remain Out Time_ have been given the `device_class: "duration"`. The transmitted value is the respective duration in minutes, so that the `unit_of_measurement: "min"` is. Due to the `device_class`, Home Assistant automatically converts this into a time specification in h:min:s.
        + _SOC Set_ is already specified by Zendure with `unit_of_measurement: "%"`, but the sensor then delivers e.g. 1000 % if the upper charge limit is 100 %. I have no idea why this should be the case. I have divided the value accordingly by 10.
        + I have added a `device` block to all entries and used my serial number as a unique `identifier`. Through this block, Home Assistant knows that they are entities of the same device and creates this device accordingly, so that you can also find it under _Devices & Services_.
        + The Zendure broker also plays out two values for which there are no instructions. I have simply added them. These are `wifiState` and `inverseMaxPower`. What the first one represents should be clear. The second represents the maximum acceptable input power to the inverter.
        + Of course, you can already give each entity its own symbol here. Add an `icon` line for the respective entity, e.g. `icon: mdi:flash`.
  4. **Advantages and disadvantages**
     | Method | Advantages | Disadvantages |
     | -------- | -------- | -------- |
     | (1.) No previous use of MQTT in Home Assistant | Quickly added and little effort | Subsequent adjustment of entities possible but possibly circumstantial. Warnings in log about missing `dict object`[^1] |
     | (2. iii.) Already MQTT in use. Automatic creation of entities by Home Assistant | Slight additional effort compared to (1.), but not otherwise possible in Home Assistant | Subsequent adjustment of entities possible but possibly awkward. Warings in log about missing `dict object`[^1]|
     | (2. iv.) Already MQTT in use. Manual creation of entities | Full customisation possibilities given. Exclude warnings | May be more cumbersome and complex than (2. iii.). Not self-explanatory, difficult to comprehend for someone who is not so familiar with the subject |



[^1]: The only "solution" I know of so far is to suppress the corresponding [Warning](warning.yaml). 
