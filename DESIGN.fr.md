# DESIGN.md

[English](DESIGN.md) | [简体中文](DESIGN.zh-CN.md) | [繁體中文](DESIGN.zh-TW.md) | [Русский](DESIGN.ru.md) | [Français](DESIGN.fr.md) | [Deutsch](DESIGN.de.md) | [日本語](DESIGN.ja.md) | [한국어](DESIGN.ko.md)

Le langage visuel de l'application Android VirtualAP, qui est le langage visuel de Droidspaces
transposé avec les composants qui l'implémentent. Le backend est un script shell sans
interface, donc rien de tout cela ne s'applique à `backend/`.

Les composants d'abord, les valeurs ensuite : l'inventaire de réutilisation dans
[CONTRIBUTING.fr.md](./CONTRIBUTING.fr.md) liste ce qui existe, et ce fichier sert au cas où
rien là-bas ne couvre ce dont vous avez besoin. Ne codez jamais en dur une couleur hexadécimale, une taille sp ou un rayon
choisi à l'œil.

## À quoi ressemble l'application, et pourquoi

Rien dans cette application n'est surélevé. La profondeur s'exprime en montant la couleur de surface d'un
niveau et en dessinant une bordure de 1dp ; si vous songez à une ombre, vous avez quitté le design.
`Surface` est la primitive : les boutons sont des `Surface` cliquables, pas des `Button` Material, et
les boîtes de dialogue sont `Dialog { Surface { } }`, pas `AlertDialog`. Les composants Material ne sont pas bannis,
mais ils arrivent avec leurs propres défauts d'élévation et de rayon et doivent être argumentés.

La couleur est dérivée, jamais littérale. `ui/theme/Theme.kt` compose un schéma complet à partir de trois
couleurs de palette, ou reprend un schéma Material You entier. La couleur dynamique est active par défaut,
des palettes de repli sont livrées pour les appareils anciens et pour les utilisateurs qui la désactivent, et l'AMOLED réduit
chaque surface au noir, donc un `Color(0xFF...)` codé en dur casse les quatre chemins d'un coup.

Les bordures et la teinte portent l'état, les remplissages restent discrets. Un point d'accès en marche n'est pas une carte verte ;
c'est une carte normale avec un bouton teinté et une bordure d'accent.

## Couleur

Les rôles viennent de `MaterialTheme.colorScheme`. Le schéma lui-même est construit dans `ui/theme/Theme.kt`
à partir des palettes de `ui/theme/Color.kt`, les deux seuls fichiers qui nomment une couleur littéralement.

| Surface | Rôle |
| --- | --- |
| Fond d'écran | `background` |
| Carte posée sur le fond | `surfaceContainer` |
| Carte ou pastille imbriquée dans une autre surface, et lignes de réglages interactives | `surfaceContainerHigh` |
| Coque de boîte de dialogue et de feuille inférieure | `surfaceContainer` |

| Texte | Rôle |
| --- | --- |
| Principal | `onSurface` |
| Secondaire, complémentaire, métadonnées | `onSurfaceVariant` avec alpha `0.7f` |
| Tertiaire, volontairement discret | `onSurfaceVariant` avec alpha `0.6f` |
| Contenu désactivé | `onSurface` avec alpha `0.38f` |
| Remplissage de conteneur désactivé | `onSurface` avec alpha `0.12f` |

Les bordures font toujours 1dp. Ce qui change, c'est l'alpha :

| Bordure | Couleur |
| --- | --- |
| Carte | `outlineVariant` à `0.35f` |
| Coque de boîte de dialogue | `outlineVariant` à `0.4f` |
| Enveloppe de pastille d'action et autres surfaces imbriquées | `outlineVariant` à `0.2f` |
| Séparateur (`HorizontalDivider`) | `outlineVariant` à `0.3f` |
| Champ de texte, non focalisé | `outlineVariant` à `0.5f` |
| Champ de texte, focalisé | `primary` à `0.8f` |
| Bouton teinté, assorti à son accent | le rôle d'accent à `0.2f` |

### Ce que les couleurs signifient

Le cycle de vie du point d'accès est cohérent dans toute l'application et les nouveaux états doivent le rejoindre plutôt
que d'inventer un vocabulaire parallèle.

| État | Accent |
| --- | --- |
| En marche, sain, action de démarrage | `primary` |
| Arrêté, en échec, destructif, action d'arrêt | `error` |
| Action de redémarrage | `secondary` |
| En cours, redémarrage, téléchargement | `tertiary` |
| Inactif, inconnu, rien à signaler | `onSurfaceVariant` à `0.6f` |
| Désactivé | remplissage `onSurface` à `0.08f`, accent `outlineVariant` |

Un bouton teinté prend son remplissage dans le rôle de conteneur correspondant à l'alpha `0.4f`, par exemple
`primaryContainer.copy(alpha = 0.4f)` pour le démarrage et `errorContainer.copy(alpha = 0.4f)` pour l'arrêt,
avec une bordure de 1dp de l'accent à `0.2f`.

La console colore une ligne `[WARN]` en `tertiary` et une ligne `[ERROR]` en `error`, selon le niveau
que `classifyLine` lit sur la balise ; le backend émet du texte brut, sans codes d'échappement.

## Typographie

L'échelle est celle de Material 3, `Typography()` dans `ui/theme/Type.kt`. Utilisez les styles nommés et
passez la graisse sur place : un titre est `titleLarge` avec `FontWeight.Bold`, un titre de carte
`titleMedium` SemiBold. Il n'y a aucune surcharge de `fontSize` dans l'application.

Le texte d'interface est `FontFamily.Default`. Tout ce que la machine a produit, une ligne de journal, une valeur de configuration, un
chemin, est en JetBrains Mono, importé depuis `ui/theme/Type.kt`. Ne déclarez pas de famille de police
locale.

| Élément | Style |
| --- | --- |
| Titre d'écran dans une `TopAppBar` | `titleLarge`, Bold |
| Titre de boîte de dialogue | `titleLarge`, Bold, aligné à gauche |
| Titre de carte | `titleMedium`, SemiBold |
| En-tête de section dans un écran | `titleSmall`, Bold, `primary` |
| Libellé de bouton | `labelLarge`, SemiBold |
| Texte courant dans une boîte de dialogue | `bodyMedium` |
| Texte d'aide, métadonnées, sous-titres | `bodySmall` avec `onSurfaceVariant` à `0.7f` |
| Libellé de pastille de statut | `labelSmall`, Black, `letterSpacing = 0.5.sp`, majuscules |
| Sortie machine | `bodySmall` avec la famille JetBrains Mono |

## Espacement

L'échelle est **4, 8, 12, 16, 20, 24, 32**. Rien d'autre. Si un écart semble vouloir 14, il
veut 12 ou 16.

| Écart | Valeur |
| --- | --- |
| Bord d'écran, listes et tableaux de bord | 16 horizontal |
| Bord d'écran, parcours assistant et configuration | 24 horizontal |
| Entre les cartes d'une liste | 16 |
| Entre les blocs d'une carte | 12 |
| Remplissage interne de carte | 16 |
| Remplissage interne de boîte de dialogue | 24 |
| Icône à son libellé, pastille à pastille | 8 |
| Lignes de métadonnées serrées | 4 |

Il n'y a pas de barre d'onglets. L'écran principal se pose au-dessus de `PrimaryActionBottomBar` dans un `Scaffold`,
donc la hauteur mesurée de la barre arrive comme remplissage interne du Scaffold et la liste la dégage
sans que personne ne saisisse un nombre de dégagement. Gardez-le ainsi : une barre qui n'est pas le
`bottomBar` du Scaffold a besoin d'une valeur littérale, et la valeur littérale dérive.

### Les en-têtes de cartes s'alignent

La carte du réseau actif n'a pas de titre : sa première ligne est le nom du réseau à gauche et la
pastille de statut à droite, au-dessus d'un séparateur, à une hauteur minimale de `CardHeaderHeight` et
dans `CardContentPadding`, tous deux issus de `ui/component/CardMetrics.kt`. La hauteur est de 48dp
parce que la ligne contient le bouton icône du QR Wi-Fi, et 48dp est le plancher tactile.

Une carte qui reçoit bien un en-tête titre-et-pastille prend les mêmes deux valeurs, donc son séparateur et sa
pastille tiennent la même ligne que celle-ci. Si vous les changez, vous déplacez chaque en-tête de carte de
l'application, ce qui est le sens voulu.

## Forme

| Élément | Rayon |
| --- | --- |
| Carte | 20 |
| Coque de boîte de dialogue | 24 |
| Feuille inférieure, coins supérieurs seulement | 24 |
| Bouton principal pleine largeur | 20 |
| Menu déroulant | 20 |
| Bouton d'action | 16 |
| Champ de texte | 16 |
| Enveloppe de pastille d'action | 12 |
| Pastille de statut, petit badge | 8 |

## La pastille d'action

Une rangée d'actions liées se pose dans une enveloppe teintée :

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

Les boutons sont plus arrondis que la boîte qui les contient, et c'est là le design : l'encart de 4dp plus
le rayon intérieur plus grand font lire les boutons comme posés en relief sur l'enveloppe plutôt que
nichés à ras dedans. Aligner les deux rayons aplatit l'effet.

Le sélecteur Interface ou Conteneur de l'écran principal (`SegmentedSelector` dans
`ui/screen/MainScreen.kt`) est cette géométrie utilisée comme sélecteur : le segment sélectionné prend
le remplissage teinté et la bordure d'accent, l'autre reste transparent. La rangée Clear et Copy au-dessus
de la console de journal (`LogActionRow`) n'est pas cela : c'est une simple rangée de deux boutons de 38dp sans
enveloppe.

## Boîtes de dialogue

Chaque boîte de dialogue est `DsDialog`, qui possède la mise en page autant que le cadre : une `Surface` pleine largeur
dans les gouttières de 24dp, bornée à la hauteur de l'écran moins celles-ci, et une colonne à remplissage de 24dp
dont le corps défile. La borne vient de `LocalConfiguration`, pas de la fenêtre, donc une boîte de dialogue ouverte
pendant une rotation se mesure contre la configuration vivante. Les appelants ne fixent pas de largeur, ni
leur propre remplissage, ni leur propre défilement, et ne passent jamais `wrapContentHeight()` ; ce qui est propre à chaque boîte de dialogue
passe par le `modifier`, `imePadding()` pour un champ de texte ou un plafond `heightIn(max = ...)`.

Le titre est en `titleLarge` Bold, aligné à gauche, et c'est la première chose du contenu.

## Actions de boîte de dialogue

À propos, une page d'information, se ferme depuis son en-tête avec `DialogCloseButton`. Une boîte de dialogue avec un choix,
le sélecteur de langue, se termine par `DialogFooterRow` : deux boutons de 48dp de poids égal au rayon 16, un
renvoi tonal discret et une confirmation pleine, des libellés sur une ligne qui finissent par des points de suspension. Il va dans le
slot `footer` de `DsDialog`, mesuré avant le corps défilant pour que le paysage ne puisse pas le comprimer. La première
boîte de dialogue destructive reprend la confirmation `error` de Droidspaces.

## Icônes et cibles tactiles

| Usage | Taille |
| --- | --- |
| Icône de tête sur une carte ou une ligne de liste | 20 |
| Action de barre d'outils, icône dans un bouton | 18 |
| En ligne avec du texte `bodySmall` | 16 |
| En-tête de boîte de dialogue | 24 |
| Glyphe hero du parcours de configuration | 96, dans le hero de 240 |
| Point de statut | 6 |

Tout ce qui est tapable fait au moins **48dp**. C'est un plancher d'accessibilité, pas une préférence de style,
et c'est la seule règle de ce fichier qu'un argument de design n'a pas le droit de contourner.

## Alignement

Les lignes sont en `Alignment.CenterVertically`.

Les titres de cartes et les titres de boîtes de dialogue sont alignés à gauche, toujours. Une ligne de carte qui associe un titre à un
contrôle utilise `Arrangement.SpaceBetween`.

Le texte centré est réservé aux états plein écran : listes vides, chargement, hero de configuration. Si le contenu
est centré dans une carte, c'est probablement une erreur.

## Animation et chargement

Les indicateurs de chargement viennent de `LoadingIndicator` avec un `LoadingSize`, jamais d'un `.size(n.dp)` brut sur un
indicateur de progression. Le chargement plein écran est `FullScreenLoading` ; `LoadingSize.Hero` (144) est
le loader hero du parcours de configuration. Les durées d'animation viennent de `AnimationUtils` ; les changements d'état du hero
utilisent `MaterialTheme.motionScheme`, que le thème règle sur `MotionScheme.expressive()`.

### Le parcours de configuration

Le carrousel d'accueil, la vérification de root et l'installateur du backend partagent une géométrie, qui vit
dans `ui/component/SetupHero.kt`. `SetupPage` est le squelette sous le Scaffold : une ligne de 40dp
au-dessus (le compteur de pages sur le carrousel d'accueil, vide ailleurs), le slot du cadre, qui centre
son contenu ou le fait défiler quand la fenêtre est trop courte, et une ligne de 44dp en dessous (les points de page).
Chaque écran garde les deux lignes pour que le cadre et le bouton ne bougent jamais d'un écran à l'autre.
`SetupFrame` est le cadre : une boîte hero de 240dp, puis le titre et le texte dans un bloc de texte
fixe. `SetupHero` est le hero d'un écran qui fait un travail, un
`MaterialShapes.Cookie12Sided` de 240dp dans `primaryContainer` ou `errorContainer` contenant le loader ou
un glyphe de 96dp, et seules sa couleur et son contenu changent d'un état à l'autre. Le hero du carrousel d'accueil
est sa propre `Shape` à la place, une explosion douce qui se pose en cookie, en trèfle et en soleil quand les
pages glissent dessus. `SetupActionBar` est le bouton unique. Construisez un nouvel écran de configuration à partir de ceux-ci
et le hero, le titre et le bouton atterrissent sur les mêmes lignes que les autres.

## Quand la règle ne s'applique pas

Parfois elle ne s'appliquera pas, et la réponse n'est pas d'utiliser discrètement un autre nombre.

Dites-le dans la PR, et laissez un commentaire sur place expliquant ce que la déviation apporte, comme
le font les exceptions décidées ci-dessous. Une déviation non documentée est une dérive et le prochain contributeur
la « corrigera ». Une déviation documentée est une décision, et elle survit.

## Exceptions décidées

Examinées, volontairement laissées telles quelles. Ne les rouvrez pas sans une raison que la raison d'origine
laisse de côté.

- **La plaque du QR est `Color.White`** dans la feuille du QR Wi-Fi, et `QrCodeGenerator.encode` dessine
  des modules noirs sur blanc. Un scanner a besoin du contraste que la spécification suppose ; une plaque thématisée sous une
  palette sombre se lit comme un code cassé.
- **La carte de statut répond à un tap** (`ActiveNetworkCard` dans `ui/screen/MainScreen.kt`) bien qu'elle
  n'ouvre rien. C'est la carte que l'utilisateur tapote pour vérifier que l'application est vivante, donc elle produit un effet
  et retire le focus du champ au lieu de rester morte sous un doigt. Le coût est que TalkBack la lit comme un seul
  nœud actionnable.
- **La feuille du QR Wi-Fi, la feuille des appareils et la feuille des journaux** dans
  `ui/screen/MainScreen.kt`, et les feuilles de liste d'appareils dans `ui/component/DeviceList.kt`,
  restent des `ModalBottomSheet` Material plutôt que des
  `Dialog { Surface { } }`. Toutes sont du contenu à fermeture par glissement qui
  appartient au bord inférieur, et la feuille des journaux bloque la fermeture par geste pendant qu'une commande
  tourne, pour que la feuille ne puisse pas laisser derrière elle un voile qui mange les touches. Leur coque est
  `surfaceContainer` avec des coins supérieurs de 24dp, conformément au tableau des formes.
- **Le bouton du parcours de configuration** (`SetupActionBar` dans `ui/component/SetupHero.kt`) est une pastille pleine
  sans surface de barre et sans séparateur, pas le bouton de barre de 20dp. Il se pose seul sur le fond
  sous un hero plutôt qu'au-dessus d'un contenu défilant. Ses couleurs désactivées restent égales à ses couleurs
  activées : il n'est désactivé que pendant qu'une vérification ou une installation est en cours, et le griser pour
  ce moment se lit comme un clignotement.
- **Les alphas de remplissage ténus de 0.03 et 0.06** dans `LogActionRow` restent non partagés. Ils sont invisibles
  isolément, et les aligner signifierait inventer un jeton pour quelque chose que personne
  ne peut voir.
