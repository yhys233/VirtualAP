# DESIGN.md

[English](DESIGN.md) | [简体中文](DESIGN.zh-CN.md) | [繁體中文](DESIGN.zh-TW.md) | [Русский](DESIGN.ru.md) | [Français](DESIGN.fr.md) | [Deutsch](DESIGN.de.md) | [日本語](DESIGN.ja.md) | [한국어](DESIGN.ko.md)

Die visuelle Sprache der VirtualAP-Android-App, die die Designsprache von Droidspaces
ist, übernommen zusammen mit den Komponenten, die sie umsetzen. Das Backend ist ein Shell-Skript ohne
UI, daher gilt nichts davon für `backend/`.

Komponenten zuerst, Werte zweitens: Die Wiederverwendungs-Bestandsliste in
[CONTRIBUTING.de.md](./CONTRIBUTING.de.md) listet, was existiert, und diese Datei ist für den Fall, dass
nichts dort abdeckt, was Sie brauchen. Hardcoden Sie nie eine Hex-Farbe, eine sp-Größe oder einen Radius, den Sie
nach Augenmaß gewählt haben.

## Wie die App aussieht, und warum

Nichts in dieser App hat Elevation. Tiefe wird ausgedrückt, indem die Flächenfarbe um eine
Stufe angehoben und ein 1dp-Rand gezeichnet wird; wenn Sie zu einem Schatten greifen, haben Sie das Design verlassen.
`Surface` ist das Primitiv: Buttons sind klickbare Surfaces, keine Material-`Button`s, und
Dialoge sind `Dialog { Surface { } }`, keine `AlertDialog`s. Material-Komponenten sind nicht verboten,
aber sie kommen mit ihren eigenen Erhebungs- und Radius-Standards und müssen begründet werden.

Farbe wird abgeleitet, nie wörtlich. `ui/theme/Theme.kt` mischt ein vollständiges Schema aus drei
Palettenfarben, oder übernimmt ein Material-You-Schema ganz. Dynamische Farbe ist standardmäßig an,
Fallback-Paletten werden für ältere Geräte und für Nutzer ausgeliefert, die sie abschalten, und AMOLED kollabiert
jede Fläche zu Schwarz, daher bricht ein hardcodiertes `Color(0xFF...)` alle vier Pfade auf einmal.

Ränder und Tönung tragen Zustand, Füllungen bleiben ruhig. Ein laufender Zugangspunkt ist keine grüne Karte;
er ist eine normale Karte mit einem getönten Button und einem Akzentrand.

## Farbe

Rollen kommen aus `MaterialTheme.colorScheme`. Das Schema selbst wird in `ui/theme/Theme.kt`
aus den Paletten in `ui/theme/Color.kt` gebaut, den einzigen zwei Dateien, die eine Farbe wörtlich benennen.

| Fläche | Rolle |
| --- | --- |
| Bildschirmhintergrund | `background` |
| Karte auf dem Hintergrund | `surfaceContainer` |
| Karte oder Pill in einer anderen Fläche verschachtelt, und interaktive Einstellungszeilen | `surfaceContainerHigh` |
| Dialog- und Bottom-Sheet-Hülle | `surfaceContainer` |

| Text | Rolle |
| --- | --- |
| Primär | `onSurface` |
| Sekundär, unterstützend, Metadaten | `onSurfaceVariant` mit Alpha `0.7f` |
| Tertiär, bewusst leise | `onSurfaceVariant` mit Alpha `0.6f` |
| Deaktivierter Inhalt | `onSurface` mit Alpha `0.38f` |
| Deaktivierte Container-Füllung | `onSurface` mit Alpha `0.12f` |

Ränder sind immer 1dp. Was sich ändert, ist das Alpha:

| Rand | Farbe |
| --- | --- |
| Karte | `outlineVariant` mit `0.35f` |
| Dialog-Hülle | `outlineVariant` mit `0.4f` |
| Aktions-Pill-Hülle und andere verschachtelte Flächen | `outlineVariant` mit `0.2f` |
| Trennlinie (`HorizontalDivider`) | `outlineVariant` mit `0.3f` |
| Textfeld, nicht fokussiert | `outlineVariant` mit `0.5f` |
| Textfeld, fokussiert | `primary` mit `0.8f` |
| Getönter Button, passend zu seinem Akzent | die Akzentrolle mit `0.2f` |

### Was die Farben bedeuten

Der Lebenszyklus des Zugangspunkts ist in der ganzen App konsistent, und neue Zustände sollten ihm beitreten, statt
ein paralleles Vokabular zu erfinden.

| Zustand | Akzent |
| --- | --- |
| Laufend, gesund, Start-Aktion | `primary` |
| Gestoppt, fehlgeschlagen, destruktiv, Stopp-Aktion | `error` |
| Neustart-Aktion | `secondary` |
| In Arbeit, Neustart, Herunterladen | `tertiary` |
| Leerlauf, unbekannt, nichts zu melden | `onSurfaceVariant` mit `0.6f` |
| Deaktiviert | Füllung `onSurface` mit `0.08f`, Akzent `outlineVariant` |

Ein getönter Button nimmt seine Füllung aus der passenden Container-Rolle mit Alpha `0.4f`, zum Beispiel
`primaryContainer.copy(alpha = 0.4f)` für Start und `errorContainer.copy(alpha = 0.4f)` für Stopp,
mit einem 1dp-Rand des Akzents mit `0.2f`.

Die Konsole färbt eine `[WARN]`-Zeile `tertiary` und eine `[ERROR]`-Zeile `error`, aus der Stufe,
die `classifyLine` vom Tag liest; das Backend gibt reinen Text aus, keine Escape-Codes.

## Typografie

Die Skala ist die eigene von Material 3, `Typography()` in `ui/theme/Type.kt`. Nutzen Sie die benannten Stile und
geben Sie das Gewicht an der Stelle an: Ein Titel ist `titleLarge` mit `FontWeight.Bold`, eine Kartenüberschrift
`titleMedium` SemiBold. Es gibt nirgendwo in der App eine `fontSize`-Überschreibung.

UI-Text ist `FontFamily.Default`. Alles, was die Maschine erzeugt hat, eine Log-Zeile, ein Konfigurationswert, ein
Pfad, ist JetBrains Mono, importiert aus `ui/theme/Type.kt`. Deklarieren Sie keine lokale
Schriftfamilie.

| Element | Stil |
| --- | --- |
| Bildschirmtitel in einer `TopAppBar` | `titleLarge`, Bold |
| Dialogtitel | `titleLarge`, Bold, linksbündig |
| Kartentitel | `titleMedium`, SemiBold |
| Abschnittsüberschrift in einem Bildschirm | `titleSmall`, Bold, `primary` |
| Button-Beschriftung | `labelLarge`, SemiBold |
| Fließtext in einem Dialog | `bodyMedium` |
| Hilfetext, Metadaten, Untertitel | `bodySmall` mit `onSurfaceVariant` mit `0.7f` |
| Status-Pill-Beschriftung | `labelSmall`, Black, `letterSpacing = 0.5.sp`, Großbuchstaben |
| Maschinenausgabe | `bodySmall` mit der Familie JetBrains Mono |

## Abstände

Die Skala ist **4, 8, 12, 16, 20, 24, 32**. Nichts anderes. Wenn sich ein Abstand nach 14 anfühlt, will er
12 oder 16.

| Abstand | Wert |
| --- | --- |
| Bildschirmrand, Listen und Dashboards | 16 horizontal |
| Bildschirmrand, Assistenten und Einrichtungsabläufe | 24 horizontal |
| Zwischen Karten in einer Liste | 16 |
| Zwischen Blöcken in einer Karte | 12 |
| Innenabstand der Karte | 16 |
| Innenabstand des Dialogs | 24 |
| Icon zu seiner Beschriftung, Chip zu Chip | 8 |
| Enge Metadaten-Zeilen | 4 |

Es gibt keine Tab-Leiste. Der Hauptbildschirm sitzt über `PrimaryActionBottomBar` in einem `Scaffold`,
so kommt die gemessene Höhe der Leiste als Innenabstand des Scaffolds an und die Liste weicht ihr aus,
ohne dass jemand eine Freiraumzahl tippt. Halten Sie es so: Eine Leiste, die nicht die `bottomBar` des Scaffolds ist,
braucht ein Literal, und das Literal driftet.

### Kartenköpfe richten sich aus

Die Karte des aktiven Netzwerks hat keinen Titel: Ihre erste Zeile ist der Netzwerkname links und die
Status-Pill rechts, über einer Trennlinie, bei einer Mindesthöhe von `CardHeaderHeight` und
innerhalb von `CardContentPadding`, beide aus `ui/component/CardMetrics.kt`. Die Höhe ist 48dp,
weil die Zeile den Wi-Fi-QR-Icon-Button enthält, und 48dp ist die Touch-Untergrenze.

Eine Karte, die einen Titel-und-Pill-Kopf bekommt, nimmt dieselben zwei Werte, damit ihre Trennlinie und
Pill dieselbe Linie halten wie diese. Wenn Sie sie ändern, verschieben Sie jeden Kartenkopf in
der App, und das ist die beabsichtigte Richtung.

## Form

| Element | Radius |
| --- | --- |
| Karte | 20 |
| Dialog-Hülle | 24 |
| Bottom Sheet, nur obere Ecken | 24 |
| Primärbutton über die volle Breite | 20 |
| Dropdown-Menü | 20 |
| Aktionsbutton | 16 |
| Textfeld | 16 |
| Aktions-Pill-Hülle | 12 |
| Status-Pill, kleines Badge | 8 |

## Die Aktions-Pill

Eine Reihe verwandter Aktionen sitzt in einer getönten Hülle:

```kotlin
Surface(
    shape = RoundedCornerShape(12.dp),                    // wrapper
    color = MaterialTheme.colorScheme.surfaceContainerHigh,
    border = BorderStroke(1.dp, MaterialTheme.colorScheme.outlineVariant.copy(alpha = 0.2f))
) {
    Row(Modifier.padding(4.dp), horizontalArrangement = Arrangement.spacedBy(4.dp)) {
        Surface(
            modifier = Modifier.weight(1f).height(48.dp),
            shape = RoundedCornerShape(16.dp),            // button, larger than the wrapper
            color = accentContainer.copy(alpha = 0.4f),
            border = BorderStroke(1.dp, accent.copy(alpha = 0.2f))
        ) { /* icon + labelLarge SemiBold */ }
    }
}
```

Die Buttons sind runder als die Box, in der sie sitzen, und das ist das Design: Der 4dp-Einsatz plus
der größere Innenradius lässt die Buttons so wirken, als stünden sie vor der Hülle, statt bündig
in ihr zu verschachteln. Die beiden Radien anzugleichen flacht den Effekt ab.

Der Schalter Interface oder Container auf dem Hauptbildschirm (`SegmentedSelector` in
`ui/screen/MainScreen.kt`) ist diese Geometrie als Selektor genutzt: Das gewählte Segment nimmt
die getönte Füllung und den Akzentrand, das andere bleibt transparent. Die Zeile Leeren und Kopieren über
der Log-Konsole (`LogActionRow`) ist das nicht: Sie ist eine bloße Zeile aus zwei 38dp-Buttons ohne
Hülle.

## Dialoge

Jeder Dialog ist `DsDialog`, das sowohl das Layout als auch den Rahmen besitzt: eine `Surface` über die volle Breite
innerhalb der 24dp-Ränder, begrenzt auf die Bildschirmhöhe minus diese, und eine 24dp gepolsterte Spalte,
deren Körper scrollt. Die Begrenzung kommt aus `LocalConfiguration`, nicht aus dem Fenster, damit ein Dialog, der
über eine Drehung offen ist, gegen die lebende Konfiguration misst. Aufrufer setzen keine Breite, ihren
eigenen Innenabstand oder ihr eigenes Scrollen, und übergeben nie `wrapContentHeight()`; was pro Dialog gilt, reist
auf dem `modifier`, `imePadding()` für ein Textfeld oder eine `heightIn(max = ...)`-Obergrenze.

Der Titel ist `titleLarge` Fett, linksbündig, und ist das Erste im Inhalt.

## Dialog-Aktionen

Über, eine Infoseite, schließt aus seinem Kopf mit `DialogCloseButton`. Ein Dialog mit einer Auswahl,
die Sprachauswahl, endet mit `DialogFooterRow`: zwei gleichgewichtige 48dp-Buttons mit Radius 16, ein
leises tonales Verwerfen und ein gefülltes Bestätigen, einzeilige Beschriftungen, die ellipsieren. Er kommt in den
`footer`-Slot von `DsDialog`, vor dem scrollenden Körper gemessen, damit Querformat ihn nicht quetschen kann. Der erste
destruktive Dialog kopiert Droidspaces' `error`-Bestätigung zurück.

## Icons und Touch-Ziele

| Verwendung | Größe |
| --- | --- |
| Führendes Icon auf einer Karte oder Listenzeile | 20 |
| Toolbar-Aktion, Icon in einem Button | 18 |
| Inline mit `bodySmall`-Text | 16 |
| Dialogkopf | 24 |
| Einrichtungs-Hero-Glyph | 96, im 240-Hero |
| Status-Punkt | 6 |

Alles Antippbare ist mindestens **48dp**. Das ist eine Barrierefreiheits-Untergrenze, keine Stilpräferenz,
und es ist die eine Regel in dieser Datei, die ein Design-Argument nicht überschreiben darf.

## Ausrichtung

Zeilen sind `Alignment.CenterVertically`.

Kartentitel und Dialogtitel sind linksbündig, immer. Eine Kartenzeile, die einen Titel mit einem
Steuerelement paart, nutzt `Arrangement.SpaceBetween`.

Zentrierter Text ist für Vollbildzustände reserviert: leere Listen, Laden, Einrichtungs-Heroes. Wenn Inhalt
in einer Karte zentriert ist, ist er wahrscheinlich falsch.

## Bewegung und Laden

Spinner kommen aus `LoadingIndicator` mit einer `LoadingSize`, nie ein rohes `.size(n.dp)` an einem
Fortschrittsindikator. Laden über den ganzen Bildschirm ist `FullScreenLoading`; `LoadingSize.Hero` (144) ist
der eine Hero-Loader des Einrichtungsablaufs. Animationstimings kommen aus `AnimationUtils`; Hero-Zustandsänderungen
nutzen `MaterialTheme.motionScheme`, das das Theme auf `MotionScheme.expressive()` setzt.

### Der Einrichtungsablauf

Der Willkommens-Pager, die Root-Prüfung und der Backend-Installer teilen eine Geometrie, die in
`ui/component/SetupHero.kt` lebt. `SetupPage` ist das Skelett unter dem Scaffold: eine 40dp-Zeile
darüber (der Seitenzähler auf dem Willkommens-Pager, anderswo leer), der Frame-Slot, der seinen Inhalt zentriert
oder scrollt, wenn das Fenster zu kurz ist, und eine 44dp-Zeile darunter (die Seitenpunkte).
Jeder Bildschirm behält beide Zeilen, damit Frame und Button sich zwischen Bildschirmen nie bewegen.
`SetupFrame` ist der Frame: eine 240dp-Hero-Box, dann der Titel und der Text in einem festen Text-
block. `SetupHero` ist der Hero eines Bildschirms, der Arbeit leistet, ein 240dp-
`MaterialShapes.Cookie12Sided` in `primaryContainer` oder `errorContainer`, der den Loader oder
ein 96dp-Glyph hält, und nur seine Farbe und sein Inhalt ändern sich zwischen Zuständen. Der Hero des Willkommens-Pagers
ist stattdessen seine eigene `Shape`, ein weicher Burst, der sich zu einem Cookie, einem Kleeblatt und einer Sonne setzt, während die
Seiten darüber gleiten. `SetupActionBar` ist der eine Button. Bauen Sie einen neuen Einrichtungsbildschirm aus diesen
und der Hero, der Titel und der Button landen auf denselben Zeilen wie bei den anderen.

## Wenn die Regel nicht passt

Manchmal wird sie es nicht, und die Antwort ist nicht, still eine andere Zahl zu verwenden.

Sagen Sie es im PR, und hinterlassen Sie an der Stelle einen Kommentar, der erklärt, was die Abweichung bringt, so wie
die entschiedenen Ausnahmen unten. Eine undokumentierte Abweichung ist Drift und der nächste Beitragende
wird sie "reparieren". Eine dokumentierte ist eine Entscheidung, und sie überlebt.

## Entschiedene Ausnahmen

Angesehen, bewusst unangetastet gelassen. Öffnen Sie diese nicht erneut ohne einen Grund, den der ursprüngliche
übersieht.

- **Die QR-Platte ist `Color.White`** im Wi-Fi-QR-Sheet, und `QrCodeGenerator.encode` zeichnet
  schwarze Module auf Weiß. Ein Scanner braucht den Kontrast, den die Spezifikation annimmt; eine thematisierte Platte unter einer
  dunklen Palette wirkt wie ein kaputter Code.
- **Die Statuskarte antwortet auf einen Tipp** (`ActiveNetworkCard` in `ui/screen/MainScreen.kt`), obwohl
  sie nichts öffnet. Sie ist die Karte, die ein Nutzer anstupst, um zu prüfen, ob die App lebt, also wellt sie
  und lässt den Feldfokus fallen, statt tot unter einem Finger zu liegen. Der Preis ist, dass TalkBack sie als einen
  handlungsfähigen Knoten liest.
- **Das Wi-Fi-QR-Sheet, das Geräte-Sheet und das Log-Sheet** in
  `ui/screen/MainScreen.kt`, und die Gerätelisten-Sheets in `ui/component/DeviceList.kt`,
  bleiben Material `ModalBottomSheet`s statt
  `Dialog { Surface { } }`. Alle sind per Ziehen verwerfbarer Inhalt, der
  an die Unterkante gehört, und das Log-Sheet blockiert die Gestenverwerfung, während ein Befehl
  läuft, damit das Sheet nicht einen berührungsfressenden Scrim zurücklässt. Ihre Hülle ist
  `surfaceContainer` mit 24dp oberen Ecken, passend zur Formtabelle.
- **Der Button des Einrichtungsablaufs** (`SetupActionBar` in `ui/component/SetupHero.kt`) ist eine volle Pill
  ohne Leistenfläche und ohne Trennlinie, nicht der 20dp-Leistenbutton. Er sitzt allein auf dem Hintergrund
  unter einem Hero statt über scrollendem Inhalt. Seine deaktivierten Farben bleiben gleich zu seinen aktivierten
  Farben: Er ist nur deaktiviert, während eine Prüfung oder Installation läuft, und ihn für
  diesen Moment auszugrauen wirkt wie ein Flackern.
- **Blasse Füll-Alphas von 0.03 und 0.06** in `LogActionRow` bleiben ungeteilt. Sie sind unsichtbar
  in Isolation, und sie zusammenzuschnappen würde bedeuten, einen Token für etwas zu erfinden, das niemand
  sehen kann.
