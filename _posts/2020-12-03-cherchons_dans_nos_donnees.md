---
layout: post
title: Cherchons dans nos données !
category: [LIS Advent Calendar]
tags: [Catmandu, SRU, SRW, Solr]
published: true
comments: true
--- 

Pour l'article d'aujourd'hui, je vous propose de créer un serveur
[SRU](https://fr.wikipedia.org/wiki/Search/Retrieve_via_URL)
permettant d'interroger nos données. Pour cela, j'utiliserai le même
jeu de données que dans l'article d'hier, je continuerai à utiliser
[Catmandu](https://librecat.org/) pour la manipulation des données, et
j'utiliserai [Solr](https://lucene.apache.org/solr/) pour la partie
indexation des données.

Pour ceux et celles qui n'ont jamais entendu parlé du SRU, il s'agit
d'un protocole définissant une API de recherche dans un base de
données, typiquement un catalogue de bibliothèque. Vous pouvez ainsi
interroger le catalogue de la [BnF](https://www.bnf.fr/fr) en SRU :
<http://catalogue.bnf.fr/api/test.do>. Ce protocole se veut un
successeur du vénérable Z39.50. 

Sans plus attendre, commençons !

## Indexation des données

Si nous partons du même jeu de données qu'hier, il y a malgré tout un
petit travail de renommage des
colonnes. [*Fix*](https://github.com/LibreCat/Catmandu/wiki/Fix-language)
fait le travail de la manière suivante :

```perl
vacuum()
move_field("BL record ID", identifier)
move_field("Title", title)
move_field("Languages", language)
move_field("Publisher", publisher)
move_field("Physical description", format)
move_field("Notes", description)
move_field("Name", creator)
retain("identifier", "title", "publisher", "description", "creator", "language", "format")
```

Ce qui nous permet de passer de la notice suivante :

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

à cette notice :

```yaml
---
creator: Archer, William
format: 126 pages (8°)
identifier: '000105518'
language: English
publisher: Watts
title: 'God and Mr. Wells: a critical examination of ''God, the Invisible King.'''
...
```

Ce travail effectué, je vais pouvoir les intégrer dans le moteur de
recherche retenu pour ce prototype : Solr. 

Pour cela, nous utiliserons encore Catmandu, et nous découvrirons la
possibilité de le configurer via un fichier, nous évitant ainsi de
tout saisir dans une ligne de commande. 

Voici le fichier de configuration pour Solr, nommé `catmandu.yml` :

```
---
store:
  sru:
    package: Solr
    options:
      url: http://localhost:8983/solr/sru
      bags:
        data:
          cql_mapping:
            default_index: basic
            indexes:
              creator:
                op:
                  'any': true
                  'all': true
                  '=': true
                  'exact': true
                field: 'creator'
              title:
                op:
                  'any': true
                  'all': true
                  '=': true
                  'exact': true
                field: 'title'
```

En bref :

* Nous configurons un *store* nommé *sru* ;
* Nous renseigons l'URL permettant de se connecter à Solr ;
* Nous définissons un *bag* nommé *data* ;
* Et nous définissons deux correspondances entre le recherche CQL du
  SRU et les champs de nos données. 
  
Avec ce fichier de configuration, nous pouvons maintenant lancer
l'indexation des données avec la commande suivante :

```shell
catmandu import CSV --file records.csv --fix sru_dc.fix to sru
```

Il est temps maintenant de bosser sur l'API SRU

## Exposition des données en SRU

Pour cette partie, je vous pouvoir exploiter la richesse du
[CPAN](https://metacpan.org) et utiliser un module qui se chargera de
faire le boulot :
[`Dancer::Plugin::Catmandu::SRU`](https://metacpan.org/pod/Dancer::Plugin::Catmandu::SRU). Et
tout ce que je vais utiliser est dans la documentation !

J'ai besoin d'un fichier de configuration, nommé `config.yml` pour
l'application Dancer (un microframework permettant de créer facilement
des applications web) :

```yaml
charset: "UTF-8"
plugins:
    'Catmandu::SRU':
        store: sru
        bag: data
        default_record_schema: dc
        limit: 200
        maximum_limit: 500
        record_schemas:
            -
                identifier: "info:srw/schema/1/dc-v1.1"
                name: dc
                template: dc.tt
```

Les données seront uniquement exposées en Dublin Core, mais je peux
évidemment ajouter ce qui m'intéresse. Le tout est de fournir un
*template* pour convertir les notices dans le bon format. Ici le
fichier est nommé `dc.tt` et contient :

```html
<srw_dc:dc xmlns:srw_dc="info:srw/schema/1/dc-schema"
           xmlns:dc="http://purl.org/dc/elements/1.1/"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="info:srw/schema/1/dc-schema http://www.loc.gov/standards/sru/recordSchemas/dc-schema.xsd">
[%- FOREACH var IN ['title' 'creator' 'description' 'publisher' 'format' 'identifier' 'language'] %]
    [%- FOREACH val IN $var %]
    <dc:[% var %]>[% val | html %]</dc:[% var %]>
    [%- END %]
[%- END %]
</srw_dc:dc>
```

Enfin, le dernier élément est le script qui va lancer le service : 

```perl
#!/usr/bin/env perl

use Dancer;
use Catmandu;
use Dancer::Plugin::Catmandu::SRU;
  
Catmandu->load;
Catmandu->config;
  
my $options = {};
 
sru_provider '/sru', %$options;
  
dance;
```

Le script se trouve dans un fichier nommé `app.pl`, et nous pouvons le
lancer de la manière suivante : 

```shell
perl app.pl
```

Nous pouvons alors voir :

```text
>> Dancer 1.3513 server 484576 listening on http://0.0.0.0:3000
>> Dancer::Plugin::Catmandu::SRU (0.0503)
== Entering the development dance floor ...
```

Il ne reste plus qu'à visiter avec notre navigateur l'URL
<http://localhost:3000> :

```xml
<explainResponse xmlns="http://www.loc.gov/zing/srw/">
  <version>1.1</version>
  <record>
    <recordSchema>http://explain.z3950.org/dtd/2.1/</recordSchema>
    <recordPacking>xml</recordPacking>
    <recordData>
      <explain xmlns="http://explain.z3950.org/dtd/2.1/">
	<serverInfo protocol="SRU" method="GET" transport="http">
	  <host>localhost</host>
	  <port>3000</port>
	  <database>sru</database>
	</serverInfo>
	<indexInfo>
	  <index>
	    <title>title</title>
	    <map>
	      <name>title</name>
	    </map>
	  </index>
	  <index>
	    <title>creator</title>
	    <map>
	      <name>creator</name>
	    </map>
	  </index>
	</indexInfo>
	<schemaInfo>
	  <schema name="dc" identifier="info:srw/schema/1/dc-v1.1">
	    <title>dc</title>
	  </schema>
	</schemaInfo>
	<configInfo>
	  <default type="numberOfRecords">200</default>
	  <setting type="maximumRecords">500</setting>
	</configInfo>
      </explain>
    </recordData>
  </record>
  <echoedExplainRequest/>
</explainResponse>
```

Cela fonctionne correctement !

Nous pouvons maintenant tester une recherche en CQL : `title any
wells` et nous obtenons la réponses suivantes :

```xml
<searchRetrieveResponse xmlns="http://www.loc.gov/zing/srw/">
  <version>1.1</version>
  <numberOfRecords>318</numberOfRecords>
  <records>
    <record>
      <recordSchema>info:srw/schema/1/dc-v1.1</recordSchema>
      <recordPacking>xml</recordPacking>
      <recordData>
	<srw_dc:dc xmlns:srw_dc="info:srw/schema/1/dc-schema" xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="info:srw/schema/1/dc-schema http://www.loc.gov/standards/sru/recordSchemas/dc-schema.xsd">
	  <dc:title>H.G. Wells</dc:title>
	  <dc:creator>Young, Kenneth</dc:creator>
	  <dc:publisher>British Council ; Longman</dc:publisher>
	  <dc:format>58 pages, plates, 1 portrait, 22 cm</dc:format>
	  <dc:identifier>9683608</dc:identifier>
	  <dc:language>English</dc:language>
	</srw_dc:dc>
      </recordData>
      <recordPosition>1</recordPosition>
    </record>
	
	.
	.
	.
	
    <record>
      <recordSchema>info:srw/schema/1/dc-v1.1</recordSchema>
      <recordPacking>xml</recordPacking>
      <recordData>
	<srw_dc:dc xmlns:srw_dc="info:srw/schema/1/dc-schema" xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="info:srw/schema/1/dc-schema http://www.loc.gov/standards/sru/recordSchemas/dc-schema.xsd">
	  <dc:title>H.G. Wells</dc:title>
	  <dc:creator>Murray, Brian, Professor of writing</dc:creator>
	  <dc:description>'A Frederick Ungar book'</dc:description>
	  <dc:publisher>Continuum</dc:publisher>
	  <dc:format>190 pages, 22 cm</dc:format>
	  <dc:identifier>8100925</dc:identifier>
	  <dc:language>English</dc:language>
	</srw_dc:dc>
      </recordData>
      <recordPosition>2</recordPosition>
    </record>
  </records>
  <extraResponseData/>
  <echoedSearchRetrieveRequest>
    <version>1.1</version>
    <query>title any wells</query>
    <xQuery>
      <searchClause xmlns="http://www.loc.gov/zing/cql/xcql/">
	<index>title</index>
	<relation>
	  <value>any</value>
	</relation>
	<term>wells</term>
      </searchClause>
    </xQuery>
  </echoedSearchRetrieveRequest>
</searchRetrieveResponse>
```

Comme la requête retournait les 200 premiers résultats sur un total de
318, j'ai supprimé un grande partie de la réponse. Mais cela
fonctionne ! 

J'espère avoir montré qu'il n'est pas trop difficile d'exposer un jeu
de données avec un protocole comme le SRU ! 
