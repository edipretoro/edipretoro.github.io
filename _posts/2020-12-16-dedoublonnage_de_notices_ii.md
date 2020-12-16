---
layout: post
title: Dédoublonnage de notices bibliographiques (2e partie)
category: [LIS Advent Calendar]
tags: [Python, Algorithme, Data Matching, Dédoublonnage]
published: true
comments: true
--- 

Nous avons vu hier qu'il était possible de créer une clé sur base des
propriétés d'une notice catalographique, cette clé pouvant ensuite
être utilisée pour dédoublonner les notices. L'algorithme du *BibHash*
offrait même deux niveaux de clé : le niveau 0 qui était une
normalisation des propriétés de la notice, et le niveau 1 qui était
une signature MD5 de la clé de niveau 0. Nous allons voir aujourd'hui
qu'il est possible d'utiliser un autre type de signatures, les
*SimHashes*, qui permet d'estimer rapidement si deux documents sont
identiques.

## L'implémentation de l'algorithme

La tâche est assez facile aujourd'hui puisqu'un module implémentant
les *SimHashes* existe déjà :
[simhash](https://pypi.org/project/simhash/). 

## Utilisation du module

Nous allons reprendre le code d'hier, et essayer de résoudre les
problèmes constatés :

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
```

Nous avons maintenant un *BibHash* pour chacun des ouvrages. Il est
temps de voir à quoi ressemble un *SimHash* :

```python
    print(f"{book1_key.level0()} → {simhash.Simhash(book1_key.level0()).value}")
    print(f"{book2_key.level0()} → {simhash.Simhash(book2_key.level0()).value}")
    print(f"{book3_key.level0()} → {simhash.Simhash(book3_key.level0()).value}")
    print(f"{book4_key.level0()} → {simhash.Simhash(book4_key.level0()).value}")
```

qui nous donne le résultat suivant :

```text
lenomdelarose [u.eco] 1982 → 6125459656537179394
nomdelarosele [e.umberto] 1982 → 5854760840262058026
lenomdelarose [u.eco] 1982 → 6125459656537179394
schismatrice [b.sterling] 1985 → 3315960671757815760
```

Comme nous pouvons le voir, les trois premières clés sont plus proches
entre elles que de la quatrième. 

Pour exploiter cela, nous allons utiliser une autre fonctionnalité du
module : le *Simhashindex*.

```python
    catalog = {
        "book1": simhash.Simhash(book1_key.level0()),
        "book2": simhash.Simhash(book2_key.level0()),
        "book3": simhash.Simhash(book3_key.level0()),
        "book4": simhash.Simhash(book4_key.level0())
    }

    index = simhash.SimhashIndex(
        [(k, v) for k, v in catalog.items()],
        k=25
    )

    print(index.get_near_dups(catalog["book3"]))
```

Ce qui nous imprime le résultat suivant :

```text
['book2', 'book1', 'book3']
```

C'est-à-dire les *doublons* de notre petit catalogue. 

Quelques remarques :

* Dans l'exemple d'utilisation du *Simhashindex*, j'ai utilisé une
  valeur élevée pour`k`, qui est un paramètre pour régler la tolérance
  pour identifier les potentiels doublons, et ce n'est pas
  nécessairement souhaitable puisque le nombre de faux positifs
  augmente aussi. Une solution plus satisfaisante serait de ne pas
  utiliser les *BibHashes* et de créer une autre méthode ;
* Ce *Simhashindex* est vraiment pratique puisqu'il peut servir à
  mettre en place une vérification avant l'ajout d'une nouvelle
  notice.
