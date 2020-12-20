---
layout: post
title: Convertir un thésaurus sous forme de liste alphabétique en SKOS
category: [LIS Advent Calendar]
tags: [Python, SKOS, pyparsing]
published: true
comments: true
---

Aujourd'hui, je vous propose de voir comment extraire des données d'un
texte ayant une structure cohérente, et ce sera l'occasion d'illustrer
comment j'ai créé la version SKOS (Turtle) du thésaurus Promosanthés
utilisé hier. 

## Le problème

J'ai choisi d'utiliser le thésaurus Promosanthés pour générer les
synonymes nécessaires à la configuration d'Elasticsearch, mais je n'ai
trouvé que les versions alphabétique et hiérarchique du
thésaurus. Elles sont évidemment suffisantes pour une utilisation dans
une bibliothèque ou un centre de documentation, mais ce n'est pas un
export SKOS que je pourrais intégrer facilement dans un logiciel
spécialisé. 

Le problème est de passer de la représentation suivante :

```text
AA
Employé pour:
alcooliques anonymes
TG: ORGANISME
TA: GROUPE D'ENTRAIDE

ABANDON
TG: HISTOIRE FAMILIALE
TA: PRIVATION MATERNELLE, PRIVATION PATERNELLE

ABANDON DE TRAITEMENT
TG: CADRE THERAPEUTIQUE
.
.
.
```

à une représentation utilisable par un ordinateur comme, par exemple :

```json
  {
    "id": "0000000001",
    "Employé pour": "alcooliques anonymes",
    "TG": [
      "0000001909"
    ],
    "TA": [
      "0000001243"
    ],
    "term": "AA"
  },
  {
    "id": "0000000002",
    "TG": [
      "0000001306"
    ],
    "TA": [
      "0000002159",
      "0000002160"
    ],
    "term": "ABANDON"
  },
  {
    "id": "0000000003",
    "TG": [
      "0000000422"
    ],
    "term": "ABANDON DE TRAITEMENT"
  }
  .
  .
  .
```

## La solution

Pour réaliser ce genre d'extraction de structure sur base d'un texte,
je vais devoir en analyser la syntaxe. À savoir :

* Les entrées du thésaurus sont en majuscule ;
* La structure thésaurale est composée d'un code et du caractère « : »
  ;
* etc.

## Le code

Pour réaliser ce genre de tâche de *parsing* en Python, un module
incontournable est
[`pyparsing`](https://pyparsing-docs.readthedocs.io/en/latest/#). 

```python
#!/usr/bin/env python

import argparse
import itertools
from pathlib import Path

import pyparsing as pp


def gen_id():
    counter = itertools.count(start=1)

    def new_counter():
        return f"{next(counter):010}"

    return new_counter


def extract_thesaurus(alpha_thesaurus):
    lines = Path(alpha_thesaurus).read_text()

    EOL = pp.LineEnd().suppress()
    term = pp.Regex("[^a-z:,\(\)\n]{2,}").setResultsName("term")
    entry = pp.LineStart() + term.setResultsName("entry") + EOL
    thes_op = pp.oneOf(["TG", "TS", "TA", "EM", "Employé pour", "Définition"])
    thes = (
        thes_op.setResultsName("thes")
        + pp.Literal(":").suppress()
        + pp.SkipTo(entry | thes_op).setResultsName("entry")
    )
    line = thes | entry

    r = line.searchString(lines)

    final_thesaurus = {}
    promosanthes_id = gen_id()
    for e in r:
        if len(e) == 1:
            last_entry = e[0]
            final_thesaurus.setdefault(last_entry, {}).update({"id": promosanthes_id()})
        if len(e) == 2:
            if e[0] in ("TA", "TG", "TS", "EM"):
                final_thesaurus[last_entry][e[0]] = [
                    value.strip()
                    for value in e[1].strip().replace("\n", " ").split(",")
                ]
            else:
                final_thesaurus[last_entry][e[0]] = e[1].strip().replace("\n", " ")

    for term, data in final_thesaurus.copy().items():
        final_thesaurus[term]["term"] = term
        tg_id = []
        for tg in data.get("TG", []):
            if tg in final_thesaurus:
                tg_id.append(final_thesaurus[tg]["id"])
        final_thesaurus[term]["TG"] = tg_id

        ts_id = []
        for ts in data.get("TS", []):
            if ts in final_thesaurus:
                ts_id.append(final_thesaurus[ts]["id"])
        final_thesaurus[term]["TS"] = ts_id

        ta_id = []
        for ta in data.get("TA", []):
            if ta in final_thesaurus:
                ta_id.append(final_thesaurus[ta]["id"])
        final_thesaurus[term]["TA"] = ta_id

        em_id = []
        for em in data.get("EM", []):
            if em in final_thesaurus:
                em_id.append(final_thesaurus[em]["id"])
        final_thesaurus[term]["EM"] = em_id

    return final_thesaurus


def convert2skos(thes, skos_filename):
    with Path(skos_filename).open("w") as skos:
        skos.write(
            """@prefix skos: <http://www.w3.org/2004/02/skos/core#> .
@prefix thesaurus: <http://localhost/thesaurus/skos/term#> .

"""
        )
        for term in thes.values():
            skos.write(f"thesaurus:{term['id']} a skos:Concept ;\n")
            for b in term["TG"]:
                skos.write(f"  skos:broader thesaurus:{b} ;\n")
            for n in term["TS"]:
                skos.write(f"  skos:narrower thesaurus:{n} ;\n")
            for r in term["TA"]:
                skos.write(f"  skos:relatedMatch thesaurus:{r} ;\n")
            skos.write(f'  skos:prefLabel "{term["term"]}"@fr .\n')
            skos.write("\n")


if __name__ == "__main__":
    parser = argparse.ArgumentParser(
        description="Generating synonyms for Elasticsearch based on a SKOS file (Turtle representation)."
    )

    parser.add_argument(
        "-t",
        "--thesaurus",
        help="Specifiy the filename for the thesaurus.",
        required=True,
    )
    parser.add_argument(
        "-s",
        "--skos",
        help="Specifiy the filename where to output the SKOS version of the thesaurus.",
        required=False,
    )
    args = parser.parse_args()

    thesaurus = extract_thesaurus(args.thesaurus)
    convert2skos(thesaurus, args.skos)
```

L'outil s'utilise de la manière suivante : 

```shell
./alpha2skos.py -t promosanthes.txt -s promosanthes.ttl
```

Et voici un extrait du fichier produit : 

```text
@prefix skos: <http://www.w3.org/2004/02/skos/core#> .
@prefix thesaurus: <http://localhost/thesaurus/skos/term#> .

thesaurus:0000000001 a skos:Concept ;
  skos:broader thesaurus:0000001909 ;
  skos:relatedMatch thesaurus:0000001243 ;
  skos:prefLabel "AA"@fr .

thesaurus:0000000002 a skos:Concept ;
  skos:broader thesaurus:0000001306 ;
  skos:relatedMatch thesaurus:0000002159 ;
  skos:relatedMatch thesaurus:0000002160 ;
  skos:prefLabel "ABANDON"@fr .

thesaurus:0000000003 a skos:Concept ;
  skos:broader thesaurus:0000000422 ;
  skos:prefLabel "ABANDON DE TRAITEMENT"@fr .
```

Quelques remarques :

* Le *parsing* n'est pas complet puisque je me suis concentré sur les
  caractéristiques dont j'avais besoin (termes génériques et
  spécifiques ;
* Idem pour la production du fichier Turtle. 

Malgré ces limitations, cela reste une bonne illustration de
l'extraction de la structure d'un document à destination d'être
humain. 
