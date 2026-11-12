# Contributing to VirtualAP

[English](CONTRIBUTING.md) | [简体中文](CONTRIBUTING.zh-CN.md) | [繁體中文](CONTRIBUTING.zh-TW.md) | [Русский](CONTRIBUTING.ru.md) | [Français](CONTRIBUTING.fr.md) | [Deutsch](CONTRIBUTING.de.md) | [日本語](CONTRIBUTING.ja.md) | [한국어](CONTRIBUTING.ko.md)

Ein Feature, das nicht existiert, ist besser als ein kaputtes. VirtualAP läuft als Root auf
gerooteten Telefonen, auf Wi-Fi-Chips, deren Treiber sich über grundlegende Dinge uneinig sind, also ist ein Patch, der
auf Ihrem Telefon funktioniert und auf dem eines anderen bricht, eine Regression, kein Beitrag. Und
zehn Zeilen, die die Arbeit erledigen, schlagen hundert, die dieselbe Arbeit tun: Löschen vor dem Hinzufügen, Wiederverwenden
vor dem Schreiben.

## Was gelten muss

Die Regeln für jeden, der dieses Repository ändert, Agenten eingeschlossen, stehen in
[AGENTS.de.md](./AGENTS.de.md): der Stil, das Commit-Format, die Engstellen und die harten
Einschränkungen. Kurz gesagt: nur Android, die Engine ist ein single-threaded C auf musl, das mit
dem Kernel selbst spricht, eine Zuständigkeit pro Modul, die App läuft auf Android 8, und die Binaries
stammen nur aus `scripts/build-static.sh` und, für `virtualap`, dem `Makefile` der Wurzel.

VirtualAP läuft auf Qualcomm-, Broadcom-FullMAC- (Samsung) und MediaTek-Chips. Ein Fix für den
Quirk eines Chips wird nur akzeptiert, wenn das Backend sich zur Laufzeit daran anpasst, durch Erkennung, einen
bedingten Pfad oder einen Fallback, ohne das Verhalten auf den anderen zu ändern;
`nl80211_sta_freq`, das die Frequenz des assoziierten BSS liest, weil Samsung den Kanal auslässt,
und `ensure_5g_ir`, das nur feuert, wenn die Phy NO-IR meldet, sind das Muster. Wenn der Fix nicht so
gegatet werden kann, gehört er in einen Fork. Testen Sie auf einem echten Telefon, es gibt keinen Emulator für einen Wi-Fi-Chip, und sagen Sie in
Ihrem PR, auf welchen Geräten, Treibern, Bändern und Upstreams Sie getestet haben; eine ungetestete Behauptung zählt
als ungetestet.

## Bauen

```
./scripts/build-static.sh    # from the repository root: both arches, needs Docker
./build.sh [release]         # from Android/: the APK, needs JDK 17 or newer, ANDROID_HOME and musl-cross gcc
```

Was die Skripte tun, und was CI tut, steht im Abschnitt Bauen von
[How-It-Works.de.md](./Documentation/How-It-Works.de.md). Ein Release ist ein Tag: `make bump NEW=x.y.z`,
committen Sie es als `VirtualAP: bump vX.Y.Z`, dann pushen Sie `main` und das Tag `vX.Y.Z`.

## Code-Stil

Die vier Regeln stehen in AGENTS.md, Stil. Darüber hinaus:

- Backend: Die Coreutils sind `$ECHO`, `$GREP`, `$SED` und Verwandte, nie ohne Pfad, und nie in Anführungszeichen.
  Ausgabe ist `log`, `warn`, `error`. Unterbefehle sind `cmd_*`, Droidspaces-Helfer `ds_*`,
  Funk-Probes `phy_*`, Politik `pick_*` und `resolve_*`. Der Abbau spiegelt den Aufbau Schritt für Schritt
  und jeder Schritt toleriert Abwesenheit. Jeder Daemon loggt in seine eigene Datei unter `logs/`.
- App: Zustand lebt in einem ViewModel, nicht in einem zustandsbehafteten Composable und nicht in einem `util`-
  Singleton. Listen rendern mit `LazyColumn` und stabilen Keys. Farben kommen aus
  `MaterialTheme.colorScheme`, Typografie aus `MaterialTheme.typography`, Timings aus
  `AnimationUtils`, jeder andere visuelle Wert aus [DESIGN.de.md](./DESIGN.de.md). Jeder Wert, der
  eine Root-Shell erreicht, geht durch `Backend.quote()` oder einen Allow-List-Validator.

Commits werden signiert und mit Präfixen versehen, wie AGENTS.md, Commits, sagt.

## Übersetzungen

Übersetzer arbeiten auf Weblate, das in die `values-<lang>/`-Ordner committet; ein Pull Request
berührt nur das Englische in `res/values/`. Wie die Komponenten eingerichtet sind, steht im
Abschnitt Übersetzungen von [How-It-Works.de.md](./Documentation/How-It-Works.de.md).

Einen String hinzufügen:

- Legen Sie ihn in `strings.xml` unter den Kommentar für seinen Bildschirm, oder in `plurals.xml`, wenn eine Anzahl
  seine Formulierung ändert. Kotlin liest ihn mit `stringResource`, oder `getString` außerhalb der
  Komposition.
- Platzhalter sind positionell, `%1$s`, damit eine Übersetzung sie umordnen kann. Sagen Sie, was jeder
  ist, in einem Kommentar direkt über dem String, den Weblate dem Übersetzer zeigt; tun Sie dasselbe
  für ein kurzes Wort, das zwei Dinge bedeuten könnte.
- Eine ganze Phrase pro String. Kleben Sie nie übersetzte Teile zusammen.
- Pfade, Befehle, Preference-Schlüssel und was die Engine ausgibt, bleiben im Code, auf Englisch. Ein
  Wert in der Datei, den keine Sprache ändert, ein Produktname oder eine Beispieladresse, ist
  `translatable="false"`.

## Wiederverwendungs-Bestandsliste

Greppen Sie diese Liste, bevor Sie etwas Neues schreiben. Wenn etwas Ähnliches existiert, erweitern Sie es, statt
ein Geschwister hinzuzufügen. Android-Pfade sind relativ zu
`Android/app/src/main/java/com/virtualap/app/`; Engine-Module sind Pfade unter `src/`, der Header
jedes Ordners unter `src/include/<folder>/`.
Diese Liste beantwortet "was rufe ich auf"; DESIGN.md beantwortet "wie soll es aussehen", wenn
nichts hier passt. Die gemeinsamen Komponenten wurden aus Droidspaces portiert; wenn ein Aufrufer für eine
seiner anderen Komponenten auftaucht, kopieren Sie die Upstream-Datei, statt eine neue zu schreiben.

Formulare und Eingaben:

- `DsDropdown(label, selected, options, displayName, onSelect, ...)` in
  `ui/component/DsDropdown.kt`: jedes Auswahlfeld. Bauen Sie nie `ExposedDropdownMenuBox` von Hand.
- `DsTextFieldDefaults.colors()` in `ui/component/DsTextFieldDefaults.kt`: jedes
  `OutlinedTextField`.
- `rememberClearFocus()`, das `DsDialog` nutzt, und `ClearFocusOnClickOutside`, die Wurzel des Haupt-
  bildschirms, in `ui/util/FocusUtils.kt`: die Tastatur bei einem Tipp außerhalb schließen.

Dialoge und Sheets:

- `DsDialog(onDismiss, modifier, footer) { }` in `ui/component/DsDialog.kt`: jeder Dialog. Setzen Sie nie
  eine Breite, Innenabstand oder Scrollen.
- `DialogFooterRow(dismissLabel, confirmLabel, onDismiss, onConfirm, confirmEnabled)` in
  `ui/component/DialogFooterRow.kt`: die zwei Buttons in einem Dialog mit einer Auswahl, im `footer`-
  Slot.
- `DialogCloseButton(onClick, enabled)` in `ui/component/DialogCloseButton.kt`: das 36dp große Schließen-
  Quadrat in einer Kopfzeile, für Inhalt, der von oben schließt.
- `LogActionRow(logs, isBlocking, onClear)` in `ui/component/LogActionRow.kt`: die Zeile Leeren und
  Kopieren über einer Log-Konsole; `onClear = null` ergibt nur Kopieren.
- `DeviceRow`, `DeviceSheet(title, onDismiss, content)`, `BlockedList`, `AllowedList`,
  `BlockedDevicesSheet`, `AllowedDevicesSheet` und `rememberDeviceAction(reload)` in
  `ui/component/DeviceList.kt`: ein Gerät mit seiner Aktion, das Sheet, in dem sie sitzen, die beiden Listen,
  und das Ausführen einer Änderung daran mit laut gesagtem Fehler.
- `AddDeviceDialog` und `AllowOnlyDialog` in `ui/component/AddDeviceDialog.kt`: das MAC-Feld,
  das nur Hexziffern annimmt, und wer herausfällt, wenn nur erlaubte Geräte beitreten dürfen.
- `Caption(text)` in `ui/component/Caption.kt`: eine Zeile sekundären Textes.
- Das Wi-Fi-QR-Sheet, das Geräte-Sheet und das Log-Sheet sind privat in `ui/screen/MainScreen.kt`, der Über-
  Dialog und die Sprachauswahl in `ui/screen/SettingsScreen.kt`. Importieren oder kopieren Sie sie nicht.

Leisten, Einrichtung, Rückmeldung:

- `PrimaryActionBottomBar(label, icon, onClick, ...)` in
  `ui/component/PrimaryActionBottomBar.kt`: die Leiste Starten und Stoppen. Die Überladung mit einem
  `content`-Slot ist für einen Button, der seinen Inhalt austauscht.
- `SetupPage`, `SetupFrame`, `SetupHero`, `SetupActionBar` in `ui/component/SetupHero.kt`:
  jeder Bildschirm des Einrichtungsablaufs, der Willkommens-Pager, die Root-Prüfung und der Installer. Siehe
  DESIGN.md, "Der Einrichtungsablauf".
- `PullToRefreshWrapper(onRefresh) { }` in `ui/component/PullToRefreshWrapper.kt`: jeder Pull-to-
  Refresh-Körper eines Bildschirms.
- `ToggleCard` in `ui/component/ToggleCard.kt`: eine Switch-Zeile in ihrer eigenen Karte, die
  Optionen des Hauptbildschirms. `SwitchItem` in `ui/component/SwitchItem.kt`: die flache Switch-Zeile in einer
  gruppierten Fläche, der Bildschirm Einstellungen.
- `StatusPill(label, color)` in `ui/component/StatusPill.kt`: jeder kleine Status-Chip.
  `SectionHeader(text)` in `ui/component/SectionHeader.kt`: eine Überschrift über einer Gruppe von Karten.
- `CardContentPadding`, `CardHeaderHeight` in `ui/component/CardMetrics.kt`: jede Karte mit einem
  Titel-und-Pill-Kopf, damit die Trennlinien ausgerichtet sind.
- `LoadingIndicator(size, color)` mit `LoadingSize`, und `FullScreenLoading(message)`, in
  `ui/util/LoadingIndicator.kt`: jeder Spinner und jeder Ladezustand über den ganzen Bildschirm.
- `TerminalConsole(logs, isProcessing, maxHeight)` und `ShimmerAnimation(enabled) { }` in
  `ui/component/TerminalConsole.kt`: die scrollende Log-Ansicht und ihr Rahmen.

Theme:

- `VirtualAPTheme(darkTheme, dynamicColor, amoledMode, themePalette)` in `ui/theme/Theme.kt`:
  die eine Theme-Wurzel, angewendet in `MainActivity`.
- `rememberThemeState()` und `ThemeState` in `ui/theme/ThemeStateHolder.kt`: die live
  Theme-Präferenzen.
- `ThemePalette` in `ui/theme/Color.kt`: eine Akzentpalette hinzufügen, hier und nirgendwo sonst.
  `Theme.kt` und `Color.kt` sind die einzigen Dateien, die eine Farbe wörtlich benennen.
- `MaterialTheme.typography.*` und `JetBrainsMono` in `ui/theme/Type.kt`: alle Textstile,
  und die Mono-Schrift für Log- und Code-Text.
- `AnimationUtils` in `util/AnimationUtils.kt`: die Tween-Spezifikationen. Nie ein wörtliches `tween(300)`.
- `AccentColorPicker` und `ColorPaletteSwatch` in `ui/component/`: die Palettenauswahl.
- Es gibt kein Objekt für Abstands- oder Form-Token; die Zahlen leben in DESIGN.md und werden als
  wörtliche dp geschrieben.

Navigation:

- `Screen` und das `when` darüber in `MainActivity.kt`: der eine Ort, an dem ein Bildschirm der obersten
  Ebene gewählt wird, zuerst das Willkommens-Flag, dann Root- und Installationszustand, dann das Flag Einstellungen. Ein neuer
  Bildschirm kommt dorthin.

Shell, Root und das Backend:

- `Backend.quote(value)` und `Backend.startAp` in `util/Backend.kt`: jeder dynamische Wert, der
  einen Root-Befehl erreicht, und das Befehlspräfix der Engine (mit seinem Fallback auf das Skript, das
  eine ältere Version extrahierte, bis der Einrichtungsablauf es ersetzt).
- `APManager.getStatus/start/stop/getInterfaces/getCaps/isInstalled` in `util/APManager.kt`:
  das Ansteuern von `virtualap`, und `getContainers`/`parseContainers`, die Droidspaces direkt nach seinen
  laufenden Containern fragen. Setzen Sie einen dieser Befehle nie von Hand zusammen.
- `Hotspot.start(cfg)/stop()/refresh()`, `phase`, `status`, `actionLogs` in `util/Hotspot.kt`:
  das Starten oder Stoppen des AP von überall und das Lesen des Sitzungszustands; postet außerdem die
  laufende Benachrichtigung und weckt die Kachel.
- `APConfig` mit `isValid()`, `isValidIpv4`, `isValidDnsList`, `channelsForBand` in
  `util/APConfig.kt`: die gespeicherte Hotspot-Konfiguration, die eine Startvalidierung, und die Adress-
  prüfungen, die die Felder mit ihr teilen.
- `BackendLogger` und `classifyLine(line)` in `util/BackendLogger.kt`: die Log-Senke, die die AP-
  Aufrufe nehmen, und die Einstufung `[ERROR]`/`[WARN]`.
- `PreferencesManager` in `util/PreferencesManager.kt`: die eigenen Einstellungen der App.
  `APManager.loadConfig/saveConfig`: die Hotspot-Konfiguration, ganz gelesen und geschrieben in `ap.conf`.
  `Constants` in `util/Constants.kt`: jeder Pfad und jeder Preference-Schlüssel.
- `RootChecker.checkRootAccess(reprompt)` und `RootStatus` in `util/RootChecker.kt`;
  `VirtualAPInstaller.deviceArch/bundledPayloadVersion/payloadUpdateAvailable/install` in
  `util/VirtualAPInstaller.kt`; `ModuleInstaller.install` in `util/ModuleInstaller.kt`, das Boot-
  modul; `QrCodeGenerator.wifiPayload/encode` in
  `util/QrCodeGenerator.kt`, der einzige zxing-Aufrufer.
- `HotspotTileService`, `HotspotStopReceiver` und `HotspotBootReceiver`: der Schalter der
  Schnelleinstellungen, die Stopp-Aktion der Benachrichtigung, und das Wort des Bootmoduls, dass es einen Hotspot gestartet hat.
  Keines hält eigenen Zustand.
- `LocaleHelper.availableLanguages/currentLanguage/changeLanguage` in `util/LocaleHelper.kt`:
  die Sprache der App, der einzige Aufrufer von `AppCompatDelegate.setApplicationLocales`.
- `AppViewModel` besitzt das Willkommens-Flag, den Root-Status und den Installationszustand; `APViewModel` besitzt die editierbare Konfiguration,
  die Interface- und Container-Listen und das mitgelesene Log, und reicht Start und Stopp an
  `Hotspot` weiter. Der 3-s-Status-Poll lebt in `MainScreen`, an dessen Lifecycle gebunden.
- Die globale libsu-Konfiguration lebt in `VirtualAPApplication.kt`, und nur dort.

Engine (`src/`):

- `core/`: `proc` (jedes ausgeführte Programm, die Pid-Dateien, und die Sperre, die einen Start oder
  Stopp zur Zeit laufen lässt), `io` (das Log: `vap_log`, `vap_warn`, `vap_error`, `log_stamp`;
  `read_file`, `write_file`, `write_atomic`; die Parser `valid_ip4`, `prefix_parse`,
  `subnet_from_gw`, und `csv_each` für die Kommalisten von `-d`).
- `cli/`: `options` (die Tabelle `vap_keys`, `opt_set`, `opts_parse`, `print_usage`), `conf`
  (`ap.conf` und `run.state`).
- `net/`: `nl` (der rtnetlink-Transport, `nl_dump`, und die Namespace-Helfer `netns_open`,
  `in_netns`), `link` (Links, Adressen und Nachbarn), `route` (Routen, Policy-Regeln, und
  netds Tabellen: `upstream_resolve`, `upstream_v6_iface`), `ipt/`, getrennt gehalten, weil auf
  musl die netfilter-Header des Kernels und die Netzwerk-Header von libc nicht in einer Datei zusammenkommen können:
  `ipt` (die `ip_tables`-Engine), `ipt_rules` (FORWARD, MASQUERADE, die Anwesenheitsprüfung, die
  NAT-Probe, `ipt_bin`, der TTL-Fix).
- `radio/`: `driver` (nl80211 über genetlink: ap0 anlegen und löschen, die Phy, die
  Frequenz der Station, die Breite; der Steuer-Socket von wpa; `wifi_mac`, die Werks-MAC), `channel`
  (Band-, Kanal- und Breitenpolitik, und `ensure_5g_ir`).
- `mode/`: `ap` (ap0, die Daemons, die Funkplanung, das Signal-Hold, das beide Modi teilen, und
  `hostapd` und `dnsmasq`: ihre Konfigurationen und Starts, die Client-Anzahl), `routed` (mit den
  festgepinnten Policy-Regeln, die beide Familien nutzen), `ipv6` (`ipv6_decide`, Aufbau, Abbau, NAT66 und
  das gemeinsame /64 mit seiner Client-Erkennung), `managed` (der Droidspaces-Container: seine Pid,
  die OpenWrt-Provisionierung, die RPS-Masken), `maclist` (die Dateien der blockierten und erlaubten Geräte,
  die `hostapd`s `deny_mac_file` und `accept_mac_file` sind, und das Mitteilen an ein laufendes
  `hostapd`).
- `monitor/`: `monitor` (die Schleife des Daemons), `upstream` (`reprobe`).
- `cmd/`: die Befehle, die die App aufruft, verteilt von `main.c`: `start` (`start` und
  `stop`), `status` (die schreibgeschützten: `status`, `clients`, `interfaces`, `caps`) und
  `block` (`block`, `unblock`, `blocked`, `allow`, `disallow`, `allowed`, `allow-only`).

## Einen PR öffnen

Forken Sie, arbeiten Sie in einem Branch, öffnen Sie den PR gegen `main`, und bleiben Sie reaktionsfähig: Ein unbeantworteter PR wird
geschlossen, und Feedback wird im selben PR behandelt, nicht in einem neuen. Es gibt kein CLA; mit
dem Einreichen stimmen Sie zu, dass Ihr Beitrag unter der Lizenz des Projekts verbreitet werden darf.

Jeder PR nennt:

1. Welches reale Problem er löst. "Ich wollte das" ist keines; sagen Sie, was bricht, fehlschlägt oder
   für echte Nutzer auf echter Hardware fehlt.
2. Screenshots oder Terminal-Ausgabe, die zeigen, dass es funktioniert.
3. Die getesteten Umgebungen: für die App Gerät, Android-Version und Root-Anbieter; für das
   Backend Gerät, SoC und Wi-Fi-Chip, Treiber, Band und Breite, Upstream (mobile Daten,
   Wi-Fi, VPN), und ob der verwaltete Modus ausprobiert wurde.
4. Für UI-Änderungen die befolgten DESIGN.md-Regeln, und von welcher Sie abgewichen sind und
   warum. Ein Radius oder eine Farbe, die ohne Grund von DESIGN.md abweicht, wird zurückgeschickt.

Was gemergt wird: Fixes mit einer Reproduktion und einer verifizierten Lösung, über Geräte hinweg
validierte Features, Sicherheits-Härtung, gemessene Leistungsverbesserungen, Löschungen und
Dokumentationskorrekturen. Was abgelehnt wird: eine Engine-Änderung, die ein Werkzeug für etwas startet, das
netlink oder die eigenen Dateien des Kernels erledigen können, oder die ein Modul über seine eine Zuständigkeit hinaus wachsen lässt; ein
vorgefertigtes oder dynamisch gelinktes Binary;
ein Feature, das kein echter Nutzer gemeldet hat oder das nur auf einer Hardware-Konfiguration funktioniert; Code,
den sein Autor nicht erklären kann; eine App-Änderung, die Android 8 bricht; eine zweite Kopie von etwas
in der Bestandsliste; jeder dynamische Wert, der eine Root-Shell ohne Anführungszeichen erreicht; alles, was
regressiert, was funktionierte, so nützlich das neue Verhalten auch sei.

Ein gemergtes Feature gehört Ihnen weiterhin: Wenn eine neue Android-Version oder ein Treiber es bricht, werden
Sie erwartet, es zu reparieren, und ein Feature, dessen Autor unerreichbar ist, wird entfernt. Ein Beitragender,
dessen PRs immer wieder aus denselben Gründen abgelehnt werden, kein echtes Problem, ungetestet, oder das Aufblähen
der Codebasis, wird blockiert; es gibt keine Strike-Zählung, die Entscheidung liegt beim Maintainer.

## Sicherheit

Alles, was einen Hotspot-Client, oder einen in die App eingegebenen Wert, unquotiert die Root-Shell
erreichen lässt, ist eine Schwachstelle. Öffnen Sie kein öffentliches Issue; melden Sie es privat an
droidcasts@protonmail.com oder [t.me/ravindu](https://t.me/ravindu) mit einer Reproduktion, den
betroffenen Konfigurationen und dem Gerät oder Treiber, und halten Sie die öffentliche Offenlegung zurück, bis ein Fix
draußen ist. Härtungs-Patches werden zuerst geprüft.
