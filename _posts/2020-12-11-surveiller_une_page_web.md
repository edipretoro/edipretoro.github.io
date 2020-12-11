---
layout: post
title: Comment surveiller une page web ?
category: [LIS Advent Calendar]
tags: [Python, urlwatch]
published: true
comments: true
--- 

Pour le billet d'aujourd'hui, je vais m'intéresser à un problème
classique de la vie d'un documentaliste : surveiller efficacement une
page web. Alors, évidemment, je parle ici de surveiller un page web
qui n'offre pas de fil RSS, ou encore de surveiller une partie précise
d'une page web. Et je ne vais pas non plus vous présenter des bouts de
code qui font le boulot, je vais plutôt vous présenter un outil que
j'utilise pour cela. Il s'agit
d'[*urlwatch*](https://urlwatch.readthedocs.io/). 

Pour installer l'outil, et pour autant que Python soit déjà installé
sur votre machine, il vous suffit de saisir la commande classique :
`pip install urlwatch`. Il s'agit d'un programme qui va lire une liste
de tâches, stockées dans un fichier nommé `urls.yaml`, et qui va
générer un rapport : que ce soit un affichage dans le terminal, un
mail ou une communication via un *bot* Slack, Matrix, Telegram,
etc. Et pour que cet outil soit vraiment utile, il faut programmer une
exécution régulière, que ce soit via `cron` sur Unix ou `Windows Task
Scheduler`. 

## Un job simple : surveiller l'évolution des prix de serveurs

Pour éviter de laisser passer une éventuelle bonne affaire, je suis la
page des serveurs Kimsufi avec la configuration suivante :

```yaml
---
name: Prix des serveurs Kimsufi
navigate: https://www.kimsufi.com/fr/serveurs.xml
filter:
  - html2text:
      method: lynx
      width: 400
```

Cette configuration, utilisant l'option `navigate` va utiliser
[`pyppeteer`](https://pypi.org/project/pyppeteer/) pour récupérer la
page web avec la version *headless* de Google Chrome. De cette
manière, la page est chargée avec un navigateur qui interprétera
l'éventuel JavaScript. Cette page sera ensuite stockée et comparer
d'une exécution d'`urlwatch` à l'autre avec une méthode équivalente à
l'outil `diff`, bien connu des programmeurs. Voici un exemple de
résultat : 

```text
===========================================================================
01. CHANGED: Prix des serveurs Kimsufi
===========================================================================

---------------------------------------------------------------------------
CHANGED: Prix des serveurs Kimsufi ( https://www.kimsufi.com/fr/serveurs.xml )
---------------------------------------------------------------------------
--- @   Fri, 11 Dec 2020 16:42:40 +0100
+++ @   Fri, 11 Dec 2020 21:40:59 +0100
@@ -42,7 +42,7 @@
    Victime de son succès !
    Disponible prochainement.
    KS-4 Intel  Atom N2800 2c/4t 1,86GHz 4Go DDR3 1066 MHz SoftRaid  2x2To  100 Mbps /128 13,99 € HT
-   (soit 16,79 € TTC)
+   (soit 16,79 € TTC) [icn-3600s-high.png]
    En cours de réapprovisionnement.
    Avez-vous pensé aux
    serveurs So You Start ?

---------------------------------------------------------------------------
```

## Autre exemple : suivre la publication d'une nouvelle version d'un logiciel

```yaml
---
name: Nouvelle version de dataverse
url: https://github.com/IQSS/dataverse/releases
filter:
  - xpath: '//div[contains(@class,"markdown-body")]/h1'
  - html2text: re
  - strip
```

Dans cette configuration, j'utilise une expression XPath pour
sélectionner une partie précise de la page. Dans le cas présent, les
titres de niveau 1.

## Encore un autre exemple : surveiller un fichier PDF

Pour cette configuration, je n'ai pas d'exemple concret car je ne l'ai
pas encore utilisé pour mes propres besoins. Mais je sais que la
fonctionnalité est utile, par exemple pour surveiller des textes
normatifs. Voici l'[exemple de la configuration du
logiciel](https://urlwatch.readthedocs.io/en/latest/filters.html#filtering-pdf-documents)
:

```
---
url: https://example.net/pdf-test.pdf
filter:
  - pdf2text
  - strip
```

Pour que cela fonctionne, il faut installer d'autres logiciels
(`pdftotext`) et le module Python qui sert d'interface à ce dernier.

## Et ainsi de suite...

L'outil est vraiment chouette et tout son potentiel réside dans le
fait qu'il peut analyser n'importe quelle texte produit par un
logiciel, que ce texte soit une page web, une liste de fichiers
présents dans un répertoire, etc. La documentation est bien construite
et de nombreux exemples sont disponibles pour mettre en place ses
propres surveillances.
