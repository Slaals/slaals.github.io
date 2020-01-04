---
layout: article
title: "Servir Bitcoin et Lightning Network en étant anonyme avec Tor"
description: "Vous avez un nœud Bitcoin et Lightning Network, vous participez à la décentralisation du réseau, bravo ! Cependant il réside un point noir, en particulier si votre noeud est localisé chez vous ou votre bureau... En connaissant votre adresse IP, publiée par le noeud, il est possible de vous localiser. On remédie à cela avec Tor."
image: "/assets/img/thumbnail/tor_bitcoind_lnd.png"
image_bg: "/assets/img/thumbnail/tor_bitcoind_lnd.png"
categories: [linux,bitcoin,lnd,tor]
---

<p>
  <b>Dernières modifications le {% include date_localize.html date=page.date %}</b>
</p>

{% include summary.html
  intro="Vous avez un nœud Bitcoin et Lightning Network, vous participez à la décentralisation du réseau, bravo ! Cependant il réside un point noir, en particulier si votre noeud est localisé chez vous ou votre bureau... En connaissant votre adresse IP, publiée par le nœud, il est possible de vous localiser. C'est sensible dans le cas où vous y mettez pour l'équivalent de milliers d'euros en bitcoins, une personne mal intentionnée pourrait vous pirater voire vous cambrioler. Heureusement, grâce à Tor vous allez pouvoir cacher tout cela en un temps record"
sum="<a href='#tor-il-tord'>Qu'est ce que Tor ? Pourquoi Tor ?</a>\\
<a href='#installer-tor'>Installer et activer Tor</a>\\
<a href='#configurer-tor'>Configurer Tor</a>\\
<a href='#connecter-bitcoin-tor'>Connecter votre Bitcoin à Tor</a>\\
<a href='#connecter-lnd-tor'>Connecter votre Lightning à Tor</a>\\
<div>\\
  <a href='#regler-les-droits'>Gérer les droits avec Tor</a>\\
</div>"
%}

_Ce guide est basé sur une installation avec `bitcoind` et `lnd` sur un Debian 9 Stretch  Lite, sur n'importe quelle version Debian 9 ce guide sera fonctionnel._

## <a name="tor-il-tord"></a>Qu'est ce que Tor ? Pourquoi Tor ?

Vous connaissez peut-être Tor indirectement en citant le « Deep Web », mais c'est bien plus que cela. Tor est un logiciel libre qui permet de brouiller les communications Internet (TCP) en construisant un circuit de routage IP. En d'autres termes un émetteur communique avec un récepteur en passant par un chemin incompréhensible, la communication devient intraçable d'un point de vue extérieur.

Autant vous dire que Tor est la solution propice à anonymiser les communications de votre nœud Bitcoin.

>Tor tord le chemin.

## <a name="installer-tor"></a>Installer et activer Tor

On va travailler avec le terminale de commandes, si vous avez la version Lite, vous n'avez que cela sous la main en principe.

Installer Tor se fait avec la commande `apt update && apt install tor`, on met à jour `apt` pour être sûr de télécharger la dernière version des dépendances.

Une fois installé activez le service en tapant `systemctl enable tor` et `systemctl start tor`.

## <a name="configurer-tor"></a>Configurer Tor

On a installé Tor, nous allons le configurer en prévision des usages avec `lnd` et `bitcoind`.

La configuration de Tor se trouve dans le chemin `/etc/tor/torrc` où `torrc` est un fichier de configuration. Connectez-vous en `root` pour le modifier avec `su root`, puis tapez `vim /etc/tor/torrc` ou `nano /etc/tor/torrc` si vous préférez `nano` pour éditer un fichier. Je vous montre les configurations à décommenter (en retirant le # au début de chaque ligne concernée).

```
ControlPort 9051

CookieAuthentication 1

HiddenServiceDir /var/lib/tor/bitcoin/ # Emplacement du dossier de votre routage
HiddenServicePort 8333 127.0.0.1:8333 # Routage bitcoin (8333 est le port mainnet de Bitcoin)
HiddenServicePort 8332 [::1]:8332 # Routage bitcoin ipv6
```

Modifications faites, tapez `systemctl restart tor` pour redémarrer Tor afin qu'il prenne en compte vos modifications.

Et enfin pour voir si Tor est fonctionnel, observez le avec la commande `systemctl status tor@default`, vous devriez avec quelque chose comme suit :

{% include terminal.html title="Voir le status de Tor" prompt="root@raspberry:~ $" commands="systemctl status tor<br />
● tor@default.service - Anonymizing overlay network for TCP<br />
   Loaded: loaded (/lib/systemd/system/tor@default.service; static; vendor preset: enabled)<br />
   Active: active (running) since Wed 2020-01-01 10:09:09 UTC; 7h ago<br />
  Process: 1805 ExecReload=/bin/kill -HUP ${MAINPID} (code=exited, status=0/SUCCESS)<br />
  Process: 2259 ExecStartPre=/usr/bin/tor --defaults-torrc /usr/share/tor/tor-service-defaults-torrc -f /etc/tor/torrc --RunAsDaemon 0 --verify-config (code=exited, status=0/SUCCESS<br />
  Process: 2257 ExecStartPre=/usr/bin/install -Z -m 02755 -o debian-tor -g debian-tor -d /var/run/tor (code=exited, status=0/SUCCESS)<br />
 Main PID: 2262 (tor)<br />
   CGroup: /system.slice/system-tor.slice/tor@default.service<br />
           └─2262 /usr/bin/tor --defaults-torrc /usr/share/tor/tor-service-defaults-torrc -f /etc/tor/torrc --RunAsDaemon 0<br /><br />
Jan 01 10:09:05 raspberrypi tor[2259]: Jan 01 10:09:05.113 [notice] Read configuration file /usr/share/tor/tor-service-defaults-torrc.<br />
Jan 01 10:09:05 raspberrypi tor[2259]: Jan 01 10:09:05.113 [notice] Read configuration file /etc/tor/torrc.<br />
Jan 01 10:09:05 raspberrypi tor[2259]: Configuration was valid<br />
Jan 01 10:09:05 raspberrypi tor[2262]: Jan 01 10:09:05.979 [notice] Tor 0.2.9.16 (git-9ef571339967c1e5) running on Linux with Libevent 2.0.21-stable, OpenSSL 1.1.0j and Zlib 1.2.8.<br />
Jan 01 10:09:05 raspberrypi tor[2262]: Jan 01 10:09:05.979 [notice] Tor can't help you if you use it wrong! Learn how to be safe at https://www.torproject.org/download/download#warn<br />
Jan 01 10:09:05 raspberrypi tor[2262]: Jan 01 10:09:05.979 [notice] Read configuration file /usr/share/tor/tor-service-defaults-torrc.<br />
Jan 01 10:09:05 raspberrypi tor[2262]: Jan 01 10:09:05.979 [notice] Read configuration file /etc/tor/torrc.<br />
Jan 01 10:09:05 raspberrypi tor[2262]: Jan 01 10:09:05.995 [notice] Opening Socks listener on 127.0.0.1:9050<br />
Jan 01 10:09:05 raspberrypi tor[2262]: Jan 01 10:09:05.995 [notice] Opening Control listener on 127.0.0.1:9051<br />
Jan 01 10:09:09 raspberrypi systemd[1]: Started Anonymizing overlay network for TCP." %}

Si votre service Tor n'est pas actif, vous trouverez des informations en tapant `journalctl -u tor@default`.

## <a name="connecter-bitcoin-tor"></a>Connecter votre Bitcoin à Tor

Cette configuration fonctionne avec `bitcoind` en version `0.19.0.1`, dans mon nœud j'utilise l'utilisateur `bitcoin` pour gérer ce service, je vous déconseille d'utiliser `root`.

Avant de commencer vous allez devoir chercher votre nom de domaine Tor en tapant ce qui suit :

{% include terminal.html title="Voir le status de Tor" prompt="bitcoin@raspberry:~ $" commands="cat /var/lib/tor/bitcoin/hostname
blocsiq5epwbye47.onion" %}

`blocsiq5epwbye47.onion` est mon nom de domaine (hostname).

> Si vous voulez savoir comment avoir un domaine qui commence par un mot ou pseudo comme ici "blocs", faites le moi savoir par [email](https://blocs.fr/#contact).

Pour configurer Bitcoin vous allez modifier `.bitcoin/bitcoin.conf` avec les informations suivantes :

```
# Vos configurations
# ...
# Ajoutez ou modifiez ce qui suit

proxy=127.0.0.1:9050 # Proxy sur le service Tor
externalip=blocsiq5epwbye47.onion # à remplacer par votre hostname, c'est l'adresse affichée aux autres nœuds du réseau
```

Redémarrez Bitcoin en tapant `bitcoin-cli stop && bitcoind`.

Félicitations votre nœud est désormais sur le réseau Tor. Il ne vous reste plus qu'à faire de même sur LND afin d'anonymiser nos échanges sur le réseau Lightning Network.

## <a name="connecter-lnd-tor"></a>Connecter votre Lightning à Tor

La configuration de `lnd` est un peu différente de Bitcoin. Sans plus attendre ouvrez `.lnd/lnd.conf` et ajoutez ces différentes configurations :

```
# Vos configurations
# ...
# Ajoutez ou modifiez ce qui suit

listen=localhost # Important pour éviter d'ouvrir votre IP
externalip=blocsiq5epwbye47.onion # Votre hostname Tor
nat=false # A désactiver pour éviter d'associer votre adresse IP à votre nœud

[tor]
tor.active=1
tor.socks=9050
tor.privatekeypath=/var/lib/tor/bitcoin/private_key
```

Redémarrez votre `lnd` en tapant `lncli stop` et `lnd` pour lancer le service.

Il se peut que vous ayez une erreur de droit relative à Tor. Dans la partie suivante on voit comment la solutionner.

### <a name="regler-les-droits"></a>Gérer les droits avec Tor

Vous devez donner accès à différents fichiers de Tor à l'utilisateur qui gère votre `lnd`, dans mon cas mon utilisateur est `bitcoin`.

Tapez la commande `chmod -R 0750 /var/lib/tor`. Cela va donner accès en lecture au groupe `debian-tor` sur le dossier `/var/lib/tor`.

Maintenant ajoutez l'utilisateur `bitcoin` au groupe `debian-tor` en tapant `usermod -a -G debian-tor bitcoin` en étant connecté sur `su root`.

> Tor est bien conçu, il ne nous permet pas d'appliquer les droits qu'on veut à ses dossiers pour limiter de trop les ouvrir. Tapez `journalctl -u tor@default`, si les dernières lignes parlent d'une erreur, cela signifie que votre Tor n'est plus en route. C'est un général dû à une mauvaise configuration (mauvais droits sur les dossiers relatifs à Tor, mauvaises configuration dans `torrc`, etc.)

---

A partir d'ici votre nœud est connecté au réseau en passant par Tor ! Il est toujours intéressant d'observer si tout est bien cadré par Tor afin de ne pas avoir de fuite d'IP. Tapez la commande du terminal ci-dessous :

{% include terminal.html title="Observer les ports d'écoute" prompt="root@raspberry:~ $" commands="netstat -tulpn<br />
(Not all processes could be identified, non-owned process info<br />
 will not be shown, you would have to be root to see it all.)<br />
Active Internet connections (only servers)<br />
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name<br />
tcp        0      0 127.0.0.1:9735          0.0.0.0:*               LISTEN      2043/lnd<br />
tcp        0      0 127.0.0.1:28332         0.0.0.0:*               LISTEN      653/bitcoind<br />       
tcp        0      0 127.0.0.1:8332          0.0.0.0:*               LISTEN      653/bitcoind<br />  
tcp        0      0 127.0.0.1:8333          0.0.0.0:*               LISTEN      653/bitcoind<br />  
tcp        0      0 127.0.0.1:28333         0.0.0.0:*               LISTEN      653/bitcoind<br />  
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      2043/lnd<br />
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -<br />           
tcp        0      0 127.0.0.1:10009         0.0.0.0:*               LISTEN      2043/lnd<br />            
tcp        0      0 127.0.0.1:9050          0.0.0.0:*               LISTEN      -<br />           
tcp        0      0 127.0.0.1:9051          0.0.0.0:*               LISTEN      -<br />                  
tcp6       0      0 ::1:8332                :::*                    LISTEN      653/bitcoind<br />
tcp6       0      0 :::80                   :::*                    LISTEN      -<br />      
tcp6       0      0 :::22                   :::*                    LISTEN      -<br />                   
udp        0      0 0.0.0.0:48138           0.0.0.0:*                           -<br />                   
udp        0      0 0.0.0.0:68              0.0.0.0:*                           -<br />                   
udp        0      0 0.0.0.0:5353            0.0.0.0:*                           -<br />                   
udp6       0      0 :::45175                :::*                                -<br />                   
udp6       0      0 :::5353                 :::*                                -" %}

Nous, ce qui nous intéresse ce sont `bitcoind` et `lnd`. On va passer tous les ports d'écoute en revue et comprendre si oui ou non ils sont "torrifiés" :
- `lnd 9735` : C'est le port de LND, "torrifiée" avec le clé privée de votre Tor ;
- `bitcoind 28332` : C'est le port [ZMQ](https://zeromq.org/) de Bitcoin, il sert à la configuration et n'est pas distribué ;
- `bitcoind 8333` : C'est le port de `bitcoind` qui est caché via Tor (HiddenServicePort dans `/etc/tor/torrc`) ;
- `bitcoind 8332` : C'est le port JSON-RPC de `bitcoind`, nul besoin de le cacher avec Tor si vous ne le distribuez pas ;
- `bitcoind 8332 IPV6` : C'est le port ipv6 de `bitcoind` qui est caché via Tor (HiddenServicePort dans `/etc/tor/torrc`) ;

Enfin pour voir si `lnd` a bien considéré vos modifications, tapez `lncli getinfo`, vous obtenez un JSON comme suit :

```
{
	"version": "0.8.0-beta commit=v0.8.1-beta",
	"identity_pubkey": "03a2c34daf010b3501daf704b5a321e82e1631421b7ffa18dd49014967eda82dc9",
	"alias": "blocs.fr",
	"color": "#00aaff",
	"num_pending_channels": 0,
	"num_active_channels": 6,
	"num_inactive_channels": 0,
	"num_peers": 6,
	"block_height": 610817,
	"block_hash": "0000000000000000000ff1cf20075afd0df2a52288edd20a7d1cd8a51ca3e32a",
	"best_header_timestamp": 1577900732,
	"synced_to_chain": true,
	"synced_to_graph": true,
	"testnet": false,
	"chains": [
		{
			"chain": "bitcoin",
			"network": "mainnet"
		}
	],
	"uris": [
		"03a2c34daf010b3501daf704b5a321e82e1631421b7ffa18dd49014967eda82dc9@blocsiq5epwbye47.onion:9735"
	]
}
```

Intéressez vous à la partie d'en bas, `"uris"`, qui est une liste des adresses distribuées dans le réseau. Ici je n'ai qu'une adresse qui est un .onion, voir ce qui suit le "@". La partie de gauche est la clé publique de mon lnd.

> .onion est un nom de domaine exclusif au routage en oignon comme le fait Tor. Comme vous pouvez l'imaginer, c'est réellement une référence à la plante. Cela vient du routage qui recouvre le message de plusieurs couches, comme un oignon où le coeur est enrobé de plusieurs couches. [MrBidouille en a fait une parfaite vidéo introductive](https://www.youtube.com/watch?v=zjqw1VnuKLs).

Vérifications faites, vous êtes cachés, bravo !

Merci à [Clint](https://twitter.com/clint_network) pour ses infos sur Tor et [JohnOnChain](https://twitter.com/JohnOnChain) pour m'avoir aidé à la configuration de lnd avec Tor.

{% include comments.html %}
