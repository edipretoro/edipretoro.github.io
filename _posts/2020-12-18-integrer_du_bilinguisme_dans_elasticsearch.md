---
layout: post
title: Intégrer du bilinguisme dans Elasticsearch en exploitant un thésaurus
category: [LIS Advent Calendar]
tags: [Python, Elasticsearch, SPARQL, SKOS]
published: true
comments: true
---

Aujourd'hui, je vous propose de voir comment exploiter un thésaurus
pour configurer Elasticsearch afin qu'il offre du bilinguisme dans ses
résultats de recherche. 

## Elasticsearch

[Elasticsearch](https://www.elastic.co/) est une base de données
orientée document qui s'appuie sur
[Solr](https://lucene.apache.org/solr/) et qui se positionne comme un
moteur de recherche et d'analyse pour entreprise. Ce logiciel est
particulièrement intéressant pour offrir des recherches en texte
intégral pour des bibliothèques numériques.

Elasticsearch intègre une fonctionnalité assez utile, les
[synonymes](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-synonym-tokenfilter.html),
qui permet de configurer la manière dont le moteur de recherche
indexera son contenu. En lui fournissant une liste de synonymes sous
la forme suivante :

```csv
chien,hond
maison,huis
.
.
.
```

Nous aurons un moteur de recherche qui nous retournera les documents
contenant *huis* ou *maison* pour la requête *maison*. 

Évidemment, pour que cela soit intéressant, il faut avoir une liste de
synonymes en lien avec les documents indexés. Et pour cela, rien de
tel que les thésaurus !

## Le thésaurus

Dans ce qui suit, je vais m'appuyer sur un thésaurus converti en
[SKOS](https://www.w3.org/TR/2009/REC-skos-reference-20090818/) pour
produire la liste des synonymes. Ce qui m'intéresse est la propriété
`skos:exactMatch`. 

## Le code 

Voici le code de l'outil :

```python
#!/usr/bin/env python

import argparse
import re
from pathlib import Path

from rdflib import Graph


def _stripping_square_bracket(s):
    s = re.sub(r"\[[^\[\]]+]", "", s)

    return s


def _stripping_slashes(s):
    s = s.replace('/', ' ').replace('\\', ' ')

    return s


def _stripping_comma(s):
    s = s.replace(',', '')

    return s


def synonym_cleaning(synonym):
    actions = {
        'lowercase': str.lower,
        'strip_comma': _stripping_comma,
        'strip_square_bracket': _stripping_square_bracket,
        'strip_slashes': _stripping_slashes,
    }

    for action in actions:
        synonym = actions[action](synonym)

    return synonym


def generate_bilingual(graph, output):
    qres = graph.query(f"""
        PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
        PREFIX thesaurus: <http://localhost/thesaurus/skos/term#>

        SELECT ?label ?equivalent WHERE {% raw %}{{ {% endraw %}
          ?term skos:prefLabel ?label ;
                skos:exactMatch ?concept .
          ?concept skos:prefLabel ?equivalent .
          FILTER((LANG(?label) = "fr" && LANG(?equivalent) = "nl") || (LANG(?label) = "nl" && LANG(?equivalent) = "fr")) .
        {% raw %} }} {% endraw %}
    """)

    seen = {}
    with Path(output).open(mode="w") as outfile:
        for row in qres:
            term1 = synonym_cleaning(row[0])
            term2 = synonym_cleaning(row[1])

            if term1 != term2 and term1 not in seen and term2 not in seen:
                outfile.write(f"{str(term1)},{str(term2)}\n")
                seen.update({
                    term1: 1,
                    term2: 1
                })

if __name__ == '__main__':
    parser = argparse.ArgumentParser(
        description="Generating synonyms for Elasticsearch based on a SKOS file (Turtle representation)."
    )
    
    parser.add_argument(
        '-s', '--skos',
        help="Specifiy the filename for the SKOS input (Turtle format).",
        required=True
    )
    parser.add_argument(
        '-b', '--bilingual',
        help="Specifiy the filename where to output bilingual synonyms.",
        required=False
    )
    args = parser.parse_args()

    graph = Graph().parse(args.skos, format='turtle')

    if args.bilingual:
        generate_bilingual(graph, args.bilingual)

```

L'outil s'utilise de manière suivante :

```shell
generate_bilingual_synonyms.py -s thesaurus.skos.ttl -b elasticsearch-synonyms.txt
```

Ce qui produira le fichier suivant : 

```csv
zoogdier,mammifère
straat,rue
.
.
.
```

En termes de fonctionnement :

1. L'outil travaille à partir d'un thésaurus SKOS au format Turtle ;
   voici à quoi cela peut ressembler : 
   
   ```text
   thesaurus:91321 a skos:Concept ;
skos:broader thesaurus:93681 ;
        skos:narrower thesaurus:86495 ;
        skos:narrower thesaurus:87274 ;
        skos:narrower thesaurus:86627 ;
        skos:narrower thesaurus:82401 ;
        skos:narrower thesaurus:91971 ;
        skos:narrower thesaurus:86631 ;
        skos:narrower thesaurus:82073 ;
        skos:exactMatch thesaurus:93296 ;
        skos:prefLabel "rue"@fr .
 
    thesaurus:93296 a skos:Concept ;
skos:broader thesaurus:90551 ;
         skos:narrower thesaurus:90560 ;
         skos:narrower thesaurus:91155 ;
         skos:narrower thesaurus:21429 ;
         skos:narrower thesaurus:90998 ;
         skos:narrower thesaurus:92903 ;
         skos:narrower thesaurus:91952 ;
         skos:narrower thesaurus:92396 ;
         skos:narrower thesaurus:88641 ;
         skos:narrower thesaurus:103116 ;
         skos:exactMatch thesaurus:91321 ;
         skos:relatedMatch thesaurus:3582 ;
         skos:prefLabel "straat"@nl .
    ```

2. Je crée un graphe avec
   [*RDFLib*](https://rdflib.readthedocs.io/en/stable/) sur base de ce
   fichier SKOS ;
3. Une requête SPARQL est soumise au graphe ;
4. Je traite les résultats (nettoyage des termes pour respecter le
   format des synonymes d'Elasticsearch) et j'écris cela dans un fichier. 


Une bonne manière de mettre en valeur le travail des documentalistes !
