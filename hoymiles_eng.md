## Zero feed with a Hoymiles inverter
I have implemented the zero feed for myself as a [script](script.yaml)[^1] in Home Assistant, which is automatically started via an [automation](automation.yaml) when I have to restart Home Assistant.

For the whole thing to work, the script needs to know your home consumption, i.e. what your electricity meter measures. Some use a [Shelly EM 3](https://www.shelly.com/de/products/shop/shelly-3em-1) for this. But there are several other sensors that work on the same principle. The main thing is that you can integrate them into Home Assistant.

Since I already had a modern electricity meter with an infrared interface, I decided to try [Volkzszaehler](https://wiki.volkszaehler.org/) at the time. In the meantime, however, I only use the _vzlogger_ and pull the read-out data into my Home Assistant via _RESTful_. MQTT would also work. But I haven't yet managed to reconnect it in such a way that my history isn't lost. Here, too, there are various other sensors that make use of the infrared interface.
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
### Anmerkungen
Neben dem "eigentlichen" Skript zur Anpassung des Wechselrichterlimits an den aktuellen Verbrauch habe ich noch ein paar Bausteine ergänzt:
+ Ist der Akku voll und übersteigt die aktuelle Photovoltaikleistung die meines Wechselrichters, wird das Limit auf 100 % gesetzt. Ist das aktuelle Limit kleiner als die verfügbare Photovoltaikleistung wird das Limit auf diese erhöht. Wie gut das mit dem Bypass-Modus des PV-Hubs funktioniert habe ich noch nicht testen können.
+ Ist der Ladestand des Akkus kleiner als 10 % erfolgt eine Begrenzung des Wechselrichterlimits auf 120 W um so erstmal den Akku ein wenig zu laden. Bzw. folgt so auch eine Begrenzung beim Entladen wenn die 10 % wieder unterschritten werden und keine Sonne mehr scheint. Die 120 W entsprechen dabei so ungefähr meiner Grundlast.
+ In der SolarFlow-App ist kein Energieplan eingestellt. PV-Hub Ausgangsleistung und akzeptable Wechselrichtereingangsleistung stehen auf dem was der Wechselrichter maximal kann.
[^1]: Das Skript basiert auf der Arbeit von Peter F. aus H., welcher die Nulleinspeisung als Python-Skript realisiert hat: https://gitlab.com/p3605/hoymiles-tarnkappe/-/blob/main/hoymiles_setlimiter.py?ref_type=heads
