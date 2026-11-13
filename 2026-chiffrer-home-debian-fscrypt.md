# Chiffrer son dossier personnel sur Debian avec fscrypt

Un ordinateur portable se perd ou se vole, et un disque non chiffré se lit en quelques minutes depuis une clé USB de démarrage. Chiffrer son dossier personnel est la protection minimale contre ce scénario. Cet article fait le tour des options disponibles sur Debian, puis détaille pas à pas la mise en place de **fscrypt**, la solution la plus pratique sur un système déjà installé.

## Quelles options pour chiffrer sa home ?

| Outil | Principe | Quand le choisir |
|---|---|---|
| **LUKS** | Chiffre une partition entière | `/home` est sur sa propre partition, ou vous réinstallez le système |
| **fscrypt** | Chiffrement natif d'ext4, dossier par dossier | Système déjà installé, pas envie de repartitionner |
| **eCryptfs** | Couche de chiffrement empilée sur le système de fichiers | À éviter pour une nouvelle installation |

Quelques précisions :

- **LUKS** est la solution la plus robuste. Si vous installez Debian de zéro, l'option « LVM chiffré » de l'installateur chiffre tout le disque et règle la question une fois pour toutes. Sur un système existant, en revanche, il faut formater la partition concernée.
- **eCryptfs** est l'outil qui se cache derrière la case « Chiffrer mon dossier personnel » de l'installateur de Linux Mint. Il fonctionne, mais il n'est quasiment plus maintenu depuis des années, et un [ticket sur le dépôt de Linux Mint](https://github.com/linuxmint/linuxmint/issues/258) demande d'ailleurs son remplacement par fscrypt.
- **fscrypt** offre le même confort qu'eCryptfs (le dossier se déverrouille tout seul à la connexion, avec le mot de passe de session) en s'appuyant sur le chiffrement intégré au noyau. C'est lui qu'on va mettre en place.

## Avant de commencer

Trois prérequis :

1. **Une sauvegarde complète** de votre dossier personnel, sur un support externe. On va déplacer toutes vos données : ce n'est pas le moment de s'en passer.
2. **Un système de fichiers ext4** (ou f2fs). fscrypt ne fonctionne ni avec btrfs ni avec xfs.
3. **Un accès root hors de votre session** : le mot de passe root, ou un second compte disposant de sudo.

Pour vérifier le système de fichiers et repérer la partition concernée :

```bash
findmnt -no SOURCE,FSTYPE -T /home
```

La commande affiche quelque chose comme `/dev/sda2 ext4`. Notez le nom du périphérique : dans la suite, il remplace `/dev/sdXN`.

Dans les commandes qui suivent, l'utilisateur s'appelle `alexis`. Remplacez-le par votre identifiant.

## Étape 1 : installer fscrypt

```bash
sudo apt install fscrypt libpam-fscrypt
```

Le paquet `libpam-fscrypt` configure PAM automatiquement. C'est ce module qui déverrouillera votre dossier à l'ouverture de session.

## Étape 2 : activer le chiffrement sur le système de fichiers

```bash
sudo tune2fs -O encrypt /dev/sdXN
```

Cette commande mérite une explication, car son nom peut inquiéter. `tune2fs` modifie les paramètres d'un système de fichiers ext4 existant, et `-O encrypt` ajoute la fonctionnalité « encrypt » à la liste de ses options.

Elle **ne chiffre rien** et ne touche à aucun fichier : elle autorise simplement ext4 à contenir des dossiers chiffrés. Sans ce drapeau, fscrypt ne peut rien faire. Elle s'exécute sur une partition montée, sans redémarrage.

Pour contrôler que l'option est bien active :

```bash
sudo tune2fs -l /dev/sdXN | grep -i features
```

Le mot `encrypt` doit apparaître dans la liste. À noter : une fois activée, l'option ne se retire pas simplement. Sur une Debian récente, cela ne pose aucun problème ; seul un noyau ou un GRUB très ancien ne saurait plus lire la partition.

## Étape 3 : initialiser fscrypt

```bash
sudo fscrypt setup
```

Acceptez la création du fichier `/etc/fscrypt.conf`. Si `/home` se trouve sur une partition distincte de la racine, ajoutez :

```bash
sudo fscrypt setup /home
```

## Étape 4 : chiffrer le dossier personnel

fscrypt ne sait chiffrer qu'un dossier **vide**. La méthode consiste donc à créer un nouveau dossier chiffré, à y copier les données, puis à l'échanger avec l'ancien.

Fermez votre session graphique et connectez-vous en root sur une console texte (Ctrl+Alt+F3). Puis :

```bash
loginctl terminate-user alexis
mkdir /home/alexis.new
chown alexis:alexis /home/alexis.new
fscrypt encrypt /home/alexis.new --user=alexis
```

À la question sur le type de protection, choisissez **« Your login passphrase »** et saisissez le mot de passe de session de l'utilisateur. C'est ce choix qui lie le déverrouillage du dossier à la connexion.

Copiez ensuite les données et échangez les deux dossiers :

```bash
cp -a -T /home/alexis /home/alexis.new
mv /home/alexis /home/alexis.old
mv /home/alexis.new /home/alexis
reboot
```

## Étape 5 : vérifier et nettoyer

Reconnectez-vous normalement, puis contrôlez l'état du dossier :

```bash
fscrypt status /home/alexis
```

La sortie doit indiquer `Unlocked: Yes`. Prenez le temps de vérifier que vos fichiers et vos applications fonctionnent comme avant.

Quand tout est en ordre, supprimez la copie en clair :

```bash
sudo rm -rf /home/alexis.old
```

Sur un SSD, lancez ensuite `sudo fstrim -av` pour que les blocs libérés soient réellement effacés. Sur un disque dur classique, les anciennes données restent récupérables tant qu'elles n'ont pas été réécrites.

## Ce qu'il faut savoir au quotidien

- **Changement de mot de passe.** Utilisez toujours `passwd` depuis votre propre compte : fscrypt suit le changement. Si c'est root qui modifie le mot de passe, le dossier ne se déverrouillera plus avec le nouveau, et l'ancien sera nécessaire pour rétablir la situation.
- **Connexion sans mot de passe.** L'autologin et la connexion SSH par clé ne déverrouillent pas le dossier, puisque c'est le mot de passe qui sert de clé.
- **Ce qui reste visible.** Le contenu et les noms des fichiers sont chiffrés, mais pas leur taille, leurs dates ni la structure de l'arborescence.
- **Le swap et `/tmp`.** Ils ne sont pas couverts et peuvent contenir des fragments de vos données. Chiffrez le swap, ou désactivez-le si vous n'en avez pas besoin.
- **Les sauvegardes.** Un mot de passe oublié rend les données définitivement illisibles. Le chiffrement ne remplace pas une sauvegarde, il la rend indispensable.

## En résumé

Pour une nouvelle installation, le chiffrement complet du disque proposé par l'installateur Debian reste le choix le plus simple et le plus complet. Sur un système déjà en service, fscrypt permet de protéger son dossier personnel en une demi-heure, sans repartitionner et sans rien changer à ses habitudes : on se connecte avec son mot de passe, et le dossier s'ouvre.
