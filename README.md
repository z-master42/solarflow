# Zendure SolarFlow [(click for English version)](README_eng.md)
Dies ist mein Versuch, meine gemachten Erfahrungen im Hinblick auf die Einbindung des Zendure SolarFlow-Systems, bestehend aus PV-Hub ZDSPV1200 und Energiespeicher AB1000, in das Heimautomatisierungsökosystem Home Assistant, zugänglich zu machen. Das ganze funktioniert natürlich auch mit dem PV-Hub ZDHUB2000 und dem Energiespeicher AB2000. Für das Komplettsystem AIO 2400 stellt Zendure aktuell noch keine Daten über MQTT bereit.
### Einbinden in Home Assistant
Im weiteren Kapitel [hier](solarflow.md), werde ich versuchen euch zu beschreiben, auf welchen Wegen ihr das SolarFlow-System in Home Assistant einbinden könnt.
### Nutzung in Verbindung mit einem Hoymiles-Wechselrichter
Für diejenigen, die einen Wechselrichter der Marke Hoymiles ihr eigen nennen, diesen an ihren PV-Hub angeschlossen haben und für die Steuerung und den Abruf der Wechselrichterwerte auf [OpenDTU](https://github.com/tbnobody/OpenDTU) oder [AhoyDTU](https://ahoydtu.de/) setzen, findet ihr [hier](hoymiles.md) meine Realisierung für eine Nulleinspeisung.
