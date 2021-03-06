## Transformatie van een MIM model naar een RDF model
## Inleiding

<aside class='ednote'>
  Let op: deze bjilage bevat nog delen die "under-construction" zijn. Deze delen zijn te herkennen aan 
  de woorden **"VERDER UITWERKEN"** die in de bettreffende paragrafen zijn opgenomen.
</aside>

Het MIM is een *metamodel*. Dit betekent dat in termen van het MIM een concreet informatiemodel kan worden uitgewerkt, bijvoorbeeld het informatiemodel Basisregistratie Adressen en Gebouwen. Het MIM is niet bedoeld om vervolgens in termen van dit informatiemodel een concrete dataset te vormen. Hiervoor is een transformatie nodig naar een (technisch) uitwisselings- of opslagmodel, bijvoorbeeld een XSD schema of een RDMS database definitie.

Op diezelfde manier levert het toepassen van het MIM in RDF geen ontologie of vocabulaire waarin RDF kan worden uitgedrukt in een concrete Linked Data dataset. Slechts het informatiemodel zelf is op deze manier in RDF uitgedrukt. Een afzonderlijke transformatie is nodig voor de vertaalslag naar een ontologie voor een concrete Linked Data.

Zo leidt een MIM objecttype "Schip" tot de volgende weergave in RDF:

<pre class='ex-turtle'>
@prefix vb: &lt;http://bp4mc2.org/voorbeeld/>.
@prefix mim: &lt;http://bp4mc2.org/def/mim#>.

vb:Schip a mim:Objecttype;
  rdfs:label "Schip"@nl;
.
</pre>

`vb:Schip` is in dit voorbeeld een voorkomen van de klasse `mim:Objecttype`. Dit voorkomen, `vb:Schip`, kent zelf geen voorkomens. Het is dan ook niet mogelijk om te stellen:

<pre class='ex-turtle'>
vb:Pakjesboot12 a vb:Schip.
</pre>

`vb:Schip` is immers geen klasse maar zelf een voorkomen! Om te kunnen uitdrukken dat de pakjesboot een voorkomen van de klasse Schip is, is een vertaling nodig naar een `rdfs:Class` of `owl:Class`, bijvoorbeeld door:

<pre class='ex-turtle'>
@prefix vbo: &lt;http://bp4mc2.org/voorbeeld/def#>.

vbo:Schip a rdfs:Class;
  rdfs:seeAlso vb:Schip;
.
vb:Pakjesboot12 a vbo:Schip.
</pre>

Dit document beschrijft hoe deze vertaling van het MIM model in RDF naar een RDFS-gebaseerde ontologie plaatsvindt. Daarbij zal niet alleen gebruik worden gemaakt van RDFS, maar ook van de OWL, SHACL en SKOS vocabulaires. De vertaling wordt zo veel mogelijk als SPARQL rules beschreven, zodat een machinale vertaling mogelijk is. De vertaling is beoogd als omkeerbaar. De SPARQL rules die vanuit een RDFS-gebaseerde ontologie de vertaling maken naar een MIM model in RDF, zullen daarom ook worden beschreven (in deze versie van het document zijn deze nog niet opgenomen).

## Gebruikte functies

In de SPARQL rules wordt gebruik gemaakt van een aantal SPARQL functies. In onderstaande tabel staan deze opgesomd met de specificatie van de werking.

|Functie|Specificatie|
|-------|------------|
|t:CamelCase|Codeert een tekstveld naar een URI-vorm op basis van (upper) CamelCase regels|
|t:camelCase|Codeert een tekstveld naar een URI-vorm op basis van (lower) camelCase regels|
|t:kebabcase|Codeert een tekstveld naar een URI-vorm op basis van kebabcase regels (een `-` voor spaties)|
|t:classuri|Formuleert de uri voor een klasse op basis van de naam van een MIM resource. De class URI is opgebouwd als `{namespace}#{t:CamelCase(naam)}`. De `{namespace}` is een vooraf vastgestelde waarde die gelijk is aan de te maken ontologie.|
|t:nodeshapeuri|Formuleert de uri voor een nodeshape op basis van de naam van een MIM resource. De nodeshape URI is opgebouwd als `{shape-namespace}#{t:CamelCase(term)}`. De `{shape-namespace}` is een vooraf vastgestelde waarde die gelijk is aan de te maken shapesgraph.|
|t:propertyuri|Formuleert de uri voor een property op basis van de naam van een MIM resource. De property URI is opgebouwd als `{namespace}#{t:camelCase(naam)}`. Zie ook `t:classuri`.|
|t:propertyshapeuri|Formuleert de uri voor een propertyshape op basis van de naam van een MIM resource en de naam van de MIM resource die hiervan de "bezitter" is. De propertyshape URI is opgebouwd als `{shape-namespace}#{t:CamelCase(bezittersnaam)}-{t:camelCase(naam)}`. Zie ook `t:nodeshapeuri`.|
|t:nodepropertyuri|Formuleert de uri voor een property op basis van de naam van een MIM resource en de naam van de MIM resource die hiervan de "bezitter" is. De property URI is opgebouwd als `{namespace}#{t:CamelCase(bezittersnaam)}-{t:camelCase(naam)}`. Zie ook `t:classuri`.|
|t:statementuri|Formuleert de uri voor een rdf:Statement op basis van zijn afzonderlijke elementen. Mogelijke invulling kan het maken van een hash zijn op basis van de aaneenschakeling van subject, predicate en object.|
|t:mincount|Formuleert de minimum kardinaliteit op basis van een kardinaliteitsaanduiding (zie bij mim:kardinaliteit). De waarde kan ook unbound zijn, in dat geval wordt ook de variable niet gebound en daardoor de betreffende triple niet opgevoerd.|
|t:maxcount|Formuleert de maximum kardinaliteit op basis van een kardinaliteitsaanduiding (zie bij mim:kardinaliteit). De waarde kan ook unbound zijn, in dat geval wordt ook de variable niet gebound en daardoor de betreffende triple niet opgevoerd.|

<aside id='trans-1' class='note'>
Om een unieke URI op te bouwen, is naast de naam van een modelelement, `mim:naam`, ook een namespace noodzakelijk. Deze namespace wordt afgeleid van het veld `mim:locatie` van de package waartoe het modelelement behoord. Deze namespace kan gelijk zijn aan de naam van deze package. Een vertaling is ook denkbaar, bijvoorbeeld als hetzelfde veld ook gebruikt wordt voor het bepalen van de namespace van een XSD, die vaak anders zal zijn dan de URI namespace.
</aside>

<aside id='trans-2' class='note'>
Het MIM model kent geen volgorde. Ondanks dat in de weergave attribuutsoorten in een bepaalde volgorde getoond worden binnen een objecttype en ook referentiewaarden in een bepaalde volgorde getoond worden in een referentielijst, is er in het MIM geen aspect waarin deze volgorde is opgenomen. In het getransformeerde model is het mogelijk om een volgorde te specificeren met behulp van de eigenschap `sh:index`. Deze eigenschap komt echter niet terug in het MIM model zelf. Twee MIM modellen die alleen qua volgorde verschillen moeten gezien worden als equivalent.
</aside>

> **VERDER UITWERKEN**
>
> Er zit nog een foutje in de transformatie: `rdfs:seeAlso` wordt in sommige gevallen gebruikt om meerdere resources. Bijvoorbeeld `mim:Objecttype` leidt tot zowel een `owl:Class` als een `sh:NodeShape`. Gevolg is dat bij de transformatie aspecten als `mim:naam`, `mim:definitie`, `mim:constraint` bij beide resources terecht komen. Beter zou zijn als het maar bij één van de twee terecht komt.

## Overzicht

Onderstaande tabellen geven een overzicht van alle transformaties en een referentie naar de betreffende transformatieregel

### Klassen

|MIM-klasse|Vertaling|Referentie|
|----------|---------|----------|
|`mim:Objecttype`|`owl:Class`, `sh:NodeShape`|[Objecttype](#objecttype)|
|`mim:Attribuutsoort`|`owl:ObjectProperty`, `owl:DatatypeProperty`, `sh:PropertyShape`|[Attribuutsoort](#attribuutsoort)|
|`mim:Gegevensgroep`|`sh:PropertyShape`, `owl:ObjectProperty`|[Gegevensgroep](#gegevensgroep)|
|`mim:Gegevensgroeptype`|`owl:Class`, `sh:NodeShape`|[Gegevensgroeptype](#gegevensgroeptype)|
|`mim:Generalisatie`|`rdfs:subClassOf` `rdf:Statement`|[Generalisatie](#generalisatie)|
|`mim:Relatiesoort`|`sh:PropertyeShape`, `owl:ObjectProperty`|[Relatiesoort](#relatiesoort)|
|`mim:Relatieklasse`|`rdf:Statement`|[Relatieklasse](#relatieklasse)|
|`mim:ExterneKoppeling`|`sh:PropertyShape`, `owl:ObjectProperty`|[Externe koppeling](#externe-koppeling)|
|`mim:Relatierol`|n.n.b.|[Relatierol](#relatierol)|
|`mim:Referentielijst`|n.n.b.|[Referentielijst](#referentielijst)|
|`mim:ReferentieElement`|n.n.b.|[Referentie-element](#referentie-element)|
|`mim:Enumeratie`|n.n.b.|[Enumeratie](#enumeratie)|
|`mim:Enumeratiewaarde`|n.n.b.|[Enumeratiewaarde](#enumeratiewaarde)|
|`mim:Codelist`|n.n.b.|[Codelist](#codelist)|
|`mim:Datatype`|n.v.t.|Abstracte klasse|
|`mim:PrimitiefDatatype`|`rdfs:Datatype`|[Primitief datatype](#primitief-datatype)|
|`mim:GestructureerdDatatype`|`sh:NodeShape`|[Gestructureerd datatype](#gestructureerd-datatype)|
|`mim:DataElement`|`owl:ObjectProperty`, `owl:DatatypeProperty`, `sh:PropertyShape`|[Data element](#data-element)|
|`mim:Union`|`sh:xone`, `rdf:List`|[Union](#union)|
|`mim:UnionElement`|Blank node binnen de `rdf:List`|[Union element](#union-element)|
|`mim:Domein`|`owl:Ontology`|[Domein](#domein)|
|`mim:Extern`|`owl:imports`|[Extern](#extern)|
|`mim:View`|`owl:imports`|[View](#view)|
|`mim:Id`|n.n.b.|-|
|`mim:Identificerend`|n.n.b.|-|
|`mim:Constraint`|`mim:Constraint`|[Constraint](#constraint)|
|`mim:RelatierolSource`|n.n.b.|[Relatierol](#relatierol)|
|`mim:RelatierolTarget`|n.n.b.|[Relatierol](#relatierol)|

### Eigenschappen

|MIM-eigenschap|Vertaling|Referentie|
|--------------|---------|----------|
|`mim:naam`|`rdfs:label`|[naam](#naam)|
|`mim:alias`|`skos:altLabel`|[alias](#alias)|
|`mim:begrip`|`dct:source`|[begrip](#begrip)|
|`mim:begripsterm`|`dc:source`|[begripsterm](#begripsterm)|
|`mim:definitie`|`rdfs:comment`|[definitie](#definitie)|
|`mim:toelichting`|`mim:toelichting`|[toelichting](#toelichting)|
|`mim:herkomst`|`mim:herkomst`|[herkomst](#herkomst)|
|`mim:herkomstDefinitie`|`mim:herkomstDefinitie`|[herkomst definitie](#herkomst-definitie)|
|`mim:datumOpname`|`mim:datumOpname`|[datum opname](#datum-opname)|
|`mim:indicatieMaterieleHistorie`|n.n.b.|[indicatie materiële historie](#indicatie-materi%C3%ABle-historie)|
|`mim:indicatieFormeleHistorie`|n.n.b.|[indicatie formele historie](#indicatie-formele-historie)|
|`mim:kardinaliteit`|`sh:minCount`, `sh:maxCount`|[kardinaliteit](#kardinaliteit)|
|`mim:authentiek`|`mim:authentiek`|[authentiek](#authentiek)|
|`mim:indicatieAfleidbaar`|`mim:indicatieAfleidbaar`|[indicatie afleidbaar](#indicatie-afleidbaar)|
|`mim:mogelijkGeenWaarde`|`mim:mogelijkGeenWaarde`, `sh:minCount`|[mogelijk geen waarde](#mogelijk-geen-waarde)
|`mim:locatie`|`mim:locatie`|[locatie](#locatie)|
|`mim:type`|`sh:datatype`, `sh:node`|[type](#type)|
|`mim:lengte`|`sh:maxLength`|[lengte](#lengte)|
|`mim:patroon`|`mim:patroon`|[patroon](#patroon)|
|`mim:formeelPatroon`|`sh:pattern`|[formeel patroon](#formeel-patroon)|
|`mim:uniekeAanduiding`|`mim:uniekeAanduiding`|[unieke aanduiding](#uniekeaanduiding)
|`mim:populatie`|`mim:populatie`|[populatie](#populatie)|
|`mim:kwaliteit`|`mim:kwaliteit`|[kwaliteit](#kwaliteit)|
|`mim:indicatieAbstractObject`|`mim:indicatieAbstractObject`|[indicatie abstract object](#indicatie-abstract-object)|
|`mim:identificerend`|`mim:identificerend`|[identificerend](#identificerend)|
|`mim:gegevensgroeptype`|`sh:node`|[gegevensgroeptype](#gegevensgroeptype-eigenschap)|
|`mim:unidirectioneel`|n.n.b.|[unidirectioneel](#unidirectioneel)|
|`mim:bron`|`sh:property`|[bron](#bron)|
|`mim:doel`|`sh:class`|[doel](#doel)|
|`mim:aggregatietype`|`mim:aggregatietype`|[aggregatietype](#aggregatietype)|
|`mim:subtype`|`rdfs:subClassOf`|[Generalisatie](#generalisatie)|
|`mim:supertype`|`rdfs:subClassOf`|[Generalisatie](#generalisatie)|
|`mim:code`|n.n.b.|[code](#code)|
|`mim:specificatieTekst`|`mim:specificatieTekst`|[specificatie-tekst](#specificatie-tekst)|
|`mim:specificatieFormeel`|`mim:specificatieFormeel`|[specificatie-formeel](#specificatie-formeel)|
|`mim:attribuut`|`sh:property`|[attribuut](#attribuut)|
|`mim:gegevensgroep`|`sh:property`|[gegevensgroep](#gegevensgroep-eigenschap)|
|`mim:waarde`|n.n.b.|[Enumeratie](#enumeratie)|
|`mim:constraint`|`mim:constraint`|[Constraint](#constraint)|
|`mim:element`|`sh:property`|[Data element](#data-element)|

### Instanties (datatypen)

|MIM datatype|Vertaling|Referentie|
|------------|---------|----------|
|`mim:CharacterString`|`xsd:string`|[Primitief datatype - standaard datatypen](#primitief-datatype---standaard-datatypen)|
|`mim:Integer`|`xsd:integer`|[Primitief datatype - standaard datatypen](#primitief-datatype---standaard-datatypen)|
|`mim:Real`|`xsd:decimal`|[Primitief datatype - standaard datatypen](#primitief-datatype---standaard-datatypen)|
|`mim:Boolean`|`xsd:boolean`|[Primitief datatype - standaard datatypen](#primitief-datatype---standaard-datatypen)|
|`mim:Date`|`xsd:date`|[Primitief datatype - standaard datatypen](#primitief-datatype---standaard-datatypen)|
|`mim:DateTime`|`xsd:dateTime`|[Primitief datatype - standaard datatypen](#primitief-datatype---standaard-datatypen)|
|`mim:Year`|`xsd:gYear`|[Primitief datatype - standaard datatypen](#primitief-datatype---standaard-datatypen)|
|`mim:Day`|`xsd:gDay`|[Primitief datatype - standaard datatypen](#primitief-datatype---standaard-datatypen)|
|`mim:Month`|`xsd:gMonth`|[Primitief datatype - standaard datatypen](#primitief-datatype---standaard-datatypen)|
|`mim:URI`|`xsd:anyURI`|[Primitief datatype - standaard datatypen](#primitief-datatype---standaard-datatypen)|

## Klassen

Omdat het getransformeerde model daadwerkelijk een nieuw model is, zullen de elementen in het getransformeerde model ook eigen URI's krijgen. Om de relatie tussen het originele MIM-model het het getransformeerde model op basis van RDFS te behouden, wordt de eigenschap `rdfs:seeAlso` gebruikt.

> **VERDER UITWERKEN**
>
> Gebruik van `rdfs:seeAlso` is niet erg sterk. Een dergelijke eigenschap kan voor heel veel dingen gebruikt worden. Gekeken moet worden of er niet een betere property beschikbaar is voor dergelijke vocabulaires, of dat we een eigen property moeten introduceren binnen de mim vocabulaire, bv: `mim:equivalent`.

### Objecttype

> De typering van een groep objecten (in de werkelijkheid) die binnen een domein relevant zijn en als gelijksoortig worden beschouwd.

Een `mim:Objecttype` wordt vertaald naar een `owl:Class` in combinatie met een `sh:NodeShape`.

<aside id='trans-3' class='note'>
De identificatie van een objecttype is afgeleid van de naam van het objecttype en de namespace die afgeleid is van aspecten van de package waartoe het objecttype behoord. Aangezien een objecttype binnen een package uniek behoord te zijn conform het MIM, zal hiermee ook een unieke identificatie worden verkregen. Indien in meerdere packages dezelfde namespace wordt gehanteerd, dan dienen over al deze packages ook de namen van objecttypen uniek te zijn. Een dergelijke regel geldt ook voor andere modelelementen die binnen een package vallen.
</aside>

<pre class='ex-sparql'>
CONSTRUCT {
  ?class a owl:Class.
  ?class rdfs:seeAlso ?objecttype.
  ?nodeshape a sh:NodeShape.
  ?nodeshape rdfs:seeAlso ?objecttype.
  ?nodeshape sh:targetClass ?class.
}
WHERE {
  ?objecttype a mim:Objecttype.
  ?objecttype mim:naam ?objecttypenaam.
  BIND (t:classuri(?objecttypenaam) as ?class)
  BIND (t:nodeshapeuri(?objecttypenaam) as ?nodeshape)
}
</pre>

### Attribuutsoort

> De typering van gelijksoortige gegevens die voor een objecttype van toepassing is.

Een `mim:Attribuutsoort` wordt vertaald naar een `sh:PropertyShape` in combinatie met een `owl:DatatypeProperty`. De nodekind van de propertyshape is een `sh:Literal`.

In OWL is een property anders dan in het MIM een *first class citizen*. Dit betekent dat als in twee objecttypen gebruik wordt gemaakt van een attribuutsoort die dezelfde naam heeft, dit leidt tot twee verschillende attribuutsoorten. In OWL zou dit echter leiden tot maar één attribuutsoort, tenzij daadwerkelijk sprake is van verschil in betekenis.

<aside id='trans-4' class='note'>
In een goed RDF model bestaat de mogelijkheid dat de properties worden hergebruikt over meerdere klassen. Er is nog steeds sprake van één unieke propertyshape bij één objecttype, maar in het RDF model kan vervolgens expliciet worden aangegeven dat de *betekenis* van de attribuutsoort dezelfde is als een attribuutsoort van een andere klasse. Dit is niet expliciet in het MIM uit te drukken.

De oplossing hiervoor is om naar het veld `mim:begrip` te kijken. Indien bij twee attribuutsoorten verwezen wordt naar hetzelfde `mim:begrip` en ook hun `mim:naam` hetzelfde is, dan wordt ook verondersteld dat de betekenis van deze attribuutsoorten hetzelfde is, en sprake is van dezelfde property.

Mocht het veld `mim:begrip` niet gebruikt zijn, dan wordt gekeken naar het veld `mim:definitie` in combinatie met het veld `mim:naam`. Indien twee attribuutsoorten dezelfde definitie hebben EN dezelfde naam, dan wordt ook verondersteld dat de betekenis van deze attribuutsoorten hetzelfde is, en sprake is van dezelfde property.
</aside>

<aside id='trans-5' class='note'>
De identificatie van een attribuutsoort is afgeleid van de naam van het attribuutsoort en de namespace die afgeleid is van aspecten van de package waartoe het objecttype behoord. Voor de propertyshape geldt dat deze laatste ook nog afhankelijk is van de naam van het objecttype waartoe de attribuutsoort behoord. Aangezien een attribuutsoort binnen zijn objecttype uniek behoord te zijn conform het MIM, zal hiermee ook een unieke identificatie worden verkregen. Voor de identificatie van de propertyshape geldt dat deze uniek moet zijn binnen de package als sprake is van hetzelfde begrip. Een dergelijke regel geldt ook voor andere modelelementen die binnen een objecttype vallen.
</aside>

Indien het datatype van een attribuutsoort gelijk is aan PrimitiefDatatype (of een daarvan afgeleid datatype), dan is sprake van een `owl:DatatypeProperty`. In alle andere gevallen is sprake van een `owl:Objecttype`. Zie ook de transformatie van de eigenschape `mim:type`.

>> **VERDER UITWERKEN**
>
> In een RDF model wordt soms ook gebruik gemaakt van attribuutsoorten die afkomstig zijn uit andere modellen. De URI-generatie ondersteunt dit nu niet (een attribuutsoort krijgt altijd de namespace die ook zijn bijbehorend objecttype heeft).
>
> Een mogelijk oplossing hiervoor is een afzonderlijk primitief datatype te ondersteunen. Een voorstel is gedaan om een primitief datatype van het type xsd:anyURI te gebruiken. Punt is echter wel dat een xsd:anyURI formeel gezien een literal is en geen URI verwijzing naar een resource.

De URI van de propertyshape wordt afgeleid van de naam van het modelelement dat de attribuutsoort "bezit" en de naam van de attribuutsoort. De URI van de datatypeproperty wordt afgeleid van de naam van de attribuutsoort.

<pre class='ex-sparql'>
CONSTRUCT {
  ?propertyshape a sh:PropertyShape.
  ?propertyshape sh:path ?datatypeproperty.
  ?propertyshape sh:nodekind sh:Literal.
  ?propertyshape rdfs:seeAlso ?attribuutsoort.
  ?datatypeproperty a owl:DatatypeProperty.
  ?datatypeproperty rdfs:seeAlso ?attribuutsoort.
}
WHERE {
  ?attribuutsoort a mim:Attribuutsoort.
  ?attribuutsoort mim:naam ?attribuutsoortnaam.
  ?bezitter mim:attribuut ?attribuutsoort.
  ?bezitter mim:naam ?bezittersnaam
  BIND (t:propertyshapeuri(?bezittersnaam,?attribuutsoortnaam) as ?propertyshape)
  BIND (t:propertyuri(?attribuutsoortnaam) as ?datatypeproperty)
  {
    {
      ?attribuutsoort mim:datatype/rdfs:subClassOf* mim:PrimitiefDatatype.
      BIND (owl:DatatypeProperty as ?type)
    }
    UNION
    {
      ?attribuutsoort mim:datatype ?datatype.
      FILTER NOT EXISTS {
        ?attribuutsoort mim:datatype/rdfs:subClassOf* mim:PrimitiefDatatype.
      }
      BIND (owl:ObjectProperty as ?type)
    }
}
</pre>

### Gegevensgroep

> Een typering van een groep van gelijksoortige gegevens die voor een objecttype van toepassing is.

Een `mim:Gegevensgroep` wordt vertaald naar een `sh:PropertyShape` in combinatie met een `owl:ObjectProperty`. De nodekind van de propertyshape is een `sh:BlankNode`. Gedachte hierachter is dat de gegevensgroep de verbinding is tussen een objecttype en een gegevensgroeptype. Een gegevensgroeptype is vervolgens een groep van samenhangende attribuutsoorten, wat overeen komt met een class en een nodeshape (zie ook gegevensgroeptype). Omdat een gegevensgroeptype geen eigen identiteit heeft, zal dit gemodelleerd worden als blank node.

De URI van de propertyshape wordt afgeleid van de naam van het modelelement dat de gegevensgroep "bezit" en de naam van de gegevensgroep. De URI van de objectproperty wordt afgeleid van de naam van de gegevensgroep.

<pre class='ex-sparql'>
CONSTRUCT {
  ?propertyshape a sh:PropertyShape.
  ?propertyshape sh:path ?objectproperty.
  ?propertyshape sh:nodekind sh:BlankNode.
  ?propertyshape rdfs:seeAlso ?gegevensgroep.
  ?objectproperty a owl:ObjectProperty.
  ?objectproperty rdfs:seeAlso ?gegevensgroep.
}
WHERE {
  ?gegevensgroep a mim:Gegevensgroep.
  ?gegevensgroep mim:naam ?gegevensgroepnaam.
  ?bezitter mim:gegevensgroep ?gegevensgroep.
  ?bezitter mim:naam ?bezittersnaam
  BIND (t:propertyshapeuri(?bezittersnaam,?gegevensgroepnaam) as ?propertyshape)
  BIND (t:propertyuri(?gegevensgroepnaam) as ?objectproperty)
}
</pre>

### Gegevensgroeptype

> Een groep van met elkaar samenhangende attribuutsoorten. Een gegevensgroeptype is altijd een type van een gegevensgroep.

Een `mim:Gegevensgroeptype` wordt vertaald naar een `owl:Class` en een `sh:NodeShape`, net zoals een `mim:Objecttype`.

> **VERDER UITWERKEN**
>
> Er is nu geen verschil meer tussen een gegevensgroeptype en een objecttype. Het MIM maakt echter wel onderscheid. Het verschil is alleen zichtbaar doordat een gegevensgroeptype als blank node verbonden is (zie ook [Gegevensgroep](#gegevensgroep)).

<pre class='ex-sparql'>
CONSTRUCT {
  ?class a owl:Class.
  ?class rdfs:seeAlso ?gegevensgroeptype.
  ?nodeshape a sh:NodeShape.
  ?nodeshape rdfs:seeAlso ?gegevensgroeptype.
  ?nodeshape sh:targetClass ?class.
}
WHERE {
  ?gegevensgroeptype a mim:Gegevensgroeptype.
  ?gegevensgroeptype mim:naam ?gegevensgroeptypenaam.
  BIND (t:classuri(?gegevensgroeptypenaam) as ?class)
  BIND (t:nodeshapeuri(?gegevensgroeptypenaam) as ?nodeshape)
}
</pre>

### Generalisatie

> De typering van het hiërarchische verband tussen een meer generiek en een meer specifiek modelelement van hetzelfde soort, waarbij het meer specifieke modelelement eigenschappen van het meer generieke modelelement overerft.

Generalisatie kan gebruikt worden tussen objecttypen, maar ook tussen datatypes. Aangezien zowel objecttypen als datatypen in het RDFS gebaseerde model worden getransformeerd naar een subklasse van `rdfs:Class`, kan in beide gevallen gebruik worden gemaakt van dezelfde transformatie.

Een `mim:Generalisatie` wordt vertaald naar een `rdfs:subClassOf`.

<aside id='trans-6' class='note'>
Generalisatie is in Linked Data ook mogelijk op properties, en daar ook wel gebruikelijk. Dit wordt nu formeel niet door het MIM ondersteunt. Indien in een RDF model een dergelijke situatie zich voordoet, kan dit vertaald worden naar een MIM model waarbij de aspecten `mim:subtype` en `mim:supertype` verwijzen naar een attribuutsoort of relatieklasse.
</aside>

<aside id='trans-7' class='note'>
Vertaling van een mim:Generalisatie naar een rdfs:subClassOf betekent dat wat in het MIM een metaklasse is, in Linked Data een eigenschap is geworden en geen (meta)class. Hierdoor is het niet mogelijk om extra kenmerken te verbinden aan een generalisatie. Dit betekent dat het niet mogelijk is om de generalisatie een naam of een alias te geven. Dit wordt opgelost in de transformatie door middel van reificatie met rdf:Statement. Een alternatief zou kunnen zijn om subklassen te maken van `rdfs:subClassOf`. Hiervoor is niet gekozen omdat de reificatie oplossen leidt tot een meer gebruikelijk RDF model, waarbij de reificatie kan worden gezien als een aanvullende annotatie die ook weggelaten zou kunnen worden.
</aside>

<pre class='ex-sparql'>

CONSTRUCT {
  ?subject rdfs:subClassOf ?object.
  ?statement a rdf:Statement.
  ?statement rdf:subject ?subject.
  ?statement rdf:predicate rdfs:subClassOf.
  ?statement rdf:object ?object.
  ?statement rdfs:seeAlso ?generalisatie.
}
WHERE {
  ?generalisatie a mim:Generalisatie.
  ?generalisatie mim:subtype ?subtype.
  ?generalisatie mim:supertype ?supertype.
  ?subject rdfs:seeAlso ?subtype.
  ?object rdfs:seeAlso ?supertype.
  ?subject a ?type.
  ?object a ?type.
  FILTER (?type != sh:NodeShape)
  BIND (t:statementuri(?subtype,rdfs:subClasOf,?supertype) as ?statement)
}
</pre>

### Relatiesoort

> De typering van het structurele verband tussen een object van een objecttype en een (ander) object van een ander (of hetzelfde) objecttype.

In het MIM zijn er twee specificatievormen voor relaties: op basis van `mim:Relatiesoort` of op basis van `mim:Relatierol`. Indien gekozen wordt voor `mim:Relatiesoort` dan geldt onderstaande uitwerking. Indien gekozen wordt voor `mim:Relatierol`, dan geldt de uitwerking zoals beschreven bij Relatierol. De keuze wordt bepaald door de aanwezigheid van het attribuut `mim:relatierol`.

Een `mim:Relatiesoort` wordt vertaald naar een `sh:PropertyShape` in combinatie met een `owl:ObjectProperty`. De nodekind van de propertyshape is een `sh:IRI`.

De URI van de propertyshape wordt afgeleid van de naam van het modelelement dat de relatiesoort "bezit" en de naam van de relatiesoort. De URI van de objectproperty wordt afgeleid van de naam van de relatiesoort.

<pre class='ex-sparql'>
CONSTRUCT {
  ?propertyshape a sh:PropertyShape.
  ?propertyshape sh:path ?objectproperty.
  ?propertyshape sh:nodekind sh:IRI.
  ?propertyshape rdfs:seeAlso ?relatiesoort.
  ?objectproperty a owl:ObjectProperty.
  ?objectproperty rdfs:seeAlso ?relatiesoort.
}
WHERE {
  ?relatiesoort a mim:Relatiesoort.
  ?relatiesoort mim:naam ?relatiesoortnaam.
  ?bezitter mim:bron ?relatiesoort.
  ?bezitter mim:naam ?bezittersnaam
  BIND (t:propertyshapeuri(?bezittersnaam,?relatiesoortnaam) as ?propertyshape)
  BIND (t:propertyuri(?relatiesoortnaam) as ?objectproperty)
  FILTER NOT EXISTS {
    ?relatiesoort mim:relatierol ?rol
  }
}
</pre>

### Relatieklasse

> Een relatiesoort met eigenschappen.

Een `mim:Relatieklasse` wordt vertaald naar een subklasse van `rdf:Statement`, waarbij bovendien ook de transformatieregels voor een `mim:Objecttype` en een `mim:Relatiesoort` worden gevolgd.

<pre class='ex-sparql'>
CONSTRUCT {
  ?class a owl:Class.
  ?class rdfs:subClassOf rdf:Statement.
  ?class rdfs:seeAlso ?relatieklasse.
  ?nodeshape a sh:NodeShape.
  ?nodeshape rdfs:seeAlso ?relatieklasse.
  ?nodeshape sh:targetClass ?class.
  ?nodeshape sh:property [
    sh:path rdf:predicate;
    sh:hasValue ?objectproperty;
    sh:minCount 1;
    sh:maxCount 1;
  ];
  ?nodeshape sh:property [
    sh:path rdf:subject;
    sh:class ?subject;
  ];
  ?nodeshape sh:property [
    sh:path rdf:object;
    sh:class ?object;
  ];
  ?propertyshape a sh:PropertyShape.
  ?propertyshape sh:path ?objectproperty.
  ?propertyshape sh:nodekind sh:IRI.
  ?propertyshape rdfs:seeAlso ?relatieklasse.
  ?objectproperty a owl:ObjectProperty.
  ?objectproperty rdfs:seeAlso ?relatieklasse.
}
WHERE {
  ?relatieklasse a mim:Relatiesoort.
  ?relatieklasse mim:naam ?relatieklassenaam.
  ?relatieklasse mim:bron ?bezitter.
  ?bezitter mim:naam ?bezittersnaam.
  ?bezitter mim:seeAlso ?subject.
  ?relatieklasse mim:doel ?doelklasse.
  ?doelklasse mim:seeAlso ?object.
  BIND (t:classuri(?relatieklassenaam) as ?class)
  BIND (t:nodeshapeuri(?relatieklassenaam) as ?nodeshape)
  BIND (t:propertyshapeuri(?bezittersnaam,?relatieklassenaam) as ?propertyshape)
  BIND (t:propertyuri(?relatieklassenaam) as ?objectproperty)
}
</pre>

### Externe koppeling

> Een associatie waarmee vanuit het perspectief van het eigen informatiemodel een objecttype uit het 'eigen' informatiemodel gekoppeld wordt aan een objecttype van een extern informatiemodel. De relatie zelf hoort bij het 'eigen' objecttype.

Een externe koppeling wordt op dezelfde wijze omgezet als een `mim:Relatiesoort` (zie [Relatiesoort](#relatiesoort)). Het verschil is zichtbaar doordat de betreffende objecttypes uit verschillende modellen komen. Anders dan bij UML is het daarbij niet gebruikelijk om het andere objecttype "in" het eigen model te plaatsen, maar juist om direct naar het andere objecttype te verwijzen. Eventueel kan daarbij ook nog gebruik worden gemaakt van een `owl:imports` om expliciet aan te geven dat een ander model wordt gebruikt.

<aside id='trans-8' class='note'>
Een externe koppeling gedraagd zich in een RDF model exact als een relatiesoort. Het verschil wordt zichtbaar doordat het gerelateerde objecttype in een andere package zitten met de aanduiding `mim:view` of `mim:extern`. De objecttypen in deze packages zullen dan ook niet worden omgezet. Wel wordt een extra `owl:imports` statement toegevoegd. Dit gebeurt bij de vertaling van de betreffende packages.
</aside>

#### Relatierol

In het MIM zijn er twee specificatievormen voor relaties: op basis van `mim:Relatiesoort` of op basis van `mim:Relatierol`. Indien gekozen wordt voor `mim:Relatierol` dan geldt onderstaande uitwerking. Indien gekozen wordt voor `mim:Relatiesoort`, dan geldt de uitwerking zoals beschreven bij Relatiesoort.

Een `mim:Relatiesoort` wordt vertaald naar een `sh:PropertyShape` in combinatie met een `owl:ObjectProperty`. De nodekind van de propertyshape is een `sh:IRI`.

De URI van de propertyshape wordt afgeleid van de naam van het modelelement dat de relatierol "bezit" en de naam van de relatierol. De URI van de objectproperty wordt afgeleid van de naam van de relatiesoort. Aangezien er twee relatierollen gedefinieerd kunnen worden, kan ook sprake zijn van twee properties. In dat geval zijn deze twee properties elkaars inverse.

<pre class='ex-sparql'>
CONSTRUCT {
  ?propertyshape a sh:PropertyShape.
  ?propertyshape sh:path ?objectproperty.
  ?propertyshape sh:nodekind sh:IRI.
  ?propertyshape rdfs:seeAlso ?relatierol.
  ?objectproperty a owl:ObjectProperty.
  ?objectproperty rdfs:seeAlso ?relatierol.
}
WHERE {
  ?relatierol a ?type.
  ?relatierol mim:naam ?relatiesoortnaam.
  ?relatiesoort mim:relatierol ?relatierol.
  ?bezitter mim:bron ?relatiesoort.
  ?bezitter mim:naam ?bezittersnaam
  BIND (t:propertyshapeuri(?bezittersnaam,?relatiesoortnaam) as ?propertyshape)
  BIND (t:propertyuri(?relatiesoortnaam) as ?objectproperty)
  FILTER (?type = mim:RelatierolSource || ?type = mim:RelatierolTarget)
}

CONSTRUCT {
  ?sourceproperty owl:inverseOf ?targetproperty
}
WHERE {
  ?sourceproperty a owl:ObjectProperty.
  ?sourceproperty rdfs:seeAlso ?relatierolsource.
  ?relatierolsource a mim:RelatierolSource.
  ?targetproperty a owl:ObjectProperty.
  ?targetproperty rdfs:seeAlso ?relatieroltarget.
  ?relatieroltarget a mim:RelatierolTarget.
  ?relatiesoort mim:relatierol ?relatierolsource,
                               ?relatieroltarget.
}
</pre>

### Referentielijst

> Een lijst met een opsomming van de mogelijke domeinwaarden van een attribuutsoort, die buiten het model in een externe waardenlijst worden beheerd. De domeinwaarden in de lijst kunnen in de loop van de tijd aangepast, uitgebreid, of verwijderd worden, zonder dat het informatiemodel aangepast wordt (in tegenstelling tot bij een enumeratie).

<aside id='trans-9' class='note'>
Het MIM doet geen uitspraak hoe een referentielijst op de externe locatie wordt gepubliceerd. Voor de transformatie wordt de aanname gedaan dat de externe locatie overeen komt met een verzameling van uitspraken van (een subklasse van) `skos:Concept`, en dat op de URL aangeduid met het aspect `mim:locatie` van de referentielijst deze verzameling is te vinden. Bovendien wordt daarbij uitgegaan dat deze referentielijst zelf een `skos:ConceptScheme` is, waarbij de identificatie van dit `skos:ConceptScheme` gelijk is aan de URL van de locatie. Wel kan in een concrete transformatie deze regels getuned worden naar de specifieke behoefte.

Deze constructie wordt ook toegepast bij enumeraties en codelijsten.
</aside>

> **VERDER UITWERKEN**
>
>  Er zijn drie waardelijstconstructies gebruikelijk in RDF: (1) Conceptschema, (2) Collectie en (3) Klasse. Bovenstaande invulling gaat uit van de eerste situatie. Maar ook de andere twee situaties komen voor. Hier zal nog een oplossing voor gezocht moeten worden.

### Referentie element

> Een eigenschap van een object in een referentielijst in de vorm van een gegeven.

### Enumeratie

> Een datatype waarvan de mogelijke waarden limitatief zijn opgesomd in een statische lijst.

<aside id='trans-10' class='note'>
Een enumeratie kan verschillende soorten dingen opsommen. Een lijst met waardes, bijv. een opsomming van nummers, maar ook een lijst met concepten, datatypes, of objecten. Het is dan ook niet triviaal om een goede automatische vertaling te bepalen die een enumeratie kan vertalen naar Linked Data. Om deze reden kiezen we voor een standaardtransformatie naar een klasse gelijknamig aan de enumeratieklasse, en instanties van deze klasse voor elk van de geënumereerde waardes. De geënumereeerde waardes worden ook met een `owl:oneOf` constructie begrensd door de enumeratieklasse. De SHACL gegevensregel maakt gebruikt van het `sh:in` construct om de enumeratie uit te drukken.

In de Inspire RDF Guidelines wordt voorgeschreven om een enumeratie te modelleren als rdfs:Datatype in plaats van als klasse. Dit leidt tot enumeratiewaardes die een literal zijn, met het datatype van de enumeratie. Bijvoorbeeld `"hoog"^^imgolf:NatuurwaardeValue`. De reden om hiervan af te wijken is omdat enumeraties vaker waardelijsten zijn die een object of concept modelleren, dan een lijst van letterlijke waardes. Door deze waardes als objecten te modelleren blijft het mogelijk om nieuwe uitdrukkingen te doen over de waardes.
</aside>

### Enumeratiewaarde

> Een gedefinieerde waarde, in de vorm van een eenmalig vastgesteld constant gegeven.

### Codelist

> Een referentielijst die extern wordt beheerd, en geen onderdeel is van het informatiemodel.

## Datatypen

### Primitief datatype

> Een in het eigen model gedefinieerd primitieve datatype. Deze worden wel door de modelleur gecreëerd, met een eigen naam en een eigen definitie (en eigen metagegevens).

Een primitief datatype wordt vertaald naar een `rdfs:Datatype`. Indien er geen subklasse aanwezig is naar een andere datatype, dan wordt per default een subklasse van xsd:string toegevoegd.

<pre class='ex-sparql'>
CONSTRUCT {
  ?datatype a rdfs:Datatype.
  ?datatype rdfs:subClassOf xsd:string.
  ?datatype rdfs:seeAlso ?primitiefdatatype.
}
WHERE {
  ?primitiefdatatype a mim:PrimitiefDatatype.
  ?primitiefdatatype mim:naam ?primitiefdatatypenaam.
  FILTER NOT EXISTS {
    ?generalisatie mim:subtype ?primitiefdatatype
  }
  BIND (t:classuri(?primitiefdatatypenaam) as ?datatype)
}

CONSTRUCT {
  ?datatype a rdfs:Datatype.
  ?datatype rdfs:seeAlso ?primitiefdatatype.
}
WHERE {
  ?primitiefdatatype a mim:PrimitiefDatatype.
  ?primitiefdatatype mim:naam ?primitiefdatatypenaam.
  ?generalisatie mim:subtype ?primitiefdatatype
  BIND (t:classuri(?primitiefdatatypenaam) as ?datatype)
}
</pre>

### Primitief datatype - standaard datatypen

Voor standaard datatypen maakt RDF gebruik van de XSD datatypen. Onderstaande tabel geeft de mapping weer vanuit de datatypen die in het MIM zijn gespecificeerd.

|MIM datatype|XSD Datatype|
|------------|------------|
|`mim:CharacterString`|`xsd:string`|
|`mim:Integer`|`xsd:integer`|
|`mim:Real`|`xsd:decimal`|
|`mim:Boolean`|`xsd:boolean`|
|`mim:Date`|`xsd:date`|
|`mim:DateTime`|`xsd:dateTime`|
|`mim:Year`|`xsd:gYear`|
|`mim:Day`|`xsd:gDay`|
|`mim:Month`|`xsd:gMonth`|
|`mim:URI`|`xsd:anyURI`|

Deze vertaaltabel kan worden doorgevoerd via `rdfs:seeAlso` statements, aangezien deze vervolgens gebruikt wordt in de transformatieregel voor generalisatie en in de transformatieregel voor het aspect `mim:type`.

<pre class='ex-sparql'>
CONSTRUCT {
  mim:CharacterString a mim:PrimitiefDatatype.
  mim:Integer a mim:PrimitiefDatatype.
  mim:Real a mim:PrimitiefDatatype.
  mim:Boolean a mim:PrimitiefDatatype.
  mim:Date a mim:PrimitiefDatatype.
  mim:DateTime a mim:PrimitiefDatatype.
  mim:Year a mim:PrimitiefDatatype.
  mim:Day a mim:PrimitiefDatatype.
  mim:Month a mim:PrimitiefDatatype.
  mim:URI a mim:PrimitiefDatatype.
  xsd:string rdfs:seeAlso mim:CharacterString.
  xsd:integer rdfs:seeAlso mim:Integer.
  xsd:decimal rdfs:seeAlso mim:Real.
  xsd:boolean rdfs:seeAlso mim:Boolean.
  xsd:date rdfs:seeAlso mim:Date.
  xsd:dateTime rdfs:seeAlso mim:DateTime.
  xsd:gYear rdfs:seeAlso mim:Year.
  xsd:gDay rdfs:seeAlso mim:Day.
  xsd:gMonth rdfs:seeAlso mim:Month.
  xsd:anyURI rdfs:seeAlso mim:URI.
}
WHERE {}
</pre>

### Gestructureerd datatype

> Specifiek benoemd gestructureerd datatype dat de structuur van een gegeven beschrijft, samengesteld uit minimaal twee elementen.

Een `mim:GestructureerdDatatype` wordt vertaald naar een `sh:NodeShape`. Er wordt geen `sh:Class` aangemaakt zoals bij een `mim:Objecttype`, aangezien conform het MIM een gestructureerd datatype slechts een structuur schetst en geen semantiek.

<pre class='ex-sparql'>
CONSTRUCT {
  ?nodeshape a sh:NodeShape.
  ?nodeshape rdfs:seeAlso ?gestructureerddatatype.
}
WHERE {
  ?gestructureerddatatype a mim:GestructureerdDatatype.
  ?gestructureerddatatype mim:naam ?gestructureerddatatypenaam.
  BIND (t:nodeshapeuri(?gestructureerddatatypenaam) as ?nodeshape)
}
</pre>

### Data element

> Een onderdeel/element van een Gestructureerd datatype die als type een datatype heeft.

Een `mim:DataElement` wordt op dezelfde wijze omgezet als een `mim:Attribuutsoort`, waarbij het gestructureerd datatype de "bezitter" is van het data element. De relatie tussen het gestructureerd datatype en het data element wordt direct meegenomen in de transformatie.

De URI van de propertyshape wordt afgeleid van de naam van het gestructureerde datatype dat het data element "bezit" en de naam van het data element. De URI van de datatypeproperty wordt ook op die manier afgeleid. Dit in afwijking van de wijze waarop dit bij een attribuutsoort gebeurt. Reden is het feit dat een data element echt uniek bij een gestructureerd datatype hoort, conform het metamodel (er is sprake van een compositie-aggregatie).

<pre class='ex-sparql'>
CONSTRUCT {
  ?nodeshape sh:property ?propertyshape.
  ?propertyshape a sh:PropertyShape.
  ?propertyshape sh:path ?datatypeproperty.
  ?propertyshape sh:nodekind sh:Literal.
  ?propertyshape rdfs:seeAlso ?attribuutsoort.
  ?datatypeproperty a owl:DatatypeProperty.
  ?datatypeproperty rdfs:seeAlso ?attribuutsoort.
}
WHERE {
  ?dataelement a mim:DataElement.
  ?dataelement mim:naam ?dataelementnaam.
  ?bezitter mim:element ?dataelement.
  ?bezitter mim:naam ?bezittersnaam
  BIND (t:nodeshapeuri(?bezittersnaam) as ?nodeshape)
  BIND (t:propertyshapeuri(?bezittersnaam,?dataelementnaam) as ?propertyshape)
  BIND (t:nodepropertyuri(?dataelementnaam) as ?datatypeproperty)
}
</pre>

### Union

> Gestructureerd datatype, waarmee wordt aangegeven dat er meer dan één mogelijkheid is voor het datatype van een attribuut. Het attribuut zelf krijgt als datatype de union. De union biedt een keuze uit verschillende datatypes, elk afzonderlijk beschreven in een union element.

> **VERDER UITWERKEN**
Het lijkt erop dat in het MIM een union veel beperkter is dan in standaard UML. Daardoor kan de transformatie ook eenvoudiger plaatsvinden. Daarnaast is het handig om de rdf:List afzonderlijk te modelleren, conform het MIM. Op dit moment wordt opnieuw gekeken naar de Union, om deze mogelijk toch breder in te zetten. Daarbij kan mogelijk ook gekeken worden of een eenvoudigere vertaling mogelijk is.
>
> Een union is feitelijk een onderdeel van de specificatiek van een attribuutsoort. In het MIM wordt deze als afzonderlijk modelelement opgenomen en kan daardoor ook hergebruikt of worden voorzien van extra meta-informatie. Een `min:Union`, in combinatie met het `mim:type` wordt vertaald naar een `sh:xone` waarbij de Union zelf een rdf:List is. In deze transformatieregel wordt ook de transformatie van `mim:type` meegenomen. Deze wordt hiermee niet opgenomen bij de transformatie van `mim:type` zelf (zie ook [Type](#type)). Merk op dat een empty list normaal gesproken wordt gerepresenteerd met `rdf:nil`. Dat is in ons geval niet handig, aangezien we expliciet een instantie willen aanmaken van het type `rdf:List`. Aangezien het MIM vereist dat minimaal twee union elementen aanwezig zijn, ontstaat altijd een correcte lijst.

<pre class='ex-sparql'>
CONSTRUCT {
  ?list a rdf:List.
  ?list rdf:rest rdf:nil.
  ?list rdfs:seeAlso ?union.
}
WHERE {
  ?union a mim:Union.
  ?union mim:naam ?unionnaam.
  BIND (t:nodeshapeuri(?unionnaam) as ?list)
}

CONSTRUCT {
  ?subject sh:xone ?union
}
WHERE {
  ?modelelement mim:type ?type.
  ?type rdfs:subClassOf*/rdf:type mim:Union.
  ?subject rdfs:seeAlso ?modelelement.
  ?union rdfs:seeAlso ?type.
}
</pre>

### Union element

> Een type dat gebruikt kan worden voor het attribuut zoals beschreven in de definitie van Union. Het union element is een onderdeel van een Union, uitgedrukt in een eigenschap (attribute) van een union, die als keuze binnen de Union is gerepresenteerd.

Een `mim:unionElement` wordt vertaald naar een element in de lijst van de Union.

Anders dan andere voorbeelden, wordt hier geen CONSTRUCT query gebruikt, omdat de lijst recursief wordt opgebouwd, in combinaties van DELETE en INSERT queries. Nieuwe elementen worden aan het begin van de lijst toegevoegd (er wordt geen volgorde verondersteld, zie ook het algemene issue over volgorde bovenaan). Het union element wordt zelf als blank node toegevoegd.

Onderstaand voorbeeld geeft aan hoe de conversie uiteindelijk plaatsvindt:

> **VERDER UITWERKEN**
>
> Eigenlijk is het helemaal niet mooi dat het zo ingewikkeld wordt. Maar DAT het zo gebeurt is een beperking van UML. We kunnen het ook eenvoudiger maken door het weg te laten, en in MIM expliciet op te nemen dat voor dit element deze waarden niet gelden.

```
ex:GeometrischObject a mim:Objecttype;
  mim:naam "Geometrisch object";
  mim:attribuut ex:geometrie;
.
ex:geometrie a mim:Attribuutsoort;
  mim:naam "geometrie";
  mim:datatype ex:LineOrPolygon;
.
ex:LineOrPolygon a mim:Union;
  mim:naam "Line or polygon";
  mim:element ex:Line;
  mim:element ex:Polygon;
.
ex:Line a mim:UnionElement;
  mim:naam "Line";
  mim:type gml:Line;
.
ex:Polygon a mim:UnionElement;
  mim:naam "Polygon";
  mim:type gml:Polygon;
.

shape:GeometrischObject-geometrie a sh:PropertyShape;
  rdfs:label "geometrie";
  rdfs:seeAlso ex:geometrie;
  sh:xone shape:LineOrPolygon;
.
shape:LineOrPolygon a rdf:List;
  rdfs:label "Line or polygon";
  rdfs:seeAlso ex:LineOrPolygon;
  rdf:first [
    rdfs:label "Line";
    rdfs:seeAlso ex:Line;
    sh:datatype gml:Line
  ];
  rdf:rest (
    [
      rdfs:label "Polygon";
      rdfs:seeAlso ex:Polygon;
      sh:datatype gml:Polygon
    ]
  );
.
```

De list in bovenstaand voorbeeld is niet geheel in de `()` vorm neergezet, zodat ook de extra eigenschappen zichtbaar kunnen worden gemaakt. Feitelijk is bovenstaande formeel gezien gelijk aan:

```
shape:GeometrischObject-geometrie a sh:PropertyShape;
  rdfs:label "geometrie";
  rdfs:seeAlso ex:geometrie;
  sh:xone (
    [ sh:datatype gml:Line ]
    [ sh:datatype gml:Polygon ]  
  )
.
```

De meer uitgebreide vorm is gekozen om ook de aanvullende informatie over Union en Unionelement kwijt te kunnen.

```
DELETE {
  ?endoflist rdf:rest rdf:nil
}
INSERT {
  ?list rdf:first [ rdfs:seeAlso ?unionelement ];
  ?list rdf:rest ?endoflist.
}
WHERE {
  ?unionelement a mim:UnionElement.
  ?union mim:element ?unionelement.
  ?list rdfs:seeAlso ?union.
  ?list rdf:rest* ?endoflist.
  ?endoflist rdf:rest rdf:nil.
}

DELETE {
  ?endoflist rdf:rest rdf:nil.
  ?realendoflist rdf:rest ?endoflist.
}
INSERT {
  ?realendoflist rdf:rest rdf:nil
}
WHERE {
  ?list a rdf:List.
  ?list rdf:rest* ?realendoflist.
  ?realendoflist rdf:rest ?endoflist.
  ?endoflist rdf:rest rdf:nil.
}
```

De tweede delete-insert query is een "opruimquery": aangezien we zijn begonnen met een rdf:List in plaats van een rdf:nil, moeten we het einde van de lijst er nog weer afknippen.

## Packages
> Een package is een benoemde en begrensde verzameling/groepering van modelelementen.

### Domein
> Het eigen IM.

> **VERDER UITWERKEN**
>
> Het domein betreft het eigen IM. Transformatie naar `owl:Ontology` lijkt voor de hand te liggen. In het MIM lijkt dit stereotype niet formeel beschreven. Hoe achterhalen we deze? En kunnen er meerdere packages met stereotype domein zijn binnen een model? Mogelijk moeten we dan meerdere owl:Ontologies aanmaken? Dit is gerelateerd aan het issue over dubbele namen bij bv [Objecttype](#objecttype).

### Extern
> Een groepering van constructies die een externe instantie beheert en beschikbaar stelt aan een informatiemodel en die in het informatiemodel ongewijzigd gebruikt worden.

> **VERDER UITWERKEN**
>
> Het lijkt logisch om een extern package niet te transformeren. De aanname is dat dit al door de externe instantie is gedaan. Mits er voldoende informatie in de UML aanwezig is, kan er een owl:import statement gegenereerd worden. Hiervoor lijkt minimaal noodzakelijk dat een locatie opgegeven kan worden. Wellicht dat het element `mim:locatie` dan ook toegepast zou kunnen worden op package niveau?

### View
> Een groepering van objecttypen die gespecificeerd zijn in een extern informatiemodel en vanuit het perspectief van het eigen informatiemodel inzicht geeft welke gegevens van deze objecttypen relevant zijn binnen het eigen informatiemodel.

> **VERDER UITWERKEN**
>
> In geval van een view, wordt - anders dan bij extern - wel een eigen invulling gegeven aan de structuur van het objecttype. Dit betekent dat de betekenis uit het andere model wordt overgenomen, maar niet per sé alle eigenschappen en relaties. Dit betekent concreet voor de transformatie dat er wel shapes worden aangemaakt voor view-packages, maar geen classes of properties.

## Overig

### Constraint
> Een constraint is een conditie of een beperking, die over een of meerdere modelelementen uit het informatiemodel geldt.

Een constraint (en bijbehorende gegevens) worden direct overgenomen in het vertaalde model als blank node. Het MIM kent voor een constraint twee aspecten: tekstueel en formeel. Het MIM doet daarbij geen uitspraak over de taal die voor het formele model moet worden gehanteerd. Daarmee is een transformatie niet op zijn plaats. Zie ook de [INSPIRE RDF Guidelines](http://inspire-eu-rdf.github.io/inspire-rdf-guidelines/#ref_cr_constraint) waar een vergelijkbare redenatie wordt gevolgd.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:constraint ?constraint.
  ?constraint ?prop ?obj.
}
WHERE {
  ?modelelement mim:constraint ?constraint.
  ?subject rdfs:seeAlso ?modelelement.
  ?constraint ?prop ?obj.
}
</pre>

## Properties

### naam

> De naam van een modelelement

Een `mim:naam` wordt vertaald naar een `rdfs:label`.

<aside id='trans-11' class='note'>
Het MIM geeft de mogelijkheid voor naamgevingsconventies. Zie MIM [Naamgevingsconventies](#afspraken-rondom-naamgeving-en-definities). Dit is op dit moment niet in het MIM zelf als gestructureerd aspect beschikbaar. Voor een RDF model wordt uitgegaan dat de `mim:naam` de voor mensen leesbare naam bevat. Hier wordt dus **geen** technische naam verondersteld en dit veld mag dus ook spaties bevatten.
</aside>

<aside id='trans-12' class='note'>
Het MIM in Linked Data ondersteunt, zoals elk Linked Data model, de mogelijkheid om specifiek een taal aan te geven. Indien een taal aanwezig is, dan wordt dit veld overgenomen. Ook kent het MIM de mogelijkheid om expliciet een taal op het niveau van een package aan te geven. Dit is mede gedaan omdat in UML het niet zo eenvoudig is om een taal per aspect aan te geven. Indien er in een MIM model geen taal is aangegeven, dan wordt deze taal op package niveau gebruikt op elke plek waar een aspect een string is en geen expliciete taalvermelding heeft.
</aside>

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject rdfs:label ?naam
}
WHERE {
  ?modelelement mim:naam ?naam.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### alias

> De alternatieve weergave van de naam.

Een `mim:alias` wordt vertaald naar een `skos:altLabel`

<aside id='trans-13' class='note'>
Het MIM geeft de mogelijkheid voor naamgevingsconventies. Zie MIM [Naamgevingsconventies](#afspraken-rondom-naamgeving-en-definities). Dit is op dit moment niet in het MIM zelf als gestructureerd aspect beschikbaar. Voor een RDF model wordt uitgegaan dat de `mim:alias` een voor mensen alternatieve weergave biedt. Hier wordt dus **geen** technische naam verondersteld en dit veld mag dus ook spaties bevatten.
</aside>

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject skos:altLabel ?alias
}
WHERE {
  ?modelelement mim:alias ?alias.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### begrip

> Verwijzing naar een begrip, vanuit een modelelement, waarmee wordt aangegeven op welk begrip, of begrippen, het informatiemodel element is gebaseerd.

Een `mim:begrip` wordt vertaald naar een `dct:source`

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject dct:source ?begrip
}
WHERE {
  ?modelelement mim:begrip ?begrip.
  ?subject rdfs:seeAlso ?modelelement
}
</pre>

### begripsterm

> Verwijzing naar een begrip in de vorm van de term, vanuit een modelelement, waarmee wordt aangegeven op welk begrip, of begrippen, het informatiemodel element is gebaseerd.

Een `mim:begripsterm` wordt vertaald naar een `dc:source`. Het heeft de voorkeur om geen gebruik te maken van dit aspect, maar om gebruik te maken van het aspect `mim:begrip`, waarmee een directe verwijzing kan worden gemaakt naar het begrip zelf.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject dc:source ?begrip
}
WHERE {
  ?modelelement mim:begripsterm ?begripsterm.
  ?subject rdfs:seeAlso ?modelelement
}
</pre>

### definitie

> De beschrijving van de betekenis van dit modelelement.

Een `mim:definitie` wordt vertaald naar een `rdfs:comment`

Rationale om niet te kiezen voor `skos:definition`: in de meeste Linked Data vocabulaires is het gebruikelijk om de beschrijving van een klasse op te nemen door middel van een `rdfs:comment`, wat ook de intentie is in het MIM. Het MIM is niet beoogd als een volledig begrippenkader. Het MIM biedt daarnaast de mogelijkheid om expliciet te verwijzen vanuit een modelelement naar een `skos:Concept`. Het ligt dan ook voor de hand om bij dit `skos:Concept` de werkelijke `skos:definition` op te nemen.

> **VERDER UITWERKEN**
>
> Ik meende dat het mogelijk was om in het MIM op te geven dat een modelelement *ook* een begrip is. In dat geval zou je dus een andere vertaling kunnen maken, dwz: *wel* naar een `skos:definition`. Dit zou wel beter zijn.

### toelichting

> Een inhoudelijke toelichting op de definitie, ter verheldering of nadere duiding.

Een `mim:toelichting` wordt direct, zonder aanpassing, overgenomen in het vertaalde model.

*Aanbevolen wordt om geen gebruik te maken van mim:toelichting, maar gebruik te maken van de verwijzing naar expliciet gedefinieerde begrippen, waarbij de toelichting bij het begrip zelf wordt opgenomen*.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:toelichting ?toelichting
}
WHERE {
  ?modelelement mim:toelichting ?toelichting.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### herkomst

> De registratie of het informatiemodel waaraan het modelelement ontleend is dan wel de eigen organisatie indien het door de eigen organisatie toegevoegd is.

Een `mim:herkomst` wordt direct, zonder aanpassing, overgenomen in het vertaalde model.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:herkomst ?herkomst
}
WHERE {
  ?modelelement mim:herkomst ?herkomst.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### herkomst definitie

> De registratie of het informatiemodel waaruit de definitie is overgenomen dan wel een aanduiding die aangeeft uit welke bronnen de definitie is samengesteld.

Een `mim:herkomstDefinitie` wordt direct, zonder aanpassing, overgenomen in het vertaalde model.

*Aanbevolen wordt om geen gebruik te maken van mim:herkomstDefinitie, maar gebruik te maken van de verwijzing naar expliciet gedefinieerde begrippen, waarbij de herkomst van de definitie bij het begrip zelf wordt opgenomen*.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:herkomstDefinitie ?herkomstdefinitie
}
WHERE {
  ?modelelement mim:herkomstDefinitie ?herkomstdefinitie.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### datum opname

> De datum waarop het modelelement is opgenomen in het informatiemodel.

Een `mim:datumOpname` wordt direct, zonder aanpassing, overgenomen in het vertaalde model.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:datumOpname ?datumopname
}
WHERE {
  ?modelelement mim:datumOpname ?datumopname.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### indicatie materiële historie

> Indicatie of de materiële historie van het kenmerk van het object te bevragen is.

> **VERDER UITWERKEN**
>
> Hoe gaan we dit doen? Feitelijk moet je deze indicatie omzetten naar daadwerkelijke properties.
>
> Zie voor input het NEN3610 Linked Data Profiel [7.3.4.2.4 Attribuut met stereotype «materieleHistorie»](https://geonovum.github.io/NEN3610-Linkeddata/#regels-attributen-materieleHistorie).

### indicatie formele historie

> Indicatie of de formele historie van het kenmerk van het object bijgehouden wordt en te bevragen is.

> **VERDER UITWERKEN**
>
> Hoe gaan we dit doen? Feitelijk moet je deze indicatie omzetten naar daadwerkelijke properties. Meestal doen we daarbij formele historie als onderdeel van de volledige klasse, maar niet de afzonderlijke elementen.
>
> Zie voor input het NEN3610 Linked Data Profiel [7.3.4.2.5 Attribuut met stereotype «formeleHistorie»](https://geonovum.github.io/NEN3610-Linkeddata/#regels-attributen-formeleHistorie).

### kardinaliteit

De `mim:kardinaliteit` wordt vertaald naar `sh:minCount` en `sh:maxCount`. Daarbij wordt de volgende tabel gebruikt om de string-waarde van mim:kardinaliteit om te zetten. Een `-` betekent dat de betreffende triple niet wordt opgenomen in het model.

Daarnaast wordt `min:kardinaliteit` ook direct overgenomen in het vertaalde model. De reden hiervoor is tweeledig. Enerzijds maakt het daarmee eenvoudiger om de originele kardinaliteit weer te geven. Voor niet-SHACL experts kan het verwarrend zijn dat het ontbreken van zowel sh:minCount als sh:maxCount betekent dat sprake is van een 0..* kardinaliteit. Anderzijds maakt het de terugvertaling in geval van een `min:mogelijkGeenWaarde` mogelijk, aangezien dit veld invloed kan hebben op de sh:minCount (zie ook [mogelijk-geen-waarde](#mogelijk-geen-waarde)).

|mim:kardinaliteit|sh:minCount|sh:maxCount|
|-----------------|-----------|-----------|
|1|1|1|
|*|-|-|
|0..1|-|1|
|0..*|-|-|
|1..1|1|1|
|1..*|1|-|
|a..z|a|z|

In de laatste regel moet voor a en z een geheel getal vanaf 1 worden gelezen.

<pre class='ex-sparql'>
CONSTRUCT {
  ?propertyshape sh:minCount ?mincount.
}
WHERE {
  ?modelelement mim:kardinaliteit ?kardinaliteit.
  ?modelelement mim:mogelijkGeenWaarde ?mogelijkgeenwaarde.
  ?propertyshape rdfs:seeAlso ?modelelement.
  BIND (t:mincount(?kardinaliteit) as ?mincount)
  FILTER(NOT(?mogelijkgeenwaarde))
}

CONSTRUCT {
  ?propertyshape sh:maxCount ?maxcount.
}
WHERE {
  ?modelelement mim:kardinaliteit ?kardinaliteit.
  ?propertyshape rdfs:seeAlso ?modelelement.
  BIND (t:maxcount(?kardinaliteit) as ?maxcount)
}
</pre>

### authentiek
> Aanduiding of het kenmerk een authentiek gegeven betreft.

Een `mim:authentiek` wordt direct, zonder aanpassing, overgenomen in het vertaalde model.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:authentiek ?authentiek
}
WHERE {
  ?modelelement mim:authentiek ?authentiek.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### indicatie afleidbaar
> Aanduiding dat gegeven afleidbaar is uit andere attribuut- en/of relatiesoorten.

Een `mim:indicatieAfleidbaar` wordt direct, zonder aanpassing, overgenomen in het vertaalde model.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:indicatieAfleidbaar ?indicatieafleidbaar
}
WHERE {
  ?modelelement mim:indicatieAfleidbaar ?indicatieafleidbaar.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### mogelijk geen waarde
> Aanduiding dat van een aspect geen waarde is geregistreerd, maar dat onduidelijk is of de waarde er werkelijk ook niet is.

Een `mim:mogelijkGeenWaarde` wordt direct, zonder aanpassing, overgenomen in het vertaalde model, waarbij in een enkel geval een aanpassing wordt gedaan aan de manier waarop `mim:kardinaliteit` wordt getransformeerd.

Linked Data gaat in beginsel uit van een "open word assumptie". Dit houdt onder andere in dat Linked Data er van uitgaat dat elk aspect mogelijk geen waarde kan hebben. Met SHACL kan deze assumptie worden beperkt. Zo zal bij een verplicht veld (zoals kardinaliteit 1..* of 1..1) daadwerkelijk ook een waarde aanwezig moeten zijn. Het MIM gaat in beginsel  uit van een "closed world assumptie", het veld `mim:mogelijkGeenWaarde` is juist bedoeld om deze assumptie te verruimen. Doordat het veld `mim:mogelijkGeenWaarde` altijd een waarde heeft ("Ja" of "Nee"), kan het veld ook worden gelezen als *als de waarde in de werkelijkheid bestaat, dan is deze ook aanwezig*. Dit maakt dat het veld `mim:mogelijkGeenWaarde` feitelijk in de context van Linked Data het model in vele gevallen een gesloten model maakt, vandaar ook het gebruik van SHACL waarmee we deze beperkingen kunnen opleggen. Indien sprake is van een "mogelijk geen waarde" dan is het wel noodzakelijk om de transformatieregel voor `mim:kardinaliteit` aan te passen, conform onderstaande tabel:

|`mim:mogelijkGeenWaarde`|`mim:kardinaliteit`|Aanpassing             |
|------------------------|-------------------|-----------------------|
| Nee                    | 0..x              |geen                   |
| Nee                    | 1..x              |geen                   |
| Ja                     | 0..x              |geen                   |
| Ja                     | 1..x              |sh:minCount verwijderd |
| Ja                     | n..x (met n>1)    |sh:minCount verwijderd |

"sh:minCount verwijderd" houdt in dat de kardinaliteit 0..x wordt. Deze aanpassing is opgenomen in de transformatieregel van [kardinaliteit](#kardinaliteit).

Zie ook het NEN3610 Linked Data Profiel [7.3.4.2.3 Attribuut met stereotype «voidable»](https://geonovum.github.io/NEN3610-Linkeddata/#regels-attributen-voidable) voor meer achtergrondinformatie.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:mogelijkGeenWaarde ?mogelijkgeenwaarde
}
WHERE {
  ?modelelement mim:mogelijkGeenWaarde ?mogelijkgeenwaarde.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### locatie
> Als het type van het attribuutsoort een waardenlijst is, dan wordt hier de locatie waar deze te vinden is opgegeven.

Een `mim:locatie` wordt direct, zonder aanpassing, overgenomen in het vertaalde model. Daarnaast wordt dit veld gebruikt bij het munten van de URI's van de verschillende modelelementen en het achterhalen van de inhoud van een waardenlijst.

### type
> Het datatype waarmee waarden van deze attribuutsoort worden vastgelegd.

De vertaling van een `mim:type` hangt af van de vertaling van het datatype waar naar wordt verwezen:

- Voor primitieve datatypes wordt vertaald naar een `sh:datatype`;
- Voor gestructureerde datatypes wordt vertaald naar een `sh:node`;
- Voor een enumeratie wordt vertaald naar een `sh:node`;
- Voor een referentielijst wordt vertaald naar een `sh:node`;
- Voor een codelijst wordt vertaald naar een `sh:node`;
- Voor een union wordt de vertaling opgepakt bij de transformatieregel van union zelf (zie ook [Union](#union))
- In geval van zelfgespecificeerde datatypen wordt vertaald conform het betreffende supertype.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject sh:datatype ?datatype
}
WHERE {
  ?modelelement mim:type ?type.
  ?type rdfs:subClassOf*/rdf:type mim:PrimitiefDatatype.
  ?subject rdfs:seeAlso ?modelelement.
  ?datatype rdfs:seeAlso ?type.
}

CONSTRUCT {
  ?subject sh:node ?datatype
}
WHERE {
  ?modelelement mim:type ?type.
  ?type rdfs:subClassOf*/rdf:type ?mimtype.
  ?subject rdfs:seeAlso ?modelelement.
  ?subject a sh:NodeShape.
  ?datatype rdfs:seeAlso ?type.
  FILTER (?mimtype = mim:GestructureerdDatatype
       || ?mimtype = mim:Enumeratie
       || ?mimtype = mim:Referentielijst
       || ?mimtype = mim:Codelijst
  )
}
</pre>

### lengte
> De aanduiding van de lengte van een gegeven.

Een `mim:lengte` wordt vertaald naar een `sh:maxLength`.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject sh:maxLength ?lengte
}
WHERE {
  ?modelelement mim:lengte ?lengte.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### patroon
> De verzameling van waarden die gegevens van deze attribuutsoort kunnen hebben, oftewel het waardenbereik, uitgedrukt in een specifieke structuur.

De structuur van `mim:patroon` is in woorden beschreven. Deze wordt direct, zonder aanpassing, overgenomen in het vertaalde model.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:patroon ?patroon
}
WHERE {
  ?modelelement mim:patroon ?patroon.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### formeel patroon
> Zoals patroon, formeel vastgelegd, uitgedrukt in een formele taal die door de computer wordt herkend.

`mim:formeelPatroon` wordt beschreven met `sh:pattern`.

<aside id='trans-13' class='note'>
Het MIM stelt dat het formeelPatroon door de computer moet worden herkend, zonder specifiek te zijn op welke manier. In het geval van een MIM in Linked Data model wordt uitgegaan dat hier sprake is van een reguliere expressie.
</aside>

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject sh:pattern ?formeelpatroon
}
WHERE {
  ?modelelement mim:formeelPatroon ?formeelpatroon.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### unieke aanduiding
> Voor objecttypen die deel uitmaken van een (basis)registratie of informatiemodel betreft dit de wijze waarop daarin voorkomende objecten (van dit type) uniek in de registratie worden aangeduid.

Een `mim:uniekeAanduiding` wordt direct, zonder aanpassing, overgenomen in het vertaalde model.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:uniekeAanduiding ?uniekeaanduiding
}
WHERE {
  ?modelelement mim:uniekeAanduiding ?uniekeaanduiding.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### populatie
> Voor objecttypen die deel uitmaken van een (basis)registratie betreft dit de beschrijving van de exemplaren van het gedefinieerde objecttype die in de desbetreffende (basis)­registratie voorhanden zijn.

Een `mim:populatie` wordt direct, zonder aanpassing, overgenomen in het vertaalde model.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:populatie ?populatie
}
WHERE {
  ?modelelement mim:populatie ?populatie.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### kwaliteit
> Voor objecttypen die deel uitmaken van een registratie betreft dit de waarborgen voor de juistheid van de in de registratie opgenomen objecten van het desbetreffende type.

Een `mim:kwaliteit` wordt direct, zonder aanpassing, overgenomen in het vertaalde model.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:kwaliteit ?kwaliteit
}
WHERE {
  ?modelelement mim:kwaliteit ?kwaliteit.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### indicatie abstract object
> Indicatie dat het objecttype een generalisatie is, waarvan een object als specialisatie altijd voorkomt in de hoedanigheid van een (en slechts één) van de specialisaties van het betreffende objecttype.

In een MIM conform informatiemodel kunnen zowel abstracte als concrete klassen voorkomen. In UML kun je daarvan afleiden dat je geen instanties mag hebben van abstracte klassen, maar alleen van concrete klassen. In RDF wordt geen onderscheid gemaakt tussen het abstract of concreet zijn van klassen. In RDF worden klassen beschouwd als sets van dingen. Als je een set kunt beschrijven, dan kunnen er ook dingen zijn die tot die set behoren.

Een `mim:indicatieAbstractObject` wordt direct, zonder aanpassing, overgenomen in het vertaalde model.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:indicatieAbstractObject ?indicatieabstractobject
}
WHERE {
  ?modelelement mim:indicatieAbstractObject ?indicatieabstractobject.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### identificerend
> Een kenmerk van een objecttype die aangeeft of deze eigenschap uniek identificerend is voor alle objecten in de populatie van objecten van dit objecttype.

Een `mim:identificerend` wordt direct, zonder aanpassing, overgenomen in het vertaalde model.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:identificerend ?identificerend
}
WHERE {
  ?modelelement mim:identificerend ?identificerend.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### gegevensgroeptype (eigenschap)

> De verwijzing naar het bijbehorende gegevensgroeptype.

Een `mim:gegevensgroeptype` wordt vertaald naar een `sh:class` met als waarde de URI van de class die het bijbehorende gegevensgroeptype representeert. Zie [Gegevensgroep](#gegevensgroep) en [Gegevensgroeptype](#gegevensgroeptype) voor meer uitleg.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject sh:class ?object
}
WHERE {
  ?modelelement mim:gegevensgroeptype ?gegevensgroeptype.
  ?subject rdfs:seeAlso ?modelelement.
  ?subject a sh:NodeShape.
  ?object rdfs:seeAlso ?gegevensgroeptype.
  ?object a owl:Class.
}
</pre>

### unidirectioneel

> **VERDER UITWERKEN**
> Betreft het automatisch opnemen van een inverse relatie. Daarbij dient de rolnaam gebruikt te worden voor deze inverse relatie.

Bij `<<relatiesoort>>`:

> Het gerelateerde objecttype (de target) waarvan het objecttype, die de eigenaar is van deze relatie (de source), kennis heeft. Alle relaties zijn altijd gericht van het objecttype (source) naar het gerelateerde objecttype (target).

Bij `<<Externe koppeling>>`:
> Het gerelateerde objecttype uit een extern informatiemodel (de target) waarvan het objecttype die de eigenaar van deze relatie is (de source) kennis heeft. Het aggregation type van de source is altijd ‘composition’. Alle relaties zijn altijd gericht van het objecttype (source) naar het gerelateerde objecttype (target).

### bron
> Aanduiding van het bronobject in een relatie tussen objecten. Een bronoject heeft middels een relatiesoort een relatie met een doelobject.

Een `mim:bron` wordt vertaald naar een `sh:property` die hoort bij de de NodeShape van het objecttype. Zie voor meer informatie over hoe relaties tussen objecttypen worden vertaald de paragrafen [Relatiesoort](#relatiesoort) en [Externe koppeling](#externe-koppeling).

<pre class='ex-sparql'>
CONSTRUCT {
  ?objecttype sh:property ?modelelement
}
WHERE {
  ?modelelement mim:bron ?objecttype.
  ?subject rdfs:seeAlso ?modelelement.
  ?object rdfs:seeAlso ?objecttype.
}
</pre>

### doel
> Aanduiding van het gerelateerde objecttype die het eindpunt van de relatie aangeeft. Naar objecten van dit objecttype wordt verwezen.

Een `mim:doel` wordt vertaald naar een `sh:class` met als waarde de URI van de Class die het gerelateerde objecttype representeert. Zie voor meer informatie over hoe relaties tussen objecttypen worden vertaald de paragrafen [Relatiesoort](#relatiesoort) en [Externe koppeling](#externe-koppeling).

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject sh:class ?object
}
WHERE {
  ?modelelement mim:doel ?gerelateerdobjecttype.
  ?subject rdfs:seeAlso ?modelelement.
  ?object rdfs:seeAlso ?gerelateerdobjecttype.
}
</pre>

### aggregatietype
> Aanduiding of het objecttype die de eigenaar is van een relatie het doel van relatie ziet als een samen te voegen onderdeel.

Aggregatie- en compositie-associaties worden gemodelleerd zoals simpele relatiesoorten, gebruikmakend van de specifieke naam die de associatie in het oorspronkelijke model heeft. Een `mim:aggregatietype` wordt direct, zonder aanpassing, overgenomen in het vertaalde model.

<pre class='ex-sparql'>
CONSTRUCT {
  ?subject mim:aggregatietype ?aggregatietype
}
WHERE {
  ?modelelement mim:aggregatietype ?aggregatietype.
  ?subject rdfs:seeAlso ?modelelement.
}
</pre>

### code
> De in een registratie of informatiemodel aan de enumeratiewaarde toegekend unieke code (niet te verwarren met alias, zoals bedoeld in 2.6.1).

> **VERDER UITWERKEN**
>
> We hebben nog niet gespecificeerd hoe we enumeraties vertalen. In NEN3610-LD is standaardtransformatie een transformatie naar een klasse gelijknamig aan de enumeratieklasse, en instanties van deze klasse voor elk van de geënumereerde waardes. Als we dit volgen zouden we de `mim:code` kunnen vertalen naar een `rdfs:label` of `skos:altLabel`.

### specificatie-tekst
> De specificatie van de constraint in normale tekst.

> **VERDER UITWERKEN**
>
> We hebben nog niet gespecificeerd hoe we constraints vertalen. Voorstel: alleen vertalen naar documentatie in het MIM model in RDF. In de toekomst wellicht vertalen naar SHACL.

Een `mim:specificatieTekst` wordt direct, zonder aanpassing, overgenomen in het vertaalde model, als onderdeel van de [transformatieregel voor constraints](#constraint).

### specificatie-formeel
> De beschrijving van de constraint in een formele specificatietaal, in OCL.

> **VERDER UITWERKEN**
>
> We hebben nog niet gespecificeerd hoe we constraints vertalen. Voorstel: alleen vertalen naar documentatie in het MIM model in RDF. In de toekomst wellicht vertalen naar SHACL.

Een `mim:specificatieFormeel` wordt direct, zonder aanpassing, overgenomen in het vertaalde model, als onderdeel van de [transformatieregel voor constraints](#constraint).

### attribuut

Een `mim:attribuut` wordt vertaald naar een `sh:property` die hoort bij de de NodeShape van de bezitter van het attribuut. Zie ook [Attribuutsoort](#attribuutsoort).

<pre class='ex-sparql'>
CONSTRUCT {
  ?nodeshape sh:property ?propertyshape
}
WHERE {
  ?modelelement mim:attribuut ?attribuutsoort.
  ?nodeshape rdfs:seeAlso ?modelelement.
  ?nodeshape a sh:NodeShape.
  ?propertyshape rdfs:seeAlso ?attribuutsoort.
  ?propertyshape a sh:PropertyShape.

}
</pre>

### gegevensgroep (eigenschap)

Een `mim:gegevensgroep` wordt vertaald naar een `sh:property` die hoort bij de de NodeShape van de bezitter van het attribuut. Zie ook [Gegevensgroep](#gegevensgroep).

<pre class='ex-sparql'>
CONSTRUCT {
  ?nodeshape sh:property ?propertyshape
}
WHERE {
  ?modelelement mim:gegevensgroep ?gegevensgroep.
  ?nodeshape rdfs:seeAlso ?modelelement.
  ?nodeshape a sh:NodeShape.
  ?propertyshape rdfs:seeAlso ?gegevensgroep.
  ?propertyshape a sh:PropertyShape.

}
</pre>
