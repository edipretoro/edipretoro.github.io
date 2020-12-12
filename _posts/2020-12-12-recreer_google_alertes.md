---
layout: post
title: Comment recréer les Google Alertes avec Ecosia
category: [LIS Advent Calendar]
tags: [Python, RSS, Google Alertes]
published: true
comments: true
--- 

Après avoir très brièvement traiter de la surveillance de pages web,
je vous propose de nous pencher sur un autre outil de veille : les
alertes Google. 

Contrairement à hier, je ne vais pas utiliser un outil existant, mais
je vais vous présenter un script Python qui reproduira les
fonctionnalités de Google Alertes, avec quelques modifications liés à
des préférences personnnes :

* Il n'utilisera pas [Google](https://www.google.com/), mais
  [Ecosia](https://www.ecosia.org/) ;
* Il ne m'enverra pas de mails, mais il génèrera un flux RSS auquel
  je pourrai m'abonner dans mon lecteur habituel. 
  
## Le script

```python
#!/usr/bin/env python

from dateutil import tz
from datetime import datetime
import pathlib
import yaml

import diskcache
from feedgen.feed import FeedGenerator
import requests
from lxml.html import fromstring
from ScrapeSearchEngine.ScrapeSearchEngine import Ecosia


def generate_feed(alert, entries):
    feed = FeedGenerator()
    feed.id(f"{alert['base_output']}.rss")
    feed.title(alert['name'])
    feed.description(alert['name'])
    feed.link(href=f"{alert['base_output']}.rss")

    for entry in entries:
        print(entry)
        e = feed.add_entry()
        e.id(entry['url'])
        e.link(href=entry['url'])
        e.pubdate(entry['date'])
        e.title(entry['title'])

    feed.rss_file(f"{pathlib.Path(f"{alert['base_output']}.rss").expanduser()}")


def get_title(link):
    return fromstring(requests.get(link).content).findtext(".//title").strip()


if __name__ == "__main__":
    ua = "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/39.0.2171.99 Safari/537.36"
    now = datetime.now(tz=tz.tzlocal())

    app_home = pathlib.Path().home() / ".config" / "se_alerts"
    if not app_home.exists():
        app_home.mkdir(parents=True, exist_ok=True)

    cache = diskcache.Cache(directory=app_home / "cache")

    if pathlib.Path(app_home / 'alerts.yaml).exists():
        alerts = yaml.safe_load_all(pathlib.Path(app_home / 'alerts.yaml').read_text())
    else:
        alerts = []

    for alert in alerts:
        links = Ecosia(alert["query"], ua)

        if not pathlib.Path(f"{alert['base_output']}.yaml").expanduser().exists():
            entries = []
        else:
            entries = list(yaml.safe_load_all(pathlib.Path(f"{alert['base_output']}.yaml").expanduser().read_text()))
        for link in links:
            if not link in cache:
                cache[link] = 1
                if len(entries) == 0:
                    entries.append({
                        "date": now,
                        "url": link,
                        "title": get_title(link)
                    })
                else:
                    entries.insert(0, {
                        "date": now,
                        "url": link,
                        "title": get_title(link)
                    })
            pathlib.Path(f"{alert['base_output']}.yaml").expanduser().write_text(
            yaml.dump(entries),
            encoding="utf-8",
        )

        generate_feed(alert, entries)
```

Ce script utilise plusieurs modules pour faire le boulot intéressant :

* [Scrape Search
  Engine](https://github.com/sujitmandal/scrape-search-engine) pour
  les recherches ; notez que j'ai utilisé Ecosia, mais d'autres
  moteurs de recherche sont disponibles ;
* [feedgenerator](https://github.com/lkiesow/python-feedgen) pour la
  production du flux RSS ;
* [diskcache](http://www.grantjenks.com/docs/diskcache/) pour la
  fonctionnalité de *caching* des liens, pour éviter de recevoir des
  liens déjà mentionnés. 
  
Pour le reste, les alertes sont configurées dans un fichier
`alerts.yaml`. Ce dernier contient des documents YAML ressemblant à :

```yaml
---
name: CIDOC CRM
query: cidoc crm
base_output: ~/.config/se_alerts/cidoc_crm
```

Il ne reste plus qu'à lancer ce script à l'intervalle choisi via
`crontab`, et nous avons un système d'alertes basés sur des recherches
web. 
