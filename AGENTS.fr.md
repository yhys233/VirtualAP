# AGENTS.md

[English](AGENTS.md) | [简体中文](AGENTS.zh-CN.md) | [繁體中文](AGENTS.zh-TW.md) | [Русский](AGENTS.ru.md) | [Français](AGENTS.fr.md) | [Deutsch](AGENTS.de.md) | [日本語](AGENTS.ja.md) | [한국어](AGENTS.ko.md)

Instructions pour les agents de codage IA qui travaillent dans ce dépôt. Les humains doivent lire
[CONTRIBUTING.fr.md](./CONTRIBUTING.fr.md), qui contient l'inventaire de ce qui existe déjà
ici. Le fonctionnement de l'ensemble est décrit dans
[Documentation/How-It-Works.fr.md](./Documentation/How-It-Works.fr.md).

## Ce que c'est

VirtualAP transforme un téléphone rooté en point d'accès Wi-Fi. Deux moitiés :

- `src/` est le moteur, `virtualap`, un binaire C statique qui tourne en root et en pilote deux
  autres : `hostapd` et `dnsmasq`, construits à partir des sous-modules git sous `externals/` par
  `scripts/build-static.sh`. `make aarch64 armhf` construit `virtualap`. Les trois arrivent dans
  `backend/aarch64/` et `backend/armhf/`, qui sont ignorés par git.
- `Android/` est l'application Compose qui pilote `virtualap` via un shell root.

La tâche gradle `prepareAssets` place les deux arborescences d'architecture dans une racine
d'assets générée sous `Android/app/build/`, et l'application déploie l'architecture correspondante
vers `/data/local/virtualap` au premier lancement et après chaque mise à jour de l'APK, avec un
module de démarrage pour le gestionnaire root. Il n'y a pas de chroot.

## Compilation

```
./scripts/build-static.sh    # from the repository root: both arches, needs Docker
./build.sh [release]         # from Android/: the APK, needs JDK 17 or newer, ANDROID_HOME and musl-cross gcc
```

Lancez la première avant la première compilation de l'application, puis de nouveau chaque fois
que `externals/` ou les scripts de compilation changent. La compilation de l'application lance
elle-même `make aarch64 armhf`, donc `virtualap` correspond toujours à `src/`. `prepareAssets`
échoue quand `backend/<arch>/` manque l'un des trois. `make check` lance les tests. La version
vient de `VAP_VERSION` dans `src/include/virtualap.h`. Incrémentez-la avec
`make bump NEW=x.y.z`, nulle part ailleurs. Les détails de compilation et de CI sont dans la section Building de How-It-Works.md.

## Commits

- Signez chaque commit : `git commit -s`.
- N'ajoutez jamais de ligne `Co-Authored-By:` pour un agent IA. Les co-auteurs humains sont acceptés.
- Préfixes, conformes à l'historique existant :
  - `app:` pour l'application Android, avec `app: fix:` et `app: refactor:` pour ces cas
  - `backend:` pour le moteur sous `src/`, `fix:` pour la correction d'un bug du moteur
  - `scripts:` pour la compilation des binaires statiques sous `scripts/` et `externals/`, `ci:` pour
    `.github/workflows/`, `docs:` pour la documentation
  - `fix(security):` pour tout ce qui touche à la sécurité, dans l'une ou l'autre moitié
  - `VirtualAP: bump vX.Y.Z` pour une montée de version, et rien d'autre dans ce commit

`ap:`, `build:`, `ui:`, `feat:` et `refactor:` apparaissent dans l'historique plus ancien. Ne les utilisez pas pour
les nouveaux commits.

## Style

Quatre règles. Elles s'appliquent au code, aux commentaires, aux messages de commit et à la documentation.

**Pas de tiret cadratin.** Utilisez une virgule, un point, ou réécrivez la phrase.

**Pas de commentaires en bandeau ASCII.** Pas de rangées de `-----`, pas de `=====`, pas d'en-têtes de section encadrés.

**Les commentaires sonnent comme une personne.** Dites pourquoi le code fait une chose, ou ce qui casse s'il
ne la fait pas. Omettez les commentaires qui répètent la ligne du dessous.

```sh
# Bad: bump the priority
PRIO_FROM_AP=7010

# Good: netd's own rules start at 10000, so 7010 wins before the VPN catch-all
PRIO_FROM_AP=7010
```

**Dix lignes qui marchent valent mieux que cent qui font la même chose.** Supprimez avant d'ajouter. Un
diff plus petit au bon endroit est l'objectif, pas un diff plus petit n'importe où.

Dans les documents, chaque fait est énoncé une fois, sur la page où le lecteur le cherche, et
les autres pages y renvoient. La répartition est fixe : README.md dit quoi, pour qui et les étapes ;
How-It-Works.md contient tous les faits techniques ; ce fichier contient les règles pour les agents ;
CONTRIBUTING.md dit comment un humain fait fusionner un changement et contient l'inventaire ; DESIGN.md
contient les valeurs de l'interface. Les tableaux de DESIGN.md restent, ce sont des cartes de rôles et des tailles qui ne fonctionnent
pas en prose ; ailleurs, une liste ou un paragraphe.

## Modifications de l'interface

Tout ce qui est visuel dans `Android/` suit [DESIGN.fr.md](./DESIGN.fr.md). Couleurs, typographie, espacement, rayons et
le motif de la pastille d'action y sont tous spécifiés, et ils ont été relevés dans l'application existante plutôt
qu'inventés, donc les suivre est aussi le plus petit diff.

Si une règle ne convient vraiment pas à votre cas, déviez, mais laissez un commentaire à l'endroit concerné disant ce
que la déviation apporte et dites-le dans la PR. Une déviation non documentée est une dérive et le prochain
contributeur la « corrigera ». La section « Exceptions décidées » de DESIGN.md liste les déviations qui
sont des décisions, ne « corrigez » pas celles-là.

## Avant même d'écrire

Parcourez cette liste et arrêtez-vous à la première réponse qui tient.

1. Est-ce que cela doit exister ? Un drapeau que personne n'a demandé, un réglage pour une valeur qui ne change jamais,
   une interface avec une seule implémentation : sautez, et dites-le en une ligne.
2. Est-ce que cela existe déjà ici ? Voir « Réutiliser avant d'écrire » ci-dessous. Faites un grep d'abord.
3. Est-ce que la plateforme le fait déjà ? La libc de musl et les interfaces du noyau (netlink,
   `/proc/sys`) côté moteur ; la bibliothèque standard Kotlin et Java et le framework Android côté
   application. Attention à `minSdk = 26`.
4. Une dépendance installée le couvre-t-elle ? libsu, Compose, lifecycle, zxing, appcompat côté
   application. N'ajoutez jamais une nouvelle dépendance pour ce que quelques lignes peuvent faire.
5. Est-ce que cela peut tenir en une ligne ? Alors c'est une ligne.
6. Ce n'est qu'alors que vous écrivez la plus petite chose qui marche.

Deux réponses tiennent ? Prenez la plus haute et continuez.

La liste raccourcit la solution, jamais la lecture. Remontez le flux que le changement touche avant
de choisir un échelon. Un petit diff au mauvais endroit est un second bug, pas un correctif paresseux.
Il en va de même pour les rapports de bug : un rapport nomme un symptôme, donc faites un grep de chaque appelant avant
d'éditer. Une protection dans la fonction partagée est plus petite qu'une protection dans chaque appelant, et elle
corrige les frères et sœurs que le rapport n'a pas mentionnés.

Ne simplifiez jamais jusqu'à supprimer la validation des entrées à une frontière de confiance, une vérification fail-closed,
la gestion des erreurs qui perd l'état, ou quoi que ce soit que le demandeur a nommé précisément.

## Réutiliser avant d'écrire

Faites un grep dans l'inventaire de CONTRIBUTING.md, puis dans l'arborescence. Ce sont des points de passage obligés. Contourner l'un
d'eux est un bug, pas un raccourci.

- `Backend.quote()` dans `Android/app/src/main/java/com/virtualap/app/util/Backend.kt` enveloppe
  chaque valeur dynamique qui atteint un shell root. `APManager` est la seule classe qui lance
  `virtualap`, et `Backend.startAp` est la seule façon d'écrire son chemin.
- `Hotspot` est le seul appelant de `APManager.start` et `APManager.stop`. Il possède la
  phase de démarrage/arrêt, le journal des commandes, la notification de fonctionnement et la synchronisation
  de la tuile des réglages rapides, donc l'écran et la tuile sont toujours d'accord. `APConfig.isValid()` est la seule
  validation au démarrage ; le bouton de l'écran et la tuile la sollicitent tous les deux.
- `src/` a un dossier par couche : `core`, `cli`, `net` (avec `net/ipt/`), `radio`, `mode`,
  `monitor`, `cmd`, puis `main.c`. Un fichier n'inclut que sa propre couche et celles en dessous,
  dans cet ordre. Chaque dossier a un en-tête public, `src/include/<folder>/<folder>.h`, ses
  prototypes groupés sous un libellé nommant le fichier qui les définit ; ce que seuls les fichiers
  du dossier et les tests utilisent va dans `src/<folder>/internal.h`, inclus en tant que `"internal.h"`
  (les tests le prennent par chemin). Tout le reste est `static`. Les fichiers font 200 à 800 lignes : une nouvelle
  fonction va dans le fichier de son sous-système, et un nouveau fichier est pour un nouveau sous-système ou un
  fichier au-delà de 800. `net/ipt/` a son propre en-tête parce que sur musl les en-têtes netfilter du noyau
  et les en-têtes réseau de la libc ne peuvent pas se rencontrer dans un seul fichier. Le code porté depuis
  Droidspaces et USB-Arsenal suit ces règles ; le code tiers reste dans `externals/`,
  intact.
- Le journal utilise `vap_log`, `vap_warn`, `vap_error` (`core/io`). Ils écrivent sur stdout, que
  l'application diffuse dans sa feuille de journal avec les balises `[WARN]` et `[ERROR]` qu'elle lit, et
  ajoutent une copie horodatée à `logs/ap.log`. L'application ne lit jamais ce fichier. Un
  `printf` nu est invisible dans le journal, et tout ce qui est sur stderr est perdu.
- Les programmes sont lancés par `core/proc` (`run_quiet`, `run_capture`, `run_shown`, `run_logged`,
  `run_lines`), jamais `popen` ou `system` : Android n'a pas de `/bin/sh`, et chaque enfant reçoit
  `/dev/null` pour stdin et un masque de signaux propre. Les fichiers pid passent par ses fonctions
  `pidfile_*`, qui ne signalent qu'un processus exécutant le programme attendu.
- Les réglages vivent dans une seule table, `vap_keys` dans `cli/options.c` : lettre de drapeau, clé
  `ap.conf`, valeur par défaut, texte d'usage. Un nouveau réglage est un champ dans `struct vap_opts` et une ligne
  là, et un champ dans `APConfig` ; l'analyse et le texte d'usage suivent. Chaque valeur passe par
  `opt_set`, qui refuse les retours à la ligne. `ap.conf` appartient à l'application : seul `APManager.saveConfig`
  l'écrit, et uniquement une configuration qui passe `APConfig.isValid()`. Le moteur ne fait que le lire.
- Le noyau est sollicité via `net/` (rtnetlink : `link` avec les adresses et les
  voisins, `route` avec les règles de politique et les tables de netd ; le moteur `ip_tables` dans
  `net/ipt/` : `ipt`, `ipt_rules`) et `radio/` (`driver` : nl80211 et la socket de contrôle
  wpa). Aucun module ne lance `ip`, `iw` ou `hostapd_cli` ; `iptables` n'est lancé que pour le
  correctif TTL (`ttl_*` dans `net/ipt/ipt_rules`).
- `upstream_resolve` (`net/route`) est le seul endroit où la table d'amont est résolue.
  `radio/channel`, `ensure_5g_ir` compris, est le seul endroit où vit la politique de bande et de canal :
  suivre le canal de la station, le repli DFS, le correctif NO-IR du pays. Ne comparez pas un
  numéro de canal ailleurs. `ipv6_decide` (`mode/ipv6`) est le seul endroit où l'IPv6 des clients est choisi ;
  le démarrage et le moniteur le sollicitent tous les deux.
- `Constants` contient chaque chemin et chaque clé de préférence côté application, ceux du moteur
  (`Constants.ENGINE`) compris. `VirtualAPInstaller` déploie les binaires ; il n'est pas
  dupliqué ailleurs.
- Aucun texte en Kotlin. Chaque chaîne affichée par l'application est dans `res/values/strings.xml` ou
  `plurals.xml`, lue avec `stringResource`, ou `getString` hors composition, pour que Weblate
  puisse la traduire. Les règles sont dans [CONTRIBUTING.fr.md](./CONTRIBUTING.fr.md), section Traductions.

## Contraintes strictes

- L'application cible `minSdk = 26`. Testez le comportement d'Android 8 avant de supposer qu'une API existe.
- Le moteur est du C11 (`gnu11`) sur musl, mono-thread : `setns` ne déplace que le thread
  appelant, et une socket netlink reste dans l'espace de noms où elle a été ouverte. Il se compile sans
  avertissement avec l'ensemble gcc du Makefile, que la CI transforme en erreurs.
- Les binaires sont des builds statiques musl complets issus des sous-modules `externals/`, les deux architectures. Jamais
  un binaire lié dynamiquement, jamais un précompilé venu d'ailleurs. Un nouvel outil est un nouveau sous-module
  plus un bloc dans `scripts/build-in-container.sh` ; `virtualap` est le seul binaire construit à partir du propre
  `src/` de ce dépôt, avec le gcc musl-cross et le `Makefile` racine.
- Aucune nouvelle dépendance quand une dépendance installée le couvre. L'application a déjà libsu, Compose,
  lifecycle, zxing et appcompat.
- Le pilote Broadcom FullMAC de Samsung est le plancher des hypothèses sans fil, et le
  gen4m de MediaTek le second ; ce qui manque à chacun est dans « La radio » de How-It-Works.md. Un correctif de
  particularité qui n'est pas conditionné à une détection à l'exécution n'entre pas.
