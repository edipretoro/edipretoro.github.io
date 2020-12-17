---
layout: post
title: Dédoublonnage de notices bibliographiques (3e et dernière partie)
category: [LIS Advent Calendar]
tags: [Python, Algorithme, Data Matching, Dédoublonnage]
published: true
comments: true
---

Pour ce troisième billet sur le dédoublonnage de notices
catalographiques, je vais vous présenter un algorithme qui est assez
ancien puisque les premières traces datent de 1982, en s'appuyant sur
des principes d'un autre algorithme datant de 1974. Il s'agit de
l'*Universal Standard Book Code* (USBC). 

## L'algorithme

Comme pour le *BibHash*, l'idée est de produire une clé sur base de
propriétés d'une notice catalographique. Dans
l'[article](https://academic.oup.com/comjnl/article/23/1/53/676846)
utilisé pour l'implémentation, les champs suivants sont utilisés : 

* Le titre ;
* L'éditeur commercial ;
* La date ;
* L'édition ;
* La tomaison.

Certains principes se dégagent aussi de l'article :

* Une clé se compose de différentes parties ;
* Chaque partie vient d'un champ différent ;
* Chaque partie ne conserve que la partie la plus discriminante ; cela
  signifie par exemple que pour une date, 1990, nous ne conserverons
  que les trois derniers chiffres : 990. Et quand il s'agira de
  caractères alphabétiques, nous ne garderons que les lettres les
  moins fréquentes.

## L'implémentation

J'avais déjà réalisé une [implémentation en
Perl](https://github.com/edipretoro/Biblio-Record-USBC), mais je l'ai
refait en Python :

```python
from collections import Counter
from dataclasses import dataclass, field

import regex as re
import unidecode


def default_format_mapping():
    return {
        "%w": "weight",
        "%l": "language",
        "%d": "date",
        "%t": "title",
        "%e": "edition",
        "%v": "volume",
        "%p": "publisher",
        "%c": "check_digit",
    }


@dataclass
class USBC:
    language: str = ""
    date: str = ""
    title: str = ""
    edition: str = ""
    volume: str = ""
    publisher: str = ""
    format_string: str = "%w%l%d%t%e%v%p%c"
    format_mapping: dict = field(default_factory=default_format_mapping)

    @property
    def weight(self):
        is_alpha = re.compile(r"[[:alpha:]]")

        chars = unidecode.unidecode(self.title).upper()
        cpt = 0
        for letter in chars:
            if is_alpha.search(letter):
                cpt += 1

        return cpt

    @property
    def computed_weight(self):
        return str(self.weight % 10)

    @property
    def computed_language(self):
        languages_code = {
            "english": "0",
            "german": "1",
            "germanic": "2",
            "scandinavian": "2",
            "dutch": "2",
            "french": "3",
            "italian": "4",
            "portuguese": "4",
            "spanish": "4",
            "rumanian": "4",
            "greek": "5",
            "latin": "5",
            "slavic": "6",
            "east_european": "6",
            "finnish": "6",
            "asian": "7",
            "hebrew": "7",
            "african": "8",
            "arabic": "8",
            "others": "9",
        }

        if self.language in languages_code:
            return languages_code[self.language]
        else:
            return "9"

    @property
    def computed_date(self):
        date_re = re.compile(r"(?P<date>\d+)")
        date = date_re.search(self.date)
        if date:
            self.date = f"{date.group('date')}"
            l = len(self.date) - 3
            return self.date[l:] if self.date[l:] else "000"
        else:
            self.date = "000"
            return self.date

    @property
    def computed_title(self):
        code = self._get_string_by_freq(self.title)
        if code:
            if len(code) < 7:
                code += "0" * (7 - len(code))
        else:
            code = "0" * 7

        return code[0:8]

    @property
    def computed_edition(self):
        digit_re = re.compile(r"(?P<edition>\d+)")

        if digit := digit_re.search(self.edition):
            self._edition = f"{digit.group('edition')}"
            l = len(self._edition) - 1
            return self._edition[l:]
        else:
            return "0"

    @property
    def computed_volume(self):
        digit_re = re.compile(r"(?P<vol>\d+)")

        digits = digit_re.findall(self.volume)

        if len(digits) == 0:
            return "00"
        elif len(digits) == 1:
            return f"{int(digits[0]):02}"
        elif len(digits) == 2:
            l_vol = len(digits[0])
            vol = digits[0][l_vol - 1 : l_vol]
            l_num = len(digits[1])
            num = digits[1][l_num - 1 : l_num]
        else:
            return "00"  # Wasn't described in the original article

    @property
    def computed_publisher(self):
        code = self._get_string_by_freq(self.publisher)
        if code:
            return code[0:3]
        else:
            return "00"

    def computed_check_digit(self):
        pass

    def get_usbc(self):
        usbc = ""

	    for i in range(0, len(self.format_string) - 2, 2):
            mapping = self.format_string[i:i + 2]
            usbc += getattr(self, f"computed_{self.format_mapping[mapping]}")

        return usbc

    def _get_string_by_freq(self, value):
        is_alpha = re.compile(r"[[:alpha:]]")

        chars = unidecode.unidecode(value).upper()
        freqs = Counter()
        for letter in chars:
            if is_alpha.search(letter):
                freqs.update(letter)

        by_freqs = {}
        for letter, count in freqs.items():
            by_freqs.setdefault(count, []).append(letter)

        code = "".join(["".join(sorted(chars)) for _, chars in sorted(by_freqs.items())])

        return code
```

Rien de bien compliqué au final : on nettoie, on trie, on assemble. 

## Les tests

Voici le script nous permettant de tester l'algorithme : 

```python#!/usr/bin/env python

import simhash

from usbc import USBC

if __name__ == "__main__":
    book1 = {
        "title": "Le nom de la rose",
        "author": "Umberto Eco",
        "year": "1982"
    }

    book2 = {
        "title": "Nom de la rose (Le)",
        "author": "Eco, Umberto",
        "year": "1982"
    }

    book3 = {
        "title": "Le nom de la rose",
        "author": "U. Eco",
        "year": "1982"
    }

    book4 = {
        "title": "Schismatrice +",
        "author": "Bruce Sterling",
        "year": "1985"
    }

    book1_usbc = USBC(title=book1['title'], date=book1['year'])
    book2_usbc = USBC(title=book2['title'], date=book2['year'])
    book3_usbc = USBC(title=book3['title'], date=book3['year'])
    book4_usbc = USBC(title=book4['title'], date=book4['year'])

    print(f"book1: {book1['title']} → {book1_usbc.get_usbc()}")
    print(f"book2: {book2['title']} → {book2_usbc.get_usbc()}")
    print(f"book3: {book3['title']} → {book3_usbc.get_usbc()}")
    print(f"book4: {book4['title']} → {book4_usbc.get_usbc()}")
```

Et voici le résultat : 

```text
book1: Le nom de la rose → 39982ADMNRSLO00000
book2: Nom de la rose (Le) → 39982ADMNRSLO00000
book3: Le nom de la rose → 39982ADMNRSLO00000
book4: Schismatrice + → 29985AEHMRTCI00000
```

Nous pouvons voir que *book1*, *book2* et *book3* ont la même
clé. Évidemment, dans cet exemple, je n'ai pas d'éditeur commercial,
mais l'utilisation des lettres les moins présentes, et donc les plus
discriminantes, pour constituer ce sous-code fait que je suis quasiment
certain que des versions différentes du même éditeur produiront la
même sous-clé. Et si ce n'est pas le cas, les clés USBC complètes
seront alors très similaires, et les *SimHashes* les regrouperont très
facilement.

Voici la fin de notre petite série de billets sur le dédoublonnage, en
tout cas dans le cadre de ce calendrier de l'Avent de l'informatique
documentaire, et il n'y a évidemment pas de solutions miracles, et si je
devais dédoublonner un fonds documentaire demain, j'utiliserais
probablement une combinaison de ces méthodes.
