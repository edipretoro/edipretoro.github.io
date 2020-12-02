---
layout: post
title: Migrons des données !
category: [LIS Advent Calendar]
tags: [Catmandu, Migration]
published: true
comments: true
--- 

S'il y a bien une opération que j'ai été amenée à réaliser
régulièrement au fil de ma carrière, c'est de migrer des données, et
plus particulièrement migrer des données catalographiques. Que ce soit
de passer de
[Winisis](https://wayback.archive-it.org/all/20151215051648/http://portal.unesco.org/ci/en/ev.php-URL_ID=5330&URL_DO=DO_TOPIC&URL_SECTION=201.html)
à [Koha](https://koha-community.org/),
d'[Antigone](http://web.archive.org/web/20100119160430/http://bibonline.homeip.net/webant/antigone.php)
à [PMB](https://www.sigb.net/), de [Microsoft
Access](https://www.microsoft.com/fr-fr/microsoft-365/access) à PMB,
etc. 

Et avec le temps, ma méthode a évoluée : du script *ad hoc* à
l'utilisation de [Catmandu](https://librecat.org/). Et si les scripts
m'ont été fort utiles pour faire mes armes en tant que programmeur, je
dois bien reconnaître que j'apprécie maintenant de pouvoir travailler
avec Catmandu pour faire cela ! C'est bien plus puissant, simple et
cela me fait gagner énormément de temps. 

Dans le billet d'aujourd'hui, je vous propose donc de migrer un jeu de
données avec Catmandu. Et pour comme jeu de données, je vais
simplement prendre le fichier `records.csv` d'un [des jeux de données
de la *British
Library*](https://www.bl.uk/collection-metadata/downloads) :
<https://www.bl.uk/bibliographic/downloads/HGWellsResearcherFormat_202010_csv.zip>.

## Prendre connaissance des données

La première étape est simplement de prendre connaissance des données à
disposition. Avec Catmandu, je peux afficher les données de la manière
suivante :

```shell
catmandu convert CSV --file records.csv to YAML
```

Évidemment, cette commande affiche toutes les données, ce qui n'est
pas idéal pour un premier contact. 

Mais la commande *convert* accepte un argument permettant de limiter
le nombre de données à traiter : 

```shell
catmandu convert --total 1 CSV --file records.csv to YAML
```

Le résultat est maintenant plus facile à lire :

```yaml
---
All names: Archer, William, critic [person]
Archival Resource Key: ''
BL record ID: '000105518'
BL shelfmark: 03558.de.30
BNB number: ''
Content type: Language material ; Text
Country of publication: England
Date of creation/publication: '1917'
Dates associated with name: ''
Dewey classification: ''
Edition: ''
Genre: ''
ISBN: ''
Languages: English
Material type: Volume
Name: Archer, William
Notes: ''
Number within series: ''
Physical description: 126 pages (8°)
Place of publication: London
Provenance: ''
Publisher: Watts
Role: critic
Series title: ''
Title: 'God and Mr. Wells: a critical examination of ''God, the Invisible King.'''
Topics: Wells, H. G. (Herbert George), 1866-1946
Type of name: person
Type of resource: Monograph
Variant titles: ''
...
```

Sur base de cette notice, on peut voir qu'il y a des champs vides. Une
analyse de ces champs peut amener une meilleure connaissance des
données à migrer. Catmandu permet également d'offrir une analyse
rapide via
[Catmandu-Breaker](https://metacpan.org/pod/Catmandu::Breaker). Le
traitement se fait en deux étapes :

1. La transformation des données dans le format *Breaker* via la
   commande suivante : `catmandu convert CSV --file records.csv to
   Breaker --file hg_wells.breaker`
2. L'analyse proprement dite : `catmandu breaker hg_wells.breaker`. 

Sur base du fichier, voici les résultats : 

```text
| name       | count | zeros | zeros% | min | max | mean               | variance | stdev | uniq~  | uniq%     | entropy      |
|------------|-------|-------|--------|-----|-----|--------------------|----------|-------|--------|-----------|--------------|
| #          | 1753  |       |        |     |     |                    |          |       |        |           |              |
| All        | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 570    | 32.6      | 4.8/10.6     |
| Archival   | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 60     | 3.4       | 0.4/10.8     |
| BL         | 3506  | 0     | 0.0    | 2   | 2   | 2                  | 0.0      | 0.0   | 3242   | 92.5      | 11.3/11.8    |
| BNB        | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 698    | 39.9      | 4.7/10.8     |
| Content    | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 5      | 0.3       | 0.4/10.8     |
| Country    | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 51     | 2.9       | 2.4/10.8     |
| Date       | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 197    | 11.2      | 6.4/10.1     |
| Dates      | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 93     | 5.3       | 1.7/10.8     |
| Dewey      | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 92     | 5.3       | 2.4/10.8     |
| Edition    | 208   | 1545  | 88.1   | 0   | 1   | 0.118653736451797  | 0.1      | 0.3   | 92     | 44.4      | 1.2/10.8     |
| Genre      | 371   | 1382  | 78.8   | 0   | 1   | 0.211637193382772  | 0.2      | 0.4   | 64     | 17.3      | 1.6/10.8     |
| ISBN       | 723   | 1030  | 58.8   | 0   | 1   | 0.412435824301198  | 0.2      | 0.5   | 698    | 96.6      | 4.8/10.7     |
| Languages  | 1699  | 54    | 3.1    | 0   | 1   | 0.969195664575014  | 0.0      | 0.2   | 44     | 2.6       | 1.2/10.8     |
| Material   | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 12     | 0.7       | 0.7/10.8     |
| Name       | 1693  | 60    | 3.4    | 0   | 1   | 0.965772960638905  | 0.0      | 0.2   | 285    | 16.9      | 2.6/10.7     |
| Notes      | 469   | 1284  | 73.2   | 0   | 1   | 0.267541357672561  | 0.2      | 0.4   | 419    | 89.4      | 3.0/10.8     |
| Number     | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 253    | 14.5      | 1.8/10.8     |
| Physical   | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 1178   | 67.2      | 9.1/10.6     |
| Place      | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 238    | 13.6      | 3.7/10.7     |
| Provenance | 11    | 1742  | 99.4   | 0   | 1   | 0.0062749572162008 | 0.0      | 0.1   | 12 (!) | 109.1 (!) | 0.1/10.8 (!) |
| Publisher  | 1517  | 236   | 13.5   | 0   | 1   | 0.865373645179692  | 0.1      | 0.3   | 512    | 33.8      | 7.0/10.4     |
| Role       | 273   | 1480  | 84.4   | 0   | 1   | 0.155733029092983  | 0.1      | 0.4   | 10     | 3.7       | 0.7/10.8     |
| Series     | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 330    | 18.8      | 3.5/10.7     |
| Title      | 1712  | 41    | 2.3    | 0   | 1   | 0.976611523103252  | 0.0      | 0.2   | 965    | 56.4      | 9.1/10.3     |
| Topics     | 673   | 1080  | 61.6   | 0   | 1   | 0.383913291500285  | 0.2      | 0.5   | 373    | 55.5      | 3.6/10.7     |
| Type       | 3506  | 0     | 0.0    | 2   | 2   | 2                  | 0.0      | 0.0   | 12     | 0.3       | 1.3/11.8     |
| Variant    | 1753  | 0     | 0.0    | 1   | 1   | 1                  | 0.0      | 0.0   | 246    | 14.1      | 2.4/10.7     |
Overflow warning - probably your dataset is too small for an accurate uniq~, uniq% and entropy count...
```

Cela nous permet de dégager quelques éléments :

* Nous avons 1753 notices ;
* Nous avons quelques champs qui sont régulièrement vides :
  *Provenance*, *Edition*, etc. 
  
Il y a encore probablement pas mal d'actions que nous pourrions faire
sur un jeu de données plus complexe, mais à ce stade, nous pouvons
passer à l'étape suivante. 

## Établir une table de correspondance

Pour migrer les données, je vais devoir faire correspondre un champ du
fichier d'entrée, à un champ dans le format de sortie. Dans mon cas,
je vais produire un fichier UNIMARC, qui est souvent facilement
importable dans les logiciels de gestion de bibliothèque. 

Dans le fichier `records.csv` la première ligne contient le nom des
colonnes. Je peux la récupérer de la manière suivante : 

```shell
catmandu convert --total 1 CSV --header 0 --file records.csv to YAML
```

Ce qui nous donne le résultat suivant :

```yaml
---
'0': BL record ID
'1': Type of resource
'10': Role
'11': All names
'12': Title
'13': Variant titles
'14': Series title
'15': Number within series
'16': Country of publication
'17': Place of publication
'18': Publisher
'19': Date of creation/publication
'2': Content type
'20': Edition
'21': Physical description
'22': Dewey classification
'23': BL shelfmark
'24': Topics
'25': Genre
'26': Languages
'27': Notes
'28': Provenance
'3': Material type
'4': BNB number
'5': Archival Resource Key
'6': ISBN
'7': Name
'8': Dates associated with name
'9': Type of name
...
```

Je peux maintenant compléter ce « tableau » :

```yaml
---
'0': BL record ID → 001
'12': Title → 200$a
'17': Place of publication 210$a
'18': Publisher → 210$c
'19': Date of creation/publication → 210$d
'21': Physical description → 215$a
'27': Notes → 300$a
'6': ISBN → 010$a
'7': Name → 700$a, 700$b, 700$4 → 070
...
```

Bon, comme vous pouvez le voir, j'ai fait preuve de paresse^W
pédagogie : les autres colonnes sont laissées comme exercice au
lecteur !

## La migration proprement dite

Catmandu fournit un langage, nommé
[*Fix*](https://github.com/LibreCat/Catmandu/wiki/Fix-language)
permettant de manipuler les données. 

Voici ce que cela donne :

```perl
vacuum()
marc_add('001', ind1 => ' ', ind2 => ' ', _ => $.0)
marc_add('010', ind1 => ' ', ind2 => ' ', a => $.6)
marc_add('200', ind1 => '1', ind2 => ' ', a => $.12)
marc_add('210', ind1 => ' ', ind2 => ' ', a => $.17, c => $.18, d => $.19)
marc_add('215', ind1 => ' ', ind2 => ' ', a => $.21)
marc_add('300', ind1 => ' ', ind2 => ' ', a => $.27)
split_field($.7, ', ')
if exists($.7.0)
   trim($.7.0)
   trim($.7.1)
   marc_add('700', ind1 => ' ', ind2 => '1', a => $.7.0, b => $.7.1, 4 => '070')
end
```

* La commande `vacuum()` permet de supprimer les champs vides ;
* `marc_add` permet de créer un champ ISO2709 et ses
sous-champs ;
* L'expression `$.0` signifie de prendre la valeur de la colonne 0
(donc la première colonne) ;
* `split_field` permet découper un champ sur base d'une séquence de
caractère ;
* Et enfin, `trim` permet de supprimer les éventuels espaces en début
et en fin d'une chaîne de caractère.

Et voici la commande à utiliser pour produire un fichier UNIMARC au
format ISO2709 : 

```shell
catmandu convert --start 1 CSV --header 0 --file records.csv to MARC --fix bl_records.fix --type ISO --encoding utf-8 --file bl_records.mrc
```

Et nous pouvons vérifier le résultat avec `yaz-marcdump` : 

```text
00241     2200085   4500
001 000105518
200 1  $a God and Mr. Wells: a critical examination of 'God, the Invisible King.'
210    $a London $c Watts $d 1917
215    $a 126 pages (8°)
700  1 $a Archer $b William $4 070

00322     2200097   4500
001 000128859
200 1  $a Commentary; or Reply to H. G. Wells: his views in the book The common sense of war and peace
210    $a Liverpool $d 1970
215    $a 2 parts, ff 17, 20, 33 cm
300    $a Reproduced from typewriting
700  1 $a Ashton $b Charles Frederick $4 070

00414     2200097   4500
001 000194501
200 1  $a The Journal of a Disappointed Man ... With an introduction by H. G. Wells
210    $a London $c Chatto & Windus $d 1919
215    $a x, 312 pages (8°)
300    $a Other edition: The Journal of a Disappointed Man ... With an introduction by H. G. Wells. pp. x. 346. Chatto & Windus: London, 1923. 8º
700  1 $a Barbellion $b W. N. P. $4 070
```

Et j'ai maintenant un fichier que je peux importer dans le SIGB de mon
choix. 

Évidemment, le fichier produit est loin d'être parfait (le complément
du titre fait partie du titre, tous les champs ne sont pas traités, la
description matérielle pourrait être améliorée aussi), mais en quelques
lignes de *Fix*, nous avons déjà un résultat utile et importable. La tâche
équivalente dans un langage de programmation, que cela Perl ou Python,
prendra beaucoup plus de temps à écrire. Mais je garderai ces langages
pour réaliser d'autres opérations, plus complexes, en lien avec la
migration : dédoublonnage, nettoyage des données, etc. 
