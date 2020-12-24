---
layout: post
title: Filtrer ses fils RSS
category: [LIS Advent Calendar]
tags: [Python, Document Filtering, Naïve Bayes]
published: true
comments: true
---

Pour ce dernier billet du calendrier de l'Avent 2020 de l'informatique
documentaire, je vais parler du filtrage de fils RSS. Si se tenir
informé est une évidence professionnelle, il est tout aussi évident
que ce processus doit être réfléchi de manière à éviter la noyade
informationnelle. Pour le moment, ma stratégie a été de pratiquer
autant que possible une [diète
informationnelle](https://www.oreilly.com/library/view/the-information-diet/9781449321536/)
en limitant les sources d'informations. Mais même avec cette
stratégie, je constate que je passe encore pas mal de temps à
parcourir rapidement les articles pour voir s'ils ont un intérêt pour
moi. Il est donc temps de se pencher de nouveau sur le problème, et
voir si je peux déléguer une partie du travail à un logiciel. Ce
billet se limitera à présenter la technologie que je compte utiliser
pour filtrer mes fils RSS, l'outil étant encore principalement dans
une phase exploratoire.

## Un problème connu

Filtrer des documents est un problème classique, et présent dans la
vie de tout un chacun puisque nos boîtes mail sont probablement toutes
connectées à un système de filtrage du spam. Pour filtrer les entrées
des fils RSS, je n'utiliserai pas les catégories « *spam* » et « *ham*
», mais « *à lire* » et « *à ne pas lire* ». Et la première méthode
qui me passe par la tête est l'algorithme de [classification naïve de
Bayes](https://fr.wikipedia.org/wiki/Classification_na%C3%AFve_bay%C3%A9sienne). Si
le théorème de Bayes est assez haut dans ma liste des sujets à
creuser, entre autres en lisant [*La formulaire du
savoir*](https://laboutique.edpsciences.fr/produit/1035/9782759822614/La%20formule%20du%20savoir),
j'ai pu m'appuyer sur l'ouvrage [*Programming Collective
Intelligence*](https://www.oreilly.com/library/view/programming-collective-intelligence/9780596529321/)
pour avoir une implémentation de l'algorithme à tester.

## Le code

Voici le code présent dans *Programming Collective Intelligence*
(adapté pour Python 3) :

```python
import re
import math


def getwords(doc):
    splitter = re.compile(r'[^\w]+')
    words=[
        s.lower()
        for s in splitter.split(doc)
        if len(s)>2 and len(s)<20
    ]

    return {w: 1 for w in words}


class naivebayes:
    def __init__(self, getfeatures, filename=None):
        self.fc = {}
        self.cc = {}
        self.getfeatures = getfeatures
        self.thresholds = {}
        
    def incf(self, f, cat):
        self.fc.setdefault(f, {})
        self.fc[f].setdefault(cat, 0)
        self.fc[f][cat] += 1

    def incc(self, cat):
        self.cc.setdefault(cat, 0)
        self.cc[cat] += 1

    def fcount(self, f,cat):
        if f in self.fc and cat in self.fc[f]:
            return float(self.fc[f][cat])
        return 0.0

    def catcount(self, cat):
        if cat in self.cc:
            return float(self.cc[cat])
        return 0

    def totalcount(self):
        return sum(self.cc.values())

    def categories(self):
        return self.cc.keys()
    
    def train(self, item, cat):
        features = self.getfeatures(item)
        for f in features:
            self.incf(f, cat)
        self.incc(cat)
        
    def fprob(self, f, cat):
        if self.catcount(cat) == 0: 
            return 0
        return self.fcount(f, cat) / self.catcount(cat)
        
    def weightedprob(self, f, cat, prf, weight=1.0, ap=0.5):
        basicprob = prf(f, cat)

        totals = sum([
            self.fcount(f, c) 
            for c in self.categories()
        ])

        bp = ((weight * ap) + (totals * basicprob))/(weight + totals)
        return bp
    
    def docprob(self,item,cat):
        features = self.getfeatures(item)

        p = 1
        for f in features: 
            p *= self.weightedprob(f, cat, self.fprob)
        return p
    
    def setthreshold(self, cat, t):
        self.thresholds[cat] = t

    def getthreshold(self, cat):
        if cat not in self.thresholds: 
            return 1.0
        return self.thresholds[cat]
    
    def prob(self, item, cat):
        catprob = self.catcount(cat) / self.totalcount()
        docprob = self.docprob(item, cat)
        return docprob * catprob
    
    def classify(self, item, default=None):
        probs = {}
        max = 0.0
        for cat in self.categories():
            probs[cat] = self.prob(item, cat)
            if probs[cat] > max:
                max = probs[cat]
                best = cat
        for cat in probs:
            if cat == best: 
                continue
            if probs[cat] * self.getthreshold(best) > probs[best]: 
                return default
        return best
```

Les algorithmes de classification de documents s'appuient sur les
caractéristiques de ces derniers, vous verrez souvent apparaître le
terme *features* pour désigner celles-ci. Dans notre cas, nous
utiliserons les mots présents dans le document comme *feature*, et la
fonction `getwords()` réalise ce travail. 

Voici un exemple pour le titre de ce billet : `getwords("Filtrer ses
fils RSS")` qui nous donne le résultat suivant : `{'filtrer': 1,
'ses': 1, 'fils': 1, 'rss': 1}`. 

Nous avons ensuite une classe, `naivebayes`, qui va implémenter les
différentes méthodes nécessaires.

Voici comment utiliser cette classe :

```python
filtre = naivebayes(getwords)
```

va instancier `naivebayes` en lui indiquant comment construire les
`features` d'un document. L'algorithme de classification naïve
bayésienne étant une méthode d'apprentissage supervisé, il faut
d'abord entraîner le filtre avant qu'il puisse effectuer sa tâche.

Voici un bout de code illustrant la phase d'entraînement : 

```
entries = {
    'à lire': [
        'Web Scraping avec Ruby',
        'Créer un bot Discord avec Python'
    ],
    'à ne pas lire: [
        'Guide Hacks 3DS - 3DS Hacks Guide',
        'How to Favicon in 2021: Six files that fit most needs — Martian Chronicles, Evil Martians’ team blog'
    ]
}
for tag, titles in entries.items():
    for title in titles:
        filtre.train(title, tag)
```

Sur base de cet entraînement, je peux obtenir une prédiction :

```python
filtre.classify("Introduction au langage SPARQL avec Wikidata", default="unknown")
```

qui me donnera le résultat suivant : `à lire`. 

L'implémentation actuelle me permet aussi de vérifier ce qui se passe
« sous le capot » :

* Contenu de la variable `fc` qui contient les occurrences des
  *features* par catégories :
  
```json
{'web': {'à lire': 1},
 'scraping': {'à lire': 1},
 'avec': {'à lire': 2},
 'ruby': {'à lire': 1},
 'créer': {'à lire': 1},
 'bot': {'à lire': 1},
 'discord': {'à lire': 1},
 'python': {'à lire': 1},
 'guide': {'à ne pas lire': 1},
 'hacks': {'à ne pas lire': 1},
 '3ds': {'à ne pas lire': 1},
 'how': {'à ne pas lire': 1},
 'favicon': {'à ne pas lire': 1},
 '2021': {'à ne pas lire': 1},
 'six': {'à ne pas lire': 1},
 'files': {'à ne pas lire': 1},
 'that': {'à ne pas lire': 1},
 'fit': {'à ne pas lire': 1},
 'most': {'à ne pas lire': 1},
 'needs': {'à ne pas lire': 1},
 'martian': {'à ne pas lire': 1},
 'chronicles': {'à ne pas lire': 1},
 'evil': {'à ne pas lire': 1},
 'martians': {'à ne pas lire': 1},
 'team': {'à ne pas lire': 1},
 'blog': {'à ne pas lire': 1}}
}
```

* Le contenu de la variable `cc` qui permet de compter le nombre de
  fois que les catégories ont été utilisées :
  
```json
{'à lire': 2, 'à ne pas lire': 2}
```

Évidemment, pour que ce filtre soit réellement intéressant, j'aurai
besoin de l'entraîner sur beaucoup plus de documents. Il me faudra
également permettre une rétroaction pour confirmer ou infirmer les
prédictions. Et se pose également la question des *features* :

* Le système présenté se limite aux mots du titre, mais il est
possible, et probablement souhaitable, de traiter également le contenu
de l'article ;
* Le système présenté considère que tous les mots sont sur un pied
d'égalité, mais il y a moyen d'affiner cela en utilisant des
pondérations : pour les mots titre du titre, par exemple. 

Mais cela suffit pour une présentation sommaire de la technologie !

Et c'est la fin de ce calendrier de l'Avent de l'informatique
documentaire. Je vous souhaite de joyeuses fêtes de fin d'année !
