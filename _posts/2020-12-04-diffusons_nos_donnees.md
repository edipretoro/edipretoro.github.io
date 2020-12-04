---
layout: post
title: Diffusons nos données !
category: [LIS Advent Calendar]
tags: [Catmandu, OAI-PMH, Solr]
published: true
comments: true
--- 

Après avoir converti un jeu de données, l'avoir indexé et exposé
à la recherche par le protocole SRU, il nous reste à le diffuser, en
utilisant le protocole OAI-PMH. 

Comme hier, je vais travailler sur le même jeu de données, et
j'utiliserai l'écosystème de Catmandu pour faire le travail. 

## Conversion des données

Dans le cas de l'OAI-PMH, les identifiants et les dates sont des
éléments clés pour identifier les enregistrements et offrir la
possibilité de moissonner efficacement. Mais dans le jeu de données
choisi, je n'ai pas vraiment de date utilisable dans un contexte
d'OAI-PMH. Mais ce n'est pas nécessairement un problème puisque je
créerai un champ supplémentaire lors de l'importation des
données. J'ai également décidé de ne pas utiliser l'identifiant
présent dans le fichier, mais de créer un nouveau champ `id`, cela
permettra d'illustrer une fonctionnalité supplémentaire de Catmandu. 

Voici le script *Fix* :

```perl
vacuum()
uuid(_id)
move_field("BL record ID", identifier)
move_field("Title", title)
move_field("Date of creation/publication", date)
move_field("Languages", language)
move_field("Publisher", publisher)
move_field("Physical description", format)
move_field("Notes", description)
move_field("Name", creator)
add_field(datestamp, "2020-12-04T18:37:18Z")
retain("_id", "identifier", "title", "publisher", "description", "creator", "date", "language", "format", "datestamp")
```

Et je peux indexer les données dans un *store* de la même manière
qu'hier :

```shell
catmandu import CSV --file records.csv --fix oai.fix to oai
```

Voici la configuration du *store* :

```yaml
---
store:
  oai:
    package: Solr
    options:
      url: http://localhost:8983/solr/oai
      bags:
        data:
          cql_mapping:
            default_index: basic
            indexes:
              _id:
                op:
                  'any': true
                  'all': true
                  '=': true
                  'exact': true
                field: '_id'
              datestamp:
                op:
                  '=': true
                  '<': true
                  '<=': true
                  '>=': true
                  '>': true
                  'exact': true
                field: 'datestamp'
      index_mappings:
        publication:
          properties:
            datestamp: {type: date, format: date_time_no_millis}
```

## Diffusion des données en OAI-PMH

Pour la partie diffusion, le CPAN nous offre une fois de plus ce qu'il
nous faut avec le module `Dancer::Plugin::Catmandu::OAI`. Et comme
hier, la documentation contient tout ce qu'il nous faut ! 

Tout d'abord, le code de l'application :

```perl
#!/usr/bin/env perl
 
use Dancer;
use Catmandu;
use Dancer::Plugin::Catmandu::OAI;
 
Catmandu->load;
Catmandu->config;
 
my $options = {};
 
oai_provider '/oai' , %$options;
 
dance;
```

La configuration de l'application [Dancer](http://perldancer.org/) : 

```yaml
charset: "UTF-8"
plugins:
  'Catmandu::OAI':
    store: oai
    bag: data
    datestamp_field: datestamp
    repositoryName: "LIS Advent Calendar Provider"
    uri_base: "http://localhost:3000/oai"
    adminEmail: edipretoro@gmail.com
    earliestDatestamp: "1970-01-01T00:00:01Z"
    cql_filter: "datestamp>1970-01-01T00:00:01Z"
    deletedRecord: persistent
    repositoryIdentifier: lis.advent
    limit: 200
    delimiter: ":"
    sampleIdentifier: "oai:lis.advent:1585315"
    metadata_formats:
      -
        metadataPrefix: oai_dc
        schema: "http://www.openarchives.org/OAI/2.0/oai_dc.xsd"
        metadataNamespace: "http://www.openarchives.org/OAI/2.0/oai_dc/"
        template: oai_dc.tt
```

Rien de vraiment compliqué dans la mesure où cela ressemble à la
configuration pour le serveur SRU :

* Nous renseignons le nom du champ contenant la date avec
  `datestamp_field` ;
* Nous fixons le nom du dépôt avec `repositoryName`, son URL avec
  `uri_base`, l'adresse mail de l'administrateur avec `adminEmail`,
  etc. ;
* Et comme pour le serveur SRU, nous renseignons les formats de
  métadonnées disponibles en renseignant un *template* pour formater
  les enregistrements. 
  
Voici d'ailleurs le *template* utilisé :

```text
<oai_dc:dc xmlns="http://www.openarchives.org/OAI/2.0/oai_dc/"
           xmlns:oai_dc="http://www.openarchives.org/OAI/2.0/oai_dc/"
           xmlns:dc="http://purl.org/dc/elements/1.1/"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://www.openarchives.org/OAI/2.0/oai_dc/ http://www.openarchives.org/OAI/2.0/oai_dc.xsd">
[%- FOREACH var IN ['title' 'creator' 'description' 'publisher' 'date' 'format' 'identifier' 'language'] %]
    [%- FOREACH val IN $var %]
    <dc:[% var %]>[% val | html %]</dc:[% var %]>
    [%- END %]
[%- END %]
</oai_dc:dc>
```

Et il ne nous plus qu'à lancer le serveur avec la commande suivante :
`perl oai_app.pl`. 

`curl http://localhost:3000/oai?verb=Identify` nous donne la réponse
suivante : 

```xml
<OAI-PMH xmlns="http://www.openarchives.org/OAI/2.0/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.openarchives.org/OAI/2.0/ http://www.openarchives.org/OAI/2.0/OAI-PMH.xsd">
  <responseDate>2020-12-04T19:56:22Z</responseDate>
  <request verb="Identify">http://localhost:3000/oai</request>
  <Identify>
    <repositoryName>LIS Advent Calendar Provider</repositoryName>
    <baseURL>http://localhost:3000/oai</baseURL>
    <protocolVersion>2.0</protocolVersion>
    <adminEmail>edipretoro@gmail.com</adminEmail>
    <earliestDatestamp>1970-01-01T00:00:01Z</earliestDatestamp>
    <deletedRecord>persistent</deletedRecord>
    <granularity>YYYY-MM-DDThh:mm:ssZ</granularity>
    <description>
      <oai-identifier xmlns="http://www.openarchives.org/OAI/2.0/oai-identifier" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.openarchives.org/OAI/2.0/oai-identifier http://www.openarchives.org/OAI/2.0/oai-identifier.xsd">
	<scheme>oai</scheme>
	<repositoryIdentifier>lis.advent</repositoryIdentifier>
	<delimiter>:</delimiter>
	<sampleIdentifier>oai:lis.advent:1585315</sampleIdentifier>
      </oai-identifier>
    </description>
  </Identify>
</OAI-PMH>
```

`curl http://localhost:3000/oai?verb=ListMetadataFormats` nous donne :

```xml
<OAI-PMH xmlns="http://www.openarchives.org/OAI/2.0/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.openarchives.org/OAI/2.0/ http://www.openarchives.org/OAI/2.0/OAI-PMH.xsd">
  <responseDate>2020-12-04T19:58:12Z</responseDate>
  <request verb="ListMetadataFormats">http://localhost:3000/oai</request>
  <ListMetadataFormats>
    <metadataFormat>
      <metadataPrefix>oai_dc</metadataPrefix>
      <schema>http://www.openarchives.org/OAI/2.0/oai_dc.xsd</schema>
      <metadataNamespace>http://www.openarchives.org/OAI/2.0/oai_dc/</metadataNamespace>
    </metadataFormat>
  </ListMetadataFormats>
</OAI-PMH>
```

`curl
http://localhost:3000/oai?verb=ListIdentifiers&metadataPrefix=oai_dc`
nous donne : 

```xml
<OAI-PMH xmlns="http://www.openarchives.org/OAI/2.0/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.openarchives.org/OAI/2.0/ http://www.openarchives.org/OAI/2.0/OAI-PMH.xsd">
  <responseDate>2020-12-04T19:59:39Z</responseDate>
  <request metadataPrefix="oai_dc" verb="ListIdentifiers">http://localhost:3000/oai</request>
  <ListIdentifiers>
    <header>
      <identifier>oai:lis.advent:1271d386-b292-4584-9ecf-a6dc7ee49048</identifier>
      <datestamp>2020-12-04T18:37:18Z</datestamp>
    </header>
    <header>
      <identifier>oai:lis.advent:2de9badb-f4fc-4aa7-9ce3-647b7b725ce5</identifier>
      <datestamp>2020-12-04T18:37:18Z</datestamp>
    </header>
    <header>
      <identifier>oai:lis.advent:160411b4-f1ae-4475-80d1-dc7e7115810a</identifier>
      <datestamp>2020-12-04T18:37:18Z</datestamp>
    </header>
    <header>
      <identifier>oai:lis.advent:3be2ed92-f046-407e-a6f4-8b7013cf6135</identifier>
      <datestamp>2020-12-04T18:37:18Z</datestamp>
    </header>

	.
	.
	.
	
    <header>
      <identifier>oai:lis.advent:058d5bf4-3340-459e-a820-d7cfba1e3aeb</identifier>
      <datestamp>2020-12-04T18:37:18Z</datestamp>
    </header>
    <header>
      <identifier>oai:lis.advent:7ef2a9f9-36b2-44fd-8b98-d7140be1d1e4</identifier>
      <datestamp>2020-12-04T18:37:18Z</datestamp>
    </header>
    <resumptionToken completeListSize="1694">g6VzdGFydMzIol9uzMiiX22mb2FpX2Rj</resumptionToken>
  </ListIdentifiers>
</OAI-PMH>
```

Et enfin, pour obtenir les métadonnées, nous pouvons utiliser la
commande `curl
http://localhost:3000/oai?verb=ListRecords&metadataPrefix=oai_dc` qui nous
donne :

```xml
<OAI-PMH xmlns="http://www.openarchives.org/OAI/2.0/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.openarchives.org/OAI/2.0/ http://www.openarchives.org/OAI/2.0/OAI-PMH.xsd">
  <responseDate>2020-12-04T20:01:38Z</responseDate>
  <request metadataPrefix="oai_dc" verb="ListRecords">http://localhost:3000/oai</request>
  <ListRecords>
    <record>
      <header>
	<identifier>oai:lis.advent:1271d386-b292-4584-9ecf-a6dc7ee49048</identifier>
	<datestamp>2020-12-04T18:37:18Z</datestamp>
      </header>
      <metadata>
	<oai_dc:dc xmlns="http://www.openarchives.org/OAI/2.0/oai_dc/" xmlns:oai_dc="http://www.openarchives.org/OAI/2.0/oai_dc/" xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.openarchives.org/OAI/2.0/oai_dc/ http://www.openarchives.org/OAI/2.0/oai_dc.xsd">
	  <dc:title>God and Mr. Wells: a critical examination of 'God, the Invisible King.'</dc:title>
	  <dc:creator>Archer, William</dc:creator>
	  <dc:publisher>Watts</dc:publisher>
	  <dc:format>126 pages (8°)</dc:format>
	  <dc:identifier>105518</dc:identifier>
	  <dc:language>English</dc:language>
	</oai_dc:dc>
      </metadata>
    </record>
    <record>
      <header>
	<identifier>oai:lis.advent:2de9badb-f4fc-4aa7-9ce3-647b7b725ce5</identifier>
	<datestamp>2020-12-04T18:37:18Z</datestamp>
      </header>
      <metadata>
	<oai_dc:dc xmlns="http://www.openarchives.org/OAI/2.0/oai_dc/" xmlns:oai_dc="http://www.openarchives.org/OAI/2.0/oai_dc/" xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.openarchives.org/OAI/2.0/oai_dc/ http://www.openarchives.org/OAI/2.0/oai_dc.xsd">
	  <dc:title>Commentary; or Reply to H. G. Wells: his views in the book The common sense of war and peace</dc:title>
	  <dc:creator>Ashton, Charles Frederick</dc:creator>
	  <dc:description>Reproduced from typewriting</dc:description>
	  <dc:format>2 parts, ff 17, 20, 33 cm</dc:format>
	  <dc:identifier>128859</dc:identifier>
	  <dc:language>English</dc:language>
	</oai_dc:dc>
      </metadata>
    </record>

	.
	.
	.
	
    <record>
      <header>
	<identifier>oai:lis.advent:7ef2a9f9-36b2-44fd-8b98-d7140be1d1e4</identifier>
	<datestamp>2020-12-04T18:37:18Z</datestamp>
      </header>
      <metadata>
	<oai_dc:dc xmlns="http://www.openarchives.org/OAI/2.0/oai_dc/" xmlns:oai_dc="http://www.openarchives.org/OAI/2.0/oai_dc/" xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.openarchives.org/OAI/2.0/oai_dc/ http://www.openarchives.org/OAI/2.0/oai_dc.xsd">
	  <dc:title>The history of Mr. Polly</dc:title>
	  <dc:creator>Wells, H. G. (Herbert George)</dc:creator>
	  <dc:publisher>Longmans</dc:publisher>
	  <dc:format>xxiv, 289 pages, 18 cm</dc:format>
	  <dc:identifier>10923543</dc:identifier>
	  <dc:language>English</dc:language>
	</oai_dc:dc>
      </metadata>
    </record>
    <resumptionToken completeListSize="1694">g6JfbaZvYWlfZGOiX27MyKVzdGFydMzI</resumptionToken>
  </ListRecords>
</OAI-PMH>
```

Comme hier, j'ai supprimé une grande partie de certaines réponses, mais
comme vous pouvez le voir, cela fonctionne. 

Ici encore, j'espère avoir illustré qu'il peut être assez simple de
mettre en place un serveur OAI-PMH pour diffuser ses données, même si
ces dernières sont dans un format qui peut sembler compliqué de
premier abord. Avec les solutions présentées, un fichier Excel peut
rapidement devenir la source d'un serveur SRU ou d'un dépôt OAI-PMH !
