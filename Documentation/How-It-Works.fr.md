# Comment fonctionne VirtualAP

[English](How-It-Works.md) | [简体中文](How-It-Works.zh-CN.md) | [繁體中文](How-It-Works.zh-TW.md) | [Русский](How-It-Works.ru.md) | [Français](How-It-Works.fr.md) | [Deutsch](How-It-Works.de.md) | [日本語](How-It-Works.ja.md) | [한국어](How-It-Works.ko.md)

VirtualAP crée une interface AP virtuelle, `ap0`, sur la puce Wi-Fi du téléphone, lance `hostapd`
dessus, puis travaille dans l'un de deux modes. Tout ce qui suit est fait par `virtualap`, un
binaire C statique tournant en root, construit à partir de `src/` ; l'application ne fait qu'appeler ses commandes
(`start`, `stop`, `status`, `clients`, `interfaces`, `caps`). Il parle au noyau directement via
netlink (rtnetlink pour les liens, adresses, routes et règles ; nl80211 pour la radio) et via
la socket brute `ip_tables`. Il lance `hostapd` et `dnsmasq`, `iptables` pour le correctif TTL
uniquement, et, en mode géré, `droidspaces` pour trouver le conteneur et exécuter la configuration OpenWrt
dedans. Quels conteneurs tournent, la liste que l'application propose pour le mode géré, l'application
la demande à Droidspaces lui-même (`droidspaces show --json`, ou `show --format` sur une version
antérieure à `--json`).

## Mode routé (« Interface » dans l'application)

VirtualAP possède toute la couche réseau du hotspot. Il donne à `ap0` l'adresse de passerelle
(`192.168.42.1/24` sauf si vous en avez choisi une autre), sert DHCP et DNS avec son propre `dnsmasq`
(baux de `.10` à `.50` pendant 12 heures, DNS transmis aux serveurs que vous avez saisis, ou à
`1.1.1.1` et `8.8.8.8` par défaut), et NAT les clients vers l'amont.

Le trafic est dirigé par des règles de routage par politique épinglées au-dessus de celles d'Android, qui commencent à
la priorité 10000, pour qu'une règle attrape-tout d'un VPN ne puisse pas prendre le trafic du hotspot :

```
client -> ap0
       -> ip rule pref 7010: from all iif ap0 lookup <upstream table>
       -> MASQUERADE (-s <subnet> ! -d <subnet>)
       -> internet or VPN tunnel

reply  -> ip rule pref 7000: to <subnet> lookup main -> ap0
```

En mode Auto, l'amont est le premier de ces choix dont l'interface est active : la règle de réseau par défaut
de netd d'Android, le réseau qui a Internet en ce moment, lue dans la liste de règles du noyau,
la forme dans laquelle Android 15 l'écrit (une plage d'uid sur tous les uid) comprise ; la route par défaut
de la table main, pour les ROM qui la peuplent ; un balayage des noms de liaison montante connus (`wlan*`,
`eth*`, `v4-*`, `rmnet*`, `*ccmni*`), IPv4 seulement parce que la porteuse IMS toujours active les correspond
en IPv6. L'IPv4 passe par les trois avant que l'IPv6 soit essayée. Les interfaces de hotspot, de partage de connexion et de VPN
ne sont jamais choisies en Auto. `-o` désactive tout cela et prend à la place une liste séparée par des virgules
dans l'ordre de priorité, sans exclusions : la première entrée active et dotée d'une route
gagne, les routes IPv4 sur toute la liste avant l'IPv6. Une route ne compte que si elle est plus large qu'un /8, donc une route par défaut ou le
`0.0.0.0/5` d'un VPN en tunnel partagé compte, et le propre sous-réseau d'une interface non.

`interfaces`, que la liste déroulante de l'application affiche, liste ce que `-o` accepterait : les liens avec
une telle route dans n'importe quelle table, l'une ou l'autre famille, donc la porteuse IMS apparaît aussi. Il laisse de côté ce que
VirtualAP, Droidspaces et USB Arsenal créent et le `dummy0` de netd.

Aucun amont est un état normal, pas une erreur : l'AP se lève avec la seule règle 7000, les
clients sont refusés tout de suite, et le moniteur ajoute la règle 7010 quand un amont apparaît. Les
règles de pare-feu passent par la socket brute `ip_tables` sous le
verrou xtables d'Android, le même verrou que tiennent netd et `iptables -w`, donc aucun des deux côtés n'écrase jamais
le changement de l'autre.

La route du sous-réseau de l'AP est aussi recopiée dans la table `local_network` d'Android (97). Sans
cela, une réponse d'un conteneur Droidspaces (source `172.28.0.0/16`) vers un client du hotspot
tomberait dans la table WAN et sortirait par la liaison montante physique au lieu de `ap0`, donc
les clients ne pourraient pas atteindre les services à l'intérieur des conteneurs.

### IPv6

Les clients n'obtiennent de l'IPv6 que quand l'amont a une adresse globale ; un réseau Wi-Fi qui ne
distribue que des ULA laisse l'AP en IPv4 seul. Il y a deux façons d'en donner, essayées dans cet ordre :

- NAT66, l'approche de Droidspaces, quand le noyau a `ip6table_nat` avec MASQUERADE et que la
  table de l'amont a une route par défaut IPv6. `ap0` prend `::1` dans un /64 privé numéroté
  d'après le sous-réseau IPv4 (`fd76:6170:42::/64` pour `192.168.42.1`), `dnsmasq` l'annonce
  avec lui-même comme serveur DNS, et les clients sont masqués vers l'amont. Il
  survit intact à un changement d'amont ou de préfixe.
- Le propre /64 de l'amont, comme le fait le partage de connexion d'Android, sur les noyaux sans NAT IPv6
  (le GKI d'origine n'en livre aucun) quand l'amont détient un /64 global. `ap0` prend une adresse
  EUI-64 dedans et `dnsmasq` annonce le préfixe hors lien, donc un client envoie tout
  à la passerelle. Chaque adresse de client est routée en /128 vers `ap0` et une réponse est donnée pour elle sur
  l'amont avec une entrée de proxy NDP. Le moniteur trouve les adresses en pingant `ff02::1`
  depuis `ap0` toutes les 3 secondes, la seule chose qui les révèle ; il n'accepte que les réponses à
  son propre echo, jamais pour une adresse qui est celle du téléphone ou d'un voisin sur
  l'amont. La règle 7001 enterre un paquet d'amont pour une adresse du /64 qu'aucun client
  ne détient, ce qui sur le cellulaire rebondirait sinon jusqu'à la mort de sa limite de sauts. Sans le
  moniteur, le préfixe n'est pas annoncé.

Les deux ajoutent la règle 7002, qui donne aux propres paquets du téléphone vers l'adresse lien-local d'un client une
route : Android range les propres routes d'une interface dans une table qu'aucune règle ne lit, et la réponse de
`dnsmasq` à une sollicitation de routeur est l'un de ces paquets. L'IPv6 est du meilleur effort : quand une étape
échoue, l'AP reste en IPv4 seul.

## Le moniteur

Un démarrage réussi laisse `virtualap` tourner en moniteur (`run/monitor.pid`), un processus
sans threads, après le moniteur de routes de Droidspaces et le démon de partage de connexion d'USB-Arsenal. Il
écoute les événements de lien, d'adresse, de route et de règle du noyau, regarde 200 ms après le dernier
événement d'une rafale, et regarde toutes les 3 secondes quoi qu'il arrive, parce que netd réécrit `iptables`
sans événement. Chaque regard :

- re-résout l'amont et déplace la règle 7010 et `run.state` vers un nouveau, pour qu'un passage du Wi-Fi aux
  données mobiles ne laisse pas les clients sans Internet, et un démarrage sans réseau
  obtient son premier amont de la même façon. Si rien n'est trouvé, Auto laisse la règle tranquille,
  il n'y a rien de mieux vers quoi la pointer, et `-o` la retire jusqu'au retour d'une interface listée,
  donc son trafic ne sort jamais par autre chose ;
- remet toute règle ou entrée de pare-feu que netd a purgée, et `ip_forward`, seulement quand elle manque,
  pour que ses propres écritures ne le réveillent plus jamais ;
- refait l'IPv6, l'annonce de `dnsmasq` et le correctif TTL avec, quand l'IPv6 de l'amont
  a changé : un nouveau préfixe, une autre interface, IPv6 qui apparaît ou disparaît.

En mode géré, il surveille le veth à la place et branche un nouveau dans le conteneur quand
le conteneur redémarre. Il sort quand l'AP est parti ; `stop` le tue avant tout le reste.

## Mode géré (`-K <container>`)

Le LAN du hotspot est remis à un conteneur Droidspaces en cours. VirtualAP ne donne à `ap0` aucune
adresse, ne lance aucun `dnsmasq` et n'installe aucun NAT : il construit un chemin de couche 2 et le conteneur
est le routeur.

```
client -> ap0 (bridge port, no address)
       -> vap-br0 -> vaplan0 (inside the container)
       -> container LAN (DHCP, DNS, firewall)
       -> container NAT -> container WAN -> internet
```

`ap0` est asservi à un pont hôte, `vap-br0`, qui ne porte aucune adresse et a l'IPv6
désactivé pour qu'il ne se configure jamais à partir des annonces du conteneur. Une paire veth
rejoint le pont côté hôte ; l'autre extrémité est créée dans l'espace de noms réseau du conteneur,
déjà nommée `vaplan0` et avec sa MAC, en une seule requête. Un lien créé sur
l'hôte et renommé après le déplacement apparaît d'abord sous son ancien nom sur les noyaux antérieurs à 5.x,
et le netifd d'OpenWrt ne le regarde plus. Quand le conteneur est OpenWrt, VirtualAP
le provisionne avec un script `uci` exécuté via `droidspaces run` : une interface `vaplan` statique
sur la passerelle configurée, un /64 IPv6 issu de l'ULA du routeur épinglé au numéro du sous-réseau
IPv4 pour que netifd ne puisse pas le renuméroter, un pool DHCP avec annonces de routeur, et un
rechargement du pare-feu iptables, ou une règle MASQUERADE quand l'image n'a pas de paquet
pare-feu. Tout autre conteneur reçoit `vaplan0` et le configure lui-même.

La paire veth tourne en MTU 1508 tandis que `ap0` et les clients restent à 1500. Le firmware Broadcom de Samsung
remonte environ un quart des trames reçues avec le MIC CCMP de 8 octets encore sur la
fin, et un pont ou un veth libère silencieusement toute trame plus longue que son MTU plus 18 octets, donc
à 1500 une trame pleine taille d'un client était perdue une fois sur quatre et les téléversements dans le
conteneur rampaient à environ 1 Mo/s. La pile IP dans le conteneur retire le MIC, et la
RA épingle la MTU IPv6 des clients à 1500 avec `ra_mtu`, parce qu'odhcpd annonce sinon les
1508 du port.

Les deux extrémités du veth reçoivent RPS, sur chaque CPU sauf le CPU0. Un veth met chaque paquet en file sur le CPU
qui l'a envoyé, et un flux arrive depuis plusieurs, parce que le pilote Wi-Fi répartit son
travail de réception sur les cœurs et que TCP émet depuis le cœur qui a pris le dernier ACK. Sans RPS
l'extrémité distante voit le flux dans le désordre, et TCP retransmet des données qui n'ont jamais été perdues.
Le masque de `vaplan0` est écrit par un enfant qui entre dans l'espace de noms réseau du conteneur et
y monte un `/sys` qui lui est propre, jamais via `/proc/<pid>/root`, dont les liens symboliques sont choisis par le
conteneur. Le noyau ne garde que les CPU en ligne quand le masque est écrit, donc les cœurs
parqués à ce moment restent dehors jusqu'au redémarrage de l'AP.

Un veth neuf a une nouvelle MAC aléatoire à chaque démarrage, et Windows indexe ses profils réseau sur la
MAC de la passerelle, donc l'application peut en épingler une : une MAC aléatoire conservée entre les démarrages, ou la propre
MAC Wi-Fi du téléphone. Quand le conteneur redémarre, le
moniteur branche un nouveau veth, et la configuration OpenWrt, qui a survécu au redémarrage, le
récupère. L'ancien espace de noms emporte généralement le veth avec lui, mais il survit à son dernier processus
tant que quelque chose en tient une référence, et alors l'extrémité hôte reste, pairée à rien. Donc le
moniteur vérifie les deux : l'extrémité hôte est là, et le processus dans lequel le démarrage l'a branchée est
toujours dans l'espace de noms où elle a été branchée.

## Le correctif TTL

Avec `-T 1`, les paquets transférés sortent avec TTL 64 (limite de sauts 64 pour l'IPv6), donc pour
l'opérateur ils ressemblent à ceux du téléphone. Les règles vivent dans une chaîne mangle propre,
`vap_ttl`, filtrée sur le sous-réseau source des clients dans `FORWARD`, qui ne voit que les paquets
transférés après que le noyau a retiré son saut. Le backend sonde la cible TTL du noyau
(`CONFIG_NETFILTER_XT_TARGET_HL`) en ajoutant la vraie règle dans une chaîne vers laquelle rien ne saute ;
l'application grise l'option quand la sonde échoue. Ce sont les seules règles encore écrites par
le binaire `iptables`.

## La radio

La plupart des puces de téléphone ne peuvent émettre de balise que sur le canal auquel la station est connectée, donc le
backend suit le canal de la station quand il correspond à la bande. Le canal de la station vient
de la fréquence du BSS auquel elle est associée dans les résultats de scan, ce que lit
`iw dev wlan0 link`, parce que le pilote Broadcom de Samsung omet le canal des
informations d'interface. Une station sur un canal DFS (52 à 144) retombe en 2,4 GHz : les puces de téléphone
ne peuvent pas y émettre de balise. Une station sur une autre bande, 6 GHz compris, laisse l'AP à son propre
défaut. Sans station, la 5 GHz démarre sur le canal 36 et la 2,4 GHz sur le canal 6.

Le firmware Broadcom FullMAC de Samsung démarre dans le domaine réglementaire mondial « 99 », et le framework
ne pousse parfois jamais un vrai pays, donc chaque canal 5 GHz reste NO-IR et `hostapd`
meurt au bout d'environ cinq secondes, après la vérification de vivacité. Le wiphy est auto-géré, donc
`iw reg set` et le `country_code` de `hostapd` sont ignorés ; le seul levier est la commande `COUNTRY`
privée du pilote via la socket de contrôle du supplicant, envoyée avec le pays que le
framework rapporte (`ensure_5g_ir`). Elle ne se déclenche que quand le phy signale le canal comme
NO-IR, et si le canal est encore NO-IR ensuite l'AP retombe en 2,4 GHz.
`hostapd.conf` conserve `country_code=US`.

La largeur suit ce que la puce et le canal peuvent faire, d'après les capacités de cette bande : 40 MHz
en 2,4 GHz seulement quand on le demande par son nom, jusqu'à 80 MHz en 5 GHz quand la bande liste VHT et
que le canal se trouve dans un bloc de 80 MHz. HT40 et VHT exigent que `hostapd` saute son scan de coexistence
20/40 MHz, que les puces FullMAC ne peuvent pas exécuter sur une interface AP ; l'option
`no_pri_sec_switch` qui fait cela est un patch dans notre fork de `hostapd`, et la compilation
vérifie que le patch est présent. Si `hostapd` refuse 40 MHz en 2,4 GHz, le backend réessaie
à 20.

Le fork ne dit jamais non plus à `hostapd` qu'un pilote a une liste de contrôle MAC, et la compilation vérifie
cela aussi. La liste de contrôle d'un pilote peut contenir moins d'adresses que la liste, et au-delà `hostapd` saute sa
propre vérification tandis que le pilote ne détient rien. Donc les listes MAC sont vérifiées dans `hostapd` sur chaque
puce : un appareil listé s'associe un instant et est retiré avant sa poignée de main. Les versions
Broadcom de Samsung compilent leur liste de contrôle hors du code et gen4m n'en a pas, donc elles fonctionnent déjà ainsi.
Le qcacld de Qualcomm en a une, donc c'est le pilote que cela change, et aucun téléphone qcacld ne l'a exécuté.

Le pilote gen4m de MediaTek n'a pas de `dump_station`, donc le nombre de clients vient de la
propre liste de stations de `hostapd` via sa socket de contrôle, liée à une adresse abstraite puisque Android n'a
pas de `/tmp`, avec un délai d'une seconde pour qu'un `hostapd` bloqué ne puisse pas bloquer `status`.

`clients` parcourt cette même liste et affiche `mac ip name` pour chaque station qui a fini sa
poignée de main, `-` pour ce qui n'est pas connu. Le compte dans `status` est le même parcours, donc un appareil
avec le mauvais mot de passe n'est ni dans l'un ni dans l'autre. L'application ne le demande
que pendant que sa feuille des appareils connectés est ouverte, puisque le mode géré lance `droidspaces`, et
ce lancement a une limite de cinq secondes parce que libsu exécute une commande à la fois. L'adresse
et le nom viennent du bail le plus récent dans le /24 de la passerelle : notre propre `dnsmasq.leases`
en mode routé, le `/tmp/dhcp.leases` du conteneur en mode géré. Celui-là est lu via
`/proc/<pid>/root`, puisque le `/tmp` d'OpenWrt est un tmpfs dans l'espace de noms de montage du conteneur, un
composant à la fois et seulement comme fichier ordinaire sur un tmpfs, donc un lien symbolique, un FIFO ou un montage bind
que le conteneur y aurait laissé est refusé. Un client qui n'envoie pas de nom d'hôte, ou un conteneur
sans ce fichier, laisse ces champs à `-`.

Un appareil sur une adresse statique garde le bail de l'adresse qu'il avait, donc la table ARP du noyau
le corrige : `/proc/net/arp` en mode routé, `/proc/<pid>/net/arp` en mode géré, où le
LAN des clients se termine dans le conteneur. Un bail tient tant que la table le confirme ou ne connaît pas
d'autre adresse pour cette MAC, et la table peut donner une adresse à un appareil sans bail. Elle
n'a qu'un appareil qui a envoyé quelque chose, et elle marque aussi une entrée périmée comme complète, donc un
appareil peut afficher une adresse qu'il vient de quitter.

`block <mac> [name]` ajoute un appareil à `/data/local/virtualap/blocklist`, et `unblock <mac>`
l'en retire ; `blocked` affiche `mac name` pour chacun, `-` pour un nom absent. `allow`, `disallow` et
`allowed` font de même pour `allowlist`. `allow-only on` exclut tout appareil qui n'est pas sur la
liste d'autorisation, déconnecte ceux qui le sont, et est refusé tant que la liste est vide ; le dernier appareil
retiré de la liste le désactive, puisque personne ne pourrait se joindre autrement. Sans argument il affiche `on` ou
`off`. Cet état est le fichier `allow_only`, à côté des listes, parce que
comme elles il change une commande à la fois, hotspot allumé ou éteint. Un appareil n'est que sur une liste, et
l'ajouter à l'une le retire de l'autre, parce que `hostapd` vérifie d'abord la liste d'autorisation et laisserait
entrer un appareil présent sur les deux.

Les listes sont hors de `ap.conf` parce que l'application les modifie une entrée à la fois, pas comme un réglage qu'un
démarrage transmet. Les fichiers sont les `deny_mac_file` et `accept_mac_file` propres à `hostapd`, nommés dans sa
configuration seulement quand ils existent, puisque `hostapd` ne démarrera pas sur une liste qu'il ne peut ouvrir. Une ligne est
`mac #name` : `hostapd` lit ce qui suit une MAC comme un id de VLAN via `atoi`, et `#` rend cela
0, pas de VLAN. Un hotspot en marche entend un changement via la socket de contrôle : `DENY_ACL ADD_MAC`
déconnecte l'appareil s'il est connecté, et de même `ACCEPT_ACL DEL_MAC` une fois que seuls les appareils autorisés
peuvent se joindre. Un démarrage réécrit d'abord les fichiers, en journalisant et en abandonnant toute ligne dont il n'est pas sûr
que `hostapd` puisse la lire, et chacun contient environ 400 appareils.

L'application bloque depuis sa feuille des appareils sans confirmation, parce que Unblock est à côté, et
garde les deux listes dans les Paramètres, pour quand le hotspot est éteint. Son champ MAC ne peut contenir que
des chiffres hexadécimaux, et refuse ce que le moteur refuse : tous les zéros et les adresses de groupe. L'une ou l'autre liste
ne contient que cette MAC. Un appareil qui change la sienne échappe à un blocage et sort d'une liste d'autorisation,
et les téléphones utilisent une adresse aléatoire pour chaque réseau par défaut, donc celle à saisir est
celle des détails Wi-Fi de l'appareil pour ce hotspot.

## Sur le téléphone

Tout est sous `/data/local/virtualap` :

- `bin/` : `virtualap`, `hostapd` et `dnsmasq`, binaires statiques musl complets pour l'architecture du téléphone.
  `iptables` est celui d'Android.
- `ap.conf` : les réglages du hotspot en lignes `KEY='value'`. L'application l'écrit chaque fois que ce qui est à
  l'écran est valide et avant chaque démarrage, et un formulaire à moitié rempli reste à l'écran. Le
  moteur ne fait que le lire, et un drapeau le remplace pour cette exécution. Il contient le mot de passe Wi-Fi, donc
  il n'est lisible que par root.
- `blocklist`, `allowlist` et `allow_only` : les appareils bloqués et autorisés et si seuls
  ceux-là peuvent se joindre, voir [ci-dessus](#la-radio). Aucun n'existe avant le premier usage.
- `.boot_mode` : `1` quand le hotspot démarre au boot. Le basculeur des Paramètres l'écrit et le module de démarrage
  le lit.
- `run.state` : ce que l'AP en marche doit démonter, la passerelle, l'amont, le mode et l'interface
  IPv6 et les valeurs sysctl à restaurer. `status` base « running » sur lui et
  le fichier pid de `hostapd` ; `stop` le supprime avant de sortir.
- `run/` : les fichiers pid (`hostapd`, `dnsmasq`, le moniteur), les `hostapd.conf`
  et `dnsmasq.conf` générés, les baux, la liste des clients IPv6, la socket de contrôle de `hostapd` et le
  verrou qui empêche deux démarrages ou arrêts de tourner en même temps.
- `logs/` : `ap.log`, une copie horodatée de chaque ligne que l'application a montrée et de ce que
  le moniteur a fait, tronquée à chaque démarrage, plus `hostapd.log` et `dnsmasq.log`, parce que les deux
  démons emporteraient sinon leur raison d'échec avec eux, et `boot-module.log`, ce que le
  module de démarrage a fait au dernier boot.

On n'agit sur un fichier pid que quand le processus qu'il nomme exécute le programme attendu, donc un
pid réutilisé après un redémarrage n'est jamais signalé. L'application déploie `bin/` au premier lancement et de nouveau
chaque fois qu'un APK livre des binaires différents : la compilation écrit `PAYLOAD_VERSION`, un hachage des
binaires de chaque architecture et du module de démarrage, à côté d'eux, et l'application le compare à celui
qu'elle a déployé en dernier.

Déployer le backend installe aussi `/data/adb/modules/virtualap`, le module de démarrage, de la même façon que
Droidspaces installe le sien. Un téléphone sans gestionnaire root s'en passe, et le basculeur des Paramètres,
qui n'écrit que `.boot_mode`, n'a alors rien pour le lire. Le `service.sh` du module tourne en late
start et attend `sys.boot_completed`. Avec `.boot_mode` actif, il donne 25
secondes à l'interface Wi-Fi pour apparaître et, en mode géré, 25 de plus au conteneur pour tourner, en sautant le démarrage si
l'un des deux ne le fait pas. Il n'attend pas le réseau, puisque le hotspot suit un amont qui vient
plus tard. Le démarrage lit `ap.conf`, qui se trouve hors du stockage chiffré de l'application, donc le démarrage au boot
n'a pas besoin d'un déverrouillage. Le résultat s'affiche comme la description du module dans le gestionnaire
root. Après un démarrage réussi il attend le premier déverrouillage, avant lequel l'application ne peut pas tourner,
et envoie un broadcast à `HotspotBootReceiver`, qui affiche la notification de fonctionnement.

## Compilation

`hostapd` et `dnsmasq` sont construits à partir des sous-modules git sous `externals/`, nos propres forks,
par `scripts/build-static.sh`. Chaque architecture est construite dans un conteneur Alpine 3.23 de cette architecture,
sous QEMU sur un hôte x86, avec `-static -no-pie` pour que le résultat soit un ELF autonome sans
interpréteur. Le fork de `dnsmasq` retire la chute de privilèges `getpwnam("root")`, puisque Android
n'a pas d'entrée root dans `/etc/passwd` et que musl statique n'en synthétisera pas. `virtualap`
ne lie que la libc et n'a pas besoin de conteneur : le `Makefile` racine le construit à la
manière de Droidspaces, `make aarch64 armhf` avec le gcc musl-cross dans
`~/toolchains/<triplet>-cross` (ou `MUSL_CROSS`), que la tâche gradle `buildBackend` exécute
avant chaque compilation de l'application. La sortie atterrit dans `backend/<arch>/`, qui est ignoré par git, et la
tâche gradle `prepareAssets` place les trois binaires dans les assets de l'APK, en échouant quand l'un
manque.

`src/` a un dossier par couche, chacun avec un en-tête public sous `src/include/<folder>/`
et, là où ses fichiers partagent plus que cela, un `internal.h` privé à côté de ses sources :
`core`, `cli`, `net` (rtnetlink, et le moteur `ip_tables` dans `net/ipt/`), `radio`
(nl80211, la socket de contrôle, la politique de canal), `mode` (routé, géré, IPv6, `hostapd` et
`dnsmasq`), `monitor` et `cmd`. `make check` exécute les assertions de `tests/`, celles de netlink
contre le noyau dans un espace de noms utilisateur et réseau jetable, donc ni root ni
téléphone ne sont nécessaires ; construits pour le téléphone et exécutés là en root, ils exercent aussi le vrai
`ip_tables`.

La CI construit `hostapd` et `dnsmasq` pour chaque architecture sur un runner ARM, avec un cache sur
les arbres `externals/` et `scripts/`. Le job de l'APK récupère ensuite les chaînes d'outils musl publiées par
Droidspaces (vérifiées contre leur sha256 et mises en cache sur leur URL de release), exécute
`make check` et `make aarch64 armhf` avec les avertissements en erreurs, et construit l'APK avec
JDK 17. Un tag `v*` poussé qui correspond à `VAP_VERSION` dans `src/include/virtualap.h` publie
une release avec l'APK, un fichier `SHA256SUMS` et un changelog de chaque commit depuis le
tag précédent ; le tag doit correspondre et la clé de signature doit être présente, sinon l'exécution s'arrête.
La CI n'exécute ni formateur ni linter.

`make bump NEW=x.y.z` réécrit `VAP_VERSION`, et le Makefile, gradle et la CI le lisent tous
de là. Le versionCode de l'APK est le code de version (majeur·10000 + mineur·100 + correctif)
fois 10000 plus `git rev-list --count HEAD`, donc l'APK de chaque commit s'installe par-dessus le dernier.
Le compte a besoin de l'historique complet, c'est pourquoi le job de l'APK se checkout avec `fetch-depth: 0` : un
clone superficiel compte 1 et un arbre sans `.git` compte 0. Le `module.prop` du module de
démarrage prend le code de version seul, parce qu'il est haché dans `PAYLOAD_VERSION`.

## Traductions

Le texte de l'application est dans `Android/app/src/main/res/values/strings.xml` et ses pluriels dans
`plurals.xml` à côté, deux composants Weblate, `Strings` et `Plurals`, la même disposition que
Droidspaces. Les deux utilisent le format de fichier Android String Resource, le masque de fichier
`Android/app/src/main/res/values-*/strings.xml` ou `plurals.xml`, le fichier de base dans `values/`
comme fichier de langue de base monolingue et comme modèle pour les nouvelles traductions, et les
codes de langue de style Android (`values-pt-rBR`). Activez l'add-on Cleanup translation files de Weblate :
une clé retirée d'un fichier de base et laissée dans une traduction est `ExtraTranslation` du lint,
qui est fatal, donc la compilation de release échoue jusqu'à ce qu'elle parte. Une traduction manquante ne fait que
retomber sur l'anglais.

L'application suit la langue du système jusqu'à ce que le sélecteur de langue des Paramètres dise le contraire. La liste du sélecteur
et la liste de langues par application d'Android 13 sont un seul fichier : `generateLocaleConfig` écrit
`res/xml/_generated_res_locale_config.xml` à partir des dossiers `values-*`, `res/resources.properties`
dit que `res/values` est l'anglais, et `LocaleHelper` lit le résultat, donc une langue fusionnée depuis
Weblate est proposée à la compilation suivante, aussi peu qu'il en soit traduit. Le choix passe par
AppCompat, c'est pourquoi `MainActivity` est un `AppCompatActivity` : Android 13 et ultérieur le garde dans
le système, et sur 8 à 12 AppCompat le stocke lui-même et enveloppe l'activité, pas l'application
ni les services, donc là la notification et la tuile des réglages rapides restent dans la langue du système. Sur n'importe quelle
version, une notification déjà affichée prend une nouvelle langue à sa prochaine mise à jour.
Le journal reste en anglais, c'est ce que
le moteur a imprimé. JetBrains Mono, pour ce journal et le mot de passe sur la feuille du QR, est la seule
police fournie ; chaque libellé traduit est dans la police système, donc n'importe quel script s'affiche.
