---
layout: post
title: Un calendrier de l'Avent pour l'informatique documentaire
category: [LIS Advent Calendar]
tags: [Catmandu, Échantillonage]
published: true
comments: true
--- 

Avec le mois de décembre arrive une tradition bien connue des
gourmands : le calendrier de l'Avent. Dans mon cas, cette tradition
n'éveille pas nécessairement mes papilles gustatives, mais elle
réveille à coup sûr ma curiosité. En effet, dans le monde
informatique, c'est l'occasion d'avoir une
[lecture](http://www.perladvent.org/2020/)
[quotidienne](https://24daysindecember.net/), ou encore un [challenge
de programmation](https://adventofcode.com/2020). 

J'ai souvent eu envie d'un calendrier de l'Avent sur des thématiques
en lien avec les sciences de l'information, et plus particulièrement
l'informatique documentaire. Cette année, je vais essayer de réussir
ce challenge. Dans les prochains jours, je vous proposerai donc des
articles traitant d'informatique dans le contexte des sciences de
l'information et des
[GLAM](https://fr.wikipedia.org/wiki/GLAM_(culture)). Les sujets
seront tirés de mon expérience professionnelle, et iront de
l'explication d'un outil à la présentation d'un algorithme.

Et après ces quelques mots d'introduction, il est temps de passer au
premier article de ce calendrier : *l'échantillonnage d'un jeu de
données*. 

## Échantillons!

Il m'est arrivé par le passé d'avoir un jeu de données dont la taille
rendait difficile le traitement : imaginons la mise en place d'un
processus de dédoublonnage pour 12 millions de notices. Dans les
premières phases du projet (exploration des données, mise en place
d'un algorithme, etc.), il est plus facile et rapide de travailler sur
un échantillon.

La méthode naïve pour créer un échantillon est de lire toutes les
données en mémoire, puis de tirer au hasard les *n* éléments de
l'échantillon. Rien de bien compliqué, n'est-ce pas ? Si ce n'est
qu'il faut avoir une machine avec un chouya plus de mémoire vive que
le jeu de données à traiter. 

Une méthode plus efficace est l'échantillonnage par réservoir
([*reservoir
sampling*](https://en.wikipedia.org/wiki/Reservoir_sampling)), où il
s'agira de lire les données, et de remplir au fur et à mesure un
réservoir qui sera notre échantillon final. L'astuce résidant dans la
manière de remplir efficacement le réservoir. Voici une implémentation
simple :

```python
import random


def reservoir_sample(data: list, n: int) -> list:
    reservoir = []
    # On commencer par remplir le réservoir
    for i in range(n):
        reservoir.append(data[i])

	# On distribue les données aléatoirement
    for i in range(n, len(data)):
        j = random.randint(0, i)
        if j < n:
            reservoir[j] = data[i]
                
    return reservoir
```

Et pour tester tout cela, voici un script utilisant la fonction : 

```python
import pathlib

if __name__ == "__main__":
    data = pathlib.Path('/usr/share/dict/french').read_text().split()
    print(reservoir_sample(data, 15))
```

Ce qui nous affichera : `['coquillas', 'guidez', 'recalasse',
'queutées', 'lutâtes', 'éprissions', 'fauconnés', 'sublimassent',
'ruinez', 'résédas', 'routasse', 'ameubliras', 'accidenteraient',
'bifferiez', 'tropical']`

Comme je le disais, il s'agit d'une implémentation simple, et vous
trouverez dans l'[article
Wikipedia](https://en.wikipedia.org/wiki/Reservoir_sampling) d'autres
implémentations plus performantes. 

Pour revenir à l'anecdote initiale, à savoir un jeu de données de
taille importante, je devais donc trouver un moyen efficace
de créer un échantillon avec l'outil que j'utilise habituellement pour
manipuler des données catalographiques, à savoir
[Catmandu](https://librecat.org/). Mais n'ayant pas trouvé l'outil *ad
hoc*, j'ai fini par implémenter un *exporter* réalisant ce travail :

```perl
package Catmandu::Exporter::Sample;

use namespace::clean;
use Catmandu::Sane;
use Algorithm::Numerical::Sample;
use Moo;

with 'Catmandu::Exporter';

our $VERSION = '0.01';

has sampler => ( is => 'lazy' );
has as => ( is => 'ro', default => sub { 'JSON' } );
has sample_size => ( is => 'ro', default => sub { 100 } );

sub _build_sampler {
        my ( $self ) = @_;
        Algorithm::Numerical::Sample::Stream->new( -sample_size => $self->sample_size );
}

sub add {
        my ( $self, $data ) = @_;
        $self->sampler->data( $data );
}

sub commit {
        my ( $self ) = @_;
        my @sample = $self->sampler->extract();
        my $exporter = Catmandu->exporter(
                $self->as,
                file => $self->file,
        );
        $exporter->add_many( \@sample );
}

1;
```

Je n'ai pas eu à implémenter l'algorithme d'échantillonnage puisqu'un
module le faisait déjà :
[`Algorithm::Numerical::Sample`](https://metacpan.org/pod/Algorithm::Numerical::Sample).

Quelques lignes de code, et je pouvais lancer une commande comme :

```shell
catmandu convert MARC --file big_data.mrc --type ISO --encoding utf-8
to Sample --sample_size 1000 --file small_data.mrc
```

Je n'ai jamais pris le temps de publier ce code sur
[GitHub](https://github.com/) et le [CPAN](https://www.cpan.org/),
mais j'imagine que c'est la prochaine étape ! 
