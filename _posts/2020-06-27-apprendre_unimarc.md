---
layout: post
title: Un petit outil pour apprendre l'UNIMARC
category: [Informatique documentaire]
tags: [Catalographie, UNIMARC, Python, Anki]
published: true
--- 

Je n'ai pas nécessairement pris la peine lors du dernier billet
d'expliquer ce qu'était l'UNIMARC, partant du principe que l'article
intéresserait principalement les « initiés », et que cette technologie
serait déjà connue. Cette semaine, je vous propose de voir comment
entretenir sa connaissance de l'UNIMARC, l'améliorer ou tout
simplement la construire. 

## Une norme...

Pour les besoins de ce billet, je vais limiter l'UNIMARC au cadre
suivant :

1. L'UNIMARC est composé de champs ;
2. Chaque champ peut être composé de sous-champs ;
3. Les champs et les sous-champs ont des libellés ;
4. Certains champs ou sous-champs sont obligatoires ;
5. Certains champs ou sous-champs sont répétables. 

Et pour retenir ce genre de faits, rien de tel que la méthode des
*flashcards* : vous notez la question au recto d'une fiche en papier,
la réponse au verso, et régulièrement, vous piochez quelques fiches et
vous vous testez. À force de répétition, vous allez finir par
mémoriser ces codes. 

Et même si les fiches en papier sont parfaitement utilisables, la
technologie offre une alternative intéressante avec le logiciel de
*flashcards*. Celui que j'utilise s'appelle
[Anki](https://apps.ankiweb.net/), et ce qui est vraiment intéressant
est d'avoir ce logiciel sur son *smartphone*. Pour cela, il suffit
d'installer
[AnkiDroid](https://play.google.com/store/apps/details?id=com.ichi2.anki&hl=fr). 

Maintenant, il faut écrire les différentes *flashcards*... Et
évidemment, le but de ce billet est d'illustrer comment automatiser
cela. 

## Des outils...

La première étape pour cette automatisation est de voir si ces données
existent déjà sous une forme utilisable à partir d'un ordinateur.
C'est le cas puisque [Koha](https://koha-community.org/) propose des
grilles de catalogage, dont l'UNIMARC, et que ces dernières sont
stockées dans une base de données SQL. Voici le fichier qui nous
intéresse : `framework_DEFAULT.sql`
[(GitHub)](https://github.com/fredericd/Koha/blob/master/installer/data/mysql/fr-FR/marcflavour/unimarc_complet/Obligatoire/framework_DEFAULT.sql).

La deuxième étape consiste à vérifier qu'il est possible de générer
des *flashcards* sans passer par une interface graphique. C'est
possible en Python avec
[`genanki`](https://github.com/kerrickstaley/genanki). 

Il ne reste plus qu'à orchestrer tout cela !

## Un peu de glue...

Dans un premier temps, je vais importer les données dans une base de
données [SQLite](https://www.sqlite.org/index.html). Il s'agit d'un
projet dont la durée de vie est très courte, c'est-à-dire qu'une fois
les *flashcards* créées, je n'en aurai plus besoin, et très simple
également. SQLite est parfait dans ces situations, et beaucoup
d'autres ! 

### Importation et accès aux données

J'ai adapté le schéma SQL pour SQLite : 

``` sql
CREATE TABLE IF NOT EXISTS marc_tag_structure (
    tagfield TEXT,
    liblibrarian TEXT,
    libopac TEXT,
    repeatable INTEGER,
    mandatory INTEGER,
    authorised_value INTEGER,
    frameworkcode INTEGER
);

CREATE TABLE IF NOT EXISTS marc_subfield_structure (
    tagfield TEXT,
    tagsubfield TEXT,
    liblibrarian TEXT, 
    libopac TEXT,
    repeatable INTEGER,
    mandatory INTEGER,
    kohafield TEXT,
    tab INTEGER,
    authorised_value TEXT,
    authtypecode TEXT,
    value_builder TEXT,
    isurl INTEGER,
    hidden INTEGER,
    frameworkcode INTEGER,
    seealso INTEGER,
    link TEXT, 
    defaultvalue TEXT
);
```

Et j'ai importé les données : `sqlite3 unimarc.db <
framework_DEFAULT.sql`.

J'ai ensuite créé les classes *SQLAlchemy* nécessaires pour accéder à
la base de données :

``` python
from sqlalchemy import Column, Text, Integer
from sqlalchemy.ext.declarative import declarative_base


Base = declarative_base()

class Field(Base):
    __tablename__ = 'marc_tag_structure'
    rowid = Column(Integer, primary_key=True)
    tagfield = Column(Text)
    liblibrarian = Column(Text)
    libopac = Column(Text)
    repeatable = Column(Integer)
    mandatory = Column(Integer)
    authorised_value = Column(Integer)
    frameworkcode = Column(Integer)

class SubField(Base):
    __tablename__ = 'marc_subfield_structure'
    rowid = Column(Integer, primary_key=True)
    tagfield = Column(Text)
    tagsubfield = Column(Text)
    liblibrarian = Column(Text) 
    libopac = Column(Text)
    repeatable = Column(Integer)
    mandatory = Column(Integer)
    kohafield = Column(Text)
    tab = Column(Integer)
    authorised_value = Column(Text)
    authtypecode = Column(Text)
    value_builder = Column(Text)
    isurl = Column(Integer)
    hidden = Column(Integer)
    frameworkcode = Column(Integer)
    seealso = Column(Integer)
    link = Column(Text) 
    defaultvalue = Column(Text)
```

Rien de bien compliqué, ni même de complet puisque je n'ai même pas
pris la peine de déclarer les associations. 

### Traitement des données et production des *flashcards*

Avec l'accès aux données de Koha, il ne nous reste plus qu'à les
transformer. 

Commençons par importer les modules nécessaires :

``` python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from unimarc2anki.db import Field, SubField
import genanki
import random
```

`unimarc2anki.db` est le module contenant les classes *SQLAlchemy*
mentionnées plus haut. 

Créons un identifiant aléatoire pour notre *modèle* :

``` python
model_id = random.randrange(1 << 30, 1 << 31)
```

Et créons le modèle pour nos *flashcards* :

``` python
model = genanki.Model(
    model_id,
    "Révision des champs et sous-champs d'UNIMARC",
    fields=[
        {'name': 'Question'},
        {'name': 'Réponse'}
    ],
    templates=[
        {
            'name': 'Carte',
            'qfmt': '{{Question}}',
            'afmt': '{{FrontSide}}<hr id="answer" />{{Réponse}}'
        }
    ]
)
```

Créons un identifiant aléatoire pour notre *deck* :

``` python
deck_id = random.randrange(1 << 30, 1 << 31)
```

Et créons le *deck* :

``` python
deck = genanki.Deck(
    deck_id,
    "Révision des champs et sous-champs d'UNIMARC"
)
```

Connectons-nous à la base de données créée précédemment :

``` python
e = create_engine('sqlite:///./data/koha_unimarc.db', echo=True)
Session = sessionmaker(bind=e)
s = Session()
```

Nous pouvons maintenant traiter les champs...

``` python
for i in fields:
    # Knowing the label
    note = genanki.Note(
        model=model,
        fields=[f'À quoi correspond le champ « {i.tagfield} » ?', f'{i.liblibrarian}']
    )
    deck.add_note(note)

    # Knowing if it is mandatory
    note = genanki.Note(
        model=model,
        fields=[f'Le champ « {i.tagfield} » est-il obligatoire ?', "Oui" if i.mandatory == 1 else "Non"]
    )
    deck.add_note(note)

    # Knowing if it is repeatable
    note = genanki.Note(
        model=model,
        fields=[f'Le champ « {i.tagfield} » est-il répétable ?', "Oui" if i.repeatable == 1 else "Non"]
    )
    deck.add_note(note)
```

... et les sous-champs :

``` python
subfields = s.query(SubField)
for i in subfields:
    # Knowing the label
    note = genanki.Note(
        model=model,
        fields=[f'À quoi correspond le champ « {i.tagfield}${i.tagsubfield} » ?', f'{i.liblibrarian}']
    )
    deck.add_note(note)

    # Knowing if it is mandatory
    note = genanki.Note(
        model=model,
        fields=[f'Le sous-champ « {i.tagfield}${i.tagsubfield} » est-il obligatoire ?', "Oui" if i.mandatory == 1 else "Non"]
    )
    deck.add_note(note)

    # Knowing if it is repeatable
    note = genanki.Note(
        model=model,
        fields=[f'Le sous-champ « {i.tagfield}${i.tagsubfield} » est-il répétable ?', "Oui" if i.repeatable == 1 else "Non"]
    )
    deck.add_note(note)
```

L'algorithme est le même pour les champs et les sous-champs :

1. Je récupère les données de la base de données ;
2. Pour chacun des enregistrements, je créerai trois *flashcards* :
   1. Une première pour vérifier sur le libellé est connu ;
   2. Une deuxième pour vérifier s'il s'agit d'un élément obligatoire
      ;
   3. Et enfin, une troisième question pour vérifier si l'élément est
      répétable. 
      
Notre script est bientôt fini puisqu'il ne nous reste plus qu'à écrire
le *deck* dans un fichier :

``` python
genanki.Package(deck).write_to_file('unimarc.apkg')
```

Le script complet est disponible sous forme de [Gist sur
GitHub](https://gist.github.com/edipretoro/cca3bdb3f258da42304e2870ed1db142).

## Mais est-ce vraiment utile ?

Alors si je ne doute pas de l'utilité des *flashcards*, étudier
l'entièreté de l'UNIMARC n'est probablement pas le but de tout un
chacun. Néanmoins, vu que nous avons maintenant une base de données à
notre disposition, il est assez simple de faire évoluer l'outil vers
des besoins plus spécifiques.

Et à tout hasard, voici le [*deck*](/assets/unimarc.apkg)...
