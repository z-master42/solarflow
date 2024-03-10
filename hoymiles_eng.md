## Zero feed with a Hoymiles inverter
I have implemented the zero feed for myself as a [script](script.yaml)[^1] in Home Assistant, which is automatically started via an [automation](automation.yaml) when I have to restart Home Assistant.

For the whole thing to work, the script needs to know your home consumption, i.e. what your electricity meter measures. Some use a [Shelly EM 3](https://www.shelly.com/de/products/shop/shelly-3em-1) for this. But there are several other sensors that work on the same principle. The main thing is that you can integrate them into Home Assistant.

Since I already had a modern electricity meter with an infrared interface, I decided to try [Volkzszaehler](https://wiki.volkszaehler.org/) at the time. In the meantime, however, I only use the _vzlogger_ and pull the read-out data into my Home Assistant via  MQTT. Here, too, there are various other sensors that could make use of the infrared interface.
+ In Home Assistant, go to _Settings_ - _Automations & Scenes_ - _Scripts_ and create a new empty script. Use the three dots at the top right to switch from visual to YAML editing mode and simply copy my template into it. Of course, you may have to change the entity names I used to yours.
  
  **Note**: I have adapted the script a bit for myself. See the notes below the code block. You can find a script that only implements the pure zero feed [here](script_nulleinspeisung.yaml).
+ If you also want to take over the automation in this way, proceed almost exactly the same way. Go to _Automations_, create a new empty automation, switch to YAML mode using the three dots at the top right and copy my template into it.
+ Don't forget to save both, of course, but Home Assistant will also remind you of this when you want to leave the page.
+ The script must then be started once, go to the _Scripts_ page again and click on the three dots at the corresponding entry. Then click on _Execute_.
```yaml
alias: Anpassung Wechselrichter Leistung
sequence:
  - repeat:
      while: []
      sequence:
        - if:
            - condition: state
              entity_id: binary_sensor.hoymiles_hm_600_reachable
              state: "on"
              alias: Wechselrichter erreichbar
          alias: "-"
          then:
            - variables:
                altes_limit: >-
                  {{
                  states('number.hoymiles_hm_600_limit_nonpersistent_absolute')
                  | float(1) }}
                grid_sum: >-
                  {{ states('sensor.volkszaehler_stromzahler_aktuelle_leistung')
                  | float(1) }}
                maximum_wr: "{{ 600 | float(1) }}"
                minimum_wr: "{{ 50 | float(1) }}"
                minimum_pack: "{{ 10 | int(1) }}"
                maximum_pack: "{{ 100 | int (1) }}"
                lower_limit_wr: "{{ 120 | float(1) }}"
                pack_level: "{{ states('sensor.solarflow_electric_level') | int(1) }}"
                solar_input: "{{ states('sensor.solarflow_solar_input_power') | float(1) }}"
              alias: Variablen definieren
            - if:
                - condition: template
                  value_template: "{{ pack_level == maximum_pack }}"
                  alias: Akkustand = 100 %
              then:
                - if:
                    - condition: template
                      value_template: "{{ solar_input > maximum_wr }}"
                      alias: Solarleistung > maximale Wechselrichterleistung
                  then:
                    - service: number.set_value
                      data:
                        value: "{{ maximum_wr }}"
                      target:
                        entity_id: number.hoymiles_hm_600_limit_nonpersistent_absolute
                      alias: Setze Limit auf 600 W
                  else:
                    - if:
                        - condition: template
                          value_template: "{{ altes_limit <= solar_input }}"
                          alias: Limit <= Solarleistung
                      then:
                        - service: number.set_value
                          data:
                            value: "{{ solar_input }}"
                          target:
                            entity_id: >-
                              number.hoymiles_hm_600_limit_nonpersistent_absolute
                          alias: Setze Limit = Solarleistung
                      alias: "-"
                      else:
                        - variables:
                            setpoint: "{{ (grid_sum + altes_limit - 5.0) | float(1) }}"
                          alias: Neues Limit = Aktueller Verbrauch + Altes Limit - 5
                        - service: number.set_value
                          data:
                            value: "{{ setpoint }}"
                          target:
                            entity_id: >-
                              number.hoymiles_hm_600_limit_nonpersistent_absolute
                          alias: Setze Limit = Verbrauch
                  alias: "-"
              alias: "-"
              else:
                - if:
                    - condition: template
                      value_template: "{{ pack_level >= minimum_pack}}"
                      alias: Akkustand >= 10 %
                  then:
                    - variables:
                        setpoint: "{{ (grid_sum + altes_limit - 5.0) | float(1) }}"
                      alias: Neues Limit = Aktueller Verbrauch + Altes Limit - 5
                    - if:
                        - condition: template
                          value_template: "{{ setpoint > maximum_wr }}"
                          alias: Neues Limit > 600 W
                      then:
                        - service: number.set_value
                          data:
                            value: "{{ maximum_wr }}"
                          target:
                            entity_id: >-
                              number.hoymiles_hm_600_limit_nonpersistent_absolute
                          alias: Setze Limit auf 600 W
                      else:
                        - if:
                            - condition: template
                              value_template: "{{ setpoint < minimum_wr }}"
                              alias: Neues Limit < 50
                          then:
                            - service: number.set_value
                              data:
                                value: "{{ minimum_wr }}"
                              target:
                                entity_id: >-
                                  number.hoymiles_hm_600_limit_nonpersistent_absolute
                              alias: Setze Limit auf 50 W
                          else:
                            - if:
                                - condition: template
                                  value_template: "{{ setpoint != altes_limit }}"
                                  alias: Neues Limit != Altes Limit
                              then:
                                - service: number.set_value
                                  data:
                                    value: "{{ setpoint | float(1) }}"
                                  target:
                                    entity_id: >-
                                      number.hoymiles_hm_600_limit_nonpersistent_absolute
                                  alias: Setzte neues Limit
                              alias: "-"
                          alias: "-"
                      alias: "-"
                  else:
                    - variables:
                        setpoint: "{{ (grid_sum + altes_limit - 5.0) | float(1) }}"
                      alias: Neues Limit = Aktueller Verbrauch + Altes Limit - 5
                    - if:
                        - condition: template
                          value_template: "{{ setpoint > lower_limit_wr }}"
                          alias: Neues Limit > 120 W
                      then:
                        - service: number.set_value
                          data:
                            value: "{{ lower_limit_wr }}"
                          target:
                            entity_id: >-
                              number.hoymiles_hm_600_limit_nonpersistent_absolute
                          alias: Setze Limit auf 120 W
                      else:
                        - if:
                            - condition: template
                              value_template: "{{ setpoint < minimum_wr }}"
                              alias: Neues Limit < 50
                          then:
                            - service: number.set_value
                              data:
                                value: "{{ minimum_wr }}"
                              target:
                                entity_id: >-
                                  number.hoymiles_hm_600_limit_nonpersistent_absolute
                              alias: Setze Limit auf 50 W
                          else:
                            - if:
                                - condition: template
                                  value_template: "{{ setpoint != altes_limit }}"
                                  alias: Neues Limit != Altes Limit
                              then:
                                - service: number.set_value
                                  data:
                                    value: "{{ setpoint | float(1) }}"
                                  target:
                                    entity_id: >-
                                      number.hoymiles_hm_600_limit_nonpersistent_absolute
                                  alias: Setzte neues Limit
                              alias: "-"
                          alias: "-"
                      alias: "-"
                  alias: "-"
        - delay:
            hours: 0
            minutes: 0
            seconds: 5
            milliseconds: 0
          alias: Warte 5 s
mode: single
icon: phu:huawei-solar-inverter
```
### Notes
In addition to the "actual" script for adjusting the inverter limit to the current consumption, I have added a few more modules:
+ If the battery is full and the current photovoltaic power exceeds that of my inverter, the limit is set to 100 %. If the current limit is less than the available photovoltaic power, the limit is increased to this. I have not yet been able to test how well this works with the PV hub's bypass mode.
+ If the battery charge level is less than 10 %, the inverter limit is set to 120 W in order to charge the battery a little first. This is also followed by a limit when discharging if the battery falls below 10 % again and there is no more sunshine. The 120 W roughly corresponds to my base load.
+ No energy plan is set in the SolarFlow app. PV hub output power and acceptable inverter input power are set to what the inverter can do at maximum.
[^1]: The script is based on the work of Peter F. from H., who realised the zero feed as a Python script: https://gitlab.com/p3605/hoymiles-tarnkappe/-/blob/main/hoymiles_setlimiter.py?ref_type=heads
