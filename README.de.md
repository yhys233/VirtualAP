# VirtualAP

[English](README.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md) | [Русский](README.ru.md) | [Français](README.fr.md) | [Deutsch](README.de.md) | [日本語](README.ja.md) | [한국어](README.ko.md)

Verwandeln Sie ein Android-Telefon mit Root in einen echten Wi-Fi-Router.

VirtualAP betreibt einen richtigen Access Point auf dem Wi-Fi-Chip Ihres Telefons. Sie wählen
den Netzwerknamen, das Passwort, Band und Kanal, die Gateway-Adresse, die Ihre Geräte sehen,
und wohin deren Datenverkehr geht: mobile Daten, Wi-Fi oder ein VPN.

## Was Sie tun können

- **OpenWrt als Router für Ihren Hotspot betreiben.** Firewall-Zonen, Portweiterleitungen,
  Verkehrsregeln, Bandbreitenlimits und die LuCI-Weboberfläche, alles auf Ihrem Telefon.
  Siehe verwalteter Modus unten.
- **Ein VPN teilen.** Jedes Gerät im Hotspot läuft über das VPN Ihres Telefons. Auf dem
  Fernseher, der Konsole oder dem Laptop ist nichts zu installieren.
- **Wi-Fi weiterreichen.** Bleiben Sie mit einem Wi-Fi-Netz verbunden und strahlen Sie es als
  Ihr eigenes wieder aus.
- **Ein festes Gateway behalten.** Die Gateway-Adresse ändert sich nie, daher funktionieren
  Portweiterleitungen und gespeicherte SSH-Ziele weiter.
- **Das Funkmodul abstimmen.** 2.4 oder 5 GHz, Kanal und Breite, WPA2 oder WPA3, versteckter
  Netzwerkname.
- **Tethering vor Ihrem Anbieter verbergen** mit dem optionalen TTL-Fix, auf Kerneln, die ihn
  unterstützen.

## Verwalteter Modus: OpenWrt betreibt Ihren Hotspot

Das zeichnet VirtualAP aus. Übergeben Sie den Hotspot einem OpenWrt-Container in
[Droidspaces](https://github.com/ravindu644/Droidspaces-OSS), und OpenWrt verwaltet jedes
angeschlossene Gerät so, wie es ein echter Wi-Fi-Router tut. Öffnen Sie die von Ihnen gewählte
Gateway-Adresse im Browser, und Sie sind in LuCI.

Mit OpenWrt als Verwalter können Sie:

- eine Firewall mit beliebig vielen Zonen aufbauen
- Portweiterleitungen und Verkehrsregeln einrichten
- Bandbreite und Geschwindigkeit pro Gerät begrenzen
- ein Gästenetz betreiben, das Ihre anderen Geräte nicht erreicht
- jedes verbundene Gerät sehen und ihm eine feste Adresse und einen Namen geben
- die DNS-Server für das gesamte Netz wählen und Werbeblocker für alle Geräte aus der
  Paketquelle von OpenWrt hinzufügen
- Ihren Geräten IPv6 zusätzlich zu IPv4 geben
- Ihre Droidspaces-Container hinter denselben Router stellen, im selben Netz wie Ihre
  Wi-Fi-Geräte oder von ihnen abgeschottet
- alles Weitere installieren, was OpenWrt bietet, mit `opkg`

Die WAN-Seite von OpenWrt ist Droidspaces, und Droidspaces schaltet Uplinks in Echtzeit
zwischen mobilen Daten, Wi-Fi und VPN-Tunneln um. Ihr Hotspot bekommt das gratis:

1. Legen Sie in Droidspaces `tun0` als erstes Upstream-Interface des OpenWrt-Containers fest,
   gefolgt von `wlan0` und `rmnet0`. Ihr VPN läuft noch nicht, daher existiert `tun0` nicht.
2. Starten Sie VirtualAP. Der Datenverkehr Ihres Hotspots verlässt das Telefon über Wi-Fi oder
   mobile Daten.
3. Starten Sie Ihre VPN-App. Droidspaces sieht, dass `tun0` erscheint, und verschiebt OpenWrt
   dorthin, sodass jedes Gerät im Hotspot einen Augenblick später hinter dem VPN liegt. Nichts
   muss neu gestartet werden.

OpenWrt läuft in einem Container auf dem Kernel des Telefons selbst, ohne virtuelle Maschine
dazwischen.

Der verwaltete Modus benötigt:

- Root-Zugriff
- installiertes und funktionierendes
  [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS)
- den OpenWrt-Container aus dem
  [offiziellen Droidspaces-rootfs-Repository](https://github.com/Droidspaces/Droidspaces-rootfs-builder/releases),
  installiert und laufend

## Interface-Modus

Der einfache Modus, ohne Container und ohne Weboberfläche. VirtualAP vergibt Adressen selbst,
gibt Ihren Geräten IPv6, wenn der Upstream es hat, und sendet sie über das von Ihnen gewählte
Interface hinaus oder folgt dem Netz, das Android gerade nutzt. Das genügt, um ein VPN zu
teilen oder Wi-Fi weiterzureichen.

## Voraussetzungen

- Ein Telefon mit Root
- Android 8.0 oder neuer
- Ein ARM-Telefon, 64-Bit oder 32-Bit. Eine APK deckt beides ab.
- Für den Autostart des Access Points: Magisk oder KernelSU.
- Für den verwalteten Modus: Droidspaces und sein OpenWrt-Container (siehe oben)
- Für den TTL-Fix: ein Kernel, der ihn unterstützt. Die App prüft das und blendet die Option
  aus, wenn sie fehlt; ohne ihn funktioniert der Hotspot genauso.

## Erste Schritte

1. Laden Sie die neueste APK von [Releases](https://github.com/ravindu644/VirtualAP/releases)
   herunter und installieren Sie sie.
2. Öffnen Sie die App und gewähren Sie Root. Sie richtet alles Nötige selbst ein,
   einschließlich des optionalen Magisk/KernelSU-Bootmoduls. Nichts zu flashen oder neu zu
   starten.
3. Geben Sie einen Netzwerknamen und ein Passwort ein.
4. Wählen Sie den Upstream: ein **Interface** (oder Auto) oder Ihren OpenWrt-**Container** für
   den verwalteten Modus.
5. Starten Sie den Hotspot und verbinden Sie Ihre Geräte.

Aktivieren Sie **Start access point at boot** in den Einstellungen, damit er nach einem
Neustart zurückkehrt.

Öffnen Sie im verwalteten Modus `http://<your gateway address>` von einem verbundenen Gerät,
um LuCI zu erreichen.

## Mehr

- [So funktioniert es](Documentation/How-It-Works.md): das Routing, IPv6, die Bridge, das
  Funkmodul, die Dateien auf dem Telefon und wie die Binaries und die APK gebaut werden
- [Mitwirken](CONTRIBUTING.md)

## Lizenz

GNU General Public License v3.0. Siehe [LICENSE](LICENSE).

VirtualAP entstand aus dem Skript
[`start-hotspot`](https://github.com/ravindu644/Ubuntu-Chroot/blob/main/tools/start-hotspot)
aus [Ubuntu-Chroot](https://github.com/ravindu644/Ubuntu-Chroot); die App wurde mit Hilfe
eines KI-Assistenten geschrieben.
