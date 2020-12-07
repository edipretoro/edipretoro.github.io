---
layout: post
title: Extraire des données de son SIGB
category: [LIS Advent Calendar]
tags: [Python, PMB, SQLAlchemy]
published: true
comments: true
--- 

Certains SIGB mettent à disposition un *Software Development Kit*
(SDK) ou des *Application Programming Interface*s (API) pour accéder
programmatiquement aux données présentes dans la base de données du
logiciel. C'est le cas, par exemple, pour Vubis ou Adlib. Mais souvent,
ces SIGB s'appuient sur un *Système de gestion de bases de données*
(SGBD) et il est souvent possible d'accéder à ce dernier pour extraire
des données. Et autant dire l'évidence, mais il vaut mieux se limiter
à de l'extraction de données, et pas de l'écriture directe dans la
base de données. En effet, sans avoir une connaissance exhaustive de
la logique de l'application, il est très difficile de savoir si la
simple écriture d'une donnée dans une colonne « suffit » pour le
SIGB. Dans l'article d'aujourd'hui, je vais vous expliquer comment
j'ai extrait les données nécessaires à la mise en place du système de
recommandation de lectures. 

## Vive la documentation !

La première étape consiste à prendre le temps de connaître son SIGB,
et d'en lire sa documentation technique. Dans mon cas, je travaillais
sur un [PMB](https://www.sigb.net), et la base de données est
documentée. J'ignore si cette documentation est à jour, mais elle m'a
suffit.

## Choix des outils

La deuxième étape est de voir quels sont les outils disponibles pour
extraire les données. Dans le cas d'un PMB, je peux utiliser des
requêtes SQL et écrire un script qui traitera les résultats de ces
requêtes. C'est assez simple à mettre en place, mais il y a malgré
tout un défaut : dans trois mois, quand je relirai le code de la
requête, j'ai intérêt à avoir toutes les subtilités du schéma de PMB
en mémoire, sinon, cela va être la galère.

Un autre outil disponible quand on travaille avec des bases de données
relationnelles, c'est l'utilisation d'*Object-relational mapping*
(ORM) qui va abstraire le schéma de la base de données sous forme
d'objets. Alors, comme tout outil, l'ORM a ses avantages et ses
inconvénients. Je ne prétendrai pas que l'ORM est plus rapide qu'une
requête écrite à la main dans ma situation, je n'ai pas réalisé de
*benchmarks*. Mais dans ma situation, je veux un code qui soit facile
à relire dans trois mois, et qui puisse éventuellement évoluer. 

En [Python](https://www.python.org/), un ORM assez connu est
[*SQLAlchemy*](https://www.sqlalchemy.org/), c'est celui que nous
utiliserons. 

## Création des classes nécessaires pour extraire les données

Voici le code pour extraire les données nécessaires au projet d'hier : 

```python
from sqlalchemy import (
    Column, Integer, String, DateTime, Date, Text, Float, ForeignKey
)
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from sqlalchemy.orm.session import object_session


PMBOrm = declarative_base()


class Record(PMBOrm):
    __tablename__ = "notices"
    id = Column("notice_id", Integer, primary_key=True)
    main_title = Column("tit1", String)
    parallel_title = Column("tit3", String)
    other_title = Column("tit4", String)
    isbn = Column("code", String)
    copies = relationship("Copy")


class Copy(PMBOrm):
    __tablename__ = "exemplaires"
    id = Column("expl_id", Integer, primary_key=True)
    notice = Column("expl_notice", Integer, ForeignKey("notices.notice_id"))
    barcode = Column("expl_cb", String)


class Borrowed(PMBOrm):
    __tablename__ = "pret_archive"
    id = Column("arc_id", Integer, primary_key=True)
    borrower = Column(
        "arc_id_empr",
        Integer,
        ForeignKey("empr.id_empr")
    )
    copy = Column(
        "arc_expl_id",
        Integer,
        ForeignKey("exemplaires.expl_id"),
        primary_key=True
    )


class Borrowing(PMBOrm):
    __tablename__ = "pret"
    borrower = Column("pret_idempr", Integer, ForeignKey("empr.id_empr"))
    copy = Column(
        "pret_idexpl",
        Integer,
        ForeignKey("exemplaires.expl_id"),
        primary_key=True
    )


class Patron(PMBOrm):
    __tablename__ = "empr"
    id = Column("id_empr", Integer, primary_key=True)
    barcode = Column("empr_cb", String)
    last_name = Column("empr_nom", String)
    first_name = Column("empr_prenom", String)
    borrowing = relationship("Borrowing")
    borrowed = relationship("Borrowed")

    def full_name(self):
        return f"{self.last_name}, {self.first_name}"

    def borrowed_titles(self):
        session = object_session(self)
        b = set(
            [loan.copy for loan in self.borrowed]
            +
            [loan.copy for loan in self.borrowing]

        )

        titles = []
        for copy in b:
            results = session.query(
                Copy, Record
            ).filter(
                Copy.notice == Record.id
            ).filter(
                Copy.id.in_(b)
            ).all()

            for c, r in results:
                titles.append(r.main_title)

        return set(titles)
```

Sans rentrer dans des détails inutiles, SQLAlchemy nous permet de
créer des classes Python, en renseignant un parent commun, dans notre
cas, `PMBOrm`. Chaque classe doit renseigner la table du schéma via un
attribut `__tablename__` et doit définir à minima une clé
primaire. Les colonnes de la table sont décrites via la fonction
`Column()`. Je peux ensuite ajouter des méthodes à ces classes, comme
par exemple `full_name()` qui me permet d'avoir le nom complet d'un
usager.

Dans le cas de ce *mapping*, je me suis volontairement limité dans le
nombre de colonnes gérés par tables. Et j'en ai profité pour donner
des noms plus explicites à certaines colonnes : `tit1` → *main_title*,
`tit3` → *parallel_title*, etc. 

## Utilisation de l'ORM pour construire le jeu de données

Une fois l'ORM mis en place, je peux écrire le script qui va extraire
les données du SGBD, et le formater dans le format souhaité :

```python
#!/usr/bin/env python

import json
import os

from dotenv import find_dotenv, load_dotenv

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from pmb_orm import (Patron, Record, Copy, Borrowing, Borrowed)


if __name__ == "__main__":
	# Récupération des données de connexion dans un fichier de configuration
    load_dotenv(find_dotenv())
	
	# Création d'une connexion vers le SGBD
    e = create_engine(os.getenv("DB_DSN"))
    PMB = sessionmaker(bind=e)
    session = PMB()

	# Construction du jeu de données
    dataset = {}
    patrons = session.query(Patron).all()
    for patron in patrons:
        dataset[patron.full_name] = {
            title: 1
            for title in patron.borrowed_titles
        }

	# Affichage du jeu de données
    print(json.dumps(dataset, indent="  "))
```

La première partie du script consiste en l'importation des différents
modules nécessaires (les `import`s et les `from`s). Puis le script
établit une connexion avec le SGBD, et le traitement peux finalement
commencer :

* Nous récupérons tous les lecteurs ;
* Pour chacun des lecteurs, nous récupérons les titres des ouvrages
  empruntés et nous lui attribuons la valeur *1*. 
  
Et enfin, nous affichons les données. 

Voici un exemple, au final assez simple, pour extraire des données
d'un SIGB, et ainsi exploiter les données précieuses contenues dans ce
dernier.
