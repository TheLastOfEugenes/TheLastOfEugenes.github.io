---
title: 'SQLi: "Tiny SQL"'
description: "Complicated sqli challenge using the alternative database manager: tinysql"
tags:
  - CTF
date: 2026-02-21T17:52
tasks:
  - 
parent files:
  - 
category: challenges
---

# Recherches

On nous donne le code de ce que l'utilisateur a codé pour remplacer SQL. On voit dans ce code une fonction login:
```
def login():
    if request.method == 'GET': return render_template('login.html')

    # user lookup syntax `S:[user]:[pass]#comment`
    # results = [id, user, pass]
    code, results = conn.query('S:' + request.form['user'] +  ':' + request.form['pass'])
    if code == 'e':
        return render_template('500.html'), 500
    if len(results) != 3: return render_template('login.html'), 403
    session['username'] = results[1]
    print (session['username'])
    return redirect('/')
```

Et ensuite on regarde
```
@app.route('/forum/post/<int:post_id>', methods=['GET'])
def forum(post_id):
    if not 'username' in session or session['username'] is None:
        return redirect('/login')
    post_id = post_id % 4
    # id lookup syntax `S:[id]#comment`
    code, author = conn.query('S:' + str(post_id))
    match post_id:
        case 0:
            title = 'how 2 beat da game?'
            description = 'i dont get how 2 beat the game. its so hard'
        case 1:
            title = 'the game makes computer mean??'
            description = 'every time i run the game it says "all your base are belong to us. please send us $40 in cash (bitcoin hasnt been invented yet)" \n\nbtw i downloaded it from the pirate bay if thats relevant.'
        case 2:
            title = 'an analytical discussion on the DOOM franchise'
            description = 'in this essay i will explain the way the playstyle of doom 2 while on the surface similar to the original fails to convey the antiwar story hidden in the subtext.'
        case 3:
            title = 'flag'
            description = FLAG
        case _:
            title = "this should not be seen"
            description = "this should relaly not be sean"
    return render_template('post.html', title=title, desc=description, author=author[1])
```

La clé de tout ça repose sur `conn.query`, une fonction qui repose sur un appel au backend. Tant que l'on a pas le bon résultat on ne pourra pas accéder au post 3. On sait que la bdd utilisée est tinySQL. 

```
pip install git+https://github.com/withmartian/TinySQL@main
```

On tente différentes entrées
```
$
{
}
\
"
`
;
%00
```

# Documentation