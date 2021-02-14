---
layout: post
title: Réaliser un collage de ses lectures
category: Lecture
tags: [Python]
published: true
comments: true
---

Le passage d'une année à une autre est souvent l'occasion de faire le
bilan : nouvelles compétences, calories dépensées en faisant du sport,
etc. Dans ce billet, je vais me pencher sur le bilan de mes lectures
de 2020. J'utilise les services de [Babelio](https://www.babelio.com/)
pour découvrir de nouveaux titres qui pourraient me plaire,
j'enregistre en conséquence les livres que je lis dans cette
plateforme, en tout cas les lectures « détente ». Et Babelio offre la
possibilité d'exporter nos lectures via l'URL suivante :
<https://www.babelio.com/export_bib.php>. 

## Récupération et découverte des données

Après avoir récupéré le fichier CSV généré, je peux commencer à
l'explorer. Pour cela, je vais utiliser
[`pandas`](https://pandas.pydata.org/) : 

```python
import pandas as pd


books = pd.read_csv("./Biblio_export799712.csv", encoding="ISO-8859-1", sep=";")
print(*books.columns.tolist(), sep='\n')
```

qui me donne le résultat suivant, à savoir les colonnes du fichier CSV :

```
ISBN
Titre
Auteur
Editeur
Date de publication
Date d`entrée dans Babelio
Statut
Note
```

Pour obtenir la liste des titres lus en 2020, je pourrai écrire le
code suivant : 

```python
print(*books[(books["Date d`entrée dans Babelio"] > "2020-01-01") & (books["Statut"] == "Lu")]['Titre'], sep='\n')
```

Et si je veux connaître le nombre de livres lus durant cette période : 

```python
print(len(books[(books["Date d`entrée dans Babelio"] > "2020-01-01") & (books["Statut"] == "Lu")]))
```

## Récupération des couvertures

Pour réaliser le collage, j'ai besoin des couvertures. Récupérer les
couvertures sur base d'un ISBN peut se révéler un challenge en soi,
mais dans mon cas, le code suivant m'a permis d'en récupérer une
majorité : 

```python
import isbnlib
import urllib


amazon_url = "http://images-eu.amazon.com/images/P/{}.jpg"
for isbn in books[(books['Statut'] == "Lu") & (books["Date d`entrée dans Babelio"] > "2020-01-01")]["ISBN"]:
    if isbnlib.to_isbn10(isbn):
        urllib.request.urlretrieve(amazon_url.format(isbnlib.to_isbn10(isbn)), "covers/{}.jpg".format(isbnlib.to_isbn13(isbn)))
        print(amazon_url.format(isbnlib.to_isbn10(isbn)))
    else:
        print(isbn)
```

J'ai ensuite récupéré à la main les couvertures manquantes. 

## Création du collage

Avec les fichiers des couvertures dans le répertoire « *covers* », je
peux maintenant créer le collage :

```python
import glob
import random
import cv2
import numpy as np
import matplotlib.pyplot as plt



files = glob.glob("covers/*.jpg")    # Obtenir la liste des fichiers de couvertures
random.shuffle(files)                # Mélanger ces fichiers
rows = 5                             # Configurer le nombre de lignes pour le collage
images = [cv2.imread(img, 1) for img in files]           # Lire les couvertures
images = [cv2.resize(img, (50, 75)) for img in images]   # Redimmensionner les images
column_matrix = []                   # Variables pour le traitement
last = 0
for i in range(rows, len(files), rows):   # Début du traitement
    if len(images[last:i]) == rows:
        column_matrix.append(np.vstack([*images[last:i]]))
    last = i
	
collage = np.hstack(column_matrix)
plt.figure(figsize=(5, 5))
plt.axis("off")
plt.imshow(collage[:,:,::-1])
plt.savefig("collage_covers.png", bbox_inches='tight')
```

Ce qui me donne l'image suivante : 

![Livres lus en 2020](/assets/img/collage_covers-2020.png)

ce qui me fera une photo de profil idéale pour Babelio :D
