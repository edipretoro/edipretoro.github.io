---
layout: post
title: Générer un fil RSS sur base d'une page web
category: [LIS Advent Calendar]
tags: [Python, SKOS, pyparsing]
published: true
comments: true
---

J'ai brièvement parlé lors de billets précédents de quelques tâches du
documentaliste, que ce soit de la veille avec l'utilisation de Google
Alertes, ou la surveillance d'une page web. Pour le billet
d'aujourd'hui, je vous propose de créer un fil RSS sur base d'une page
web.

## Le problème

Si la majorité des sites web offre des fils RSS, ce n'est pas
nécessairement le cas de tous. Ou alors, la partie du site web qui
vous intéresse n'est pas présent dans le fil RSS. Une solution
consiste alors à écrire un outil qui récupère la page, l'analyse et en
extrait les parties intéressantes. 

Pour ce billet, je vais utiliser [Lirtuel](http://www.lirtuel.be/) qui
offre effectivement un [fil
RSS](http://www.lirtuel.be/resource_feeds/recent_feed.atom) pour les
acquisitions récentes, mais qui n'en offre pas pour une recherche dans
l'outil, ou pour une catégorie de livres. 

## Le code

Voici le code de l'outil :

```python
#!/usr/bin/env python

import argparse
from datetime import datetime
from urllib.parse import urljoin
from zoneinfo import ZoneInfo

import bs4
import requests
from feedgen.feed import FeedGenerator


class Lirtuel:
    base_url = "http://www.lirtuel.be/"
    categories = {
        "fiction": "F",
        "general": "FB",
        "particularites": "FY",
        "fantasy": "FM",
        "policier": "FF",
        "sf": "FL",
        "thriller": "FH",
        "historique": "FV",
        "horreur": "FK",
        "sciencessociales": "J",
        "nature": "WN",
        "sciences": "P",
    }

    def __init__(self, query):
        self.query = query

    def feed(self, filename):
        if self.query in self.categories:
            search_results = requests.get(
                urljoin(self.base_url, "/resources"),
                params={
                    "utf8": "%E2%9C%93",
                    "category": self.categories.get(self.query),
                    "sort_by": "created_at_desc",
                },
            )
        else:
            search_results = requests.get(
                urljoin(self.base_url, "/resources"),
                params={"utf8": "✓", "q": self.query},
            )
        self._request_url = search_results.url
        soup = bs4.BeautifulSoup(search_results.content, "html.parser")
        feed = self._gen_feed(soup.find_all(class_="list-details__item"))
        feed.atom_file(filename, pretty=True)

    def _gen_feed(self, entries):
        feed = FeedGenerator()
        feed.id(self._request_url)
        feed.title(f"Lirtuel Feed for {self.query}")
        feed.description(f"Lirtuel Feed for {self.query}")
        feed.link(href=self._request_url, rel="self")

        for entry in entries:
            date = datetime.now(tz=ZoneInfo("Europe/Brussels"))
            url = urljoin(self.base_url, entry.h3.a["href"])
            title = " ".join(entry.h3.a.text.split())
            author = entry.find(attrs={"class": "contributors"}).a.text

            e = feed.add_entry()
            e.id(url)
            e.link(href=url)
            e.title(f"{title} / {author}")
            e.pubDate(date)

        return feed


if __name__ == "__main__":
    parser = argparse.ArgumentParser(
        description="Generating a RSS feed for a query on Lirtuel"
    )

    parser.add_argument(
        "-q",
        "--query",
        help="Specify the query to submit to Lirtuel",
        required=True,
    )
    parser.add_argument(
        "-o",
        "--output",
        help="Specifiy the filename where to output the RSS file.",
        required=False,
    )
    args = parser.parse_args()

    Lirtuel(query=args.query).feed(args.output)
```

Et l'outil s'utilise de la manière suivante : 

```shell
lirtuel_rss.py -q "enola" -o enola.rss
```

qui produit le fichier suivant :

```xml
<?xml version='1.0' encoding='UTF-8'?>
<feed xmlns="http://www.w3.org/2005/Atom">
  <id>http://www.lirtuel.be/resources?utf8=%E2%9C%93&amp;q=enola</id>
  <title>Lirtuel Feed for enola</title>
  <updated>2020-12-21T20:22:17.060899+00:00</updated>
  <link href="http://www.lirtuel.be/resources?utf8=%E2%9C%93&amp;q=enola" rel="self"/>
  <generator uri="https://lkiesow.github.io/python-feedgen" version="0.9.0">python-feedgen</generator>
  <subtitle>Lirtuel Feed for enola</subtitle>
    <entry>
    <id>http://www.lirtuel.be/resources/5c19e8fd235794229908107d</id>
    <title>Les enquêtes d'Enola Holmes. Volume 5, L'énigme du message perdu / Serena Blasco</title>
    <updated>2020-12-21T20:22:17.061488+00:00</updated>
    <link href="http://www.lirtuel.be/resources/5c19e8fd235794229908107d" rel="alternate"/>
    <published>2020-12-21T21:22:17.061300+01:00</published>
  </entry>
  <entry>
    <id>http://www.lirtuel.be/resources/5c19e91d2357942299081965</id>
    <title>Les enquêtes d'Enola Holmes. Volume 1, La double disparition / Serena Blasco</title>
    <updated>2020-12-21T20:22:17.061279+00:00</updated>
    <link href="http://www.lirtuel.be/resources/5c19e91d2357942299081965" rel="alternate"/>
    <published>2020-12-21T21:22:17.061062+01:00</published>
  </entry>
</feed>
```

