# Installer et configurer isc-dhcp-server sur Debian

Attribuer une adresse IP à la main sur chaque machine, ça va pour trois postes. Au-delà, on veut un serveur DHCP : il distribue automatiquement l'adresse, le masque, la passerelle et les DNS à tout ce qui se branche sur le réseau.

Dans ce tutoriel, on installe `isc-dhcp-server` sur Debian et on le configure pas à pas : plage d'adresses, options réseau, réservations par adresse MAC, puis tests et dépannage. Comptez une quinzaine de minutes.

## Avant de commencer

### Un mot sur l'état du projet

ISC a arrêté le développement d'ISC DHCP fin 2022 au profit de son successeur, **Kea** ([annonce ISC](https://www.isc.org/blogs/isc-dhcp-eol/)). Le paquet `isc-dhcp-server` reste disponible dans Debian 12 (Bookworm) et Debian 13 (Trixie), où il est marqué « deprecated ».

Concrètement :

- pour un labo, un réseau domestique, une formation ou un parc existant, il fait toujours très bien le travail ;
- pour un nouveau déploiement en production appelé à durer, regardez plutôt Kea (`kea-dhcp4-server`).

### Prérequis

- une machine Debian 12 ou 13 avec un accès `root` ou `sudo` ;
- une adresse IP **statique** sur l'interface qui servira le DHCP ;
- aucun autre serveur DHCP actif sur le même réseau (pensez à désactiver celui de votre box ou routeur, sinon les deux se feront concurrence).

### Le réseau d'exemple

Tout au long de l'article, on utilise ces valeurs. Adaptez-les à votre réseau.

| Élément | Valeur |
|---|---|
| Réseau | `192.168.10.0/24` |
| Serveur DHCP | `192.168.10.1` |
| Interface | `enp0s3` |
| Passerelle | `192.168.10.254` |
| DNS | `9.9.9.9`, `1.1.1.1` |
| Plage distribuée | `192.168.10.100` à `192.168.10.200` |
| Domaine | `exemple.lan` |

## Étape 1 : donner une IP statique au serveur

Un serveur DHCP ne peut pas dépendre d'une adresse elle-même obtenue en DHCP. Repérez d'abord le nom de votre interface :

```bash
ip -br a
```

Puis éditez `/etc/network/interfaces` :

```
auto enp0s3
iface enp0s3 inet static
    address 192.168.10.1/24
    gateway 192.168.10.254
```

Appliquez la configuration :

```bash
sudo systemctl restart networking
```

> Si votre machine utilise NetworkManager ou systemd-networkd plutôt qu'ifupdown, configurez l'adresse statique avec cet outil. L'important est le résultat, pas la méthode.

## Étape 2 : installer le paquet

```bash
sudo apt update
sudo apt install isc-dhcp-server
```

À la fin de l'installation, le service tente de démarrer et **échoue**. C'est normal : il n'a encore ni interface ni sous-réseau à servir. On s'en occupe tout de suite.

## Étape 3 : choisir l'interface d'écoute

Le fichier `/etc/default/isc-dhcp-server` indique sur quelle(s) interface(s) le serveur répond :

```bash
sudo nano /etc/default/isc-dhcp-server
```

Renseignez l'interface pour IPv4 et laissez IPv6 vide si vous ne l'utilisez pas :

```
INTERFACESv4="enp0s3"
INTERFACESv6=""
```

Pour plusieurs interfaces, séparez-les par un espace : `INTERFACESv4="enp0s3 enp0s8"`.

## Étape 4 : configurer dhcpd.conf

Toute la configuration se trouve dans `/etc/dhcp/dhcpd.conf`. Gardez une copie du fichier d'origine, il contient des exemples commentés utiles :

```bash
sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.orig
sudo nano /etc/dhcp/dhcpd.conf
```

Voici une configuration minimale complète :

```
# --- Options globales ---
option domain-name "exemple.lan";
option domain-name-servers 9.9.9.9, 1.1.1.1;

default-lease-time 3600;    # durée d'un bail par défaut : 1 h
max-lease-time 86400;       # durée maximale : 24 h

authoritative;              # ce serveur fait autorité sur ce réseau
ddns-update-style none;     # pas de mise à jour DNS dynamique

# --- Sous-réseau servi ---
subnet 192.168.10.0 netmask 255.255.255.0 {
    range 192.168.10.100 192.168.10.200;
    option routers 192.168.10.254;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.168.10.255;
}
```

Ce que fait chaque directive :

| Directive | Rôle |
|---|---|
| `option domain-name` | suffixe DNS envoyé aux clients |
| `option domain-name-servers` | serveurs DNS envoyés aux clients |
| `default-lease-time` | durée du bail (en secondes) si le client n'en demande pas |
| `max-lease-time` | durée maximale qu'un client peut obtenir |
| `authoritative` | le serveur corrige les clients qui présentent une adresse invalide pour ce réseau |
| `subnet … netmask …` | déclare le réseau servi ; il doit correspondre à celui de l'interface d'écoute |
| `range` | première et dernière adresse distribuées |
| `option routers` | passerelle par défaut |

Deux pièges de syntaxe classiques : chaque ligne se termine par un **point-virgule**, et chaque accolade ouverte doit être refermée.

## Étape 5 : réserver une adresse pour une machine

Une imprimante ou un NAS doit toujours recevoir la même adresse. On associe pour cela son adresse MAC à une IP fixe, en ajoutant un bloc `host` dans `dhcpd.conf` :

```
host imprimante-bureau {
    hardware ethernet 00:11:22:33:44:55;
    fixed-address 192.168.10.50;
}
```

Choisissez une adresse **en dehors** de la plage `range` (ici `.50`, alors que la plage commence à `.100`) pour éviter qu'elle soit aussi distribuée dynamiquement à quelqu'un d'autre.

Pour trouver l'adresse MAC d'une machine : `ip link` sous Linux, `ipconfig /all` sous Windows.

## Étape 6 : vérifier, démarrer, ouvrir le pare-feu

Testez toujours la syntaxe avant de (re)démarrer le service :

```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

Aucune erreur affichée ? Démarrez le service et activez-le au démarrage :

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

Vous devez voir `active (running)`.

Si un pare-feu est actif sur le serveur, autorisez le port DHCP (UDP 67). Avec `ufw` par exemple :

```bash
sudo ufw allow 67/udp
```

## Étape 7 : tester depuis un client

Branchez une machine sur le réseau et demandez un bail :

- **Linux** : `sudo dhclient -v enp0s3`, ou débranchez/rebranchez simplement l'interface ;
- **Windows** : `ipconfig /release` puis `ipconfig /renew`.

Côté serveur, suivez l'échange en direct :

```bash
sudo journalctl -u isc-dhcp-server -f
```

Un échange réussi affiche les quatre étapes dans l'ordre : `DHCPDISCOVER`, `DHCPOFFER`, `DHCPREQUEST`, `DHCPACK`.

Pour lister les baux attribués :

```bash
dhcp-lease-list
```

Les baux sont stockés dans `/var/lib/dhcp/dhcpd.leases`. Ne modifiez pas ce fichier à la main pendant que le service tourne.

## Dépannage : les erreurs fréquentes

Le premier réflexe en cas de problème :

```bash
sudo journalctl -u isc-dhcp-server -n 50 --no-pager
```

| Message ou symptôme | Cause probable | Solution |
|---|---|---|
| `No subnet declaration for enp0s3` | aucun bloc `subnet` ne correspond au réseau de l'interface | vérifier l'IP de l'interface et le bloc `subnet` |
| `Not configured to listen on any interfaces` | `INTERFACESv4` vide ou nom d'interface faux, ou interface sans IP | corriger `/etc/default/isc-dhcp-server`, vérifier `ip -br a` |
| Erreur de syntaxe avec un numéro de ligne | point-virgule ou accolade manquant | corriger la ligne indiquée (ou celle juste avant) |
| Le service échoue au sujet de `dhcpd6` | `INTERFACESv6` renseigné sans configuration IPv6 | laisser `INTERFACESv6=""` |
| Les clients reçoivent une adresse d'un autre réseau | un second serveur DHCP répond (box, routeur) | désactiver l'autre serveur |
| Aucun `DHCPDISCOVER` dans les journaux | le client n'est pas sur le même segment, ou pare-feu | vérifier câblage/VLAN et port UDP 67 |

## Conclusion

Votre serveur distribue maintenant des adresses sur le réseau, avec des réservations pour les machines qui en ont besoin. À retenir :

1. IP statique sur le serveur ;
2. interface dans `/etc/default/isc-dhcp-server` ;
3. sous-réseau, plage et options dans `/etc/dhcp/dhcpd.conf` ;
4. `dhcpd -t` avant chaque redémarrage ;
5. `journalctl` dès que quelque chose cloche.

Pour aller plus loin : plusieurs sous-réseaux, relais DHCP entre VLAN, démarrage PXE… et, à terme, la migration vers Kea, puisque ISC DHCP ne reçoit plus de mises à jour.
