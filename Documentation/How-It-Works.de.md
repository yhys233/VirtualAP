# How VirtualAP works

[English](How-It-Works.md) | [简体中文](How-It-Works.zh-CN.md) | [繁體中文](How-It-Works.zh-TW.md) | [Русский](How-It-Works.ru.md) | [Français](How-It-Works.fr.md) | [Deutsch](How-It-Works.de.md) | [日本語](How-It-Works.ja.md) | [한국어](How-It-Works.ko.md)

VirtualAP erstellt eine virtuelle AP-Schnittstelle, `ap0`, auf dem Wi-Fi-Chip des Telefons, führt `hostapd`
darauf aus, und arbeitet dann in einem von zwei Modi. Alles unten wird von `virtualap` erledigt, einem
statischen C-Binary, das als Root läuft, gebaut aus `src/`; die App ruft nur seine Befehle auf
(`start`, `stop`, `status`, `clients`, `interfaces`, `caps`). Es spricht direkt mit dem Kernel über
netlink (rtnetlink für Links, Adressen, Routen und Regeln; nl80211 für das Funkmodul) und über
den rohen `ip_tables`-Socket. Es startet `hostapd` und `dnsmasq`, `iptables` nur für den TTL-Fix,
und, im verwalteten Modus, `droidspaces`, um den Container zu finden und die OpenWrt-Einrichtung
darin auszuführen. Welche Container laufen, die Liste, die die App für den verwalteten Modus anbietet,
fragt die App Droidspaces selbst (`droidspaces show --json`, oder `show --format` bei einem Release von
vor `--json`).

## Gerouteter Modus ("Interface" in der App)

VirtualAP besitzt die gesamte Netzwerkschicht des Hotspots. Es gibt `ap0` die Gateway-Adresse
(`192.168.42.1/24`, sofern Sie keine andere gewählt haben), bedient DHCP und DNS mit seinem eigenen `dnsmasq`
(Leases von `.10` bis `.50` für 12 Stunden, DNS weitergeleitet an die Server, die Sie eingegeben haben, oder an
`1.1.1.1` und `8.8.8.8` standardmäßig), und NATet die Clients über den Upstream hinaus.

Der Verkehr wird mit Policy-Routing-Regeln gelenkt, die über Androids eigenen festgepinnt sind, welche bei
Priorität 10000 beginnen, sodass eine Catch-all-Regel eines VPNs den Verkehr des Hotspots nicht übernehmen kann:

```
client -> ap0
       -> ip rule pref 7010: from all iif ap0 lookup <upstream table>
       -> MASQUERADE (-s <subnet> ! -d <subnet>)
       -> internet or VPN tunnel

reply  -> ip rule pref 7000: to <subnet> lookup main -> ap0
```

Bei Auto ist der Upstream der erste von diesen, dessen Schnittstelle up ist: Androids netd-
Standardnetzwerk-Regel, das Netz, das gerade Internet hat, gelesen aus der Regelliste des Kernels,
die Form, in der Android 15 sie schreibt (ein UID-Bereich über jede UID) eingeschlossen; die Standardroute
der Haupttabelle, für ROMs, die sie füllen; ein Scan der bekannten Uplink-Namen (`wlan*`,
`eth*`, `v4-*`, `rmnet*`, `*ccmni*`), nur IPv4, weil der immer aktive IMS-Träger sie über IPv6
trifft. IPv4 durchläuft alle drei, bevor IPv6 versucht wird. Hotspot-, Tether- und VPN-
Schnittstellen werden bei Auto nie gewählt. `-o` schaltet all das ab und nimmt stattdessen eine kommagetrennte
Liste in Prioritätsreihenfolge, ohne Ausschlüsse: Der erste Eintrag, der up ist und eine Route hat,
gewinnt, IPv4-Routen über die ganze Liste vor IPv6. Eine Route zählt nur, wenn sie breiter als ein /8 ist, also zählt eine Standardroute oder die
`0.0.0.0/5` eines Split-Tunnel-VPNs, das eigene Subnetz einer Schnittstelle nicht.

`interfaces`, das die Dropdown-Liste der App zeigt, listet, was `-o` akzeptieren würde: die Links mit
einer solchen Route in irgendeiner Tabelle, beide Familien, damit auch der IMS-Träger auftaucht. Es lässt weg, was
VirtualAP, Droidspaces und USB Arsenal anlegen, und netds `dummy0`.

Kein Upstream ist ein normaler Zustand, kein Fehler: Der AP kommt allein mit Regel 7000 hoch, die
Clients werden sofort abgewiesen, und der Monitor fügt Regel 7010 hinzu, wenn einer erscheint. Die Firewall-
Regeln gehen durch den rohen `ip_tables`-Socket unter
Androids xtables-Sperre hinein, derselben Sperre, die netd und `iptables -w` halten, sodass keine Seite je
die Änderung der anderen überschreibt.

Die Route des AP-Subnetzes wird auch in Androids `local_network`-Tabelle (97) gespiegelt. Ohne
das würde eine Antwort von einem Droidspaces-Container (Quelle `172.28.0.0/16`) an einen Hotspot-Client
auf die WAN-Tabelle durchfallen und über den physischen Uplink statt über `ap0` hinausgehen, sodass
Clients Dienste in Containern nicht erreichen könnten.

### IPv6

Clients bekommen IPv6 nur, wenn der Upstream eine globale Adresse hat; ein Wi-Fi-Netz, das
nur ULAs herausgibt, lässt den AP IPv4-only. Es gibt zwei Wege, es zu geben, in dieser Reihenfolge versucht:

- NAT66, Droidspaces' Ansatz, wenn der Kernel `ip6table_nat` mit MASQUERADE hat und die
  Tabelle des Upstreams eine IPv6-Standardroute hat. `ap0` nimmt `::1` in einem privaten /64, nummeriert
  nach dem IPv4-Subnetz (`fd76:6170:42::/64` für `192.168.42.1`), `dnsmasq` kündigt es
  mit sich selbst als DNS-Server an, und die Clients werden aus dem Upstream masqueradiert. Es
  übersteht einen Wechsel des Upstreams oder Präfixes unberührt.
- Das eigene /64 des Upstreams, wie Androids Tethering es tut, auf Kerneln ohne IPv6-NAT
  (Stock-GKI liefert keines), wenn der Upstream ein globales /64 hält. `ap0` nimmt eine EUI-64-
  Adresse darin und `dnsmasq` kündigt das Präfix off-link an, sodass ein Client alles
  an das Gateway sendet. Jede Client-Adresse wird als /128 zu `ap0` geroutet und auf dem
  Upstream mit einem NDP-Proxy-Eintrag beantwortet. Der Monitor findet die Adressen, indem er alle 3 Sekunden
  `ff02::1` von `ap0` aus pingt, das Einzige, was sie offenlegt; er akzeptiert nur Antworten auf
  sein eigenes Echo, nie für eine Adresse, die dem Telefon selbst gehört oder ein Nachbar auf dem
  Upstream ist. Regel 7001 blackholed ein Upstream-Paket für eine Adresse im /64, die kein Client
  hält, das auf Mobilfunk sonst zurückspringen würde, bis sein Hop-Limit starb. Ohne den
  Monitor wird das Präfix nicht angekündigt.

Beide fügen Regel 7002 hinzu, die den eigenen Paketen des Telefons an eine Link-Local-Adresse eines Clients eine
Route gibt: Android legt die eigenen Routen einer Schnittstelle in einer Tabelle ab, die keine Regel liest, und die
Antwort von `dnsmasq` auf eine Router Solicitation ist eines dieser Pakete. IPv6 ist Best Effort: Wenn ein Schritt
fehlschlägt, bleibt der AP IPv4-only.

## Der Monitor

Ein erfolgreicher Start lässt `virtualap` als Monitor laufen (`run/monitor.pid`), einen Prozess
ohne Threads, nach dem Routenmonitor von Droidspaces und dem Tether-Daemon von USB-Arsenal. Er
lauscht auf die Link-, Adress-, Routen- und Regel-Ereignisse des Kernels, schaut 200 ms nach dem letzten
Ereignis eines Schubs, und schaut unabhängig davon alle 3 Sekunden, weil netd `iptables` ohne
Ereignis umschreibt. Jeder Blick:

- löst den Upstream erneut auf und verschiebt Regel 7010 und `run.state` auf einen neuen, damit ein Wechsel von Wi-Fi zu
  Mobilfunk die Clients nicht ohne Internet lässt, und ein Start ohne Netz
  seinen ersten Upstream auf dieselbe Weise bekommt. Wird nichts gefunden, lässt Auto die Regel in Ruhe,
  es gibt nichts Besseres, worauf sie zeigen könnte, und `-o` entfernt sie, bis eine gelistete Schnittstelle
  zurückkehrt, damit ihr Verkehr nie über etwas anderes hinausgeht;
- legt jede Regel oder jeden Firewall-Eintrag wieder hin, den netd weggewischt hat, und `ip_forward`, nur wenn er fehlt,
  damit seine eigenen Schreibvorgänge es nie wieder wecken;
- macht IPv6 erneut, die Ankündigung von `dnsmasq` und den TTL-Fix damit, wenn sich das IPv6 des Upstreams
  geändert hat: ein neues Präfix, eine andere Schnittstelle, IPv6 erschienen oder verschwunden.

Im verwalteten Modus beobachtet er stattdessen das veth und steckt ein neues in den Container, wenn
der Container neu startet. Er beendet sich, wenn der AP weg ist; `stop` tötet ihn vor allem anderen.

## Verwalteter Modus (`-K <container>`)

Das LAN des Hotspots wird an einen laufenden Droidspaces-Container übergeben. VirtualAP gibt `ap0` keine
Adresse, führt kein `dnsmasq` aus und installiert kein NAT: Es baut einen Layer-2-Pfad und der Container
ist der Router.

```
client -> ap0 (bridge port, no address)
       -> vap-br0 -> vaplan0 (inside the container)
       -> container LAN (DHCP, DNS, firewall)
       -> container NAT -> container WAN -> internet
```

`ap0` wird an eine Host-Bridge versklavt, `vap-br0`, die keine Adresse trägt und IPv6
deaktiviert hat, sodass sie sich nie aus den Ankündigungen des Containers selbst konfiguriert. Ein veth-Paar
verbindet die Bridge auf der Host-Seite; das andere Ende wird im Netzwerk-Namespace des Containers erzeugt,
bereits `vaplan0` genannt und mit seiner MAC, in einer Anfrage. Ein Link, der auf dem
Host erzeugt und nach dem Umzug umbenannt wird, erscheint auf Kerneln vor 5.x zuerst unter seinem alten Namen,
und OpenWrts netifd schaut ihn nicht wieder an. Wenn der Container OpenWrt ist, richtet VirtualAP
ihn mit einem `uci`-Skript ein, das durch `droidspaces run` läuft: eine statische `vaplan`-
Schnittstelle am konfigurierten Gateway, ein IPv6-/64 aus der ULA des Routers, an die Nummer des IPv4-
Subnetzes gepinnt, damit netifd es nicht umnummerieren kann, ein DHCP-Pool mit Router-Ankündigungen, und ein
Neuladen der iptables-Firewall, oder eine MASQUERADE-Regel, wenn das Image kein Firewall-
Paket hat. Jeder andere Container empfängt `vaplan0` und konfiguriert es selbst.

Das veth-Paar läuft mit MTU 1508, während `ap0` und die Clients bei 1500 bleiben. Samsungs Broadcom-
Firmware übergibt etwa ein Viertel der empfangenen Frames mit dem 8-Byte-CCMP-MIC noch am
Ende, und eine Bridge oder ein veth gibt jeden Frame still frei, der länger als seine MTU plus 18 Byte ist, also
ging bei 1500 ein vollgroßer Frame eines Clients einmal in vier verloren und Uploads in den
Container krochen mit etwa 1 MB/s. Der IP-Stack im Container schneidet den MIC ab, und die
RA pinnt die IPv6-MTU der Clients mit `ra_mtu` auf 1500, weil odhcpd sonst die
1508 des Ports ankündigt.

Beide Enden des veth bekommen RPS, auf jeder CPU außer CPU0. Ein veth stellt jedes Paket in die Warteschlange auf der CPU,
die es gesendet hat, und ein Fluss kommt von mehreren an, weil der Wi-Fi-Treiber seine
Empfangsarbeit über Kerne verteilt und TCP von dem Kern sendet, der das letzte ACK nahm. Ohne RPS
sieht das ferne Ende den Fluss außer der Reihe, und TCP überträgt Daten neu, die nie verloren gingen.
Die Maske von `vaplan0` wird von einem Kind geschrieben, das in den Netzwerk-Namespace des Containers eintritt und
dort ein eigenes `/sys` einhängt, nie über `/proc/<pid>/root`, dessen Symlinks der
Container wählt. Der Kernel behält nur die CPUs online, wenn die Maske geschrieben wird, also bleiben Kerne,
die in dem Moment geparkt sind, draußen, bis der AP neu startet.

Ein frisches veth hat bei jedem Start eine neue zufällige MAC, und Windows richtet seine Netzwerkprofile an der
MAC des Gateways aus, also kann die App eine pinnen: eine zufällige MAC, über Starts hinweg behalten, oder die eigene
Wi-Fi-MAC des Telefons. Wenn der Container neu startet,
steckt der Monitor ein neues veth hinein, und die OpenWrt-Konfiguration, die den Neustart überlebt hat, nimmt es
auf. Der alte Namespace nimmt das veth normalerweise mit, aber es überlebt seinen letzten Prozess,
solange etwas eine Referenz hält, und dann bleibt das Host-Ende, in nichts hinein lugend. Also
prüft der Monitor beides: Das Host-Ende ist da, und der Prozess, in den der Start steckte, ist
noch in dem Namespace, in den er gesteckt wurde.

## Der TTL-Fix

Mit `-T 1` verlassen weitergeleitete Pakete mit TTL 64 (Hop-Limit 64 für IPv6), also sehen sie für den
Betreiber wie die eigenen des Telefons aus. Die Regeln leben in einer eigenen Mangle-Kette,
`vap_ttl`, abgeglichen auf das Quell-Subnetz der Clients in `FORWARD`, das nur weitergeleitete
Pakete sieht, nachdem der Kernel seinen Hop abgezogen hat. Das Backend prüft auf das TTL-Ziel des Kernels
(`CONFIG_NETFILTER_XT_TARGET_HL`), indem es die echte Regel in einer Kette hinzufügt, in die niemand springt; die
App graut die Option aus, wenn die Probe fehlschlägt. Dies sind die einzigen Regeln, die noch vom
`iptables`-Binary geschrieben werden.

## Das Funkmodul

Die meisten Telefon-Chips können nur auf dem Kanal beaconen, mit dem die Station verbunden ist, also
folgt das Backend dem Kanal der Station, wenn er ins Band passt. Der Kanal der Station kommt
von der Frequenz des BSS, mit dem sie in den Scan-Ergebnissen assoziiert ist, was
`iw dev wlan0 link` liest, weil Samsungs Broadcom-Treiber den Kanal aus den
Interface-Infos auslässt. Eine Station auf einem DFS-Kanal (52 bis 144) fällt auf 2.4 GHz zurück: Telefon-Chips
können dort nicht beaconen. Eine Station auf einem anderen Band, 6 GHz eingeschlossen, lässt dem AP seinen eigenen
Standard. Ohne Station startet 5 GHz auf Kanal 36 und 2.4 GHz auf Kanal 6.

Samsungs Broadcom-FullMAC-Firmware bootet in der Welt-Regdomain "99", und das Framework
schiebt manchmal nie ein echtes Land nach, also bleibt jeder 5-GHz-Kanal NO-IR und `hostapd`
stirbt nach etwa fünf Sekunden, nach der Liveness-Prüfung. Die Wiphy ist selbstverwaltet, also
werden `iw reg set` und `hostapd`s `country_code` ignoriert; der einzige Hebel ist der private
`COUNTRY`-Befehl des Treibers über den Steuer-Socket des Supplicants, gesendet mit dem Land, das
das Framework meldet (`ensure_5g_ir`). Er feuert nur, wenn die Phy den Kanal als
NO-IR meldet, und wenn der Kanal danach noch NO-IR ist, fällt der AP auf 2.4 GHz zurück.
`hostapd.conf` behält `country_code=US`.

Die Breite folgt dem, was Chip und Kanal können, aus den Fähigkeiten jenes Bandes: 40 MHz
auf 2.4 GHz nur, wenn namentlich verlangt, bis zu 80 MHz auf 5 GHz, wenn das Band VHT listet und
der Kanal in einem 80-MHz-Block sitzt. HT40 und VHT brauchen, dass `hostapd` seinen 20/40-MHz-
Koexistenz-Scan überspringt, den FullMAC-Chips auf einer AP-Schnittstelle nicht laufen lassen können; die
Option `no_pri_sec_switch`, die das tut, ist ein Patch in unserem `hostapd`-Fork, und der Build
stellt sicher, dass der Patch vorhanden ist. Wenn `hostapd` 40 MHz auf 2.4 GHz ablehnt, versucht das Backend es
mit 20 erneut.

Der Fork sagt `hostapd` auch nie, dass ein Treiber eine MAC-ACL hat, und der Build stellt auch das
sicher. Die ACL eines Treibers kann weniger Adressen halten als die Liste, und darüber hinaus überspringt `hostapd` seine
eigene Prüfung, während der Treiber nichts hält. Also werden die MAC-Listen in `hostapd` auf jedem
Chip geprüft: Ein gelistetes Gerät assoziiert für einen Moment und wird vor seinem Handshake entfernt. Samsungs
Broadcom-Builds kompilieren ihre ACL heraus und gen4m hat keine, also arbeiten sie bereits so.
Qualcomms qcacld hat eine, also ist es der Treiber, den das ändert, und kein qcacld-Telefon hat es laufen lassen.

MediaTeks gen4m-Treiber hat kein `dump_station`, also kommt die Client-Anzahl aus `hostapd`s
eigener Stationsliste über seinen Steuer-Socket, gebunden an eine abstrakte Adresse, da Android
kein `/tmp` hat, mit einem Timeout von einer Sekunde, damit ein hängendes `hostapd` `status` nicht blockieren kann.

`clients` läuft dieselbe Liste durch und gibt `mac ip name` für jede Station aus, die ihren
Handshake abgeschlossen hat, `-` für das, was nicht bekannt ist. Die Anzahl in `status` ist derselbe Durchlauf, also ist ein Gerät
mit dem falschen Passwort in keinem von beiden. Die App fragt
nur, während ihr Sheet der verbundenen Geräte offen ist, da der verwaltete Modus `droidspaces` startet, und
dieser Start hat ein Limit von fünf Sekunden, weil libsu einen Befehl zur Zeit ausführt. Die
Adresse und der Name kommen aus dem neuesten Lease im /24 des Gateways: unser eigenes `dnsmasq.leases`
im gerouteten Modus, das `/tmp/dhcp.leases` des Containers im verwalteten Modus. Jenes wird durch
`/proc/<pid>/root` gelesen, da OpenWrts `/tmp` ein tmpfs im Mount-Namespace des Containers ist, eine
Komponente zur Zeit und nur als reguläre Datei auf einem tmpfs, sodass ein Symlink, eine FIFO oder ein Bind-
Mount, den der Container dort hinterließ, abgelehnt wird. Ein Client, der keinen Hostnamen sendet, oder ein Container
ohne solche Datei, lässt diese Felder `-`.

Ein Gerät auf einer statischen Adresse behält den Lease der Adresse, die es hatte, also korrigiert es die ARP-Tabelle des Kernels:
`/proc/net/arp` im gerouteten Modus, `/proc/<pid>/net/arp` im verwalteten Modus, wo das
LAN der Clients im Container endet. Ein Lease steht, solange die Tabelle ihn bestätigt oder keine
andere Adresse für diese MAC kennt, und die Tabelle kann einem Gerät ohne Lease eine Adresse geben. Sie
hat nur ein Gerät, das etwas gesendet hat, und sie markiert auch einen veralteten Eintrag als vollständig, also kann ein
Gerät eine Adresse zeigen, die es gerade verlassen hat.

`block <mac> [name]` fügt ein Gerät zu `/data/local/virtualap/blocklist` hinzu, und `unblock <mac>`
nimmt es heraus; `blocked` gibt `mac name` für jedes aus, `-` für keinen Namen. `allow`, `disallow` und
`allowed` tun dasselbe für `allowlist`. `allow-only on` hält jedes Gerät draußen, das nicht auf der
Allowlist steht, wirft alle, die verbunden sind, heraus, und wird abgelehnt, während die Liste leer ist; das letzte Gerät
von der Liste schaltet es ab, da sonst niemand beitreten könnte. Ohne Argument gibt es `on` oder
`off` aus. Dieser Zustand ist die Datei `allow_only`, neben den Listen, weil
sie sich wie diese mit einem Befehl zur Zeit ändert, Hotspot an oder aus. Ein Gerät steht nur auf einer Liste, und
es zu einer hinzuzufügen nimmt es von der anderen, weil `hostapd` die Allowlist zuerst prüft und ein Gerät auf
beiden hereinlassen würde.

Die Listen liegen außerhalb von `ap.conf`, weil die App sie Eintrag für Eintrag bearbeitet, nicht als Einstellung, die ein
Start übergibt. Die Dateien sind `hostapd`s eigene `deny_mac_file` und `accept_mac_file`, in seiner
Konfiguration nur benannt, wenn sie existieren, da `hostapd` mit einer Liste, die es nicht öffnen kann, nicht startet. Eine Zeile ist
`mac #name`: `hostapd` liest, was einer MAC folgt, als VLAN-ID durch `atoi`, und `#` macht das
zu 0, kein VLAN. Ein laufender Hotspot hört eine Änderung über den Steuer-Socket: `DENY_ACL ADD_MAC`
wirft das Gerät heraus, wenn es verbunden ist, und `ACCEPT_ACL DEL_MAC` ebenso, sobald nur erlaubte Geräte
beitreten können. Ein Start schreibt die Dateien zuerst neu, loggt und verwirft jede Zeile, bei der er nicht sicher ist, dass
`hostapd` sie lesen kann, und jede hält etwa 400 Geräte.

Die App blockiert aus ihrem Geräte-Sheet ohne Bestätigung, weil Entsperren daneben sitzt, und
behält beide Listen in den Einstellungen, für wenn der Hotspot aus ist. Ihr MAC-Feld kann nichts
außer Hexziffern halten, und lehnt ab, was die Engine ablehnt: alles Nullen und Gruppenadressen. Entweder Liste
hält nur diese MAC. Ein Gerät, das seine eigene ändert, kommt an einer Blockierung vorbei und fällt von einer erlaubten
Liste, und Telefone nutzen standardmäßig eine zufällige Adresse für jedes Netz, also ist die einzugebende die
in den Wi-Fi-Details des Geräts für diesen Hotspot.

## Auf dem Telefon

Alles liegt unter `/data/local/virtualap`:

- `bin/`: `virtualap`, `hostapd` und `dnsmasq`, vollständig statische musl-Binaries für die
  Architektur des Telefons. `iptables` ist Androids eigenes.
- `ap.conf`: die Einstellungen des Hotspots als Zeilen `KEY='value'`. Die App schreibt sie, wann immer das, was auf
  dem Bildschirm steht, gültig ist, und vor jedem Start, und ein halb ausgefülltes Formular bleibt auf dem Bildschirm. Die
  Engine liest sie nur, und ein Flag überschreibt sie für diesen Lauf. Sie enthält das Wi-Fi-Passwort, also
  ist sie nur für Root lesbar.
- `blocklist`, `allowlist` und `allow_only`: die blockierten und erlaubten Geräte und ob nur
  diese beitreten können, siehe [oben](#das-funkmodul). Es gibt keine davon, bis sie erstmals genutzt werden.
- `.boot_mode`: `1`, wenn der Hotspot beim Booten startet. Der Schalter in den Einstellungen schreibt sie und das Boot-
  modul liest sie.
- `run.state`: was der laufende AP abgebaut haben muss, das Gateway, den Upstream, den IPv6-
  Modus und die Schnittstelle und die wiederherzustellenden sysctl-Werte. `status` leitet "running" daraus ab und
  aus der `hostapd`-Pid-Datei; `stop` entfernt sie, bevor es beendet wird.
- `run/`: die Pid-Dateien (`hostapd`, `dnsmasq`, der Monitor), die generierte `hostapd.conf`
  und `dnsmasq.conf`, die Leases, die IPv6-Client-Liste, `hostapd`s Steuer-Socket und die
  Sperre, die zwei Starts oder Stopps davon abhält, gleichzeitig zu laufen.
- `logs/`: `ap.log`, eine mit Zeitstempel versehene Kopie jeder Zeile, die die App zeigte, und dessen, was der
  Monitor tat, bei jedem Start abgeschnitten, plus `hostapd.log` und `dnsmasq.log`, weil beide
  Daemons sonst ihren Fehlergrund mitnähmen, und `boot-module.log`, was das
  Bootmodul beim letzten Boot tat.

Auf eine Pid-Datei wird nur eingewirkt, wenn der Prozess, den sie nennt, das erwartete Programm ausführt, also wird eine
nach einem Neustart wiederverwendete Pid nie signalisiert. Die App deployt `bin/` beim ersten Lauf und erneut,
wann immer eine APK andere Binaries ausliefert: Der Build schreibt `PAYLOAD_VERSION`, einen Hash der
Binaries jeder Architektur und des Bootmoduls, daneben, und die App vergleicht ihn mit dem, den
sie zuletzt deployt hat.

Das Deployen des Backends installiert auch `/data/adb/modules/virtualap`, das Bootmodul, auf dieselbe Weise
wie Droidspaces sein eigenes installiert. Ein Telefon ohne Root-Manager muss ohne es auskommen, und der Schalter in den Einstellungen,
der nur `.boot_mode` schreibt, hat dann nichts, was ihn liest. Die `service.sh` des Moduls läuft im späten
Start und wartet auf `sys.boot_completed`. Mit `.boot_mode` an gibt es der Wi-Fi-Schnittstelle 25
Sekunden zu erscheinen und, im verwalteten Modus, dem Container weitere 25 zu laufen, und überspringt den Start, wenn
eines von beiden es nicht tut. Es wartet nicht auf das Netz, da der Hotspot einem Upstream folgt, der
später kommt. Der Start liest `ap.conf`, die außerhalb des verschlüsselten Speichers der App liegt, also braucht der Boot-Start
keine Entsperrung. Das Ergebnis erscheint als Beschreibung des Moduls im Root-
Manager. Nach einem erfolgreichen Start wartet es auf die erste Entsperrung, vor der die App nicht laufen kann,
und sendet `HotspotBootReceiver` einen Broadcast, der die laufende Benachrichtigung aufstellt.

## Bauen

`hostapd` und `dnsmasq` werden aus den git-Submodulen unter `externals/` gebaut, unseren eigenen Forks,
durch `scripts/build-static.sh`. Jede Architektur baut in einem Alpine-3.23-Container dieser Architektur,
unter QEMU auf einem x86-Host, mit `-static -no-pie`, sodass das Ergebnis ein eigenständiges ELF ohne
Interpreter ist. Der `dnsmasq`-Fork lässt den Privilegienabfall per `getpwnam("root")` weg, da Android
keinen Root-Eintrag in `/etc/passwd` hat und statisches musl keinen synthetisieren wird. `virtualap`
linkt nichts außer libc und braucht keinen Container: Das `Makefile` der Wurzel baut es auf die
Droidspaces-Art, `make aarch64 armhf` mit dem musl-cross gcc in
`~/toolchains/<triplet>-cross` (oder `MUSL_CROSS`), den die Gradle-Aufgabe `buildBackend` vor
jedem App-Build ausführt. Die Ausgabe landet in `backend/<arch>/`, das in .gitignore steht, und die
Gradle-Aufgabe `prepareAssets` legt die drei Binaries in die Assets der APK, und schlägt fehl, wenn eines
fehlt.

`src/` hat einen Ordner pro Schicht, jeder mit einem öffentlichen Header unter `src/include/<folder>/`
und, wo seine Dateien mehr teilen, einer privaten `internal.h` neben seinen Quellen:
`core`, `cli`, `net` (rtnetlink, und die `ip_tables`-Engine in `net/ipt/`), `radio`
(nl80211, der Steuer-Socket, Kanalpolitik), `mode` (geroutet, verwaltet, IPv6, `hostapd` und
`dnsmasq`), `monitor` und `cmd`. `make check` führt die Asserts in `tests/` aus, die netlink-
gegen den Kernel in einem Wegwerf-Benutzer- und Netzwerk-Namespace, sodass weder Root noch ein
Telefon nötig ist; für das Telefon gebaut und dort als Root ausgeführt, üben sie auch das echte
`ip_tables`.

CI baut `hostapd` und `dnsmasq` für jede Architektur auf einem ARM-Runner, gecacht auf den
`externals/`- und `scripts/`-Bäumen. Der APK-Job holt dann Droidspaces' veröffentlichte musl-
Toolchains (gegen ihre sha256 geprüft und auf ihrer Release-URL gecacht), führt
`make check` und `make aarch64 armhf` mit Warnungen als Fehlern aus, und baut die APK mit
JDK 17. Ein gepushtes `v*`-Tag, das zu `VAP_VERSION` in `src/include/virtualap.h` passt, veröffentlicht
ein Release mit der APK, einer `SHA256SUMS`-Datei und einem Changelog jedes Commits seit dem
vorherigen Tag; das Tag muss dazu passen und der Signaturschlüssel muss vorhanden sein, oder der Lauf stoppt.
CI führt keinen Formatter und keinen Linter aus.

`make bump NEW=x.y.z` schreibt `VAP_VERSION` neu, und Makefile, Gradle und CI lesen es alle
von dort. Der versionCode der APK ist der Release-Code (major·10000 + minor·100 + patch)
mal 10000 plus `git rev-list --count HEAD`, sodass die APK jedes Commits über die letzte installiert.
Die Anzahl braucht die volle Historie, weshalb der APK-Job mit `fetch-depth: 0` auscheckt: Ein
flacher Clone zählt 1 und ein Baum ohne `.git` zählt 0. Die `module.prop` des Bootmoduls
nimmt allein den Release-Code, weil er in `PAYLOAD_VERSION` gehasht wird.

## Übersetzungen

Der Text der App steht in `Android/app/src/main/res/values/strings.xml` und ihre Plurale in
`plurals.xml` daneben, zwei Weblate-Komponenten, `Strings` und `Plurals`, dasselbe Layout wie
Droidspaces. Beide nutzen das Dateiformat Android String Resource, die Dateimaske
`Android/app/src/main/res/values-*/strings.xml` oder `plurals.xml`, die Basisdatei in `values/`
als sowohl die einsprachige Basissprachdatei als auch die Vorlage für neue Übersetzungen, und
Android-Sprachcodes (`values-pt-rBR`). Schalten Sie das Weblate-Add-on Cleanup translation files
ein: Ein Schlüssel, der aus einer Basisdatei entfernt und in einer Übersetzung belassen wird, ist Lints `ExtraTranslation`,
der tödlich ist, also schlägt der Release-Build fehl, bis er geht. Eine fehlende Übersetzung fällt
nur auf Englisch zurück.

Die App folgt der Systemsprache, bis die Sprachauswahl in den Einstellungen etwas anderes sagt. Die Liste der Auswahl
und Android 13s Liste der Sprachen pro App sind eine Datei: `generateLocaleConfig` schreibt
`res/xml/_generated_res_locale_config.xml` aus den `values-*`-Ordnern, `res/resources.properties`
sagt, dass `res/values` Englisch ist, und `LocaleHelper` liest das Ergebnis, sodass eine aus
Weblate zusammengeführte Sprache beim nächsten Build angeboten wird, wie wenig davon auch übersetzt ist. Die Wahl geht durch
AppCompat, weshalb `MainActivity` eine `AppCompatActivity` ist: Android 13 und später behalten sie im
System, und auf 8 bis 12 speichert AppCompat sie selbst und umhüllt die Activity, nicht die Anwendung
oder Dienste, also bleiben dort die Benachrichtigung und die Kachel der Schnelleinstellungen in der Systemsprache. Auf jeder
Version nimmt eine bereits aufgestellte Benachrichtigung eine neue Sprache bei ihrer nächsten Aktualisierung an.
Das Log bleibt auf Englisch, es ist das, was
die Engine ausgegeben hat. JetBrains Mono, für dieses Log und das Passwort auf dem QR-Sheet, ist die einzige
gebündelte Schrift; jede übersetzte Beschriftung steht in der Systemschrift, sodass jede Schrift zeichnet.
