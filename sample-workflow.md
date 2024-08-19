## Déchiffrement du disque:

Installation des packages Yubikey utils pour déchiffrer le disque:
```bash
sudo apt update
sudo apt install yubikey-luks yubikey-personalization
```
Configuration du slot 2 de la yubikey pour l'authentification en mode challenge-response:

```bash
ykpersonalize -2 -ochal-resp -ochal-hmac -ohmac-lt64 -oserial-api-visible
```

*Warning: Faire attention à ne configurer une seule fois le slot de la clé, autrement les challenge configurés précédement ne seront plus fonctionnels*


utilisation de la commande *lsblk* afin de retrouver le nom de la partition à chiffrer

```bash
lsblk
```

S'assurer que le slot 1 est vide afin de ne pas écraser une configuration précédente:
```bash
sudo cryptsetup luksDump /dev/nvme0n1p3* 
```
Assigner la yubikey au slot 1 (s'il est disponible, autrement utiliser un autre slot utilisable)

```bash
sudo yubikey-luks-enroll -d /dev/nvme0n1p3* -s 1
```

\* Remplacer par le nom de votre partition effective

Modification du fichier /etc/crypttab

changer la ligne actuelle: 

**nvme0n1p3_crypt UUID=abcdefab-1234-abcd-abcd-123456789abc none luks,discard**

Par la suivante: 

```bash
nvme0n1p3_crypt UUID=abcdefab-1234-abcd-abcd-123456789abc none luks,discard,keyscript=/usr/share/yubikey-luks/ykluks-keyscript
```
Avec la configuration actuelle, le systeme vous demandera d'entrer le mot de passe correspondant à la yubikey que vous aurez rentré lors de l'assignation de la yubikey sur la partition à chiffrer, autrement il est également possible de booter et de déchiffrer le disque sans aucune interaction de l'utilisateur en ajoutant la ligne suivante au fichier */etc/ykluks.cfg:*
```bash
YUBIKEY_CHALLENGE="YOUR PASSPHRASE HERE"
```

Ensuite il faut remplacer le fichier */usr/share/yubikey-luks/ykluks-keyscript* par le fichier suivant:

[ykluks-keyscript](https://github.com/espegro/yubikey-luks/blob/main/ykluks-keyscript)

Une fois fait il faut mettre à jour *initramfs* 
```bash
sudo update-initramfs -u
```

Vous pouvez maintenant redémarrer pour tester votre nouvelle configuration.

Maintenant vous pouvez déchiffrer votre disque:
* en utilisant le mot de passe initial du disque
* booter sans interaction de votre part si vous avez configuré *ykluks.cfg et ykluks-keyscript*
* booter à la page de déchiffrement du disque, entrer la yubikey puis taper le challenge configuré sur celle-ci.

Ensuite il faut configurer un second mot de passe pour déchiffrer le disque par les admin. Il suffit d'entrer la commande suivante:

```bash
sudo cryptsetup luksAddKey /dev/nvme0n1p3*
```
La commande précédente assignera le nouveau mot de passe automatiquement au prochain slot disponible.

La dernière étape pour déchiffrer le dique est d'assigner la clé admin également sur un autre slot disponible en utilisant la commande *luks-enroll:*
```bash
sudo yubikey-luks-enroll -d /dev/nvme0n1p3* -s 1
```

# Authentification deux facteurs pour s'identifier et utiliser la commande *sudo*

Installation des packages nécéssaires:
```bash
sudo apt install libpam-u2f pamu2fcfg
```

Génération du token pour l'utilisateur courant:
```bash
pamu2fcfg -u `whoami` -opam://`hostname` -ipam://`hostname`
```

Note: il est également possible de générer un token pour un autre utilisateur en remplaçant l'option: ``-u `whoami` `` par ``-u `nom_utilisateur` ``

Copier le résultat puis le coller dans le fichier suivant: */etc/u2f_mappings* 

Ensuite, éditer le fichier **/etc/pam.d/common-auth**

Ajouter la ligne suivante:

**auth sufficient\* pam_u2f.so origin=pam://$HOSTNAME appid=pam://$HOSTNAME authfile=/etc/u2f_mappings cue**

**Note: remplacer $HOSTNAME par le nom actuel de votre machine**
\* Sufficient signifie qu'il vous sera demandé d'utiliser la clé seulement si elle est connectée, autrement il faut remplacer *sufficient* par *required*

Mettre à jour PAM:
```bash
sudo pam-auth-update
```

**Tips: Il est fortement recommandé de garder un shell ouvert à côté avec l'identité root avec **sudo -i**, cela permettra de rétablir la configuration par défaut en cas d'erreur.**

**Répéter la manipulation avec la clé admin afin de pouvoir s'authentifier en tant qu'administrateur si jamais l'utilisateur à un problème.**

## Désactivtion du compte root

Il faut également désactiver le compte root. L'utilisateur peut uniquement utiliser sudo pour une élévation de droits.

Pour faire cela, il suffit d'éditer le fichier sudoers après avoir fait une sauvegarde de la configuration initiale.

```bash
sudo cp /etc/sudoers /etc/sudoers.bak
```

Ensuite il faut éditer le fichier sudoers à l'aide de la commande `sudo visudo`:

Dans la section commande aliases du fichier rajouter la ligne suivante:

```bash
Cmnd_Alias DISABLE_SU = /bin/su
```

Ensuite remplacer la ligne: *%sudo ALL=(ALL) ALL* par *%sudo ALL=(ALL) ALL, !DISABLE_SU*. 

Sauvegarder le fichier puis le fermer. 

## Désactivation de la fontion _visudo:_

Pour cela, il faut avoir un utilisateur admin, qui lui gardera accès au fichier en cas de problèmes. Il faut également s'asurer que l'utilisateur administrateur ait toutes les commandes autorisées. La ligne correspondante dans le fichier sudoers devrait ressembler à la suivante: `admin ALL=(ALL:ALL) ALL`

Une fois fait, on édite à nouveau le fichier */etc/sudoers* toujours à l'aide de `sudo visudo` et on rajoute un nouvel alias en précisant le chemin de la fonction *visudo*:

```bash
Cmnd_Alias DIABLE_VISUDO= /sbin/visudo
```

Et on édite à nouveau la ligne du groupe sudo en remplaçant *%sudo ALL=(ALL) ALL* par *%sudo ALL=(ALL) ALL, !DISABLE_SU, !DISABLE_VISUDO*. 

On peut maintenant tester la configuration en se connectant à un utilisateur membre du groupe sudo et entrer la commande `sudo visudo`. Si tout est fonctionnel l'utilisateur recevra le message suivant: "Sorry, user *user* is not allowed to execute /usr/sbin/visudo as root on *name_of_your_machine*"

## Mises à jour de sécurité automatiques:

Dans cette étape le but est d'automatiser les maj de sécurité.

Il faut donc commencer par installer le package nécessaire: 
```bash
sudo apt install unattended-upgrades
```

Ensuite, éditer le fichier suivant:

```bash
vi /etc/apt/apt.conf.d/50unattended-upgrades
```

S'assurer que les deux lignes suivantes soient bien activées et non sous forme de commentaires:

**"origin=Debian,codename=${distro_codename},label=Debian";
"origin=Debian,codename=${distro_codename},label=Debian-Security";**

Ensuite, copier le fichier */usr/share/unattended-upgrades/20auto-upgrades à l'endroit suivant: /etc/apt/apt.conf.d/20auto-upgrades*

```bash
cp /usr/share/unattended-upgrades/20auto-upgrades /etc/apt/apt.conf.d/20auto-upgrades
```

Le fichier est éditable et devrait contenir la configuration suivante:
```bash
APT::Periodic::Update-Package-Lists "1"; 
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::Download-Upgradeable-Packages "1";
```

Une fois fait, on peut donc tester la nouvelle configuration à l'aifr fr la commande suivante:
```bash
unattended-upgrades --debug --dry-run 
```












