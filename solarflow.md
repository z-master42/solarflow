## SolarFlow in Home Assistant einbinden
### Einleitung
Zendure betreibt für den Abruf von Informationen für die Produkte SuperBase V, Satellite Battery und SolarFlow einen MQTT-Broker. Dies stellt aktuell auch die einzige offizielle Möglichkeit dar, Informationen außerhalb der von Zendure bereitgestellten App abzugreifen. Da Zendure die Daten über einen eigenen Broker auspielt läuft die Verbindung zwangsweise durch Internet. Eine rein lokale Steuerung ist aktuell noch nicht möglich.
[MQTT](https://de.wikipedia.org/wiki/MQTT) ist ein offenes Netzwerkprotokoll für eine Maschine-zu-Maschine-Kommunikation. Dabei stehen in der Regel mehrere Clients in Verbindung zu einem Broker. Die ausgetauschten Nachrichten werden hierachisch abgestuft durch Topics definiert. Um entsprechende Informationen abzugreifen oder Befehle zu senden muss das entprechende Topic abonniert werden.
### Vorbereitung
Voraussetzung zur Nutzung ist ein Account bei Zendure (den ja jeder dadurch haben sollte, dass er ich die App zur Steuerung des SolarFlow installiert hat).
Für den Abruf der MQTT-Daten des eigenen SolarFlow benötigt man im weiteren einen `appKey` und ein `appSecret`.
Um diese beiden Werte zu erhalten, benötigt ihr, neben der Emailadresse mit der ihr euch bei Zendure registriert habt, die Seriennummer eures PV-Hubs.
Ich habe zum Abruf das Kommandozeilen-Tool _curl_ verwendet.

**Vorgehen auf einem Microsoft Betriebssystem**
+ Öffnet mit `Windows-Taste + R` die Eingabeaufforderung
+ Gebt `cmd` ein
+ Gebt folgenden Befehl in der Kommandozeile ein:
  ```
  curl -i -v --json "{'snNumber': 'EureHubSeriennummer', 'account': 'EureEmailadresse'}" https://app.zendure.tech/v2/developer/api/apply
  ```
+ Zuvor habt ihr natürlich eure Seriennummer und eure verwendete Emailadresse anstelle der Platzhalter eingetragen

**Vorgehen auf einem Linux Betriebssystem**
+ Öffnet mit `Strg+Alt+T` ein Terminalfenster
+ Gebt folgenden Befehl in der Kommandozeile ein:
  ```
  curl -i -X POST -H 'Content-Type: application/json' -d '{"snNumber": "EureHubSeriennummer", "account": "EureEmailadresse"}' https://app.zendure.tech/v2/developer/api/apply
  ```
+ Zuvor habt ihr natürlich eure Seriennummer und eure verwendete Emailadresse anstelle der Platzhalter eingetragen

**Antwort**

Habt ihr keine Fehler gemacht, sollte eine Antwort wie folgt erscheinen:
```
{"code":200,"success":true,"data":{"appKey":"EuerAppKey","secret":"EuerAppSecret","mqttUrl":"mqtt.zen-iot.com","port":1883},"msg":"Successful operation"}
```

Anstelle der Platzhalter findet ihr dann euren `appKey` und euer `appSecret`. Beides sind Buchstaben-Zahlen-Kombinationen.
### Einrichtung von MQTT in Home Assistant
Je nachdem wie weit ihr euch in Home Assistant schon ausgetobt habt, gibt es nun mehrere Optionen zum Ziel.

1. **Option: Noch kein MQTT-Gerät mit Home Assistant verbunden**
   
   Wenn dies eure erste Berührung mit MQTT ist geht die Sache recht schnell.
   + Fügt über _Einstellungen - Geräte & Dienste_ eine neue Integration hinzu. Sucht dort nach MQTT und wählt die ohne irgendwelche weiteren Bezeichnungen.
   + Der Benutzername ist euer `appKey` und das Passwort euer `appSecret`. Die URL des Brokers und der Port wurden euch ebenfalls mit der o.a. Antwort geliefert: `mqtt.zen-iot.com` mit Port `1883`
   + Damit auch Daten reinkommen müsst ihr wie eingangs erwähnt, noch ein Topic abonnieren. Dies geschieht hier in dem ihr auf der Konfigurationsseite `Enable Discovery` aktiviert und als `Discovery prefix` euren `appKey` eintragt.
     
   + ![grafik](https://github.com/z-master42/solarflow/assets/66380371/769a98f3-8786-42c3-8cc5-0d761df5aee7)
2. **Option: Ihr nutzt bereits einen MQTT-Broker in Home Assistant**
   
   Wenn ihr bereits Erfahrungen mit MQTT gesammelt habt, weil ihr z. B. Sensoren oder Steckdosen darüber mit Home Assistant verbunden habt, ist die Wahrscheinlichkeit sehr groß, dass ihr das Mosquitto-Addon installiert und die MQTT-Integration für die Verbindung mit diesem konfiguriert habt. Problem: In Home Assistant kann nur eine MQTT-Integration installiert werden. Lösung: Ihr müsst eine Brücke zum MQTT-Broker von Zendure bauen.
   Hierbei gibt es zwei Möglichkeiten des weiteren Vorgehens. Entweder ist lasst euch durch Home Assistant alle verfügbaren Sensoren automatisch anlegen oder ihr fügt diese manuell hinzu. Direkt vorweg: Ich habe meine manuell hinzugefügt, so hatte ich die Möglichkeit diese direkt noch anzupassen.
   1. **Check vorweg**
      + Um zu überprüfen, ob seitens des Zendure-Brokers überhaupt Daten eures SolarFlows ausgespielt werden, bietet sich das Programm [MQTT-Explorer](http://mqtt-explorer.com/) an, welches es für die gängisten Betriebssysteme gibt.
      + Erstellt dort eine neue Connection mit euren Zugangsdaten (`appKey`, `appSecret`) wie im Screenshot

        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/f4555568-a65c-43a1-8c8a-e6fdbb46bb96)
      + Unter Advanced (Button) müsst ihr dann noch euren `appKey` als Topic abonnieren, also als neue Subscription hinzufügen --> `appKey/#`

        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/59e3ea65-2927-4e6f-9bd9-815e761ed3d9)
      + Es sollten dann ziemlich zeitig Werte reinkommen. Wenn ihr alles aufklappt sieht es ungefähr so aus:

        ![grafik](https://github.com/z-master42/solarflow/assets/66380371/cb2d9c67-40ce-4bcb-bd18-054219adf43d)
        Unter der Broker-Adresse erscheint ein Eintrag der wie euer `appKey` lautet (alles in 🔴). Darunter gibt es drei weitere Einträge; `switch`, `sensor`, und ein weiterer Eintrag der euren SolarFlo bezeichnet. Wir nennen ihn daher ab jetzt `deviceID` (alles in 🔵).


   3. **Gleicher Start**
      + Der Anfang ist bei beiden Möglichkeiten gleich
      + Zunächst muss die Brücke zum Zendure-Broker aufgebaut werden. Hiefür müsst ihr auf das `share`-Verzeichnis eures Home Assistant zugreifen. Über das File Editor-Addon ist dies z. B. nicht möglich. Über dieses habt ihr nämlich nur Zugriff auf das `config`-Verzeichnis. Ich bin daher den Weg über das Samba-Addon gegangen. Möglich ist auch der Weg über SSH, hierfür gibt es auch Addons, und dann das direkte Anlegen über z. B. _nano_
      + Erstellt im Verzeichnis eine Datei und nennt diese `zendure.conf`
      + Fügt folgenden Inhalt ein
        ```
        connection external-bridge
        address mqtt.zen-iot.com:1883
        remote_username appAppKey
        remote_password appSecret
        remote_clientid appKey
        topic appKey/# both 0
        ```
      + `appKey` und `appSecret` ersetzt ihr natürlich wieder durch eure eigenen
      + In der Konfiguration des Mosquitto-Addons überprüft ihr jetzt noch ob unter _Customize_ `active` auf `true` gesetzt ist.
      + Abschließend ist das Addon neu zu starten.
      + Im Log sollte dann ein Eintrag ähnlich `Connecting bridge external-bridge (mqtt.zen-iot.com:1883)` auftauchen. Ggf. müsst ihr das Log mehrmals aktualisieren (Geduld). Sollte hingehen irgendwas mit Timeout oder so kommen, einfach das Addon noch mal neu starten.

