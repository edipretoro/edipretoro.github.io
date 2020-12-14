---
layout: post
title: Une méthode pour regroupe de potentiels doublons
category: [LIS Advent Calendar]
tags: [Python, Algorithme, Data Matching]
published: true
comments: true
--- 

J'ai déjà brièvement parlé de méthodes permettant de [regrouper]({%
post_url 2020-12-09-ameliorer_les_donnees_de_son_catalogue %}) des
[données]({% post_url
2020-12-10-ameliorer_les_donnees_de_son_catalogue_suite %}) dans le
contexte de la détection de potentiels doublons des autorités d'un
catalogue. Dans les billets à venir, j'aborderai différentes méthodes
de dédoublonnage de notices catalographiques, mais avant de rentrer
dans le vif du sujet, j'aimerais présenter une autre méthode qui
permet de regrouper des données. Mais contrairement aux deux méthodes
vues précédemment qui généraient une clé sur base des données, et qui
effectuait le regroupement sur base de celle-ci, cette méthode
effectue le regroupement en utilisant une fonction de comparaison. La
méthode s'appelle l'[algorithme de regroupement par
canopée](https://en.wikipedia.org/wiki/Canopy_clustering_algorithm).

Voici une implémentation de cet algorithme :

```python
#!/usr/bin/env python

import json
import pathlib

import jellyfish

data = pathlib.Path('/usr/share/dict/french').read_text().split()

idx = {k: k for k in data}

T1 = 0.65
T2 = 0.75

assert T1 < T2, "T1 is lower than T2"

canopies = []

while idx:
    orig_key, orig_val = idx.popitem()
    canopy = [orig_key]
    for key, val in idx.copy().items():
        dist = jellyfish.jaro_distance(orig_val, val)
        if dist > T1:
            canopy.append(key)
        if dist > T2:
            del idx[key]
                
    canopies.append(canopy)

print(f"Nous avons {len(canopies)} canopies"

total = 0
for c in canopies:
    print(c)
    if len(c) > 1:
        total += 1

print(f"Nous avons {total} canopies avec plus d'un élément")
```

Avec le fichier `/usr/share/dict/french`, un dictionnaire
francophone contenant 348358 mots, nous obtenons le résultat suivant :

```text
Nous avons 6274 canopées
Nous avons 6266 canopées avec plus d'un élément
```

L'avantage de cette méthode est de permettre de regrouper relativement
rapidement des données jugées similaires par une fonction, dans le cas
du script, la [distance de
Jaro-Winkler](https://fr.wikipedia.org/wiki/Distance_de_Jaro-Winkler),
avant de passer à un algorithme plus fin et plus coûteux en matière de
comparaison. L'algorithme agit donc comme un filtre. Et grâce aux
paramètres `T1` et `T2`, il est possible d'influencer le maillage de
ce filtre.

Pour reprendre l'exemple donné ci-dessous : nous avons trouvé 6266
groupes de termes qui valent la peine d'être comparés ultérieurement
avec une autre méthode, mais cela veut aussi dire que nous avons pu
exclure 342092 termes, et gagner ainsi pas mal de temps !


