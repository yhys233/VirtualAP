# VirtualAP

[English](README.md) | [简体中文](README.zh-CN.md) | [繁體中文](README.zh-TW.md) | [Русский](README.ru.md) | [Français](README.fr.md) | [Deutsch](README.de.md) | [日本語](README.ja.md) | [한국어](README.ko.md)

Transformez un téléphone Android rooté en véritable routeur Wi-Fi.

VirtualAP fait tourner un vrai point d'accès sur la puce Wi-Fi de votre téléphone. Vous
choisissez le nom du réseau, le mot de passe, la bande et le canal, l'adresse de passerelle que
voient vos appareils, et la destination de leur trafic : données mobiles, Wi-Fi ou VPN.

## Ce que vous pouvez faire

- **Faire tourner OpenWrt comme routeur de votre hotspot.** Zones de pare-feu, redirections de
  ports, règles de trafic, limites de bande passante et interface web LuCI, le tout sur votre
  téléphone. Voir le mode géré ci-dessous.
- **Partager un VPN.** Chaque appareil du hotspot passe par le VPN de votre téléphone. Rien à
  installer sur la TV, la console ou l'ordinateur portable.
- **Répéter le Wi-Fi.** Restez connecté à un réseau Wi-Fi et rediffusez-le comme le vôtre.
- **Garder une passerelle fixe.** L'adresse de passerelle ne change jamais : les redirections
  de ports et les cibles SSH enregistrées continuent donc de fonctionner.
- **Régler la radio.** 2,4 ou 5 GHz, canal et largeur, WPA2 ou WPA3, nom de réseau masqué.
- **Masquer le partage de connexion à votre opérateur** avec le correctif TTL optionnel, sur
  les noyaux qui le prennent en charge.

## Mode géré : OpenWrt pilote votre hotspot

C'est ce qui distingue VirtualAP. Confiez le hotspot à un conteneur OpenWrt exécuté dans
[Droidspaces](https://github.com/ravindu644/Droidspaces-OSS), et OpenWrt gère chaque appareil
connecté comme le ferait un vrai routeur Wi-Fi. Ouvrez dans un navigateur l'adresse de
passerelle que vous avez choisie : vous êtes dans LuCI.

Avec OpenWrt aux commandes, vous pouvez :

- bâtir un pare-feu avec autant de zones que vous voulez
- mettre en place des redirections de ports et des règles de trafic
- limiter la bande passante et le débit par appareil
- faire tourner un réseau invité qui ne peut pas atteindre vos autres appareils
- voir chaque appareil connecté et lui attribuer une adresse fixe et un nom
- choisir les serveurs DNS de tout le réseau et ajouter un blocage des publicités pour tous
  les appareils depuis le dépôt de paquets d'OpenWrt
- fournir IPv6 à vos appareils en plus d'IPv4
- placer vos conteneurs Droidspaces derrière le même routeur, sur le même réseau que vos
  appareils Wi-Fi ou isolés de ceux-ci
- installer tout ce qu'OpenWrt propose d'autre avec `opkg`

Le côté WAN d'OpenWrt est Droidspaces, et Droidspaces commute les liaisons montantes en temps
réel entre données mobiles, Wi-Fi et tunnels VPN. Votre hotspot en profite gratuitement :

1. Dans Droidspaces, définissez `tun0` comme première interface amont du conteneur OpenWrt,
   suivie de `wlan0` et `rmnet0`. Votre VPN ne tourne pas encore, donc `tun0` n'existe pas.
2. Démarrez VirtualAP. Le trafic de votre hotspot sort par Wi-Fi ou par données mobiles.
3. Démarrez votre application VPN. Droidspaces voit `tun0` apparaître et y bascule OpenWrt :
   un instant plus tard, chaque appareil du hotspot est derrière le VPN. Rien à redémarrer.

OpenWrt tourne dans un conteneur sur le noyau du téléphone lui-même, sans machine virtuelle
intermédiaire.

Le mode géré nécessite :

- un accès root
- [Droidspaces](https://github.com/ravindu644/Droidspaces-OSS) installé et fonctionnel
- le conteneur OpenWrt du
  [dépôt rootfs officiel de Droidspaces](https://github.com/Droidspaces/Droidspaces-rootfs-builder/releases),
  installé et démarré

## Mode interface

Le mode simple, sans conteneur ni interface web. VirtualAP distribue lui-même les adresses,
fournit IPv6 à vos appareils quand l'amont en dispose, et les fait sortir par l'interface que
vous choisissez, ou suit le réseau qu'Android utilise à ce moment-là. C'est tout ce qu'il vous
faut pour partager un VPN ou répéter le Wi-Fi.

## Prérequis

- Un téléphone rooté
- Android 8.0 ou plus récent
- Un téléphone ARM, 64 bits ou 32 bits. Un seul APK couvre les deux.
- Pour le démarrage automatique du point d'accès : Magisk ou KernelSU.
- Pour le mode géré : Droidspaces et son conteneur OpenWrt (voir ci-dessus)
- Pour le correctif TTL : un noyau qui le prend en charge. L'application le teste et grise
  l'option quand elle manque, et le hotspot fonctionne de la même façon sans lui.

## Pour commencer

1. Téléchargez le dernier APK depuis
   [Releases](https://github.com/ravindu644/VirtualAP/releases) et installez-le.
2. Ouvrez l'application et accordez root. Elle configure d'elle-même tout ce dont elle a
   besoin, y compris le module de démarrage Magisk/KernelSU optionnel. Rien à flasher ni à
   redémarrer.
3. Saisissez un nom de réseau et un mot de passe.
4. Choisissez l'amont : une **interface** (ou Auto), ou votre **conteneur** OpenWrt pour le mode
   géré.
5. Démarrez le hotspot et connectez vos appareils.

Activez **Start access point at boot** dans les paramètres pour qu'il revienne après un
redémarrage.

En mode géré, ouvrez `http://<your gateway address>` depuis un appareil connecté pour accéder à
LuCI.

## Pour aller plus loin

- [Comment ça marche](Documentation/How-It-Works.md) : le routage, IPv6, le pont, la radio, les
  fichiers sur le téléphone, et comment les binaires et l'APK sont construits
- [Contribuer](CONTRIBUTING.md)

## Licence

GNU General Public License v3.0. Voir [LICENSE](LICENSE).

VirtualAP est né du script
[`start-hotspot`](https://github.com/ravindu644/Ubuntu-Chroot/blob/main/tools/start-hotspot)
d'[Ubuntu-Chroot](https://github.com/ravindu644/Ubuntu-Chroot) ; l'application est écrite avec
l'aide d'un assistant IA.
