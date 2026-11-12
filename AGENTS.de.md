# AGENTS.md

[English](AGENTS.md) | [简体中文](AGENTS.zh-CN.md) | [繁體中文](AGENTS.zh-TW.md) | [Русский](AGENTS.ru.md) | [Français](AGENTS.fr.md) | [Deutsch](AGENTS.de.md) | [日本語](AGENTS.ja.md) | [한국어](AGENTS.ko.md)

Anweisungen für KI-Coding-Agenten, die in diesem Repository arbeiten. Menschen sollten
[CONTRIBUTING.de.md](./CONTRIBUTING.de.md) lesen, das die Bestandsliste dessen enthält, was
hier bereits existiert. Wie die Sache funktioniert, steht in
[Documentation/How-It-Works.de.md](./Documentation/How-It-Works.de.md).

## Was das ist

VirtualAP verwandelt ein gerootetes Telefon in einen Wi-Fi-Zugangspunkt. Zwei Hälften:

- `src/` ist die Engine, `virtualap`, ein statisches C-Binary, das als Root läuft und zwei
  weitere ansteuert: `hostapd` und `dnsmasq`, gebaut aus den git-Submodulen unter `externals/` durch
  `scripts/build-static.sh`. `make aarch64 armhf` baut `virtualap`. Alle drei landen in
  `backend/aarch64/` und `backend/armhf/`, die in .gitignore stehen.
- `Android/` ist die Compose-App, die `virtualap` über eine Root-Shell ansteuert.

Die Gradle-Aufgabe `prepareAssets` legt beide Architektur-Bäume in ein generiertes Assets-Wurzelverzeichnis unter
`Android/app/build/`, und die App deployt die passende Architektur beim ersten Start und nach jedem APK-Update nach
`/data/local/virtualap`, mit einem Bootmodul für den Root-Manager. Es gibt kein
chroot.

## Bauen

```
./scripts/build-static.sh    # from the repository root: both arches, needs Docker
./build.sh [release]         # from Android/: the APK, needs JDK 17 or newer, ANDROID_HOME and musl-cross gcc
```

Führen Sie den ersten Befehl vor dem ersten App-Build aus und erneut, wann immer `externals/` oder die Build-Skripte
sich ändern. Der App-Build führt `make aarch64 armhf` selbst aus, daher passt `virtualap` immer zu `src/`.
`prepareAssets` schlägt fehl, wenn in `backend/<arch>/` eines der drei fehlt. `make check` führt die
Tests aus. Die Version stammt aus `VAP_VERSION` in `src/include/virtualap.h`. Erhöhen Sie sie mit
`make bump NEW=x.y.z`, nirgendwo sonst. Die Details zu Build und CI stehen im Abschnitt Bauen von
How-It-Works.md.

## Commits

- Jeden Commit mit Sign-off versehen: `git commit -s`.
- Nie einen `Co-Authored-By:`-Trailer für einen KI-Agenten hinzufügen. Menschliche Co-Autoren sind in Ordnung.
- Präfixe, passend zur bestehenden Historie:
  - `app:` für die Android-App, mit `app: fix:` und `app: refactor:` für diese Fälle
  - `backend:` für die Engine unter `src/`, `fix:` für eine Fehlerbehebung in der Engine
  - `scripts:` für den Bau des statischen Binarys unter `scripts/` und `externals/`, `ci:` für
    `.github/workflows/`, `docs:` für Dokumentation
  - `fix(security):` für alles Sicherheitsrelevante, in beiden Hälften
  - `VirtualAP: bump vX.Y.Z` für einen Versionssprung, und sonst nichts in diesem Commit

`ap:`, `build:`, `ui:`, `feat:` und `refactor:` kommen in älteren Historien vor. Verwenden Sie sie nicht für
neue Commits.

## Stil

Vier Regeln. Sie gelten für Code, Kommentare, Commit-Nachrichten und Dokumentation.

**Keine Em-Dashes.** Verwenden Sie ein Komma, einen Punkt, oder schreiben Sie den Satz neu.

**Keine ASCII-Banner-Kommentare.** Keine Reihen aus `-----`, kein `=====`, keine gerahmten Abschnittsüberschriften.

**Kommentare klingen wie von einem Menschen.** Sagen Sie, warum der Code etwas tut, oder was bricht, wenn
er es nicht tut. Lassen Sie Kommentare weg, die nur die Zeile darunter wiederholen.

```sh
# Bad: bump the priority
PRIO_FROM_AP=7010

# Good: netd's own rules start at 10000, so 7010 wins before the VPN catch-all
PRIO_FROM_AP=7010
```

**Zehn Zeilen, die funktionieren, schlagen hundert, die dasselbe tun.** Löschen vor dem Hinzufügen. Ein
kleinerer Diff am richtigen Ort ist das Ziel, nicht ein kleinerer Diff irgendwo.

In den Dokumenten wird jeder Fakt einmal genannt, auf der Seite, auf der ein Leser ihn sucht, und
die anderen Seiten verlinken darauf. Die Aufteilung ist fest: README.md sagt, was, für wen und die Schritte;
How-It-Works.md enthält jeden technischen Fakt; diese Datei enthält die Regeln für Agenten;
CONTRIBUTING.md sagt, wie ein Mensch eine Änderung gemergt bekommt, und enthält die Bestandsliste; DESIGN.md
enthält die UI-Werte. Die Tabellen in DESIGN.md bleiben, sie sind Rollenzuordnungen und Größen, die als Prosa nicht funktionieren;
anderswo eine Liste oder ein Absatz.

## UI-Änderungen

Alles Visuelle in `Android/` folgt [DESIGN.de.md](./DESIGN.de.md). Farben, Typografie, Abstände, Radien und
das Muster der Aktions-Pill sind alle dort festgelegt, und sie wurden aus der bestehenden App abgelesen statt erfunden,
daher sind sie zu befolgen auch der kleinste Diff.

Wenn eine Regel wirklich nicht zu Ihrem Fall passt, weichen Sie ab, aber hinterlassen Sie an der Stelle einen Kommentar, der sagt, was
die Abweichung bringt, und sagen Sie es im PR. Eine undokumentierte Abweichung ist Drift, und der nächste
Beitragende wird sie "reparieren". Der Abschnitt "Entschiedene Ausnahmen" in DESIGN.md listet die Abweichungen, die
Entscheidungen sind, "reparieren" Sie diese nicht.

## Bevor Sie überhaupt schreiben

Arbeiten Sie diese Liste von oben nach unten ab und halten Sie bei der ersten Antwort an, die trägt.

1. Muss das existieren? Ein Flag, nach dem niemand gefragt hat, ein Regler für einen Wert, der sich nie ändert,
   eine Schnittstelle mit einer Implementierung: überspringen Sie es und sagen Sie es in einer Zeile.
2. Existiert es hier schon? Siehe "Vor dem Schreiben wiederverwenden" unten. Greppen Sie zuerst.
3. Erledigt es die Plattform bereits? Auf der Engine-Seite musls libc und die eigenen Schnittstellen des Kernels (netlink,
   `/proc/sys`); auf der App-Seite die Kotlin- und Java-Standardbibliothek und das Android-Framework.
   Beachten Sie `minSdk = 26`.
4. Deckt es eine installierte Abhängigkeit ab? Auf der App-Seite libsu, Compose, lifecycle, zxing, appcompat.
   Fügen Sie nie eine neue hinzu für etwas, das wenige Zeilen erledigen können.
5. Passt es in eine Zeile? Dann ist es eine Zeile.
6. Erst dann schreiben Sie das Kleinste, das funktioniert.

Zwei Antworten treffen zu? Nehmen Sie die höhere und machen Sie weiter.

Die Liste verkürzt die Lösung, nie das Lesen. Verfolgen Sie den Ablauf, den die Änderung berührt, bevor
Sie eine Stufe wählen. Ein kleiner Diff am falschen Ort ist ein zweiter Bug, keine faule Lösung. Dasselbe gilt für
Bugmeldungen: Eine Meldung nennt ein Symptom, also greppen Sie jeden Aufrufer, bevor Sie bearbeiten. Eine Absicherung in
der gemeinsamen Funktion ist kleiner als eine Absicherung in jedem Aufrufer, und sie behebt die Geschwister, die
die Meldung nicht erwähnt hat.

Vereinfachen Sie niemals eine Eingabevalidierung an einer Vertrauensgrenze weg, eine fail-closed-Prüfung, eine
Fehlerbehandlung, die Zustand verliert, oder irgendetwas, nach dem der Anfragende namentlich gefragt hat.

## Vor dem Schreiben wiederverwenden

Greppen Sie die Bestandsliste in CONTRIBUTING.md, dann den Baum. Das sind Engstellen. Eine zu
umgehen ist ein Bug, keine Abkürzung.

- `Backend.quote()` in `Android/app/src/main/java/com/virtualap/app/util/Backend.kt` umschließt
  jeden dynamischen Wert, der eine Root-Shell erreicht. `APManager` ist die einzige Klasse, die
  `virtualap` ausführt, und `Backend.startAp` ist der einzige Weg, seinen Pfad zu schreiben.
- `Hotspot` ist der einzige Aufrufer von `APManager.start` und `APManager.stop`. Er besitzt die
  Phase des Startens und Stoppens, das Befehls-Log, die laufende Benachrichtigung und die Synchronisierung der Kachel
  der Schnelleinstellungen, damit Bildschirm und Kachel immer übereinstimmen. `APConfig.isValid()` ist die einzige
  Startvalidierung; die Schaltfläche des Bildschirms und die Kachel fragen sie beide.
- `src/` hat einen Ordner pro Schicht: `core`, `cli`, `net` (mit `net/ipt/`), `radio`, `mode`,
  `monitor`, `cmd`, dann `main.c`. Eine Datei bindet nur ihre eigene Schicht und die darunter ein,
  in dieser Reihenfolge. Jeder Ordner hat einen öffentlichen Header, `src/include/<folder>/<folder>.h`, dessen
  Prototypen unter einem Label gruppiert sind, das die definierende Datei benennt; was nur die eigenen Dateien des Ordners
  und die Tests nutzen, steht in `src/<folder>/internal.h`, eingebunden als `"internal.h"`
  (Tests nehmen es per Pfad). Alles andere ist `static`. Dateien umfassen 200 bis 800 Zeilen: Eine neue
  Funktion kommt in die Datei ihres Subsystems, und eine neue Datei ist für ein neues Subsystem oder eine
  Datei jenseits von 800. `net/ipt/` hat einen eigenen Header, weil auf musl die netfilter-Header des Kernels
  und die Netzwerk-Header von libc nicht in einer Datei zusammenkommen können. Aus
  Droidspaces und USB-Arsenal portierter Code folgt diesen Regeln; eingekaufter Code bleibt in `externals/`,
  unberührt.
- Logging ist `vap_log`, `vap_warn`, `vap_error` (`core/io`). Sie geben nach stdout aus, das
  die App mit den Tags `[WARN]` und `[ERROR]`, die sie liest, in ihr Log-Sheet streamt, und
  hängen eine mit Zeitstempel versehene Kopie an `logs/ap.log` an. Die App liest diese Datei nie. Ein bloßes
  `printf` ist im Log unsichtbar, und alles auf stderr geht verloren.
- Programme werden von `core/proc` ausgeführt (`run_quiet`, `run_capture`, `run_shown`, `run_logged`,
  `run_lines`), nie `popen` oder `system`: Android hat kein `/bin/sh`, und jedes Kind bekommt
  `/dev/null` für stdin und eine saubere Signalmaske. Pid-Dateien laufen über seine `pidfile_*`-
  Funktionen, die nur einem Prozess ein Signal senden, der das erwartete Programm ausführt.
- Die Einstellungen liegen in einer Tabelle, `vap_keys` in `cli/options.c`: Flag-Buchstabe, `ap.conf`-
  Schlüssel, Standardwert, Nutzungstext. Eine neue Einstellung ist ein Feld in `struct vap_opts` und eine Zeile dort,
  und ein Feld in `APConfig`; das Parsen und der Nutzungstext folgen. Jeder Wert geht durch
  `opt_set`, das Zeilenumbrüche ablehnt. `ap.conf` gehört der App: Nur `APManager.saveConfig`
  schreibt sie, und nur eine Konfiguration, die `APConfig.isValid()` besteht. Die Engine liest sie nur.
- Der Kernel wird über `net/` angesprochen (rtnetlink: `link` mit den Adressen und den
  Nachbarn, `route` mit den Policy-Regeln und netds Tabellen; die `ip_tables`-Engine in
  `net/ipt/`: `ipt`, `ipt_rules`) und `radio/` (`driver`: nl80211 und der Steuer-Socket von wpa).
  Kein Modul startet `ip`, `iw` oder `hostapd_cli`; `iptables` wird nur für den
  TTL-Fix gestartet (`ttl_*` in `net/ipt/ipt_rules`).
- `upstream_resolve` (`net/route`) ist der einzige Ort, an dem die Upstream-Tabelle aufgelöst wird.
  `radio/channel`, `ensure_5g_ir` eingeschlossen, ist der einzige Ort, an dem Band- und Kanalpolitik lebt:
  dem Kanal der Station folgen, der DFS-Fallback, der NO-IR-Länderfix. Vergleichen Sie eine
  Kanalnummer nirgendwo sonst. `ipv6_decide` (`mode/ipv6`) ist der einzige Ort, an dem das IPv6 der Clients
  gewählt wird; der Start und der Monitor fragen es beide.
- `Constants` enthält jeden Pfad und jeden Preference-Schlüssel auf der App-Seite, den der Engine
  (`Constants.ENGINE`) eingeschlossen. `VirtualAPInstaller` deployt die Binaries; es wird nicht
  anderswo dupliziert.
- Kein Text in Kotlin. Jeder String, den die App zeigt, steht in `res/values/strings.xml` oder
  `plurals.xml`, gelesen mit `stringResource`, oder `getString` außerhalb der Komposition, damit Weblate
  ihn übersetzen kann. Die Regeln stehen in [CONTRIBUTING.de.md](./CONTRIBUTING.de.md), Übersetzungen.

## Harte Einschränkungen

- Die App zielt auf `minSdk = 26`. Testen Sie auf Android 8, bevor Sie annehmen, dass eine API existiert.
- Die Engine ist C11 (`gnu11`) auf musl, single-threaded: `setns` verschiebt nur den aufrufenden
  Thread, und ein netlink-Socket bleibt in dem Namespace, in dem er geöffnet wurde. Sie baut ohne
  Warnungen unter dem gcc-Satz des Makefiles, den CI zu Fehlern macht.
- Binaries sind vollständig statische musl-Builds aus den `externals/`-Submodulen, beide Architekturen. Nie
  ein dynamisch gelinktes Binary, nie ein vorgefertigtes von anderswo. Ein neues Werkzeug ist ein neues Submodul
  plus ein Block in `scripts/build-in-container.sh`; `virtualap` ist das eine Binary, das aus
  dem eigenen `src/` dieses Repositories gebaut wird, mit musl-cross gcc und dem `Makefile` der Wurzel.
- Keine neue Abhängigkeit, wenn eine installierte sie abdeckt. Die App hat bereits libsu, Compose,
  lifecycle, zxing und appcompat.
- Samsungs Broadcom-FullMAC-Treiber ist die Untergrenze für Funk-Annahmen, MediaTeks
  gen4m die zweite; was jedem fehlt, steht in "Das Funkmodul" in How-It-Works.md. Ein Quirk-Fix,
  der nicht auf Laufzeiterkennung basiert, kommt nicht hinein.
