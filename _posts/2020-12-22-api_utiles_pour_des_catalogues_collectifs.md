---
layout: post
title: Quelques APIs utiles pour des catalogues collectifs
category: [LIS Advent Calendar]
tags: [Catalogue collectif, API, DAIA, PAIA]
published: true
comments: true
---

Un problème classique pour les catalogues collectifs est que l'
information concernant la disponibilité des ressources est perdue, et
en terme d'UX pour le lecteur, ce n'est pas idéal. Une solution simple
est de s'appuyer sur une API pour obtenir cette information en temps
réel. Alors, pas de code aujourd'hui, mais quelques liens vers une API
qui se propose justement de résoudre ce problème :

- [La norme](https://gbv.github.io/daia/daia.html) : *Document
  Availability Information API* (DAIA) ; 
- Une implémentation côté serveur avec
  [*Plack::App::DAIA*](https://metacpan.org/pod/Plack::App::DAIA) et
  côté client avec [*DAIA*](https://metacpan.org/pod/DAIA). 
  
Le protocole est assez simple à implémenter pour le SIGB de votre
choix. 
