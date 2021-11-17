# Vejledning i opsætning af modeller og tabeller til Skadeøkonomi plugin

## Historie

Som beskrevet i tidligere afsnit er grundlaget for Skadeøkonomi plugin en række modeller, som beregner økonmiske- samt andre konsekvenser 
ved forskellige typer af oversvømmelser.

Det originale projekt var et sæt af Python - scripts beregnet til udførelse via ESRI ArcGIS.
Disse script fungerede ufhængigt af hinanden og benyttede kortlag vist i ArcGis samt brugerindtastede eller -valgte 
parameterværdier som grundlag for beregningerne. Resultatet var typisk et nyt kortlag inkl. et sæt tilhørende alfanumriske oplysninger.

Ved gennemgang af de originale scripts blev det afklaret, hvorledes disse scripts
generelt set fungerede:

1. Brugeren valgte to lag:

    - Et lag som indeholdt oversvømmelses polygoner med vandybde som alfanumrisk information
	- Et lag som indeholdt de objekter som var grundlaget for modellen, f.eks. bygninger, veje, infrastrukr, industri osv. 

2. Bruger indtastede eller valgte en række øvrige parametre, som indgik i den valgte modelberegning. Parametre kunne være minimum vanddybde, valg af oversvømmelsestype, antal timer med oversvømmelse o.lign.

3. Herefter blev modelberegningen gennemført. De i ArcGis indbyggede faciliteter blev benyttet til at foretage en overlapsanalyse mellem laget med oversvømmelsespolygoner og laget med iteresse objekter. Og for 
hvert oversvømmet objekt blev der udregnet en række værdier, f.eks. skadesberegninger og værditab, for hvert oversvømmet objekt udfra de øvrige pareamtre samt vanddybde.
ved afslutning blev resultatet præsenteret som et nyt kortlag.

## Idé grundlag for det nye system.

Det grundlæggende ønske var at udvikle et Open-Source baseret system, som kunne fungere på basis af et frit og gratis GIS produkt, i dette tilfælde QGIS. Endvidere var ønsket, at det skulle være nemmere at modificere
eksisterende og/eller tilføje nye modeller så nemt som muligt - i bedste fald uden at skulle programmere i Python.
 
Gennemgangen af de originale scripts viste, at det var muligt at samle/importere alle relevante datasæt til en spatiel database og benytte SQL til at gennemføre de spatielle søgninger og de øvrige beregninger.
 
Så følgende løsningsmodel blev gennemført

- Alle datasæt skal importeres til en spatiel database. Ved datasæt forstås oversvømmelsesdata, alle datasæt med interesseobjekter (bygninger, veje, infrafrastruktur osv.) samt lookupdata til brug for 
de forskellige beregninger

- Alle model beregninger defineres som SQL udtryk som (kun) benytter de tabeller, som forefindes i databasen.

- Alle SQL udtryk gemmes som tekst strenge i samme database i en ekstra administrations tabel som indeholder alle nødvendige informationer udover selve datatabeller og lookup tabeller.

- Systemet udformes, således det er muligt at tilføje nye modeller - uden at skulle ændre i Plugin koden - ved at tilføje et nyt SQL udtryk som beregner modelværdier. Og evt. importere nødvendige data som nye tabeller

- Alle model SQL udtryk "generaliseres" således at benyttede tabel- og feltnavne samt parameterkonstanter erstattes af "tokens", dvs. et variabelnavn. Variablenavn og -værdi gemmes i føromtalte parameter tabel.

- Føromtalte administrations tabel vil også indeholde informationer om oversættelse af "tokens" til aktuelle tabel- og feltnavne, således det er muligt at oversætte det generaliserede SQL udtryk til et reelt SQL
udtryk med referencer til de rigtige tabeller og felter.  

Et eksempel: 

Find alle bygninger, som bliver oversvømmet, men udeluk vanddybder mindre end 0.2 m. Værditabet for bygningen findes ved at gange den gennemsnitlige bygnings-kvadratmeterpris for kommunen med 0.1 (dvs. 10%)   

Tabel: "bygninger" indeholder bygningsdata. Tabellen indeholder felterne "kom_kode" (kommune kode), "geom" (bygningens geometri), "fid" (En entydig nøgleværdi for bygningen) 
Tabel: "oversvoem" indeholder oversvømmelses polygoner. Tabellen indeholder felterne: "geom" (oversvømmelsespolygon), "vanddybde" (vanddybde for polygonen) 
Tabel: "kvmpris" indeholder oplysninger om gennemsnitlige bygnings-kvadratmeterpris for kommuner. Tabellen indeholder felterne: "kom_kode" (kommunekode), "kvm_pris" (bygnings-kvadratmeter pris) 

    SELECT DISTINCT 
      b.fid,
      b.kom_kode,
      st_area(b.geom) * k.kvm_pris * 0.1 AS vaerdi_tab
    FROM data.bygninger b 
    INNER JOIN data.oversvoem o ON st_intersects(b.geom, o.geom) 
    LEFT JOIN admin.kvmpris k ON b.kom_kode = k.kom_kode
    WHERE o.vanddybde > 0.2

(Ovenstående SQL udtryk vil foretage den ønskede beregning. Det er voldsomt simplificeret i forhold til systemets rigtige forespørgsel vedr. skadesberegning og værditab på bygninger.)

Men: 

- 



Opsamling af IOT data 

Noter vedr. collector projekt.

## Data

- Data kommer fra en række sensorer forbundet til opsamler enhed via IOT (LoraWan netværk)

- Der sendes en original datapost hver 10 min.
- Efter ca. 0.5 sek gendsendes datapost, sandsynligvis som sikkehed mod datatab.
- Data består af følgende felter

    - "id" : Unikt id for hele tabellen(Serial), eks. 7 
    - "eui" : Unikt id for sensor, eks. "0004A30B00F25E63",
    - "bat" : , eks. 4.05,
    - "temp" : Temperatur (real), eks. 15.2
    - "humid" : Fugtighed (Real), eks. 82.4
    - "tryk" : Tryk målt i Pascal (Int), eks. 101965, svarende til 1019,65 milliBar  
    - "irra" : ? (int), eks. 306
    - "nedbor" : Nedbør målt i "tips" a' 0.2 mm. Tallet er et heltal mellem 0 og 255. Tælles løbende op. Efter 255 nulstilles så "256" bliver 0.
    - "stednavn" : Stednavn (Text) , eks. "Pederstrup",
    - "lat" : Breddegrad,(Real), eks. 55.734935,
    - "lon" : Længdegrad (Real),  12.336443,
    - "ts" : Timestamp (Timestamp with timezone) eks. "2021-09-22T09:34:51.910Z" (1)
    - "geom" : Position (geometyry), eks. "POINT (12.336443 55.734935)" (2)
 
 
Note 1: Format for timestamp:  YYYY-MM-DDThh:mm:ss.sTZD
- YYYY = four-digit year
- MM   = two-digit month (01=January, etc.)
- DD   = two-digit day of month (01 through 31)
- hh   = two digits of hour (00 through 23) (am/pm NOT allowed)
- mm   = two digits of minute (00 through 59)
- ss   = two digits of second (00 through 59)
- s    = one or more digits representing a decimal fraction of a second
- TZD  = time zone designator (Z or +hh:mm or -hh:mm)
	 
Note 2: Geometi. Opbevares binært, formateret som WKT. 

### Datastuktur

Rådata er ikke normaliseret. Normaliseret struktur kunne være: 

- Tabel "sensor":

    - "eui" : Unikt id for sensor, eks. "0004A30B00F25E63", PK for tabel
    - "stednavn" : Stednavn (Text) , eks. "Pederstrup",
    - "lat" : Breddegrad,(Real), eks. 55.734935,
    - "lon" : Længdegrad (Real),  12.336443,
    - "geom" : Position (geometyry), eks. "POINT (12.336443 55.734935)" (2)

- Tabel "regnvandsmaalinger"

    - "id" : Unikt id for hele tabellen(Serial), eks. 7 , PK for tabel, pricipielt kunne blive erstattet af "eui" + "ts"
    - "eui" : Unikt id for sensor, eks. "0004A30B00F25E63", FK til tabel "sensor"
    - "ts" : Timestamp (Timestamp with timezone) eks. "2021-09-22T09:34:51.910Z" (1)
    - "bat" : , eks. 4.05,
    - "temp" : Temperatur (real), eks. 15.2
    - "humid" : Fugtighed (Real), eks. 82.4
    - "tryk" : Tryk målt i Pascal (Int), eks. 101965, svarende til 1019,65 milliBar  
    - "irra" : ? (int), eks. 306
    - "nedbor" : Nedbør målt i "tips" a' 0.2 mm. Tallet er et heltal mellem 0 og 255. Tælles løbende op. Efter 255 nulstilles så "256" bliver 0.
	- "nedbor_beregnet" : Nedbørsværdi beregnet af trigger, se næste afsnit.

### Indlæsning af data 

Der er primært to knækpunkter:

- Hvis ellers netværket fungerer bliver alle data input i praksis duplikeret, fordi sensoren sender prcifikke data hver 10 minut. Men gør det 2 gange med ca. 0.4 sekunders mellemrum. Så man skalluge ud i data
- Nedbørsdata tælles op som "tips". 
    - Hvert tip svarer til 0.2 mm regn. 
	- Data bliver ikke nulstillet pr. døgn.
	- Når tip - lælleren når 256 nulstilles tælleren. 
	
    Så nebør skal specialbehandles: Der skal tages hensyn til tip værdien ved midnat og at tælleren faktisk kan nulstilles på vilkårlige tidspunkter

#### Foreslået løsning:

- De 2 normaliserede tabeller oprettes

- Der skrives en trigger til "modtager" modellen som gør følgende:

    - Splitter data op til hhv tabel ""sensor" og tabel "regnvandsmaalinger"
	
	- Data til tabel "sensor" føges indsat. Hvis det ikke lykkedes, er den specifikke sensor allerede genereret. Der foretages en update i stedet (3)
	
	1. Find sidst indsatte data i tabel "regnvandsmaalinger". Stemmer dato  for den spec 
	Data i tabel "regnvandsmaalinger" undersøges. Er der allerede indlagt andre data indenfor eks. 5 minutter smides nuv. data bort. Hvis sidst indlagte data er ældre en 5 minutter indsættes de nye data