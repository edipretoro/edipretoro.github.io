---
layout: post
title: Améliorer les données de son catalogue
category: [LIS Advent Calendar]
tags: [Python, Qualité des données, SQLAlchemy]
published: true
comments: true
--- 

Dans ce billet, nous allons explorer une méthode assez rapide pour
améliorer les données présentes dans son catalogue, et plus
particulièrement les autorités auteurs. L'idée est de récupérer tous
les auteurs présents dans son catalogue, et de voir s'il n'y a pas des
doublons. Le billet s'arrêtera à cette étape, mais il va de soi que le
vrai travail commence une fois que le constat a été fait : il faudra
modifier les notices dans le SIGB. 

## Récupérer les données

J'ai choisi de travailler une nouvelle fois avec
[SQLAlchemy](https://www.sqlalchemy.org/) et
[Python](https://www.python.org/) pour abstraire l'accès aux
données. Voici notre ORM :

```python
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base


PMBOrm = declarative_base()


class Author(PMBOrm):
    __tablename__ = "authors"
    id = Column("author_id", Integer, primary_key=True)
    last_name = Column("author_name", String)
    first_name = Column("author_rejete", String)

    @property
    def full_name(self):
        return f"{self.last_name}, {self.first_name}"
```

Et voici le script qui utilise l'ORM et qui construit notre jeu de
données : 

```python
#!/usr/bin/env python

import json
import os
import pathlib

from dotenv import find_dotenv, load_dotenv

import isbnlib

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from pmb_orm import Author


if __name__ == "__main__":
    load_dotenv(find_dotenv())
    e = create_engine(os.getenv("DB_DSN"))
    PMB = sessionmaker(bind=e)
    session = PMB()

    dataset = {}
    records = session.query(Author).all()
    for record in records:
        dataset[record.full_name] = {
            "id": record.id,
            "last_name": record.last_name,
            "first_name": record.first_name
        }

    pathlib.Path("authors.json").write_text(
        json.dumps(
            dataset,
            indent=2,
            ensure_ascii=False
        ), 
        encoding="utf-8"
    )
```

Ce qui nous produit un fichier JSON qui ressemble à :

```json
{
  "Borloo, Jean-Pierre": {
    "id": 1,
    "last_name": "Borloo",
    "first_name": "Jean-Pierre"
  },
  "Vandermeersch, Damien": {
    "id": 2,
    "last_name": "Vandermeersch",
    "first_name": "Damien"
  },
  "Kroll, Pierre": {
    "id": 3,
    "last_name": "Kroll",
    "first_name": "Pierre"
  },
  .
  .
  .
}
```

L'*id* correspond à la clé primaire dans la table *authors*, et me
permettra de retrouver facilement les auteurs problématiques si je
devais faire les modifications dans le catalogue. 

## Traiter les données

Il ne nous reste plus qu'à traiter ces données. 

Voici le script :

```python
#!/usr/bin/env python

import itertools
import json
import pathlib
import string
import re

import unidecode


def fingerprint(value):
    value = value.lower().translate(str.maketrans('', '', string.punctuation))
    value = re.sub(r'[\x00-\x1f\x7f-\x9f]', '', value)
    value = unidecode.unidecode(value)
    value = sorted(set(value.strip().split()))
    return " ".join(value)


if __name__ == "__main__":
    # Initialisation de la variable qui contiendra les potentiels doublons
    blocking_authors = {}
    
    # Chargement des données
    authors =
    json.loads(pathlib.Path("authors.json").read_text(encoding="utf-8"))
    
    # On affiche le nombre total d'auteurs dans le jeu de données
    print(f"We have {len(authors)} authors in our catalog.")
    
    # Pour chaque auteur, on va calculer une « signature » qui
    # permettra d'identifier les doublons potentiels ; les auteurs
    # avec la même signature sont ainsi rassemblés
    for author, data in authors.items():
        blocking_authors.setdefault(fingerprint(author), []).append({**data})

    # On initialise un compteur
    c = itertools.count()
    
    # On parcourt la liste des doublons potentiels, et si une
    # signature a plus d'un auteur, on incrémente le compteur, et si
    # ce n'est pas le cas, on supprime l'entrée car il ne s'agit pas
    # d'un doublon
    for fingerprint, authors in blocking_authors.copy().items():
        if len(authors) > 1:
            next(c)
        else:
            del blocking_authors[fingerprint]

    # On affiche le nombre de doublons
    print(f"We have {next(c)} authors possibly duplicated.")
    
    # On affiche un fichier JSON avec les doublons
    print(json.dumps(blocking_authors, indent=2, ensure_ascii=False))
```

Le script n'est pas nécessairement très long, ni très compliqué. Ce
qui est intéressant, c'est la fonction qui calcule la signature,
l'empreinte d'un auteur. Le but de cette fonction est de provoquer des
chaînes identiques sur base de valeurs légèrement différentes. 

Un outil qui fait un excellent travail à ce niveau-là est
[OpenRefine](https://openrefine.org/) et l'algorithme de cette
fonction est justement une implémentation de son
[algorithme](https://github.com/OpenRefine/OpenRefine/wiki/Clustering-In-Depth#fingerprint) : 

* Transformer en minuscule et supprimer les caractères de ponctuation
  → `value = value.lower().translate(str.maketrans('', '',
  string.punctuation))`
* Supprimer les caractères de contrôle → `value =
  re.sub(r'[\x00-\x1f\x7f-\x9f]', '', value)`
* Normaliser les caractères UNICODE dans leur version ASCII → `value =
  unidecode.unidecode(value)` 
* Enlever les espaces en début et fin de chaîne de caractère, découper
  la cette dernière en pièce sur base des espaces, les trier et les
  dédoublonner → `value = sorted(set(value.strip().split()))`
* Joindre les pièces avec un espace de manière à avoir une chaîne de
  caractère. 
  
Et voici le résultat de ce script : 

```text
We have 21173 authors in our catalog.
We have 23 authors possibly duplicated.
{
  "edmond marc": [
    {
      "id": 881,
      "last_name": "Edmond",
      "first_name": "Marc"
    },
    {
      "id": 14280,
      "last_name": "Marc",
      "first_name": "Edmond"
    }
  ],
  .
  .
  .
  "asbl communes de des et union villes wallonie": [
    {
      "id": 1887,
      "last_name": "Union des villes et communes de Wallonie (asbl)",
      "first_name": "-"
    },
    {
      "id": 9362,
      "last_name": "Union des villes et des communes de Wallonie asbl",
      "first_name": ""
    }
  ],
  "m pierre wolf": [
    {
      "id": 1990,
      "last_name": "M. Wolf",
      "first_name": "Pierre"
    },
    {
      "id": 3671,
      "last_name": "Wolf",
      "first_name": "Pierre M."
    }
  ],
  "ajuriaguerra de j": [
    {
      "id": 7143,
      "last_name": "de Ajuriaguerra",
      "first_name": "J."
    },
    {
      "id": 14267,
      "last_name": "Ajuriaguerra",
      "first_name": "J. de"
    }
  ],
  "conference document et numerique societe": [
    {
      "id": 16576,
      "last_name": "Conférence document numérique et société",
      "first_name": ""
    },
    {
      "id": 17358,
      "last_name": "Conférence Document numérique et société",
      "first_name": ""
    }
  ],
  "de federation la ministere walloniebruxelles": [
    {
      "id": 18049,
      "last_name": "Ministère de la fédération Wallonie-Bruxelles",
      "first_name": ""
    },
    {
      "id": 19727,
      "last_name": "Ministère de la Fédération Wallonie-Bruxelles",
      "first_name": ""
    }
  ],
  "archives des journee": [
    {
      "id": 20183,
      "last_name": "Journée des Archives",
      "first_name": ""
    },
    {
      "id": 20137,
      "last_name": "Journée des archives",
      "first_name": ""
    }
  ]
}

```

J'ai édité le fichier JSON produit de manière à avoir un résultat
représentatif et, surtout, lisible. Vous pouvez voir les empreintes
des auteurs, par exemple *archives des journee* ou *ajuriaguerra de
j*. 

Mais donc, n'avoir que 23 doublons potentiels sur 22173 auteurs, c'est
pas mal du tout ! D'autant plus que ce ne sont que des doublons
potentiels ! Il est plausible d'avoir à la fois un Marc Edmond et un
Edmond Marc. Pour le reste, ce sont des erreurs assez classiques :-)

Ce processus peut évidemment être adapté à chacune des autorités
de votre catalogue. 

Et voilà, il ne reste plus qu'à modifier ces notices d'autorités :p
