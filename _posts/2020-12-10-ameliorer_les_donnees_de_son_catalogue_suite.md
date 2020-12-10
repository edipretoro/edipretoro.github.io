---
layout: post
title: Continuons à améliorer les données du catalogue
category: [LIS Advent Calendar]
tags: [Python, Qualité des données, SQLAlchemy]
published: true
comments: true
--- 

Hier, je vous avais proposé d'améliorer les données d'un catalogue en
utilisant une technique de regroupement (*clustering*) des potentiels
doublons pour les autorités auteurs. Je vous propose de continuer sur
ce thème aujourd'hui, en voyant un nouvel algorithme, toujours inspiré
d'[OpenRefine](https://openrefine.org/), s'appuyant sur des
*n-grams*. 

## Des *n-grams* ? ...

Les *n-grams* sont une liste des séquences de sous-chaînes de
caractères de taille *n* présents dans une chaîne de caractères. Par
exemple, avec « Sciences de l'information », nous aurons :

* pour des *n-grams* de taille 1 : `['S', 'c', 'i', 'e', 'n', 'c', 'e',
  's', ' ', 'd', 'e', ' ', 'l', "'", 'i', 'n', 'f', 'o', 'r', 'm',
  'a', 't', 'i', 'o', 'n']` ;
* pour des *n-grams* de taille 2 : `['Sc', 'ci', 'ie', 'en', 'nc',
  'ce', 'es', 's ', ' d', 'de', 'e ', ' l', "l'", "'i", 'in', 'nf',
  'fo', 'or', 'rm', 'ma', 'at', 'ti', 'io', 'on']` ;
* pour des *n-grams* de taille 3 : `['Sci', 'cie', 'ien', 'enc',
  'nce', 'ces', 'es ', 's d', ' de', 'de ', 'e l', " l'", "l'i",
  "'in", 'inf', 'nfo', 'for', 'orm', 'rma', 'mat', 'ati', 'tio',
  'ion']` ;
* ... et ainsi de suite.

## La fonction de regroupement utilisant les *n-grams*

Voici une implémentation rapide de l'algorithme décrit dans la
[documentation
d'OpenRefine](https://github.com/OpenRefine/OpenRefine/wiki/Clustering-In-Depth#n-gram-fingerprint) : 

```python
import string
import re

import nltk
import unidecode


def ngram_fingerprint(value, n=2):
    value = value.lower().translate(str.maketrans('', '', string.punctuation))
    value = value.translate(str.maketrans('', '', string.whitespace))
    value = re.sub(r'[\x00-\x1f\x7f-\x9f]', '', value)
    value = unidecode.unidecode(value)
    value = sorted(set(nltk.ngrams(value, n)))

	return "".join(["".join(ngram) for ngram in value])
```

La fonction est quasiment la même, si ce n'est la suppression des
espaces (`value = value.translate(str.maketrans('', '',
string.whitespace))`) et la génération des *n-grams* (`value =
sorted(set(nltk.ngrams(value, n)))`). J'ai rendu paramétrable la
taille des *n-grams*. 

## Deuxième passage sur les autorités auteurs

J'ai légèrement modifié le script pour dédoublonner les auteurs : 

```python
#!/usr/bin/env python

import itertools
import json
import pathlib

from fingerprinting import fingerprint, ngram_fingerprint


def getting_duplicates(records, blocking=fingerprint):
    blocking_data = {}
    for key, data in records.items():
        blocking_data.setdefault(blocking(key), []).append({**data})

    c = itertools.count()

    for fingerprint, data in blocking_data.copy().items():
        if len(data) > 1:
            next(c)
        else:
            del blocking_data[fingerprint]

    return blocking_data, next(c)


if __name__ == "__main__":
    authors = json.loads(pathlib.Path("authors.json").read_text(encoding="utf-8"))
    print(f"We have {len(authors)} authors in our catalog.")

    duplicated_authors, c = getting_duplicates(authors)
    print(f"We have {c} authors possibly duplicated.")
    print(json.dumps(duplicated_authors, indent=2, ensure_ascii=False))

    duplicated_authors, c = getting_duplicates(authors, ngram_fingerprint)
    print(f"We have {c} authors possibly duplicated.")
    print(json.dumps(duplicated_authors, indent=2, ensure_ascii=False))
```

Et nous avons le résultat suivant :

```text
We have 21173 authors in our catalog.
We have 23 authors possibly duplicated.

. # Même résultat qu'hier
.
.
We have 21 authors possibly duplicated.
{
  .
  .
  .
  
  "adandueeelemerielalemammnuoyriroueuryl": [
    {
      "id": 6013,
      "last_name": "Le Roy Ladurie",
      "first_name": "Emmanuel"
    },
    {
      "id": 9236,
      "last_name": "Leroy-Ladurie",
      "first_name": "Emmanuel"
    }
  ],
  .
  .
  .
}
```

Les *n-grams* de taille 2 trouvent moins de doublons, mais en trouve
de nouveau, comme l'exemple donné. Les deux méthodes sont donc très
complémentaires. Il y aura moyen également de changer la taille des
*n-grams* générés. Par exemple, avec des *n-grams* de taille 1,
l'outil a identifié 2051 doublons potentiels. Un rapide coup d'œil sur
ces derniers m'a permis d'identifier pas mal de faux positifs.

Par curiosité, j'ai rapidement créé les scripts nécessaires pour
traiter les éditeurs et les collections, voici un tableau récapitulant
les résultats :

| Autorités     | Total    | fingerprint | ngram (n = 2) | ngram (n = 1) |
| ------        | :----:   | --------:   | ------:       | -----:        |
| *Éditeurs*    | **1751** | 6           | 7             | 101           |
| *Collections* | **1740** | 12          | 14            | 114           |

À mes yeux, cela vaut la peine d'essayer plusieurs méthodes de
regroupement !
