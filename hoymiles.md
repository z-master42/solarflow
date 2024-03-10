## Nulleinspeisung mit einem Hoymiles-Wechselrichter
Die Nulleinspeisung habe ich für mich als [Skript](script.yaml)[^1] in Home Assistant realisiert, welches über eine [Automatisierung](automation.yaml) automatisch gestartet wird, wenn ich den Home Assistant mal neustarten musste.

Damit das ganze funktioniert, muss das Skript euren Hausverbrauch kennen, spich das, was euer Stromzähler misst. Einige nutzen hierfür einen [Shelly EM 3](https://www.shelly.com/de/products/shop/shelly-3em-1). Es gibt aber noch diverse weitere Sensoren die nach dem gleichen Prinzip arbeiten. Hauptsache ist, ihr könnt sie in Home Assistant integrieren.

Da bei mir schon ein modernerer Stromzähler mit Infrarotschnittstelle vorhanden war, habe ich mich seinerzeit dafür entschieden, mich an [Volkzszaehler](https://wiki.volkszaehler.org/) zu versuchen. Mittlerweile nutze ich davon aber nur noch den _vzlogger_ und ziehe mir die ausgelesenen Daten über MQTT in meinen Home Assistant. Auch hier gibt es diverse weitere Sensoren, die sich die Infrarotschnittstelle zu nutzen machen können.
+ Geht in Home Assistant auf _Einstellungen_ - _Automatisierungen & Szenen_ - _Skripte_ und legt ein neues leeres Skript an. Wechselt über die drei Punkte oben rechts vom visuellen in den YAML-Bearbeitungsmodus und kopiert meine Vorlage einfach da rein. Ggf. müsst ihr natürlich die von mir verwendeten Entitätennamen auf eure ändern.
  
  **Hinweis**: Ich habe das Skript für mich noch etwas angepasst. Siehe dazu die Anmerkungen unter dem Code-Block. Ein Skript, welches nur die reine Nulleinspeisung umsetzt, findet ihr [hier](script_nur_nulleinspeisung.yaml).
+ Wollt ihr auch die Automatisierung so übernehmen, geht ihr fast genau so vor. Wechselt zu _Automatisierungen_ legt eine neue leere Automatisierung an, wechselt über die drei Punkte oben rechts in den YAML-Modus und kopiert meine Vorlage da rein.
+ Bei beidem natürlich Speichern nicht vergessen, aber da erinnert euch Home Assistant auch dran, wenn ihr die Seite verlassen wollt.
+ Das Skript muss dann einmalig noch gestartet werden, geht dazu noch mal auf die _Skripte_-Seite und klickt auf die drei Punkte am entsprechenden Eintrag. Dort dann auf _Ausführen_.
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
