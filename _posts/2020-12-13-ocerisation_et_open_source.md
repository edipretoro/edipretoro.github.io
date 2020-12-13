---
layout: post
title: Océrisation et open source
category: [LIS Advent Calendar]
tags: [Python, OCR]
published: true
comments: true
--- 

Au cours de mes précédents boulots comme documentaliste, une bonne
partie de mon temps consistait à scanner des documents, les océriser
de manière à obtenir un fichier PDF dont le texte serait accessible au
*Document Management System* utilisé
([Documentum](https://www.opentext.com/products-and-solutions/products/opentext-product-offerings-catalog/rebranded-products/documentum),
[SharePoint](https://www.microsoft.com/fr-be/microsoft-365/sharepoint/collaboration),
[Livelink](https://www.opentext.com/products-and-solutions/products/opentext-product-offerings-catalog/rebranded-products/livelink-is-now-part-of-the-opentext-ecm-suite),
etc), et les encoder dans ces plateformes. À l'époque, les solutions
pour océriser se limitait à [ABBYY FineReader
PDF](https://pdf.abbyy.com/fr/) et [Ascent
Capture](https://www.ecmconnection.com/doc/ascent-capture-0003), des
solutions commerciales. Et en terme d'*open source*, et bien, il n'y
avait pas grand chose. 

La situation s'est nettement améliorée avec
[Tesseract](https://tesseract-ocr.github.io/), mais évidemment il est
difficile de comparer un logiciel en ligne de commande, aussi
performant soit-il, et des suites logicielles offrant un grande
variété de fonctionnalités. Un de ces fonctionnalités, offerte par
Ascent Capture si ma mémoire est bonne, était la possibilité de
pouvoir déposer des fichiers PDF sur un disque réseau, et de les
récupérer océrisé dans un autre répertoire. 

Alors, je pourrais me lancer dans la réalisation d'un script offrant
ces fonctionnalités, mais cela existe déjà ! L'outil s'appelle
[`ocrmypdf`](https://ocrmypdf.readthedocs.io/en/latest/) et il s'agit
d'un *wrapper* autour de Tesseract. Dans le [dépôt GiHub du
projet](https://github.com/jbarlow83/OCRmyPDF/), il y a un outil nommé
[`watcher.py`](https://github.com/jbarlow83/OCRmyPDF/blob/master/misc/watcher.py)
qui offre précisément les fonctionnalités recherchées. 

L'outil se lance de la manière suivante ([tiré de la documentation
officielle](https://ocrmypdf.readthedocs.io/en/latest/batch.html#watched-folders-with-watcher-py))
:

```shell
env OCR_INPUT_DIRECTORY=/mnt/input-pdfs \
    OCR_OUTPUT_DIRECTORY=/mnt/output-pdfs \
    OCR_OUTPUT_DIRECTORY_YEAR_MONTH=1 \
    python3 watcher.py
```

Dans mon cas, le service a été ajouté sur un Raspberry Pi. Je peux
maintenant facilement océciser mes documents scannés, mais aussi les
fichiers PDF trouvés sur Internet qui ne le sont pas. 
