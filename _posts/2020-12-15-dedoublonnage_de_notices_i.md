---
layout: post
title: Dédoublonnage de notices bibliographiques (1ère partie)
category: [LIS Advent Calendar]
tags: [Python, Algorithme, Data Matching, Dédoublonnage]
published: true
comments: true
--- 

Comme premier algorithme de dédoublonnage, nous allons voir les
[*bibliographic hash
key*](https://verbundwiki.gbv.de/display/VZG/Bibliographic+Hash+Key)
qui s'inspire de l'[*interhash* de
BibSonomy](https://www.bibsonomy.org/help_en/InterIntraHash). 

L'idée est de pouvoir générer une signature identifiant une notice sur
base de quelques propriétés :

* Le titre ;
* L'auteur, ou l'éditeur scientifique ;
* L'année de publication. 

Ces propriétés devraient être présentes dans la majorité des notices.

Voici une implémentation en Python de ce *BibHash* :

```python
import hashlib
import regex as re

from unicodedata import normalize


class BibHash:
    def __init__(self, title="", author="", editor="", year=""):
        self.title = title
        self.author = author
        self.editor = editor
        self.year = year

    def level0(self):
        title = normalize("NFKC", self.title)
        author = normalize("NFKC", self.author)
        editor = normalize("NFKC", self.editor)
        year = normalize("NFKC", self.year)

        title = re.sub(r"[^0-9\p{L}]+", "", title).lower()
        year = re.sub(r"[^0-9]+", "", year)

        author = author if re.match(r"[0-9\p{L}]", author) else editor

        author = re.sub(r"[^0-9\p{L}\. ]+", "", author)
        author = author.strip()
        author = re.sub(r" +and +(and +)*", " and ", author)

        persons = [
            self.normalize_person(p) for p in re.split(r" and ", author)
        ]

        if persons:
            author = f"[{','.join(sorted(persons))}]"
        else:
            author = ""

        return f"{title} {author} {year}"

    def level1(self):
        return hashlib.md5(f"1{self.level0()}".encode()).hexdigest()

    def normalize_person(self, name):
        name = name.strip()

        if not name:
            return ""

        tokens = name.lower().split()
        first = tokens[0]
        last = tokens[-1]

        if first == last:
            return first

        return f"{first[0]}.{last}"
```

Et voici un exemple d'utilisation de ce code :

```python
#!/usr/bin/env python

from bibhash import BibHash


if __name__ == "__main__":
    book1 = {
        "title": "Le nom de la rose",
        "author": "Umberto Eco",
        "year": "1982"
    }

    book2 = {
        "title": "Nom de la rose (Le)",
        "author": "Eco, Umberto",
        "year": "1982"
    }

    book3 = {
        "title": "Le nom de la rose",
        "author": "U. Eco",
        "year": "1982"
    }

    book4 = {
        "title": "Schismatrice +",
        "author": "Bruce Sterling",
        "year": "1985"
    }

    book1_key = BibHash(**book1)
    book2_key = BibHash(**book2)
    book3_key = BibHash(**book3)
    book4_key = BibHash(**book4)

    print(book1,f"""
    → BibHash Level0: 
        {book1_key.level0()}
              Level1: 
        {book1_key.level1()}""")
    print(book2, f"""
    → BibHash Level0: {
        book2_key.level0()}
              Level1: 
        {book2_key.level1()}""")
    print(book3, f"""
    → BibHash Level0: 
        {book3_key.level0()}
              Level1: 
        {book3_key.level1()}""")
    print(book4, f"""
    → BibHash Level0: 
        {book4_key.level0()}
              Level1: 
        {book4_key.level1()}""")

```

Dont voici le résultat :

```text
{'title': 'Le nom de la rose', 'author': 'Umberto Eco', 'year': '1982'} 
    → BibHash Level0: 
        lenomdelarose [u.eco] 1982
              Level1: 
        9ba38341ae099d005cf5aa5afafe686b
{'title': 'Nom de la rose (Le)', 'author': 'Eco, Umberto', 'year': '1982'} 
    → BibHash Level0: 
        nomdelarosele [e.umberto] 1982
              Level1: 
        46ef698528c7820f19a3df2c8084464d
{'title': 'Le nom de la rose', 'author': 'U. Eco', 'year': '1982'} 
    → BibHash Level0: 
        lenomdelarose [u.eco] 1982
              Level1: 
        9ba38341ae099d005cf5aa5afafe686b
{'title': 'Schismatrice +', 'author': 'Bruce Sterling', 'year': '1985'} 
    → BibHash Level0: 
        schismatrice [b.sterling] 1985
              Level1: 
        c2b4d4fa42a9e39a01a4ceeb44e34e97
```

Comme vous pouvez le voir, il y a des avantages :

* Les BibHash de niveau 1, somme de contrôle MD5 du BibHash de niveau
  0, a une taille fixe, ce qui permet un stockage de taille fixe dans
  une base de données ;
* Les deux niveaux des BibHash sont faciles à calculer. 

Mais nous pouvons également voir que les règles de formatage du titre
ou des auteurs influence le résultat final, et génère ainsi des faux
négatifs. Mais ce problème devrai être assez limité au sein d'un même
catalogue, pour autant que les règles soient appliquées de manière
cohérente. Il faut aussi signaler que si les données sont extraites
d'un catalogue, il est alors possible de normaliser en amont les
données avant de les soumettre à l'algorithme. 
