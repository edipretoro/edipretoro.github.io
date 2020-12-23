---
layout: post
title: Une autre technologie pour créer un système de recommandation
category: [LIS Advent Calendar]
tags: [Neo4j, Graphe, Python, Systèmed de recommandation]
published: true
comments: true
---

Il y a plus de [quinze jours maintenant]({% post_url
2020-12-06-recommander_des_ressources %}), je vous présentais une
méthode permettant de mettre en place un système de recommandation de
ressources qui exploitait les données de prêts d'un SIGB. La méthode
s'appuyait sur une matrice des lecteurs et des ressources empruntées,
et le traitement de cette matrice pour déterminer les
recommandations. Mais cette méthode implique de nombreux calculs et,
par conséquent, un temps de traitement qui peut être
énorme. Aujourd'hui, je vous propose de voir une autre méthode,
qui exploite [Neo4j](https://neo4j.com/), une base de données
s'appuyant sur les graphes.

## Des graphes ???

Si la méthode précédente se basait sur une matrice, à savoir un
tableau :

| Nom du lecteur | Titre 1 | Titre 2 | Titre 3 | ... |
| ----:          | -----:  | ----:   | ----:   |     |
| Lecteur 1      | 0       | 1       | 1       | ... |
| Lecteur 2      | 1       | 0       | 0       | ... |
| Lecteur 3      | 0       | 1       | 0       | ... |
| ...            | ...     | ...     | ...     | ... |

pour cette méthode, nous allons construire une autre [structure de
données](https://fr.wikipedia.org/wiki/Structure_de_donn%C3%A9es) : un
[graphe](https://fr.wikipedia.org/wiki/Th%C3%A9orie_des_graphes). Un
graphe est composé de sommets et d'arêtes, et l'analogie la plus
simple consiste à s'imaginer des stations de métro, les sommets, et
les rails entre ces stations, les arêtes. Parcourir un graphe consiste
alors à trouver le chemin, c'est-à-dire les arêtes, entre un sommet de
départ et un sommet d'arrivée. 

Pour notre système de recommandation, nous aurons deux types de
sommets :

1. Des lecteurs ;
2. Des livres. 

Ces sommets étaient reliés par une arête représentant l'emprunt. 

Voici une représentation de mes prêts dans Neo4j :

![Mes prêts sous forme de graphe](/assets/img/20201223-graph_example.png)

## Importer les données

La première étape consiste à importer les données dans Neo4j. Voici le
code responsable de cette tâche :

```python
#!/usr/bin/env python

import os

from dotenv import find_dotenv, load_dotenv
from neomodel import (
    IntegerProperty,
    StructuredNode,
    StringProperty,
    RelationshipFrom,
    RelationshipTo,
    config,
    db,
)
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

import pmb_orm


class Book(StructuredNode):
    title = StringProperty(unique_index=True)
    isbn = StringProperty()
    patrons = RelationshipFrom("Patron", "BORROW")


class Patron(StructuredNode):
    id = IntegerProperty()
    fullname = StringProperty()
    firstname = StringProperty()
    lastname = StringProperty()
    books = RelationshipTo("Book", "BORROW")


def run():
    load_dotenv(find_dotenv())

    config.DATABASE_URL = os.getenv("NEO4J_URL")

    e = create_engine(os.getenv("DB_DSN"))
    PMB = sessionmaker(bind=e)
    session = PMB()

    patrons = session.query(pmb_orm.Patron).all()
    for patron in patrons:
        p = Patron(
            id=patron.id,
            fullname=patron.full_name,
            firstname=patron.first_name,
            lastname=patron.last_name,
        ).save()
        for book in patron.borrowed_titles:
            b = Book.nodes.get_or_none(title=book)
            if not b:
                res = session.query(pmb_orm.Record).filter(
                    pmb_orm.Record.main_title == book
                ).first()
                b = Book(title=book, isbn=res.isbn).save()
            p.books.connect(b)


if __name__ == "__main__":
    run()
```

Le code n'a rien de particulièrement compliqué :

* Je définis les deux classes qui représenteront les sommets dans
  Neo4j ;
* Je configure les connexions aux bases de données ;
* Je récupère tous les lecteurs, et j'alimente Neo4j avec les
  données.

## Obtenir les recommandations

La dernière étape consiste à obtenir les recommandations, et pour
cela, nous allons devoir interroger Neo4j. Ce dernier propose un
langage de requête nommé
[*Cypher*](https://neo4j.com/developer/cypher/). Ce langage vous
permet d'exprimer, sous une forme relativement visuelle, ce qui vous
intéresse dans le graphe. 

Dans notre cas, voici la requête Cypher pour les recommandations à
faire à un lecteur :

```cypher
MATCH (p:Patron {lastname: "DI PRETORO"})-[:BORROW]->(b)<-[b1:BORROW]-(sim_user)
WITH p, sim_user, COUNT(b1) AS sim_score
ORDER BY sim_score DESC
MATCH (sim_user)-[b2:BORROW]->(title)
WHERE NOT((p)-->(title))
RETURN title, COUNT(b2) AS score_title
ORDER BY score_title DESC
```

Et la requête permettant d'identifier les titres proches :

```cypher
MATCH (b:Book {title: "Mettre en oeuvre un plan de classement"})<-[:BORROW]-()-[b1:BORROW]->(sim_book)
RETURN b, sim_book, COUNT(b1) AS sim_score
ORDER BY sim_score DESC
```

Il ne reste plus qu'à intégrer tout cela dans un script :

```python
#!/usr/bin/env python

import argparse
import os

from dotenv import load_dotenv, find_dotenv

import pmb2neo4j


def get_recommandations_by(title):
    title_query = """
MATCH (b:Book {title: $title})<-[:BORROW]-()-[b1:BORROW]->(sim_book)
RETURN b, sim_book, COUNT(b1) AS sim_score
ORDER BY sim_score DESC
"""

    results, meta = pmb2neo4j.db.cypher_query(title_query, {"title": title})
    books = [(pmb2neo4j.Book.inflate(r[1]), r[2]) for r in results[:20]]

    return books


def get_recommandations_for(patron):
    patron_query = """
MATCH (p:Patron {lastname: $lastname})-[:BORROW]->(b)<-[b1:BORROW]-(sim_user)
WITH p, sim_user, COUNT(b1) AS sim_score
ORDER BY sim_score DESC
MATCH (sim_user)-[b2:BORROW]->(title)
WHERE NOT((p)-->(title))
RETURN title, COUNT(b2) AS score_title
ORDER BY score_title DESC
"""

    results, meta = pmb2neo4j.db.cypher_query(patron_query, {"lastname": patron})
    books = [(pmb2neo4j.Book.inflate(r[0]), r[1]) for r in results[:20]]

    return books


if __name__ == "__main__":
    load_dotenv(find_dotenv())
    pmb2neo4j.config.DATABASE_URL = os.getenv("NEO4J_URL")
    parser = argparse.ArgumentParser(
        description="Getting recommendations for a patron or based on a title."
    )

    parser.add_argument("-p", "--patron", help="Lastname of the patron", required=False)
    parser.add_argument("-t", "--title", help="Title of the book", required=False)
    args = parser.parse_args()

    if args.patron:
        for book, _ in get_recommandations_for(patron=args.patron):
            print(f"{book.title} (ISBN : {book.isbn})")

    if args.title:
        for book, _ in get_recommandations_by(title=args.title):
            print(f"{book.title} (ISBN : {book.isbn})")
```

Qui nous donnera les résultats suivants pour `recsys_neo4j.py -p "DI PRETORO"` :

```text
Mettre en oeuvre un plan de classement (ISBN : 978-2-910227-74-6)
Travail et méthodes du documentaliste (ISBN : 978-2-7101-1435-2)
Désherber en bibliothèque (ISBN : )
De BCDI 2 à PMB (ISBN : )
Management de projet (ISBN : 978-2-7081-3448-5)
Informatisation et numérisation d'un fonds photographique du Conseil International du Sport Militaire (CISM) (ISBN : )
Réussir son travail de fin d'études (ISBN : 978-2-8041-2298-0)
Techniques de veille et e-réputation (ISBN : 978-2-7460-4928-4)
UNIMARC (ISBN : 978-2-7654-0746-1)
Le métier de documentaliste (ISBN : 978-2-7654-0744-7)
La conduite de projet (ISBN : 978-2-212-54142-7)
Du nouveau à Saint-Boniface ! (ISBN : )
La bibliothécaire jeunesse, une intervenante culturelle (ISBN : 978-2-7654-0931-1)
Développer la médiation documentaire numérique (ISBN : 978-2-910227-99-9)
Espace pédagogique de la librairie La Licorne (ISBN : )
Une chouette au pays des livres (ISBN : )
Les archives électroniques (ISBN : 2-11-005131-0 (La documentation française))
Construire... et gérer son projet (ISBN : )
Thésauroglossaire des langages documentaires (ISBN : 978-2-84365-051-2)
Indice, index, indexation (ISBN : 978-2-84365-088-8)
```

Et les résultats suivants pour `recsys_neo4j.py -t 'Mettre en oeuvre un plan de classement'` :

```text
L'analyse documentaire (ISBN : 978-2-84365-030-7)
De BCDI 2 à PMB (ISBN : )
Guide pratique pour l'élaboration d'un thésaurus documentaire (ISBN : 978-2-923563-17-6)
Travail et méthodes du documentaliste (ISBN : 978-2-7101-1435-2)
Une chouette au pays des livres (ISBN : )
Conduire une politique documentaire (ISBN : 978-2-7654-0717-1)
Désherber en bibliothèque (ISBN : )
Méthodologie du recueil d'informations (ISBN : 978-2-8041-2299-7)
Organiser et faire vivre le classement (ISBN : 978-2-7101-1410-9)
Indice, index, indexation (ISBN : 978-2-84365-088-8)
A la recherche du mot-clé (ISBN : 978-2-88224-014-9)
Actualité des langages documentaires (ISBN : 978-2-84365-060-4)
Les politiques d'acquisition (ISBN : )
Introduction aux sciences de l'information (ISBN : 978-2-7071-5933-5)
Produire des contenus documentaires en ligne (ISBN : 979-10-91281-37-9)
Développer la médiation documentaire numérique (ISBN : 978-2-910227-99-9)
Le métier de documentaliste (ISBN : 978-2-7654-0744-7)
L'archivage photo (ISBN : 978-2-7440-9255-8)
Analyse des besoins (ISBN : 978-2-212-54144-1)
Bibliothécaires et enseignants partenaires dans la formation à la recherche documentaire (ISBN : )
```
