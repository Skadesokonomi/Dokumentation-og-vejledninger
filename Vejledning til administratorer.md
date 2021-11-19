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

- Alle SQL udtryk gemmes som tekst strenge i samme database i en ekstra administrations tabel - en parameter tabel - som indeholder alle nødvendige informationer for systemet udover datatabeller og lookup tabeller.

- Systemet udformes, således det er muligt at tilføje nye modeller - uden at skulle ændre i Plugin koden - ved at tilføje et nyt SQL udtryk som beregner modelværdier. Og evt. importere nødvendige data som nye tabeller

- Alle model SQL udtryk "generaliseres" således at benyttede tabel- og feltnavne samt konstanter erstattes af "tokens", dvs. et variabelnavn. Variablenavn og -værdi gemmes i føromtalte parameter tabel sammen med det generasliserede SQL udtryk.

- Parameter tabellen vil derfor indeholde informationer om oversættelse af "tokens" til aktuelle tabel- og feltnavne samt konstanter , således det er muligt at oversætte det generaliserede SQL udtryk til et reelt SQL
udtryk med referencer til de rigtige tabeller, felter og konstantværdier.  

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

(Ovenstående SQL udtryk vil foretage den ønskede beregning. Det er voldsomt simplificeret i forhold til systemets faktiske forespørgsel vedr. skadesberegning og værditab på bygninger.)

Men med ovenstående SQL udtryk gælder der følgende begrænsninger: 

- Tabellerne skal hedde et bestemt navn
- Tabellerne skal placeret i et bestemt schema.
- Felterne skal hedder noget bestemt
- Værdier for hhv. vanddybde og værditab kan ikke ændres.

For at gøre SQL udtrykkene mere fleksible generaliseres til følgende:

    SELECT DISTINCT 
      b.{f_pkid_t_building},
      b.{f_mun_code_t_building},
      st_area(b.{f_geom_t_building}) * k.{f_sqm_price_t_mun_sqmprice} * {p_loss_value} AS vaerdi_tab
    FROM {t_building} b 
    INNER JOIN {t_flood} o ON st_intersects(b.{f_geom_t_building}, o.{f_geom_t_flood}) 
    LEFT JOIN {t_mun_sqmprice} k ON b.{f_mun_code_t_building} = k.{f_mun_code_t_flood}
    WHERE o.{f_water_depth_t_flood} > {p_water_depth}

-Alle tabel- og feltnavne samt konstantværdier i SQL udtrykket erstattes af "tokens", som f.eks. "f_pkid_t_building". "tuborg" paranteserne {..} rundt om tokennavnet angiver at felt/tabel/konstanten er erstattet af et token.

- SQL udtrykket opfattes selv som et token og gemmes under eksempelvis navnet "q_building_flood_loss".  

- Tabel, felt og konstant tokens gemmes slutteligt i parametertabellen. Sammen med selve token-navnet gemmes også tilhørsforholdet til SQL udtrykket samt token værdien, dvs. det originale tabel / feltnavn eller konstant værdi. 

Få at køre en af brugeren valgt model foretager plugin'et følgende:

- Det generaliserede SQL udtryk findes i parameter tabellens vha. token navn.
- Alle øvrige tokens, som tilhører det generaliserede SQL udtryk findes også i parameter tabellen.
- Der foretages en "søg og erstat" i teksten for det generaliserede SQL udtryk som finder token navne og erstatter det med token værdier.
- SQL udtrykket - som nu indeholder korrekte tabel- og feltnavne samt konstant værdier eksekveres i databasen og resultatet vises i QGIS's kortvindue.

Metoden giver mulighed for at GIS administrator kan importere tabeller med vilkårlige tabel og feltnavne til skadeøkonomi databasen og derefter tilpasse de til modellen tilhørende tokenværdier med relevante tabel/feltnavne. 
GIS brugeren har via plugin'et mulighed for at dels at vælge en bestemt model, dels ændre på modellens konstantværdier før kørsel ved at ændre på de til modellen tilhørende token værdier for konstanter.

Resten af denne vejlening beskriver GIS administrators mulighed for opsætning af modeller med tilhørende tokens og hvorledes de stilles til rådighed for brugeren 


# Administrator vejledning for QGIS plugin "Skadesøkonomi"

Selve installationen af hhv. plugin og tilhørende database, inklusive opsætning af Postgres username/password, sikkerhedskopiering er gennegået i installationsvejledningen, så disse oplysning kan findes i denne vejledning.

### Opstart af plugin

I QGIS
