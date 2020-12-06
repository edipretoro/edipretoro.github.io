---
layout: post
title: Recommander des ressources à vos usagers
category: [LIS Advent Calendar]
tags: [Python, Système de recommandation]
published: true
comments: true
--- 

Un service qui m'a souvent manqué dans mes interactions avec les
catalogues de bibliothèque est une recommandation des livres qui
pourraient m'intéresser sur base de mes prêts passés. Cela permet de
découvrir des livres qui pourraient nous avoir échappé, et j'utilise
assez souvent cette fonctionnalité sur [Babelio](http://babelio.com/)
pour trouver ma prochaine lecture. 

Dans le cadre de cet article, je vais m'appuyer sur le code provenant
d'un livre : [*Programming Collective
Intelligence*](https://www.goodreads.com/book/show/1741472.Programming_Collective_Intelligence)
de Toby Segaran.

## Les données

En ce qui concerne les données nécessaires, elles sont assez
simples. J'ai besoin d'avoir une liste des lecteurs, et pour chaque
lecteur, une liste des titres de ressources empruntées. Dans un
fichier JSON, cela ressemblera à 

```json
{
	"Lecteur 1" : {
		"Titre 1": 1,
		"Titre 2": 1
	},
	"Lecteur 2" : {
		"Titre 3": 1,
		"Titre 4": 1
	},
	"Lecteur 3" : {
		"Titre 1": 1,
		"Titre 3": 1
	}
}
```

Le chiffre *1* à côté du titre représente juste le fait que la
ressource a été empruntée, un *0* signifiant que la ressource n'a pas
été empruntée. 

À priori, ces données ne devraient pas être trop difficiles à sortir
d'un SIGB. 

## Le code

La première étape consiste à uniformiser les données, c'est-à-dire à
renseigner les ouvrages non empruntés pour chaque utilisateur. Voici
le code Python :

```python
def fill_resources(patrons):
    all_resources = {}

	for patron in patrons:
        for resource, rating in patrons[patron].items():
            all_resources[resource] = 1

    for ratings in patrons.values():
        for item in all_resources:
            if item not in ratings:
                ratings[item] = 0.0
```

La deuxième étape est d'utiliser une méthode pour calculer la
similarité entre deux usagers. Pour cela, nous utiliserons le
coefficient de [corrélation de
Pearson](https://fr.wikipedia.org/wiki/Corr%C3%A9lation_(statistiques)#Coefficient_de_corr%C3%A9lation_lin%C3%A9aire_de_Bravais-Pearson)
:

```python
def sim_pearson(loans, p1, p2):
    si = {}
    for item in loans[p1]:
        if item in loans[p2]:
            si[item] = 1

    n = len(si)

    if n == 0:
        return 0

    sum1 = sum([loans[p1][it] for it in si])
    sum2 = sum([loans[p2][it] for it in si])

    sum1Sq = sum([pow(loans[p1][it], 2) for it in si])
    sum2Sq = sum([pow(loans[p2][it], 2) for it in si])

    pSum = sum([loans[p1][it] * loans[p2][it] for it in si])

    num = pSum - (sum1 * sum2 / n)
    den = sqrt((sum1Sq - pow(sum1, 2) / n) * (sum2Sq - pow(sum2, 2) / n))

    if den == 0:
        return 0
    r = num / den

    return r
```

La dernière partie est le calcul des recommandations : 

```python
from math import sqrt


def getRecommendations(loans, patron, similarity=sim_pearson):
    totals = {}
    simSums = {}

    for other in loans:
        if other == patron:
            continue

        sim = similarity(loans, patron, other)

        if sim <= 0:
            continue

        for item in loans[other]:
            if item not in loans[patron] or loans[patron][item] == 0:
                totals.setdefault(item, 0)
                totals[item] += loans[other][item] * sim

                simSums.setdefault(item, 0)
                simSums[item] += sim

    rankings = [(total / simSums[item], item) for item, total in totals.items()]
    rankings.sort()
    rankings.reverse()

    return rankings
```

Le principe de cette fonction est de prendre les données présentées au
début de l'article, et pour chaque lecteur, d'en calculer la
similarité avec l'utilisateur passer en argument. Ensuite, pour chaque
ressource des autres lecteurs que n'a pas été emprunté par notre
utilisateur, nous calculerons deux valeurs :

1. L'intérêt du lien : la somme des classements (la fameuse valeur *1*
   à côté du titre) multipliée par la similarité de lecteur ;
2. La somme totale des similarités par lien. 

Nous calculons ensuite un classement en divisant l'intérêt du lien par
la somme totale des similarités, nous trions et nous inversons notre
tableau que nous pouvons maintenant retourner. 

En enfin, il ne reste plus qu'à utiliser le tout via un script :

```python
if __name__ == "__main__":
    patrons = json.loads(pathlib.Path("dataset.json").read_text())
    fill_resources(patrons)
    user = "DI PRETORO, Emmanuel"
    pprint.pprint(getRecommendations(patrons, user)[0:10])
```

Ce dernier nous donnera le résultat suivant : 

```text
[(0.2248029528116556, 'Mettre en oeuvre un plan de classement'),
 (0.17032430641099586, 'Travail et méthodes du documentaliste'),
 (0.1489186043850669, 'Indice, index, indexation'),
 (0.12406349863584056, 'Thésauroglossaire des langages documentaires'),
 (0.12130765684475618, 'De BCDI 2 à PMB'),
 (0.11704262974597515, 'Le métier de documentaliste'),
 (0.1158689821989957, 'Développer la médiation documentaire numérique'),
 (0.11442466410474617, "Les fonctions de l'archivistique contemporaine"),
 (0.11317667874052578, 'Techniques de veille et e-réputation'),
 (0.11219350454890659, "Introduction aux sciences de l'information")]
```

Comme le jeu de données est basé sur des données réelles, il ne m'est
malheureusement pas possible de les communiquer. 

Vous trouverez dans ce [gist
(GitHub)](https://gist.github.com/edipretoro/0f6794246630a8ff012f7b6ad74cdfa1)
le script complet. 
