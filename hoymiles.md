## Nulleinspeisung mit einem Hoymiles-Wechselrichter
Die Nulleinspeisung habe ich für mich als [Skript](script.yaml)[^1] in Home Assistant realisiert, welches über eine [Automatisierung](automation_start.yaml) automatisch gestartet wird, wenn ich den Home Assistant mal neustarten musste.
Auf Grund von Limitierungen in Home Assistant, was "kopflose" Endlosschleifen angeht, ist noch eine zweite [Automatiserung](automation_repeat.yaml) nötig, die das Skript in periodischen Abständen startet. In dem Intervall halt, in dem ihr die Begrenzung eures Wechselrichters anpassen wollt. Fünf Sekunden haben sie als gutes Intervall bewiesen. Sicherlich gibt es schönere und elegantere Lösungen, das Ganze umzusetzten, aber diese funktioniert erstmal.

Damit das ganze funktioniert, muss das Skript euren Hausverbrauch kennen, spich das, was euer Stromzähler misst. Einige nutzen hierfür einen [Shelly EM 3](https://www.shelly.com/de/products/shop/shelly-3em-1). Es gibt aber noch diverse weitere Sensoren die nach dem gleichen Prinzip arbeiten. Hauptsache ist, ihr könnt sie in Home Assistant integrieren.

***Außerdem besonders wichtig dabei***: Der gemessene Werte (die aktuelle Leistung) muss der saldierte Wert sein, sonst funktioniert das Skript nicht.

Da bei mir schon ein modernerer Stromzähler mit Infrarotschnittstelle vorhanden war, habe ich mich seinerzeit dafür entschieden, mich an [Volkzszaehler](https://wiki.volkszaehler.org/) zu versuchen. Mittlerweile nutze ich davon aber nur noch den _vzlogger_ und ziehe mir die ausgelesenen Daten über MQTT in meinen Home Assistant. Auch hier gibt es diverse weitere Sensoren, die sich die Infrarotschnittstelle zu nutzen machen können.
+ Geht in Home Assistant auf _Einstellungen_ - _Automatisierungen & Szenen_ - _Skripte_ und legt ein neues leeres Skript an. Wechselt über die drei Punkte oben rechts vom visuellen in den YAML-Bearbeitungsmodus und kopiert meine Vorlage einfach da rein. Ggf. müsst ihr natürlich die von mir verwendeten Entitätennamen auf eure ändern.
  
+ Wollt ihr auch die Automatisierung so übernehmen, geht ihr fast genau so vor. Wechselt zu _Automatisierungen_ legt eine neue leere Automatisierung an, wechselt über die drei Punkte oben rechts in den YAML-Modus und kopiert meine Vorlage da rein.
+ Bei beidem natürlich Speichern nicht vergessen, aber da erinnert euch Home Assistant auch dran, wenn ihr die Seite verlassen wollt.

```yaml
alias: Anpassung Wechselrichter Leistung
sequence:
  - if:
      - condition: state
        entity_id: binary_sensor.wechselrichter_reachable
        state: "on"
        alias: Wechselrichter erreichbar
    alias: "-"
    then:
      - variables:
          altes_limit: >-
            {{
            states('number.wechselrichter_limit_nonpersistent_absolute') |
            float(1) }}
          grid_sum: >-
            {{ states('sensor.stromzahler_aktuelle_leistung') | float(1)
            }}
          maximum_wr: "{{ 600 | float(1) }}"
          minimum_wr: "{{ 50 | float(1) }}"
          setpoint: "{{ (grid_sum + altes_limit - 5.0) | float(1) }}"
        alias: Variablen definieren
      - if:
          - condition: template
            value_template: "{{ setpoint > maximum_wr }}"
            alias: Neues Limit > 600 W
        then:
          - service: number.set_value
            data:
              value: "{{ maximum_wr }}"
            target:
              entity_id: number.wechselrichter_limit_nonpersistent_absolute
            alias: Setze Limit auf 600 W
        else:
          - if:
              - condition: template
                value_template: "{{ setpoint < minimum_wr}}"
                alias: Neues Limit < 50
            then:
              - service: number.set_value
                data:
                  value: "{{ minimum_wr }}"
                target:
                  entity_id: number.wechselrichter_limit_nonpersistent_absolute
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
                      entity_id: number.wechselrichter_limit_nonpersistent_absolute
                    alias: Setzte neues Limit
                alias: "-"
            alias: "-"
        alias: "-"
mode: single
icon: mdi:vector-link
```
### Anmerkung
+ In der SolarFlow-App ist kein Energieplan eingestellt. PV-Hub Ausgangsleistung und akzeptable Wechselrichtereingangsleistung stehen auf dem was der Wechselrichter maximal kann.
[^1]: Das Skript basiert auf der Arbeit von Peter F. aus H., welcher die Nulleinspeisung als Python-Skript realisiert hat: https://gitlab.com/p3605/hoymiles-tarnkappe/-/blob/main/hoymiles_setlimiter.py?ref_type=heads
