---
layout: post
title: Inventaire et importation de données
category: [LIS Advent Calendar]
tags: [Catmandu, BnF, Z39.50]
published: true
comments: true
--- 

Pour le billet d'aujourd'hui, je vous propose de construire un petit
outil qui peut être bien pratique lors la réalisation
d'inventaires. 

Mais avant de plonger dans le code, quelques éléments de contexte. Il
n'y a pas si longtemps, j'étais prof dans une Haute École formant des
Bacheliers « Bibliothécaire-Documentaliste ». J'ai ainsi régulièrement
eu des rencontres avec des étudiant·e·s souhaitant informatiser un
fonds documentaire, et qui partait, dans le meilleur des cas, d'un
fichier Excel et de piles de livres. Un des conseils que j'ai
régulièrement donnés était d'abandonner le fichier Excel (dont la
qualité était souvent très mauvaise), de faire un inventaire du fonds
en scannant les code-barres EAN (qui sont aussi l'ISBN) des bouquins,
de récupérer les notices associées à ces code-barres via du Z39.50, de
nettoyer les données en fonction des besoins, puis d'importer le tout
dans un SIGB (pour autant que ce soit la solution retenue, dans
certains cas, le fichier Excel restait la seule solution
(*sic*)). Évidemment, ce conseil pouvait faire peur, puisque cela
demandait de sortir de connu (faire un inventaire avec Excel) et de
toucher à des aspects informatiques pour lesquels nos étudiant·e·s
n'étaient pas nécessairement formés. Ce billet est donc écrit sur base
de ce *use case* un peu particulier. Je compte aussi utiliser cet
outil pour gérer ma bibliothèque sans installer de SIGB :)

## Faire un inventaire

Pour l'inventaire, ma solution est assez simple puisqu'elle s'appuie
sur un lecteur de code-barres et sur un simple fichier texte. Le
résultat final ressemblera à ceci :

```text
isbn
9782804130534
9782804166465
9782804156039
9782710121480
9782212092882
9782804111489
9782843650895
9782843650871
9782804134587
9782804169107
9782804156152
9782807307858
9782804181987
9782910227838
9782710124313
9782710117643
9782212559279
```

Rien de bien compliqué : un fichier CSV avec une seule colonne nommée
`isbn`. 

## Récupération des notices en Z39.50

Pour récupérer les notices, je peux utiliser les services de
[MoCCAM-en-ligne](http://www.moccam-en-ligne.fr/), mais évidemment,
dans mon cas, je préfère développer un outil spécifique, plus proche
de mes besoins, et que je peux faire évoluer en fonction des
situations rencontrées. 

Typiquement, je veux obtenir deux fichiers après le traitement de mon
script :

1. Le fichier ISO2709 avec les données UNIMARC ;
2. La liste des ISBN qui n'ont pas été retrouvés via Z39.50, et que je
   devrai traiter autrement. 
   
Pour réaliser ce travail, je continuerai à utiliser
[Catmandu](https://librecat.org/), mais cette fois-ci, je n'utiliserai
plus son interface en ligne de commande, mais son interface Perl. 

Voici le script : 

```perl
#!/usr/bin/env perl

use Modern::Perl;
use Catmandu;

my $input_filename = shift(@ARGV) or die "Usage: $0 filename.csv";

my $importer_csv = Catmandu->importer('CSV', file => $input_filename);

my $not_found_exporter = Catmandu->exporter(
	'YAML', 
	file => "./isbn_not_found.$$.yml"
);
my $exporters = Catmandu->exporter(
	'Multi',
	exporters => [
		Catmandu->exporter('YAML', file => "./books.$$.yml"),
		Catmandu->exporter(
			'MARC',
			type => 'ISO',
			file => "./books.$$.mrc"
		)
	]
);

my @found;
my @not_found;
$importer_csv->each(sub {
	my $item = shift;
	my $record = get_record($item->{isbn});
	if (defined $record) {
	  push @found, get_record($item->{isbn});
	} else {
	  push @not_found, $item
	}
});

$exporters->add_many(\@found);
$exporters->commit;

$not_found_exporter->add_many(\@not_found);
$not_found_exporter->commit;

sub get_record() {
  my $isbn = shift(@_);
  my $importer_z3950 = Catmandu->importer(
	  'Z3950',
	  user => 'Z3950',
	  password => 'Z3950_BNF',
	  host => 'z3950.bnf.fr',
	  port => 2211,
	  databaseName => 'TOUT-ANA1-UTF8',
	  preferredRecordSyntax => 'Unimarc',
	  queryType => 'PQF',
	  handler => 'UNIMARC',
	  query => "\@attr 1=7 $isbn"
  );
  
  my $record = $importer_z3950->first;

  return $record;
}
```

Nous commençons par récupérer le nom du fichier contenant les ISBN via
la ligne `my $input_filename = shift(@ARGV) or die "Usage: $0
filename.csv";`. Si le programme ne reçoit de fichier CSV, il se
termine en affichant un message sommaire. 

Nous créons ensuite un *importer* pour ce fichier CSV et plusieurs
*exporters* : un pour les ISBN qui n'auront pas été retrouvés, et un
*exporter* multiple qui écrira les ISBN trouvés aussi bien en YAML
qu'en ISO2709. 

Nous passons au traitement proprement, à savoir un appel à la fonction
`each` de notre *importer*. Comme son nom le suggère, l'importeur va
appeler une fonction pour chacun des enregistrements du fichier
CSV. Cette fonction va elle-même récupérer l'ISBN, et appeler une
autre fonction qui effectuera la recherche et la récupération de
l'éventuelle notice. Nous vérifions si nous avons un résultat, et si
nous alimentons la bonne variable, `@found` ou `@not_found`. 

Et finalement, nous passons ces variables aux *exporters* pour produire
les fichiers nécessaires. 

## Et un petit test...

Sur base de notre inventaire, voici les résultats :

* Le fichier avec les ISBN qui n'ont pas été trouvés :

```yaml
---
isbn: '9782804156039'
...
---
isbn: '9782710121480'
...
---
isbn: '9782212092882'
...
---
isbn: '9782710124313'
...
---
isbn: '9782710117643'
...
```

* Les notices retrouvées (affichées avec `yaz-marcdump` et éditées
  pour éviter un affichage inutile) : 

```text
02098cam  2200433   4500
001 FRBNF422870840000004
003 http://catalogue.bnf.fr/ark:/12148/cb422870849
010    $a 978-2-8041-3053-4 $b br.
100    $a 20101006d2009    m  y0frey50      ba
101 0  $a fre
102    $a BE
105    $a ||||z   00|y|
106    $a r
181  0 $6 01 $a i  $b xxxe  
181    $6 02 $c txt $2 rdacontent
182  0 $6 01 $a n
182    $6 02 $c n $2 rdamedia
200 1  $a Des manuels scolaires pour apprendre $b Texte imprimé $e concevoir, évaluer, utiliser $f François-Marie Gerard, Xavier Roegiers $g avec la collab. de Christiane Bosman, Georges Hoyos, Georges Huget $g illustrations de Yolanda Georgette
205    $a 2e. éd.
210    $a Bruxelles $c De Boeck $d 2009
215    $a 1 vol. (422 p.) $c ill. $d 24 cm
225    $a Pédagogies en développement
300    $a Bibliogr. p. 399-405. Index
300    $a L'ouvrage porte par erreur : ISSN 0777-5245
312    $a Titre d'une autre édition : Concevoir et évaluer des manuels scolaires
410  0 $0 34287117 $t Pédagogies en développement $x 1371-1598 $d 2009
540 1  $a Concevoir et évaluer des manuels scolaires
606    $3 11952430 $a Manuels d'enseignement $x Édition $2 rameau
606    $3 11948606 $a Manuels d'enseignement $3 11975815 $x Évaluation $2 rameau
606    $3 11948606 $a Manuels d'enseignement $3 11975801 $x Utilisation $2 rameau
676    $a 371.32 $v 23
700    $3 12779278 $o ISNI000000011747725X $a Gerard $b François-Marie $f 1953-.... $4 070
701    $3 12413541 $o ISNI0000000116517862 $a Roegiers $b Xavier $f 1953-.... $4 070
702    $3 12400663 $o ISNI0000000116485522 $a Bosman $b Christiane $4 205
702    $3 12400664 $o ISNI0000000000382820 $a Hoyos $b Georges $4 205
702    $3 12213014 $o ISNI0000000066449274 $a Huget $b Georges $4 205
702    $3 13520888 $o ISNI0000000001105331 $a Georgette $b Yolanda $4 205
801  0 $a FR $b FR-751131015 $c 20101006 $g AFNOR $h FRBNF422870840000004 $2 intermrc
930    $5 FR-751131007:42287084001001 $a 2011-21231 $b 759999999 $c Tolbiac - Haut de Jardin - Philosophie, histoire, sciences de l'homme - Salle J - Libre accès $d N

.
.
.

01580cam  2200385   4500
001 FRBNF442422590000009
003 http://catalogue.bnf.fr/ark:/12148/cb44242259w
010    $a 978-2-212-55927-9 $b br. $d 10 EUR
020    $a FR $b 01506522
073  0 $a 9782212559279
100    $a 20141215d2014    m  y0frey50      ba
101 0  $a fre
102    $a FR
105    $a ||||z   00|y|
106    $a r
181  0 $6 01 $a i  $b xxxe  
181    $6 02 $c txt $2 rdacontent
182  0 $6 01 $a n
182    $6 02 $c n $2 rdamedia
200 1  $a J'aide mon enfant à mieux apprendre $b Texte imprimé $f Bruno Hourst $g illustrations de Jilème
210    $a Paris $c Eyrolles $d DL 2014 $e 53-Mayenne $g Impr. Jouve
215    $a 1 vol. (269 p.) $c ill. $d 21 cm
225    $a Eyrolles pratique $e vie quotidienne
300    $a Bibliogr. p. 259-261
300    $a 3e tirage 2014
410  0 $0 39092399 $t Eyrolles pratique $x 1763-2552 $d 2014
606    $3 13318807 $a Éducation $3 17804623 $x Participation des parents $2 rameau
606    $3 11932674 $a Psychologie de l'éducation $2 rameau
606    $3 12203321 $a Devoirs à la maison $2 rameau
676    $a 371.302 81 $v 23
686    $a 370 $2 Cadre de classement de la Bibliographie nationale française
700    $3 13169607 $o ISNI0000000117841832 $a Hourst $b Bruno $f 1949-.... $4 070
702    $3 13742901 $o ISNI0000000374134039 $a Jilème $f 1971-.... $4 440
801  0 $a FR $b FR-751131015 $c 20141215 $g AFNOR $h FRBNF442422590000009 $2 intermrc
930    $5 FR-751131007:44242259001001 $a 2014-302767 $b 759999999 $c Tolbiac - Rez de Jardin - Philosophie, histoire, sciences de l'homme - Magasin $d O
```

Dans la version de l'outil actuel, je n'interroge qu'un seul serveur
Z39.50, cela de la [BnF](http://www.bnf.fr), mais je pourrais
facilement ajouter d'autres serveurs pour essayer d'éviter à tout prix
de cataloguer ces ouvrages :p
