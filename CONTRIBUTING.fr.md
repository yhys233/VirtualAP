# Contribuer à VirtualAP

[English](CONTRIBUTING.md) | [简体中文](CONTRIBUTING.zh-CN.md) | [繁體中文](CONTRIBUTING.zh-TW.md) | [Русский](CONTRIBUTING.ru.md) | [Français](CONTRIBUTING.fr.md) | [Deutsch](CONTRIBUTING.de.md) | [日本語](CONTRIBUTING.ja.md) | [한국어](CONTRIBUTING.ko.md)

Une fonctionnalité qui n'existe pas vaut mieux qu'une fonctionnalité cassée. VirtualAP tourne en root sur
des téléphones rootés, sur des puces Wi-Fi dont les pilotes ne sont pas d'accord sur des choses élémentaires, donc un correctif qui
marche sur votre téléphone et casse sur celui d'un autre est une régression, pas une contribution. Et
dix lignes qui font le travail valent mieux que cent qui font le même travail : supprimez avant d'ajouter, réutilisez
avant d'écrire.

## Ce qui doit tenir

Les règles pour quiconque modifie ce dépôt, agents compris, sont dans
[AGENTS.fr.md](./AGENTS.fr.md) : le style, le format des commits, les points de passage obligés et les contraintes
strictes. En bref : Android seulement, le moteur est du C mono-thread sur musl qui parle au
noyau lui-même, un souci par module, l'application tourne sur Android 8, et les binaires
viennent uniquement de `scripts/build-static.sh` et, pour `virtualap`, du `Makefile` racine.

VirtualAP tourne sur des puces Qualcomm, Broadcom FullMAC (Samsung) et MediaTek. Un correctif pour la particularité
d'une puce n'est accepté que quand le backend s'y adapte à l'exécution, par détection, un
chemin conditionnel ou un repli, sans changer le comportement sur les autres ;
`nl80211_sta_freq` qui lit la fréquence du BSS associé parce que Samsung omet le canal,
et `ensure_5g_ir` qui ne se déclenche que quand le phy signale NO-IR, sont le modèle. Si le correctif ne peut pas être conditionné ainsi, il
appartient à un fork. Testez sur un vrai téléphone, il n'y a pas d'émulateur pour une puce Wi-Fi, et dites dans
votre PR sur quels appareils, pilotes, bandes et amonts vous avez testé ; une affirmation non testée compte
comme non testée.

## Compilation

```
./scripts/build-static.sh    # from the repository root: both arches, needs Docker
./build.sh [release]         # from Android/: the APK, needs JDK 17 or newer, ANDROID_HOME and musl-cross gcc
```

Ce que font les scripts, et ce que fait la CI, est dans la section Building de
[How-It-Works.fr.md](./Documentation/How-It-Works.fr.md). Une version est un tag : `make bump NEW=x.y.z`,
commitez-le avec `VirtualAP: bump vX.Y.Z`, puis poussez `main` et le tag `vX.Y.Z`.

## Style de code

Les quatre règles sont dans AGENTS.md, section Style. En plus d'elles :

- Backend : les coreutils sont `$ECHO`, `$GREP`, `$SED` et compagnie, jamais nus, et jamais entre guillemets.
  La sortie est `log`, `warn`, `error`. Les sous-commandes sont `cmd_*`, les aides Droidspaces `ds_*`,
  les sondes radio `phy_*`, la politique `pick_*` et `resolve_*`. Le démontage reflète le montage étape par étape
  et chaque étape tolère l'absence. Chaque démon journalise dans son propre fichier sous `logs/`.
- Application : l'état vit dans un ViewModel, pas dans un composable avec état et pas dans un singleton `util`.
  Les listes se rendent avec `LazyColumn` et des clés stables. Les couleurs viennent de
  `MaterialTheme.colorScheme`, la typographie de `MaterialTheme.typography`, les minutages de
  `AnimationUtils`, toutes les autres valeurs visuelles de [DESIGN.fr.md](./DESIGN.fr.md). Toute valeur qui
  atteint un shell root passe par `Backend.quote()` ou un validateur à liste d'autorisation.

Les commits sont signés et préfixés comme le dit AGENTS.md, section Commits.

## Traductions

Les traducteurs travaillent sur Weblate, qui commite dans les dossiers `values-<lang>/` ; une pull request
ne touche que l'anglais dans `res/values/`. La façon dont les composants sont configurés est dans la
section Translations de [How-It-Works.fr.md](./Documentation/How-It-Works.fr.md).

Ajouter une chaîne :

- Placez-la dans `strings.xml` sous le commentaire de son écran, ou dans `plurals.xml` si un compte
  change sa formulation. Kotlin la lit avec `stringResource`, ou `getString` hors
  composition.
- Les espaces réservés sont positionnels, `%1$s`, pour qu'une traduction puisse les réordonner. Dites ce que chacun
  est dans un commentaire juste au-dessus de la chaîne, que Weblate montre au traducteur ; faites de même
  pour un mot court qui pourrait signifier deux choses.
- Une phrase entière par chaîne. Ne collez jamais des morceaux traduits ensemble.
- Les chemins, commandes, clés de préférence et ce que le moteur imprime restent dans le code, en anglais. Une
  valeur du fichier qu'aucune langue ne change, un nom de produit ou une adresse d'exemple, est
  `translatable="false"`.

## Inventaire de réutilisation

Faites un grep de cette liste avant d'écrire quoi que ce soit de nouveau. Si quelque chose de proche existe, étendez-le
plutôt que d'ajouter un frère. Les chemins Android sont relatifs à
`Android/app/src/main/java/com/virtualap/app/` ; les modules du moteur sont des chemins sous `src/`, l'en-tête de chaque
dossier sous `src/include/<folder>/`.
Cette liste répond à « comment l'appeler » ; DESIGN.md répond à « à quoi cela doit ressembler » quand
rien ici ne convient. Les composants partagés ont été portés depuis Droidspaces ; quand un appelant pour l'un
de ses autres composants apparaît, copiez le fichier amont plutôt que d'en écrire un nouveau.

Formulaires et entrées :

- `DsDropdown(label, selected, options, displayName, onSelect, ...)` dans
  `ui/component/DsDropdown.kt` : tout champ de sélection. Ne construisez jamais à la main `ExposedDropdownMenuBox`.
- `DsTextFieldDefaults.colors()` dans `ui/component/DsTextFieldDefaults.kt` : chaque
  `OutlinedTextField`.
- `rememberClearFocus()`, qu'utilise `DsDialog`, et `ClearFocusOnClickOutside`, la racine de l'écran
  principal, dans `ui/util/FocusUtils.kt` : rangez le clavier sur un tap à l'extérieur.

Boîtes de dialogue et feuilles :

- `DsDialog(onDismiss, modifier, footer) { }` dans `ui/component/DsDialog.kt` : chaque boîte de dialogue. Ne fixez jamais
  une largeur, un remplissage ou un défilement.
- `DialogFooterRow(dismissLabel, confirmLabel, onDismiss, onConfirm, confirmEnabled)` dans
  `ui/component/DialogFooterRow.kt` : les deux boutons d'une boîte de dialogue avec un choix, dans le slot
  `footer`.
- `DialogCloseButton(onClick, enabled)` dans `ui/component/DialogCloseButton.kt` : le carré de fermeture de 36dp
  dans une ligne d'en-tête, pour le contenu qui se ferme par le haut.
- `LogActionRow(logs, isBlocking, onClear)` dans `ui/component/LogActionRow.kt` : la rangée Clear et
  Copy au-dessus d'une console de journal ; `onClear = null` donne Copy seul.
- `DeviceRow`, `DeviceSheet(title, onDismiss, content)`, `BlockedList`, `AllowedList`,
  `BlockedDevicesSheet`, `AllowedDevicesSheet` et `rememberDeviceAction(reload)` dans
  `ui/component/DeviceList.kt` : un appareil avec son action, la feuille où ils se trouvent, les deux listes,
  et l'exécution d'un changement sur elles avec l'échec dit à voix haute.
- `AddDeviceDialog` et `AllowOnlyDialog` dans `ui/component/AddDeviceDialog.kt` : le champ MAC
  qui n'accepte que des chiffres hexadécimaux, et qui est écarté quand seuls les appareils autorisés peuvent se connecter.
- `Caption(text)` dans `ui/component/Caption.kt` : une ligne de texte secondaire.
- La feuille du QR Wi-Fi, la feuille des appareils et la feuille des journaux sont privées de `ui/screen/MainScreen.kt`, la
  boîte de dialogue À propos et le sélecteur de langue de `ui/screen/SettingsScreen.kt`. Ne les importez pas et ne les copiez pas.

Barres, configuration, retours :

- `PrimaryActionBottomBar(label, icon, onClick, ...)` dans
  `ui/component/PrimaryActionBottomBar.kt` : la barre Start et Stop. La surcharge prenant un
  slot `content` est pour un bouton qui échange son contenu.
- `SetupPage`, `SetupFrame`, `SetupHero`, `SetupActionBar` dans `ui/component/SetupHero.kt` :
  chaque écran du parcours de configuration, le carrousel d'accueil, la vérification de root et l'installateur. Voir
  DESIGN.md, « Le parcours de configuration ».
- `PullToRefreshWrapper(onRefresh) { }` dans `ui/component/PullToRefreshWrapper.kt` : tout corps d'écran
  à tirer pour rafraîchir.
- `ToggleCard` dans `ui/component/ToggleCard.kt` : une ligne d'interrupteur dans sa propre carte, les
  options de l'écran principal. `SwitchItem` dans `ui/component/SwitchItem.kt` : la ligne d'interrupteur plate dans une
  surface groupée, l'écran Paramètres.
- `StatusPill(label, color)` dans `ui/component/StatusPill.kt` : toute petite pastille de statut.
  `SectionHeader(text)` dans `ui/component/SectionHeader.kt` : un titre au-dessus d'un groupe de cartes.
- `CardContentPadding`, `CardHeaderHeight` dans `ui/component/CardMetrics.kt` : toute carte avec un
  en-tête titre-et-pastille, pour que les séparateurs s'alignent.
- `LoadingIndicator(size, color)` avec `LoadingSize`, et `FullScreenLoading(message)`, dans
  `ui/util/LoadingIndicator.kt` : chaque indicateur de chargement et chaque état de chargement plein écran.
- `TerminalConsole(logs, isProcessing, maxHeight)` et `ShimmerAnimation(enabled) { }` dans
  `ui/component/TerminalConsole.kt` : la vue de journal défilante et son cadre.

Thème :

- `VirtualAPTheme(darkTheme, dynamicColor, amoledMode, themePalette)` dans `ui/theme/Theme.kt` :
  l'unique racine de thème, appliquée dans `MainActivity`.
- `rememberThemeState()` et `ThemeState` dans `ui/theme/ThemeStateHolder.kt` : les préférences
  de thème vivantes.
- `ThemePalette` dans `ui/theme/Color.kt` : ajouter une palette d'accent, ici et nulle part ailleurs.
  `Theme.kt` et `Color.kt` sont les seuls fichiers qui nomment une couleur littéralement.
- `MaterialTheme.typography.*` et `JetBrainsMono` dans `ui/theme/Type.kt` : tous les styles de texte,
  et la police mono pour le texte de journal et de code.
- `AnimationUtils` dans `util/AnimationUtils.kt` : les spécifications de tween. Jamais un `tween(300)` littéral.
- `AccentColorPicker` et `ColorPaletteSwatch` dans `ui/component/` : le sélecteur de palette.
- Il n'y a pas d'objet de jetons d'espacement ou de forme ; les nombres vivent dans DESIGN.md et sont écrits comme
  des dp littéraux.

Navigation :

- `Screen` et le `when` dessus dans `MainActivity.kt` : le seul endroit où un écran de premier niveau est
  choisi, le drapeau d'accueil d'abord, puis l'état de root et d'installation, puis le drapeau Paramètres. Un nouvel
  écran va là.

Shell, root et le backend :

- `Backend.quote(value)` et `Backend.startAp` dans `util/Backend.kt` : chaque valeur dynamique qui
  atteint une commande root, et le préfixe de commande du moteur (avec son repli vers le script qu'une
  version plus ancienne extrayait, jusqu'à ce que le parcours de configuration le remplace).
- `APManager.getStatus/start/stop/getInterfaces/getCaps/isInstalled` dans `util/APManager.kt` :
  piloter `virtualap`, et `getContainers`/`parseContainers`, qui demandent directement à Droidspaces ses
  conteneurs en cours. N'assemblez jamais une de ces commandes à la main.
- `Hotspot.start(cfg)/stop()/refresh()`, `phase`, `status`, `actionLogs` dans `util/Hotspot.kt` :
  démarrer ou arrêter le point d'accès depuis n'importe où et lire l'état de la session ; poste aussi la
  notification de fonctionnement et réveille la tuile.
- `APConfig` avec `isValid()`, `isValidIpv4`, `isValidDnsList`, `channelsForBand` dans
  `util/APConfig.kt` : la configuration de hotspot enregistrée, la validation unique au démarrage, et les vérifications
  d'adresse que les champs partagent avec elle.
- `BackendLogger` et `classifyLine(line)` dans `util/BackendLogger.kt` : le puits de journal que prennent les appels à l'AP,
  et l'étiquetage de niveau `[ERROR]`/`[WARN]`.
- `PreferencesManager` dans `util/PreferencesManager.kt` : les propres réglages de l'application.
  `APManager.loadConfig/saveConfig` : la configuration du hotspot, lue et écrite en entier dans `ap.conf`.
  `Constants` dans `util/Constants.kt` : chaque chemin et chaque clé de préférence.
- `RootChecker.checkRootAccess(reprompt)` et `RootStatus` dans `util/RootChecker.kt` ;
  `VirtualAPInstaller.deviceArch/bundledPayloadVersion/payloadUpdateAvailable/install` dans
  `util/VirtualAPInstaller.kt` ; `ModuleInstaller.install` dans `util/ModuleInstaller.kt`, le module
  de démarrage ; `QrCodeGenerator.wifiPayload/encode` dans
  `util/QrCodeGenerator.kt`, le seul appelant de zxing.
- `HotspotTileService`, `HotspotStopReceiver` et `HotspotBootReceiver` : le
  basculeur des réglages rapides, l'action Stop de la notification, et le mot du module de démarrage disant qu'il a démarré un hotspot.
  Aucun ne détient d'état propre.
- `LocaleHelper.availableLanguages/currentLanguage/changeLanguage` dans `util/LocaleHelper.kt` :
  la langue de l'application, le seul appelant de `AppCompatDelegate.setApplicationLocales`.
- `AppViewModel` possède le drapeau d'accueil, l'état de root et l'état d'installation ; `APViewModel` possède la configuration modifiable,
  les listes d'interfaces et de conteneurs et le journal suivi, et transmet le démarrage et l'arrêt à
  `Hotspot`. La scrutation de statut toutes les 3 s vit dans `MainScreen`, liée à son cycle de vie.
- La configuration libsu globale vit dans `VirtualAPApplication.kt`, et seulement là.

Moteur (`src/`) :

- `core/` : `proc` (chaque programme lancé, les fichiers pid, et le verrou qui permet à un démarrage ou un
  arrêt de tourner à la fois), `io` (le journal : `vap_log`, `vap_warn`, `vap_error`, `log_stamp` ;
  `read_file`, `write_file`, `write_atomic` ; les analyseurs `valid_ip4`, `prefix_parse`,
  `subnet_from_gw`, et `csv_each` pour les listes séparées par des virgules de `-d`).
- `cli/` : `options` (la table `vap_keys`, `opt_set`, `opts_parse`, `print_usage`), `conf`
  (`ap.conf` et `run.state`).
- `net/` : `nl` (le transport rtnetlink, `nl_dump`, et les aides d'espace de noms `netns_open`,
  `in_netns`), `link` (liens, adresses et voisins), `route` (routes, règles de politique, et
  les tables de netd : `upstream_resolve`, `upstream_v6_iface`), `ipt/`, gardé à part parce que sur
  musl les en-têtes netfilter du noyau et les en-têtes réseau de la libc ne peuvent pas se rencontrer dans un seul fichier :
  `ipt` (le moteur `ip_tables`), `ipt_rules` (FORWARD, MASQUERADE, la vérification de présence, la
  sonde NAT, `ipt_bin`, le correctif TTL).
- `radio/` : `driver` (nl80211 via genetlink : créer et supprimer ap0, le phy, la
  fréquence de la station, la largeur ; la socket de contrôle wpa ; `wifi_mac`, la MAC d'usine), `channel`
  (politique de bande, de canal et de largeur, et `ensure_5g_ir`).
- `mode/` : `ap` (ap0, les démons, la planification radio, le maintien de signal que les deux modes partagent, et
  `hostapd` et `dnsmasq` : leurs configurations et démarrages, le nombre de clients), `routed` (avec les
  règles de politique épinglées qu'utilisent les deux familles), `ipv6` (`ipv6_decide`, configuration, démontage, NAT66 et
  le /64 partagé avec sa découverte de clients), `managed` (le conteneur Droidspaces : son pid,
  le provisionnement OpenWrt, les masques RPS), `maclist` (les fichiers des appareils bloqués et autorisés,
  qui sont `deny_mac_file` et `accept_mac_file` de `hostapd`, et prévenir un `hostapd` en cours
  d'exécution).
- `monitor/` : `monitor` (la boucle du démon), `upstream` (`reprobe`).
- `cmd/` : les commandes que l'application appelle, distribuées depuis `main.c` : `start` (`start` et
  `stop`), `status` (les en lecture seule : `status`, `clients`, `interfaces`, `caps`) et
  `block` (`block`, `unblock`, `blocked`, `allow`, `disallow`, `allowed`, `allow-only`).

## Ouvrir une PR

Forkez, travaillez dans une branche, ouvrez la PR contre `main`, et restez réactif : une PR sans réponse est
close, et les retours sont traités dans la même PR, pas dans une nouvelle. Il n'y a pas de CLA ; en
soumettant, vous acceptez que votre contribution soit distribuée sous la licence du projet.

Chaque PR dit :

1. Quel problème du monde réel elle résout. « Je voulais ceci » n'en est pas un ; dites ce qui casse, échoue ou
   manque pour de vrais utilisateurs sur du vrai matériel.
2. Des captures d'écran ou une sortie de terminal montrant que cela fonctionne.
3. Les environnements testés : pour l'application, appareil, version d'Android et fournisseur de root ; pour le
   backend, appareil, SoC et puce Wi-Fi, pilote, bande et largeur, amont (données mobiles,
   Wi-Fi, VPN), et si le mode géré a été exercé.
4. Pour les changements d'interface, les règles de DESIGN.md que vous avez suivies, et celle dont vous avez dévié et
   pourquoi. Un rayon ou une couleur qui contredit DESIGN.md sans raison est renvoyé.

Ce qui est fusionné : les correctifs avec une reproduction et une résolution vérifiée, les fonctionnalités validées
sur plusieurs appareils, le durcissement de la sécurité, les améliorations de performance mesurées, les suppressions, et
les corrections de documentation. Ce qui est rejeté : un changement du moteur qui lance un outil pour ce que
netlink ou les propres fichiers du noyau peuvent faire, ou qui fait grossir un module au-delà de son unique souci ; un
binaire précompilé ou lié dynamiquement ;
une fonctionnalité qu'aucun utilisateur réel n'a signalée ou qui ne fonctionne que sur une configuration matérielle ; du code
que son auteur ne peut pas expliquer ; un changement d'application qui casse Android 8 ; une seconde copie de quelque chose
dans l'inventaire ; toute valeur dynamique atteignant un shell root sans guillemets ; tout ce qui régresse
ce qui fonctionnait, quelle que soit l'utilité du nouveau comportement.

Une fonctionnalité fusionnée est la vôtre désormais : quand une nouvelle version d'Android ou un pilote la casse, vous
êtes censé la corriger, et une fonctionnalité dont l'auteur est injoignable est retirée. Un contributeur
dont les PR sont sans cesse rejetées pour les mêmes raisons, aucun problème réel, non testé, ou du remplissage
pour la base de code, est bloqué ; il n'y a pas de compteur d'avertissements, la décision revient au mainteneur.

## Sécurité

Tout ce qui permet à un client du hotspot, ou à une valeur saisie dans l'application, d'atteindre le shell root
sans guillemets est une vulnérabilité. N'ouvrez pas d'issue publique ; signalez-le en privé à
droidcasts@protonmail.com ou [t.me/ravindu](https://t.me/ravindu) avec une reproduction, les
configurations affectées et l'appareil ou le pilote, et gardez la divulgation publique jusqu'à ce qu'un correctif soit
publié. Les correctifs de durcissement sont examinés en priorité.
