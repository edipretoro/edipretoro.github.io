---
layout: post
title: Calculer la similarité entre des notices UNIMARC (Partie 1)
category: [Informatique documentaire]
tags: [Catalographie, UNIMARC, Python]
published: true
comments: true
--- 

Début de la semaine, je discutais avec un ancien collègue des
difficultés liées aux corrections d'un cours comme « Catalographie »
et les possibilités techniques en matière d'automatisation de tout ou
partie de ces corrections. Ce billet fait partie d'une série de
manière à traiter différents aspects du problème. 

## Un peu de contexte

Avant de rentrer dans les considérations liées à l'automatisation,
décrivons brièvement le contexte :

* Il s'agit d'un cours enseignant
  l'[UNIMARC](https://www.transition-bibliographique.fr/unimarc/formats-unimarc/),
  l'idée étant de permettre aux futurs bibliothécaires-documentalistes
  d'apprivoiser une norme utilisée pour l'échange de notices
  bibliographiques, mais aussi comme format d'encodage ;
* Dans le cadre de l'évaluation, les étudiant·e·s doivent réaliser des
  notices UNIMARC sur base de reproductions des parties nécessaires
  d'un certain nombre de monographies ;
* La réalisation de ces notices se fait « à la main », sans l'aide de
  logiciel autre qu'un traitement de texte ou un éditeur de texte. Pas
  de
  [SIGB](https://fr.wikipedia.org/wiki/Syst%C3%A8me_de_gestion_de_biblioth%C3%A8que
  "Système intégré de gestion de bibliothèque") comme Koha ou PMB. Le
  focus est ici sur l'apprentissage d'une norme, et pas sur
  l'utilisation d'un logiciel précis ;
* La correction se déroule principalement de deux manières :
  1. L'utilisation d'un corrigé réalisé par l'enseignant·e ;
  2. La correction, sous-champ par sous-champ, de ce qui ne figure pas
     sur le corrigé, et la vérification dans la norme si ce qui a été
     fait est correct ou non ; 
* Évidemment, vu la complexité d'une norme comme l'UNIMARC, la
  réalisation d'un corrigé unique et complet est quasiment impossible,
  et un cadrage se fait via les consignes pour réduire le champ des
  possibles. Mais malgré tout, les corrections sont longues !
  
Voilà pour tout ce que vous n'avez jamais voulu savoir sur la
correction de notices UNIMARC dans le contexte d'un cours !

## Et l'automatisation dans tout ça ?

Bon, concrètement, comment peut-on obtenir un peu d'aide de
l'informatique en la matière ?

### La version minimale

À minima, lors de la comparaison du corrigé avec la notice d'un·e
étudiant·e, nous pouvons utiliser un logiciel comme
[`diff`](http://www.linux-france.org/article/man-fr/man1/diff-1.html)
sur une représentation textuelle des notices : les différences sont
mises en avant, et il ne reste plus qu'à trancher s'il s'agit d'une
erreur ou non. 

Concrètement, nous avons un corrigé (pour les besoins de ce billet, je
n'ai pas fait l'effort de produire un corrigé, j'ai juste récupéré la
[notice UNIMARC de la
BnF](https://catalogue.bnf.fr/ark:/12148/cb38805124b) via Z39.50) :

```
01203nam  2200289   4500
001 FRBNF38805124000000X
003 http://catalogue.bnf.fr/ark:/12148/cb38805124b
010    $a 2-226-13190-6 $b br. $d 24,90 EUR
020    $a FR $b 00209751
100    $a 20020314d2002    m  y0frey50      ba
101 1  $a fre $c eng
102    $a FR
105    $a ||||z   00|a|
106    $a r
181  0 $6 01 $a i  $b xxxe  
181    $6 02 $c txt $2 rdacontent
182  0 $6 01 $a n
182    $6 02 $c n $2 rdamedia
200 1  $a Dreamcatcher $b Texte imprimé $e roman $f Stephen King $g trad. de l'américain par William Olivier Desmond
210    $a Paris $c A. Michel $d 2002 $e 18-Saint-Amand-Montrond $g Bussière Camedan impr.
215    $a 684 p. $c couv. ill. en coul. $d 24 cm
454  1 $t Dreamcatcher
686    $a 803 $2 Cadre de classement de la Bibliographie nationale française
700    $3 11909772 $o ISNI0000000121446296 $a King $b Stephen $f 1947-.... $4 070
702    $3 11899860 $o ISNI0000000121235480 $a Desmond $b William Olivier $f 1939-2013 $4 730
801  0 $a FR $b FR-751131015 $c 20020314 $g AFNOR $h FRBNF38805124000000X $2 intermrc
930    $5 FR-751131010:38805124001001 $a 2002-62721 $b 759999999 $c Tolbiac - Rez de Jardin - Littérature et art - Magasin $d O
```

Et nous avons une version à corriger (ici aussi, je n'ai pas pris la
peine de produire une notice erronée, j'ai été pioché une notice dans
un jeu de données provenant d'une migration réalisée il y a quelques
années) :

```
00333 a   2200121   4500
606    $a Roman suspense
995    $r a $k R-1 $f 020969-L
010    $d 27,95 EUR $a 2-226-13190-6
702    $a Desmond $4 070 $b William Olivier
200    $a Dreamcatcher
300    $a Dreamcatcher
700    $4 070 $a KING $b Stephen
215    $c ill. en coul., couv. ill. en coul. $d 24 $a 684 p.
```

Et ensuite, en utilisant `diff`, nous obtenons :

```
diff -u corrige_sample_01.txt exemple_sample_01.txt
--- corrige_sample_01.txt	2020-06-20 10:19:41.000000000 +0200
+++ exemple_sample_01.txt	2020-06-20 10:19:53.000000000 +0200
@@ -1,24 +1,10 @@
-01203nam  2200289   4500
-001 FRBNF38805124000000X
-003 http://catalogue.bnf.fr/ark:/12148/cb38805124b
-010    $a 2-226-13190-6 $b br. $d 24,90 EUR
-020    $a FR $b 00209751
-100    $a 20020314d2002    m  y0frey50      ba
-101 1  $a fre $c eng
-102    $a FR
-105    $a ||||z   00|a|
-106    $a r
-181  0 $6 01 $a i  $b xxxe  
-181    $6 02 $c txt $2 rdacontent
-182  0 $6 01 $a n
-182    $6 02 $c n $2 rdamedia
-200 1  $a Dreamcatcher $b Texte imprimé $e roman $f Stephen King $g trad. de l'américain par William Olivier Desmond
-210    $a Paris $c A. Michel $d 2002 $e 18-Saint-Amand-Montrond $g Bussière Camedan impr.
-215    $a 684 p. $c couv. ill. en coul. $d 24 cm
-454  1 $t Dreamcatcher
-686    $a 803 $2 Cadre de classement de la Bibliographie nationale française
-700    $3 11909772 $o ISNI0000000121446296 $a King $b Stephen $f 1947-.... $4 070
-702    $3 11899860 $o ISNI0000000121235480 $a Desmond $b William Olivier $f 1939-2013 $4 730
-801  0 $a FR $b FR-751131015 $c 20020314 $g AFNOR $h FRBNF38805124000000X $2 intermrc
-930    $5 FR-751131010:38805124001001 $a 2002-62721 $b 759999999 $c Tolbiac - Rez de Jardin - Littérature et art - Magasin $d O
+00333 a   2200121   4500
+606    $a Roman suspense
+995    $r a $k R-1 $f 020969-L
+010    $d 27,95 EUR $a 2-226-13190-6
+702    $a Desmond $4 070 $b William Olivier
+200    $a Dreamcatcher
+300    $a Dreamcatcher
+700    $4 070 $a KING $b Stephen
+215    $c ill. en coul., couv. ill. en coul. $d 24 $a 684 p.
 

Diff finished.  Sat Jun 20 10:20:03 2020
```

Si vous utilisez un outil en ligne comme
[Diffchecker](https://www.diffchecker.com/), vous en aurez une [version
plus agréable à l'œil](https://www.diffchecker.com/4kU7i1aX) :

![Résultat d'un diff entre deux notices
UNIMARC](/assets/img/diff_between_unimarc_records.png)

Vous constaterez que si la mise en évidence est efficace, la notion de
« différence » entre les deux notices n'est pas adaptée. C'était
pourtant la méthode utilisée. 

### Une première exploration

Étant donné le contexte spécifique, et tout particulièrement le fait
que le jeu de données est très limité (une petite vingtaine de notices
provenant des étudiant·e·s correspondant à un corrigé), nous allons
pouvoir commencer par une approche très naïve de la comparaison de
notices, et plus généralement d'enregistrements provenant de bases de
données : la comparaison champ par champ avec un algorithme de
similarités du texte.

Voici le code permettant de calculer la similarité entre deux notices
:

``` python
from typing import Callable, Iterable, Tuple
import jellyfish
import pymarc


def record_walker(record):
    for f in record.get_fields():
        for sf in f:
            yield (f.tag, *sf)


def get_records_similarity(rec_a, rec_b, sim_method = jellyfish.jaro_winkler_similarity):
    last = 0
    sim = 0
    for idx, d in enumerate(record_walker(rec_a), start=1):
        for field in rec_b.get_fields(d[0]):
            subfield = field.get_subfields(d[1])
            if subfield:
                subfield = subfield[0]
                sim += sim_method(d[2], subfield)

            last = idx

    return sim / last
```

Deux fonctions : la première, `record_walker`, est juste un raccourci
permettant de générer une liste de tuples sur base d'une notice,
c'est-à-dire un objet de type `pymarc.Record`). Ces tuples contiennent
le code de champ, le code de sous-champ et la valeur stockée. Eh oui !
Comme c'est une première implémentation, l'algorithme ne tient pas
compte des indicateurs. La seconde, `get_records_similarity()`, est la
fonction qui fait le travail proprement dit. Elle prend deux notices
en argument, et permet un troisième argument qui est une fonction.
C'est cette fonction qui sera utilisée pour calculer la similarité
entre les différentes parties des notices. Par défaut, nous utilisons
la fonction `jaro_winkler_similarity` du *package* **jellyfish**, qui
retourne un nombre compris entre 0 et 1 : 0 signifiant qu'il n'y a rien
de commun entre les deux chaînes de caractère, et 1 signifiant
qu'elles sont identiques. 

Pour le reste, l'algorithme employé est assez simple :

* Nous parcourons les champs et sous-champs de la première notice ;
* Pour chaque champ, nous obtenons le champ correspondant dans la
  deuxième notice ;
* Pour ce champ, nous obtenons les sous-champs correspondants ;
* Et nous additionnons les résultats du calcul de similarité entre les
  valeurs de ces sous-champs ;
* Quand nous avons fini de parcourir les champs de la première notice,
  nous retournons le rapport entre la somme des similarités entre
  champs, et le nombre de champs dans la première notice. 

### Une première application

Voici un utilitaire en ligne de commande qui affichera le percentage
de similarité entre une notice servant de référence et d'autres
notices :

``` python
if __name__ == '__main__':
    arg_parser = argparse.ArgumentParser(description=__doc__)
    arg_parser.add_argument(
        '-b', '--basis',
        required=True,
        type=pathlib.Path,
        help="The path to the bibliographic to use as a basis for the "
             "computation."
    )
    arg_parser.add_argument(
        '-r', '--records',
        required=True,
        type=pathlib.Path,
        action="append",
        help="The path to the bibliographic records to compute similarities "
             "with."
    )
    cli = arg_parser.parse_args()

    with cli.basis.open(mode='rb') as a:
        rec_a = next(pymarc.MARCReader(a, to_unicode=True))
        for record in cli.records:
            with record.open(mode='rb') as b:
                rec_b = next(pymarc.MARCReader(b, to_unicode=True))
                print(f"Similarity between {cli.basis.name} and "
                      f"{record.name} is "
                      f"{get_records_similarity(rec_a, rec_b):.2%}")
```

Voici un exemple d'exécution du script avec les deux notices
présentées plus haut :

``` shell
./sim_unimarc_records.py -b corrige_sample_01.mrc -r exemple_sample_01.mrc
```

Et le résultat :

``` text
Similarity between corrige_sample_01.mrc and exemple_sample_01.mrc is 20.88%
```

Comme vous pouvez le constater, le score obtenu est assez bas. C'est
logique dans la mesure où la notice de la BnF est bien plus complète.
Voici le résultat si la comparaison se fait dans l'autre sens :

``` text
Similarity between exemple_sample_01.mrc and corrige_sample_01.mrc is 61.40%
```

Pour une première exploration, ce n'est pas si mal. Et il y a de la
marge pour améliorer l'algorithme !

Voici la version complète du script, sous forme de [Gist sur
GitHub](https://gist.github.com/edipretoro/37c843fc25760e2f2845f9cd45162ffb).
Un outil en moins de 100 lignes de Python qui permet de se faire une première
idée des similitudes entre notices bibliographiques, que ce soit pour
se faciliter la vie en matière de correction, ou pour détecter
d'éventuels plagiats.
