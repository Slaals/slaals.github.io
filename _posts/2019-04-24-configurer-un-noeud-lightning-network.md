---
layout: article
title: "Configurer son nœud Bitcoin Lightning Network"
description: "Dans l'article précédent j'ai écrit au sujet de l'installation d'un nœud Bitcoin Lightning Network avec un Raspberry Pi. Ce dernier est sommaire et va à l'essentiel. Dans cet article il va s'agir d'optimiser la configuration du Raspberry Pi afin d'améliorer sa sécurité et rendre l'administration moins laborieuse."
image: "/assets/img/thumbnail/conf_node_title.jpg"
image_bg: "/assets/img/thumbnail/conf_node.jpg"
---

{% include summary.html intro="<a href='/articles/installer-un-noeud-lightning-network' target='_blank'>Dans l'article précédent</a> j'ai écrit au sujet de l'installation d'un nœud Bitcoin Lightning Network avec un Raspberry Pi. Ce dernier est sommaire et va à l'essentiel. Dans cet article il va s'agir d'optimiser la configuration du Raspberry Pi afin d'améliorer sa sécurité et rendre l'administration moins laborieuse."
sum="<a href='#fixer-ip'>Fixer l'adresse IP locale du nœud</a>\\
<a href='#securiser-ssh'>Sécuriser les accès distants SSH</a>\\
<a href='#enregistrer-cle-ssh'>Enregistrer votre clé pour les connexions futures</a>\\
<a href='#fail2ban'>Ajouter une couche supplémentaire à l'authentification SSH</a>\\
<a href='#activer-redemarrage-auto-des-services'>Activer le redémarrage automatique des services</a>\\
<div>\\
  <a href='#bitcoind'>bitcoind</a>\\
  <a href='#lnd'>lnd</a>\\
</div>"
%}

_Notez que tout ce qui va suivre devra se faire avec l'utilisateur administrateur. Pour s'y connecter tapez une fois `su root` puis tapez son mot de passe._

## <a name="fixer-ip"></a>Fixer l'adresse IP locale du nœud

_Ce qui suit s'applique qu'aux connexions réseau filaire et non Wifi_

Votre nœud possède une adresse IP locale afin d'être accéssible dans votre réseau, c'est d'ailleurs grâce à cette dernière que vous pouvez y accéder avec SSH. Cependant il est fort probable que son adressage se fasse dynamiquement par défaut, par conséquent l'adresse de votre Raspberry pourrait changer. Il est préférable d'éviter cette inconvenance alors on va forcer une adresse IP statique.

Ouvrez le fichiers de configuration DHCP en tapant `nano /etc/dhcpcd.conf`, ajoutez à la fin du fichier ces informations :

```
interface eth0
static ip_address=192.168.1.42/24 # C'est l'IP qu'aura la machine
static routers=<Votre passerelle> # Passerelle, explications plus bas
static domain_name_servers=<Votre passerelle> 8.8.8.8
```

Ici vous allez devoir faire un travail d'enquête pour connaître votre réseau et savoir quelle est la passerelle. Suivez le terminal ci-dessous :

{% include terminal.html title="Trouver l'adresse IP de la passerelle" prompt="pi@raspberrypi:~ $" commands="ip route<br/>default via <span class='term-highlight'>192.168.1.1</span> dev eth0 src 192.168.1.42 metric 202<br/>
192.168.1.0<span class='term-highlight'>/24</span> dev eth0 proto kernel scope link src 192.168.1.42 metric 202
" %}

Trouvé ! `192.168.1.1` est la passerelle de mon réseau, c'est ce que vous devez mettre dans la configuration DHCP plus haut. Notez également le `/24` qui est un masque de réseau et que vous devez aussi utiliser dans votre configuration DHCP.

> Sans entrer dans les détails le masque `/24` (ou `/n`) est ce qui va déterminer la plage d'adresses disponibles. Avec une adresse type `192.168.1.0/24` vous avez 254 adresses disponibles allant de `192.168.1.1` à `192.168.1.254`. C'est important à considérer car cela signifie que vous ne pouvez pas mettre n'importe quelle adresse, il est important de respecter le format. Pour être sûr de ne pas vous tromper, conservez les 3 premiers chiffres de l'IP et choisissez le dernier : `x.x.x.y` où "x" sont les valeurs de base de l'IP (`192.168.1` selon l'exemple) et "y" est une valeur choisie arbitrairement et comprise entre 1 et 254.

Votre nœud a maintenant une IP statique.

## <a name="securiser-ssh"></a>Sécuriser les accès distants SSH

SSH est un protocole de réseau qui vous permet de vous connecter à distance sur une machine grâce à un canal chiffré. Cela signifie que la machine distante exécute un service qui reçoit les connexions SSH, ce service s'appelle `sshd`. SSH en lui même est plutôt bien sécurisé, cependant il est possible de couvrir un peu mieux les accès.

Sans plus attendre trouvez le fichier en tapant `sudo nano /etc/ssh/sshd_config`.

Voyons quelles modifications vous pouvez apporter afin de mieux sécuriser les accès SSH. Vous noterez que la plupart des lignes sont commentées grâce au caractère "#", cela signifie juste que les lignes comportant ce symbole au début sont neutralisées, le travail consiste essentiellement à dé-neutraliser certaines lignes.

`Port 2310` : Mettez le port que vous souhaitez, le changer est toujours une couche de sécurité peu coûteuse. Ici j'ai mis 2310.

`LoginGraceTime 2m` : Limite le temps de connexion inactif à 2 minutes. Vous pouvez l'augmenter mais le plus bas le mieux.

`StrictModes yes` : Assure les accès aux fichiers en fonction de l'utilisateur connecté.

`PermitRootLogin no` : Mettez PermitRootLogin à `no`, cette option bloquera les connexions SSH à l'utilisateur administrateur du Raspberry. Si vous avez suivi [l'installation que je recommande](https://blocs.fr/articles/installer-un-nœud-lightning-network) vous avez un utilisateur nommé `bitcoin` qui a pour rôle d'administrer les services bitcoind et lnd. C'est sur ce dernier que vous vous connecterez via SSH et non sur l'utilisateur root.

`MaxAuthTries 3` : Nombre d'essais pour se connecter via SSH limité à 3. Cela bloque les attaques par brute force automatisé.

`MaxSessions 1` : Si vous êtes la seule personne à gérer votre machine, et que vous ne comptez pas créer plusieurs connexions SSH simultanées, alors limitez le nombre de sessions à 1. Il faut garder ce nombre le plus petit possible en fonction de vos besoins.

`PubkeyAuthentication yes` : Cette option vous servira plus tard. Elle n'ajoute pas de sécurité supplémentaire, c'est une option qui vous facilitera l'accès à votre machine.

Tapez CTRL + O et CTRL + X; et enfin redémarrez SSHd afin d'appliquer ces modifications en tapant `sudo service sshd restart`.

### <a name="enregistrer-cle-ssh"></a>Enregistrer votre clé pour les connexions futures

Dans la configuration précédente vous aviez activé l'option PubkeyAuthentication, cette option va vous permettre d'enregistrer la clé publique de votre machine sur le Raspberry. Cela vous évitera de taper sans cesse le mot de passe de connexion.

Cela se passe en un commande. Suivez le terminal ci-dessous. Notez que cela doit se faire depuis votre machine locale (votre mac, votre pc, etc). Si vous êtes sur Windows vous n'aurez pas cette commande.

{% include terminal.html title="Enregistrer se clé publique SSH" prompt="monpc@oumonmac:~ $" commands="ssh-copy-id -i ~/.ssh/id_rsa.pub bitcoin@192.168.1.42" %}

Ensuite vous devrez taper le mot de passe de l'utilisateur "bitcoin". Notez que l'IP `192.168.1.42` ([comment trouver l'IP de mon nœud ?](/articles/installer-un-noeud-lightning-network#ifconfig)) doit être l'IP de votre nœud distant. Cette action aura pour effet de recenser la clé publique de votre machine dans le Raspberry Pi, il vous reconnaîtra pour les connexions futures et ne vous demandera plus de mot de passe.

> Si vous êtes sur Windows en utilisant PuTTy, vous pouvez générer et trouver votre clé publique avec [PuTTyGen](https://www.digitalocean.com/docs/droplets/how-to/add-ssh-keys/create-with-putty/). Copiez cette clé qui commence par "ssh-rsa" si vous l'avez créé avec le chiffrement RSA. Cliquez sur "Save private key" et enregistrez le fichier sur votre machine. Revenez sur PuTTy, sur le panneau de gauche trouvez "Connection / SSH / Auth" puis dans la section "Private key file for authentication" chargez le fichier de votre clé privée, c'est un fichier .ppk. Connectez vous à votre Raspberry en utilisant PuTTy. Une fois connecté tapez `sudo nano .ssh/authorized_keys` et collez la clé publique que vous aviez copié, puis sauvegardez le fichier. C'est terminé désormais votre Raspberry connaît votre clé publique et autorisera les connexions futures depuis votre machine Windows.

## <a name="fail2ban"></a>Ajouter une couche supplémentaire à l'authentification SSH

Pour l'instant il est toujours possible de forcer l'accès à votre nœud avec une attaque par dictionnaire, qui consiste à tester des millions de mot de passe jusqu'à trouver un accès. Il existe un service nommé `fail2ban` qui va permettre de protéger en limitant les tentatives de connexion.

Pour l'installer tapez `apt-get update && apt-get install -y fail2ban`. Ensuite vous allez devoir configurer une chose si vous avez modifié le port SSH comme je l'ai montré [juste au dessus](#securiser-ssh). Cherchez le fichier de configuration de fail2ban `nano /etc/fail2ban/jail.d/defaults-debian.conf`. Voyez la section `[sshd]` mettez ce qui suit (si vous n'avez rien dans le fichier, ajoutez également ce qui suit) :

```
[sshd]
enabled = true
port    = 2310 # Notez que c'est le port que j'ai mis plus haut dans sshd_config
```

Sauvegardez en tapant CTRL + O et CTRL + X pour fermer et enfin tapez `systemctl restart fail2ban`.

> fail2ban est un service polyvalent qui sécurise nombreuse authentifications, dont le service sshd et nous l'utilisons pour ça.

## <a name="activer-redemarrage-auto-des-services"></a>Activer le redémarrage automatique du montage du disque, de bitcoind et lnd

Cette partie est cruciale. Si votre Raspberry est redémarré à cause d'une coupure de courant par exemple, il redémarrera... Mais il ne relancera pas bitcoind ni lnd. Ce qui aurait pour effet d'éteindre le service et vous obliger à intervenir pour qu'il persévère.

D'abord il va falloir faire en sorte que le disque soit toujours monté sur le dossier .bitcoin car c'est ce dernier qui recense la blockchain et bitcoind en a besoin. Le montage de disque ne se fait jamais automatiquement. Alors voilà comment configurer cela :

{% include terminal.html title="Activer l'automount" prompt="pi@raspberry:~ $" commands="df -h<br />Filesystem      Size  Used Avail Use% Mounted on<br />
/dev/root        15G  5.6G  8.4G  40% /<br />
devtmpfs        460M     0  460M   0% /dev<br />
tmpfs           464M     0  464M   0% /dev/shm<br />
tmpfs           464M   12M  452M   3% /run<br />
tmpfs           5.0M  4.0K  5.0M   1% /run/lock<br />
tmpfs           464M     0  464M   0% /sys/fs/cgroup<br />
/dev/mmcblk0p1   44M   22M   22M  51% /boot<br />
<span class='term-highlight'>/dev/sda        458G  253G  182G  59% /home/bitcoin/.bitcoin</span><br />
tmpfs            93M     0   93M   0% /run/user/1001\\
echo '/dev/sda /home/bitcoin/.bitcoin ext4 defaults,noatime 0 0' >> /etc/fstab
" %}

Soyez vigilant ici. J'ai surligné mon montage actuel qui est `/dev/sda` monté sur `/home/bitcoin/.bitcoin`, vous devez mettre ce chemin dans le `echo` en dessous. Notez également l'annotation `ext4` qui est le système de fichier que j'ai choisi lorsque j'avais formaté le disque. Si vous avez un doute suivez le terminal ci-dessous

{% include terminal.html title="Trouver le système de fichier d'un disque" prompt="pi@raspberry:~ $" commands="file -sL /dev/sda<br />/dev/sda: Linux rev 1.0 <span class='term-highlight'>ext4</span> filesystem data, UUID=a2a0ec66-f768-4cf0-9442-6c2878ab60e1, volume name BLOCK-STORAGE (needs journal recovery) (extents) (64bit) (large files) (huge files)" %}

Où `/dev/sda` est mon disque à monter. On voit en surligné `ext4` qui est le système de fichier.

### bitcoind

Le redémarrage automatique d'un service sur Linux se fait toujours de la même manière, il faut écrire le service. Et pour ce faire tapez `nano /etc/systemd/system/bitcoind.service` et entrez la configuration suivante :

```
[Unit]
Description=Bitcoin daemon
After=network.target

[Service]
ExecStartPre=/bin/sh -c 'sleep 30'
ExecStart=/usr/local/bin/bitcoind -daemon -conf=/home/bitcoin/.bitcoin/bitcoin.conf -pid=/home/bitcoin/.bitcoin/bitcoind.pid # Mettez le chemin vers votre dossier .bitcoin
PIDFile=/home/bitcoin/.bitcoin/bitcoind.pid # Mettez le chemin vers votre dossier .bitcoin
User=bitcoin # Votre utilisateur qui gère le service
Group=bitcoin # Votre groupe qui gère le service
Type=forking
KillMode=process
Restart=always
TimeoutSec=120
RestartSec=30

[Install]
WantedBy=multi-user.target
```
CTRL + O et CTRL + X pour enregistrer et quitter puis tapez `systemctl enable bitcoind`. La machine est lancée, le service bitcoind démarrera au lancement du Raspberry.

### lnd

Encore une fois, il faut créer un service pour le service de lightning network, tapez `nano /etc/systemd/system/lnd.service` et voilà la configuration que je recommande :

```
[Unit]
Description=LND
Wants=bitcoind.service
After=bticoind.service # Il démarre après bitcoind

[Service]
ExecStart=/usr/local/bin/lnd

User=bitcoin # Votre utilisateur qui gère le service
Group=bitcoin # Votre utilisateur qui gère le service
Type=simple
KillMode=process
LimitNOFILE=128000
TimeoutSec=300
Restart=always
RestartSec=120

[Install]
WantedBy=multi-user.target
```

Et enfin pour l'activer `systemctl enable lnd`.

---

Félicitations, votre Raspberry est autonome, il relancera les services en cas de redémarrage inopiné, sans votre aide, tout seul comme un grand.

Dans un article futur on va voir comment on peut administrer son nœud avec une interface graphique !
