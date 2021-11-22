# Vejledning i opsætning af modeller og tabeller til Skadeøkonomi plugin

## Historie

Som beskrevet i tidligere afsnit er grundlaget for Skadeøkonomi plugin en række modeller, som beregner økonomiske 
og andre konsekvenser ved forskellige typer af oversvømmelser.

Det originale projekt var et sæt af Python - scripts beregnet til udførelse via ESRI ArcGIS. Disse script fungerede 
ufhængigt af hinanden og benyttede kortlag vist i ArcGis samt brugerindtastede eller -valgte parameterværdier som 
grundlag for beregningerne. Resultatet var typisk et nyt kortlag inkl. et sæt tilhørende alfanumriske oplysninger.

Ved gennemgang af de originale scripts blev det afklaret, hvorledes disse scripts generelt set fungerede:

1. Brugeren valgte to lag:

    - Et lag som indeholdt oversvømmelses polygoner med vandybde som alfanumrisk information
	- Et lag som indeholdt de objekter som var grundlaget for modellen, f.eks. bygninger, veje, infrastrukr, industri 
osv. 

2. Bruger indtastede eller valgte en række øvrige parametre, som indgik i den valgte modelberegning. Parametre kunne 
være minimum vanddybde, valg af oversvømmelsestype, antal timer med oversvømmelse o.lign.

3. Herefter blev modelberegningen gennemført. De i ArcGis indbyggede faciliteter blev benyttet til at foretage en
overlapsanalyse mellem laget med oversvømmelsespolygoner og laget med iteresse objekter. Og for hvert oversvømmet 
objekt blev der udregnet en række værdier, eksempelvis skadesberegninger og værditab, baseret på øvrige parametre
samt vanddybde. Ved afslutning blev resultatet præsenteret som et nyt kortlag.

## Idé grundlag for det nye system.

Det grundlæggende ønske var at supplere de originale ArcGis baserede scripts  med et Open-Source baseret system, som 
kunne fungere på basis af et frit og gratis GIS produkt, i dette tilfælde QGIS. Endvidere var ønsket, 
at det skulle være nemmere at modificere eksisterende og/eller tilføje nye modeller så nemt som muligt - 
i bedste fald uden at skulle programmere i Python.
 
Gennemgangen af de originale scripts viste, at det var muligt at samle/importere alle relevante datasæt til 
en spatiel database og derefter benytte SQL til at gennemføre modelberegningerne.
 
Så følgende løsningsmetode blev gennemført:

- Alle datasæt importeres til en spatiel database. Ved datasæt forstås eks. oversvømmelsesdata, alle datasæt med 
interesseobjekter (bygninger, veje, infrafrastruktur osv.) samt lookupdata til brug for de forskellige beregninger

- Alle model beregninger defineres som SQL udtryk som benytter de tabeller, som forefindes i databasen.

- Systemet udformes, således det er muligt at tilføje nye modeller - uden at skulle ændre i Plugin koden - ved at 
tilføje et nyt SQL udtryk som beregner modelværdier. Og evt. importere nødvendige data som nye tabeller

- Alle model SQL udtryk "generaliseres" således at benyttede tabel- og feltnavne samt konstanter erstattes af "tokens", 
dvs. et variabelnavn. Variablenavn og -værdi gemmes i føromtalte parameter tabel sammen med det generasliserede SQL 
udtryk.

- Alle de generaliserede SQL udtryk gemmes som tekst strenge i databasen i en speciel administrations tabel - en 
parameter tabel, som - udover SQL udtrykkene - indeholder alle nødvendige informationer for systemet, dvs. parametre, 
hjælpetekster osv. 

- Parameter tabellen vil også indeholde informationer om oversættelse af "tokens" til aktuelle tabel- og feltnavne 
samt konstanter , således det er muligt at oversætte det generaliserede SQL udtryk til et reelt SQL
udtryk med referencer til de faktiske tabeller, felter og konstantværdier.  

Et eksempel: 

Find alle bygninger, som bliver oversvømmet, men udeluk vanddybder mindre end 0.2 m. Værditabet for bygningen findes 
ved at gange den gennemsnitlige bygnings-kvadratmeterpris for kommunen med 0.1 (dvs. 10%)   

Tabel: "bygninger" indeholder bygningsdata. Tabellen indeholder felterne "kom_kode" (kommune kode), "geom" (bygningens 
geometri), "fid" (En entydig nøgleværdi for bygningen) 
Tabel: "oversvoem" indeholder oversvømmelses polygoner. Tabellen indeholder felterne: "geom" (oversvømmelsespolygon), 
"vanddybde" (vanddybde for polygonen) 
Tabel: "kvmpris" indeholder oplysninger om gennemsnitlige bygnings-kvadratmeterpris for kommuner. Tabellen indeholder 
felterne: "kom_kode" (kommunekode), "kvm_pris" (bygnings-kvadratmeter pris) 

(Nedenstående SQL udtryk vil foretage den ønskede beregning. Det er voldsomt simplificeret i forhold til systemets 
faktiske forespørgsel vedr. skadesberegning og værditab på bygninger.)


    SELECT DISTINCT 
      b.fid,
      b.kom_kode,
      st_area(b.geom) * k.kvm_pris * 0.1 AS vaerdi_tab
    FROM data.bygninger b 
    INNER JOIN data.oversvoem o ON st_intersects(b.geom, o.geom) 
    LEFT JOIN admin.kvmpris k ON b.kom_kode = k.kom_kode
    WHERE o.vanddybde > 0.2

Men med ovenstående SQL udtryk gælder der følgende begrænsninger: 

- Tabellerne skal hedde et bestemt navn
- Tabellerne skal placeret i et bestemt schema.
- Felterne skal hedder noget bestemt
- Konstantværdier for hhv. vanddybde og værditab kan ikke ændres.

For at gøre SQL udtrykkene mere fleksible generaliseres udtrykket til følgende:

    SELECT DISTINCT 
      b.{f_pkid_t_building},
      b.{f_mun_code_t_building},
      st_area(b.{f_geom_t_building}) * k.{f_sqm_price_t_mun_sqmprice} * {p_loss_value} AS vaerdi_tab
    FROM {t_building} b 
    INNER JOIN {t_flood} o ON st_intersects(b.{f_geom_t_building}, o.{f_geom_t_flood}) 
    LEFT JOIN {t_mun_sqmprice} k ON b.{f_mun_code_t_building} = k.{f_mun_code_t_flood}
    WHERE o.{f_water_depth_t_flood} > {p_water_depth}

-Alle tabel- og feltnavne samt konstantværdier i SQL udtrykket erstattes af "tokens", som f.eks. "f_pkid_t_building". 
"tuborg" paranteserne {..} rundt om tokennavnet angiver at navnet er et token og ikke et reelt tabel/feltnavn eller 
konstantværdi.
- Selve SQL udtrykket gives også et navn, f.eks. "q_building_flood_loss" og gemmes i parametertabellen. 

- Tabel, felt og konstant tokens gemmes også i parametertabellen. Sammen med de øvrige token oplysningerne gemmes også 
tilhørsforholdet til SQL udtrykket representeret ved dets token navn. 

Få at udføre en af brugeren valgt model foretager plugin'et nu følgende:

- Det generaliserede SQL udtryk findes i parameter tabellens vha. udtrykkets navn.
- Alle øvrige tokens, som tilhører det generaliserede SQL udtryk findes også i parameter tabellen.
- Der foretages en "søg og erstat" i teksten for det generaliserede SQL udtryk som finder token navne og erstatter disse  
med token værdier.
- SQL udtrykket - som nu indeholder korrekte tabel- og feltnavne samt konstant værdier eksekveres i databasen og 
resultatet vises i QGIS's kortvindue.

Metoden giver mulighed for at GIS administrator kan importere tabeller med vilkårlige tabel- og feltnavne til 
skadeøkonomi databasen og derefter tilpasse de til modellen tilhørende tokenværdier med relevante tabel/feltnavne. 

GIS brugeren har via plugin'et mulighed for at dels at vælge en bestemt model, dels ændre på modellens konstantværdier 
før kørsel ved at ændre på de til modellen tilhørende token værdier for konstanter.

# Parameter tabel

Placering og navn på parametertabel i databasen angives af administrator vha. valg og indtastningsfelter i Plugin. Se 
senere i denne vejledning. 

Parameter tabellen en den centrale informationstabel for Skadeøkonomi plugin'et. 

Den indeholder *alle* nødvendige oplysninger til at beskrive og udføre de forskellige modeller, bl.a. generaliserede 
SQL udtryk for modellerne, alle "token" navne inkl. oversættelse for tabeller, felter, søge konstanter osv. 

Endvidere indeholder tabellen en lang række andre administrative oplysninger nødvendige for at plugin'et kan 
vise modelopbygningerne korrekt og at brugerne kan vælge modeller og sætte søgeværdier for for de enkelte modeller.

Tabellen indeholder derfor en række forskellige oplysninger udover navn/værdi par. De forskellige kolonner og deres indhold 
beskrives senere i dette afsnit.

## Struktur og feltbeskrivelse

|Feltnavn|Type|Forklaring|
|---|---|---|
|name|tekst|Navn for token; skal være unikt i hele tabellen   |
|parent|tekst|Navn på den token, som denne parameterpost er bundet til. Hvis den ikke er bundet til nogen post, så efterlades den blank.|
|value|tekst|Værdi af token; alle værdier er præsenteret som en tekststreng uanset type.|
|type|karakter|Token type; kan være een af følgende: "G", "I", "O", "P" eller "T"; se afsnit *Brug af "type" feltet|
|minval|tekst| For type lig med "I" (heltal) eller "R" (Reelt tal): Den mindste værdi, som felt "value" kan indeholde|
|maxval|tekst| For type lig med "I" (heltal) eller "R" (Reelt tal): Den største værdi, som felt "value" kan indeholde  |
|lookupvalues|tekst| For type = I" eller "R" (Reelt tal): Step værdi i indtastningsboks ved værdirettelse; For type = "O": Liste over mulige værdier for felt "value" i combobox|
|default|tekst| Hvis parameter representerer et modelnavn, skal feltet indeholde navnet på den bagved liggende query|
|explanation|tekst| Tekst, som dokumenterer hvad parameteren benyttes til; vises, når markøren placeres over parameter-navn eller værdifelt i fanebladet.|
|sort|tekst| Sorteringsrækkefølge indenfor en gruppe; mindste tal øverst|
|checkable|karakter| Hvis sat til "T", vil faneblads representation for parameteren indeholde et afkrydsningsfelt|


## Retningslinjer for værdisætning af parametertabel.



### Navne for tokens
De fleste (men ikke alle) navne i parameter tabellen kan angives med en valgfri værdi. Men det anbefales *kraftigt* at overholde følgende 
konvetioner ved tilføjelse af nye modeller, forespørgsler, tabeller og felter:

- Alle token navne for *tabeller* starter med "t_". Resten af navnet reflekterer funktionen af tabellen. Ved navngivning af token for de 
oprindelige 11 modeller er det valgt at lade funktionsbeskrivelsen være på engelsk - for at markere disse data som 
"noget GIS administarorer behandler".

    |Token navn|Peger på|
    |---|---|
    |t_bioscore| Bioscore tabel|
    |t_build_usage| Bygningsanvendelse (administrationstabel)|
    |t_building| Bygningstabel|
    |t_company| Firma tabel|
    |t_damage| Skadefunktioner (administrations tabel)|
    |t_flood| Oversvømmelses tabel|
    |t_human_health| Oplysninger om beboere|
    |t_infrastructure| Infrastruktur tabel|
    |t_publicservice| Public service områder|
    |t_recreative| Rekreative områder|
    |t_road_traffic| Vejdata tabel|
    |t_sqmprice| Gennemsnitlig kvm.pris fordelt på kommuner (administrations tabel)|
    |t_tourism| Turisme tabel|

- Token navn for generaliserede SQL forespørgsler starter med "q_", efterfulgt af query funktion på engelsk.

    Eksempel: "q_building": Skademodel fro bygninger. Generelt set følger query-navne samme mønster som for tabeller. 

- Tokennavne for felter til tabeller og forespørgsler starter med "f_", efterfulgt af en funktionsbeskrivelse for feltet og afsluttes 
med token navn for den tabel/query, som feltet tilhører.

    Der findes pt. følgede funktionsbeskrivelser for felter

    |Token navn prefix|Funktion|
    |---|---|
    |f_pkey_| Feltet er primary-key felt|
    |f_geom_| Feltet er geometry felt|
    |f_usage_code_| Feltet indeholder en bbr-kode|
    |f_category_ | Feltet indeholder en hovedkategori for bygningsanvendelse|
    |f_risk_| Feltet indeholder et beregnet risiko beløb|
    |f_loss_| Feltet indeholder et beregnet værditab|
    |f_damage_| Feltet indeholder et beregnet skadesbeløb|
    |f_muncode_| Feltet indeholder en kommunekode|
	
   Øvrige felt token-navne bør beskrive feltets funktion på engelsk

   Nogle eksempler: "f_pkid_t_building": primary key for bygningstabel; "f_geom_t_building": geometri felt for bygningstabel;  

De ovenstående regler er *anbefalinger* ikke *absolutte krav* undtaget følgende: Angivelser for hhv. primary key og geometri 
felter for queries *skal* følgen formlerne; ellers vil systemt fejle ved genereringen af resultat tabeller. 

### Brug af "type" feltet

"type" feltet bestemmer dels hvad der kan opbevares i "value" feltet; dels hvilken visuel funktion parameteren har.

"type" kan være e af følgende værdier:

- "G": Parameter benyttes som en gruppe, dvs. opdeling af paramterposte i logiske sammenhænge. Den indeholder ingen værdi, men paramter
 navnet vises i fanebladenes træstruktur. Paramtre, som skal vises under denne gruppe skal tildeles gruppe parameterens navn som 
 værdi i felt "parent"

    Eksempel: I faneblad "Generelt" findes en parameterpost "Name templates". Parametrene "Cell layername" "Group name template", 
	"Main groupname" o.a. har alle fået tildelt "Name templates" som "parent". Det giver følgende resultat: 
	
	(Billede)
	
	
- "I": Parameter skal indeholde et heltal. Når man dobbeltklikker på paramternavnet eller værdi feltet i fanebladet, vil der
 vises en bruger dialog, hvor værdien kan ændres. Dialogen bliver udformet, således der kun kan indtastes et heltal. Minimu og maksimum 
 værdier bestemmes af værdierne indtastet i felterne "minval" og "maxval". Stepstørrelsen i det tilhørende rullefelt bestemmes af værdien 
 i felt "lookupvalues"
 
     Eksempel: Parameter "Cell size" har type "I", værdi 100, minval: 10, maxval: 1000, lookvalue: 50. I dialogen vise værdien 100, som kan ændres fra 10 til 1000. Og bruges scroll-pilene ændres værdier i skridt af 50. 

- "R": Parameter skal indeholde et reelt tal. Når man dobbeltklikker på paramternavnet eller værdi feltet i fanebladet, vil der
 vises en bruger dialog, hvor værdien kan ændres. Dialogen bliver udformet, således der kun kan indtastes et reelle tal. Minimum og 
 maksimum værdier bestemmes af værdierne indtastet i felterne "minval" og "maxval". Stepstørrelsen i det tilhørende rullefelt bestemmes 
 af værdien i felt "lookupvalues"

- "O" Parameter skal indeholde een bestemt værdi udvalgt fra en forudbestemt liste. Listen er værdisat i felt "lookupvalues" hvor den optræder 
som en tekst, som er sammensat af de mulige værdier adskilt af tegnet "¤" (SHIFT-4 på et dansk tastatur) . Ved dobbeltklik på enten navn 
eller værdifelt i fanebladene vil der vises en valgboks, som indeholder elementerne fra valglisten.

    Eksempel: Parameter "Skadetype", har type "O" og lookupvalues er sat til "*Skadebeløb¤Værditab¤Skadebeløb og værditab¤Intet (0 kr.)*". Ved 
	dobbeltklik vises en comboboks, som indeholder de 4 valgmuligheder.  
	
- "P": Paramter kan indeholde en multilinje tekst. Der er ingen begrænsninger i antal linjer eller linjernes længde. Ved dobbelklik på navn eller værdifelt 
vises en multiline tekstboks med den nuv. tekst.

- "T": Paramter kan indeholde en enkelt linje tekst. Der er ingen begrænsninger linjens længde. Ved dobbelklik på navn eller værdifelt 
vises en tekstboks med den nuv. tekst.

### Opbygning af hieraki vha. "parent" feltet

Data i paramtertabellen arrangeres i et hieraki, hvor stort set alle parameterposter "ejes" af en anden post. 

Eksempel: En parameterpost beskriver hvorledes du oversætter et token navn til en aktuel tabel i databasen. Denne post vil 
"eje" en række underposter, hvor de enkelte underposter beskriver hvorledes token for feltnavne i den specifikke tabel 
oversættes til reelle feltnavne.

Denne facilitet benyttes også til at visuelt at placere de enkelter parameterposter korrekt i plugin'ets fanebladssystem: Plugin'et har 5 faneblade, 
som viser data fra parametertabellen: "Generel", "Forespørgsler", "Data", "Modeller" og "Rapporter".

Hvert faneblad representeres af en gruppepost (parameter type "G") i parametertabellen med samme navn som fanebladet. Disse gruppe poster fungerer som 
"rod" for hvert faneblad. Alle parameterposter - og deres underposter - , som har en af disse "rod"-poster som parent placeres i det tilsvarende faneblad.   








# Administrator vejledning for QGIS plugin "Skadesøkonomi"

Selve installationen af hhv. plugin og tilhørende database, inklusive opsætning af Postgres username/password, 
sikkerhedskopiering er gennegået i installationsvejledningen, så disse oplysning kan findes i denne vejledning.

### Opstart af plugin

I QGIS

|Gruppe|Navn|Værdi|Funktion|
|---|---|---|---|
| |General| |Hovedgrupper til administration af grundlæggende parametre for systemet|
| |Queries| |Hovedgrupper til administration af Forespørgsler|
| |Data| |Hovedgruppe for administration af Tabeller|
| |Reports| |Hovedgruppe til administration og kørsel af Rapporter|
| |Models| |Hovedgruppe for administration og kørsel af  Modeller|
|Biodiversitet|Biodiversitet, kort| |Sæt hak såfremt modellen skal identificere særlige levesteder for rødlistede arter som bliver berørt i forbindelse med den pågældende oversvømmelseshændelse. Her vises levestederne geografisk på et kort.|
|Biodiversitet|Biodiversitet, opsummering| |Sæt hak såfremt modellen skal identificere særlige levesteder for rødlistede arter som bliver berørt i forbindelse med den pågældende oversvømmelseshændelse. Her opsummeres de berørte levesteder i en tabel.|
|Bygninger|Værditab, skaderamte bygninger (%)|10.0|Her angives størrelsen på reduktionen i salgspris for de bygninger som bliver berørt af den pågældende oversvømmelse. Tabet beregnes som en procentsats, som angives af brugeren, af den gennemsnitlige kommunale m2 pris for solgte boliger i løbet af de seneste år. Det anbefales at anvende værdien 10% såfremt man ikke har bedre data.|
|Bygninger|Skadeberegninger, Bygninger| |Skadeberegning for bygninger, forskellige skademodeller, med eller uden kælderberegning|
|Bygninger|Værditab nabobygninger| ||
|Data|t_bioscore|fdc_data.biodiversitet|Parametergruppe til tabel "biodiversitet"|
|Data|t_build_usage|fdc_admin.bbr_anvendelse|Parametergruppe til opslagstabel "bbr-anvendelse"|
|Data|t_building|fdc_data.bygninger|Parametergruppe til tabel "Bygninger"|
|Data|t_company|fdc_data.industri|Parametergruppe til tabel "industri"|
|Data|t_damage|fdc_admin.skadefunktioner|Parametergruppe til opslagstabel "skadesfunktioner"|
|Data|t_flood|fdc_data.oversvoem|Parametergruppe til tabel "oversvømmelser"|
|Data|t_human_health|fdc_data.mennesker|Parametergruppe til tabel "mennesker"|
|Data|t_infrastructure|fdc_data.kritisk_infrastruktur|Parametergruppe til tabel "kritisk infrastruktur"|
|Data|t_publicservice|fdc_data.offentlig_service| |
|Data|t_recreative|fdc_data.rekreative_omr|Parametergruppe til tabel "rekreative områder"|
|Data|t_road_traffic|fdc_data.vejnet|Parametergruppe til tabel "vejnet"|
|Data|t_sqmprice|fdc_admin.kvm_pris|Parametergruppe til opslagstabel "kommunal kvm. pris"|
|Data|t_tourism|fdc_admin.turisme|Parametergruppe til opslagstabel "turisme"|
|General|Cell size|100| |
|General|Name templates| |Navne skabeloner til diverse formål|
|General|SQL command templates| |SQL skabeloner til diverse formål|
|Generelle modelværdier|Returperiode for hændelse i dag (år)|20|Indtast returperioden under nuværende klima for den oversvømmelseshændelse som der beregnes skader og risiko for. Returperioden angives i år.|
|Generelle modelværdier|Returperiode for hændelse i fremtiden (år)|5|Indtast returperioden om 100 år under fremtidigt klima for den oversvømmelseshændelse som der beregnes skader og risiko for. Dette kan f.eks. være under klimascenarie RCP4.5 eller RCP8.5 Returperioden angives i år.|
|Generelle modelværdier|Medtag i risikoberegninger|Skadebeløb|Her vælger man om det kun er skadeomkostningen eller skadeomkostning inkl. værditab for bygninger som inkluderes i risikoberegningen.|
|Generelle modelværdier|Minimum vanddybde (meter)|0.3|Her angives den minimale vanddybde på terræn som der skal til for at der opstår økonomiske tab i forbindelse med oversvømmelsen. Denne værdi angives i m, og anvendes kun for de sektorer hvor der ikke er angivet en alternativ minimum vanddybde.|
|Industri|Industri, personale i bygninger| |Sæt hak såfremt modellen skal identificere de virksomheder som bliver berørt af den pågældende oversvømmelse, og angive antallet af medarbejdere per virksomhed.|
|Industri|Industri, punkter fejlplaceret| ||
|Kritisk infrastruktur|Oversvømmet infrastruktur| |Sæt hak såfremt modellen skal identificere kritisk infrastruktur som bliver berørt i forbindelse med den pågældende oversvømmelseshændelse.|
|Mennesker og helbred|Humane omkostninger | |Sæt hak såfremt der skal beregnes økonomiske tab ifm. oprydning, sygedage og feriedage som en konsekvens af ens ejendom har været oversvømmet.|
|Models|Generelle modelværdier| |Afsnit, hvor parametre, der bruges i mange forskellige modeller, kan værdisættes|
|Models|Bygninger| |Skademodeller for Bygninger|
|Models|Kritisk infrastruktur| |Skademodeller for Kritisk infrastruktur|
|Models|Offentlig service| |Skademodeller for Offentlig service|
|Models|Vej og trafik| |Skademodeller for Vej og trafik|
|Models|Biodiversitet| |Skademodeller for Biodiversitet|
|Models|Industri| |Skademodeller for Industri|
|Models|Rekreative områder| |Skademodeller for Rekreative områder|
|Models|Turisme| |Skademodeller for Turisme|
|Models|Mennesker og helbred| |Skademodeller for Mennesker og helbred|
|Name templates|Cell layername|celler| |
|Name templates|Name of model value section|Generelle modelværdier| |
|Name templates|Group name template|Kørsel: {time_stamp}| |
|Name templates|Main groupname|Modeller| |
|Name templates|Model_layergroup|Resultater fra modelkørsler| |
|Name templates|Result_schema|fdc_results|Name of schema to place result tables in|
|Offentlig service|Oversvømmet offentlig service| |Sæt hak såfremt modellen skal identificere offentlig service som bliver berørt i forbindelse med den pågældende oversvømmelseshændelse.|
|q_bioscore_alphanumeric|f_pkey_q_bioscore_alphanumeric|fid| |
|q_bioscore_spatial|f_geom_q_bioscore_spatial|geom_clip| |
|q_building|f_cellar_damage_q_building|skadebeloeb_kaelder_kr| |
|q_building|f_damage_q_building|skadebeloeb_kr| |
|q_building|f_loss_q_building|vaerditab_kr| |
|q_building|f_risk_q_building|risiko_kr| |
|q_building|f_geom_q_building|geom_byg|Field name for geometry column|
|q_building|f_pkey_q_building|fid|Name of primary keyfield for query|
|q_comp_build|f_geom_q_comp_build|geom_byg|Field name for building geometry|
|q_comp_build|f_pkey_q_comp_build|fid| |
|q_comp_nobuild|f_geom_q_comp_nobuild|geom| |
|q_comp_nobuild|f_pkey_q_comp_nobuild|fid| |
|q_human_health|f_damage_q_human_health|omkostning_kr| |
|q_human_health|f_risk_q_human_health|risiko_kr| |
|q_human_health|f_geom_q_human_health|geom| |
|q_human_health|f_pkey_q_human_health|fid| |
|q_infrastructure|f_geom_q_infrastructure|geom| |
|q_infrastructure|f_pkey_q_infrastructure|objectid| |
|q_publicservice|f_geom_q_publicservice|geom| |
|q_publicservice|f_pkey_q_publicservice|objectid| |
|q_recreative|f_damage_q_recreative|total_omkost_kr| |
|q_recreative|f_risk_q_recreative|risiko_kr| |
|q_recreative|f_geom_q_recreative|geom| |
|q_recreative|f_pkey_q_recreative|id| |
|q_road_traffic|f_damage_q_road_traffic|pris_total_kr| |
|q_road_traffic|f_risk_q_road_traffic|risiko_kr| |
|q_road_traffic|f_geom_q_road_traffic|geom| |
|q_road_traffic|f_pkey_q_road_traffic|id| |
|q_surrounding_loss|f_loss_q_surrounding_loss|vaerditab_kr| |
|q_surrounding_loss|f_risk_q_surrounding_loss|risiko_kr| |
|q_surrounding_loss|f_geom_q_surrounding_loss|geom_byg| |
|q_surrounding_loss|f_pkey_q_surrounding_loss|fid| |
|q_tourism_alphanumeric|f_pkey_q_tourism_alphanumeric|fid| |
|q_tourism_spatial|f_damage_q_tourism_spatial|tot_kr| |
|q_tourism_spatial|f_risk_q_tourism_spatial|risiko_kr| |
|q_tourism_spatial|f_geom_q_tourism_spatial|geom|Field name for geometry field in tourism query|
|q_tourism_spatial|f_pkey_q_tourism_spatial|id| |
|Queries|q_infrastructure|SELECT DISTINCT
  k.*
  FROM {t_infrastructure} k
  INNER JOIN {t_flood} o on st_intersects(k.{f_geom_t_infrastructure},o.{f_geom_t_flood})
  WHERE o.{f_depth_t_flood} >= {Minimum vanddybde (meter)}| |
|Queries|q_publicservice|SELECT DISTINCT
  k.*
  FROM {t_publicservice} k
  INNER JOIN {t_flood} o on st_intersects(k.{f_geom_t_publicservice},o.{f_geom_t_flood})
  WHERE o.{f_depth_t_flood} >= {Minimum vanddybde (meter)}| |
|Queries|q_bioscore_alphanumeric|  SELECT
  row_number() over () as {f_pkey_q_bioscore_alphanumeric},    b.{f_bioscore_t_bioscore} as bioscore,
  sum(st_area(st_intersection(b.{f_geom_t_bioscore},o.{f_geom_t_flood})))::NUMERIC(12,2) AS sum_area_m2,
  min(o.{f_depth_t_flood})::NUMERIC(12,2) as min_depth_m,
  max(o.{f_depth_t_flood})::NUMERIC(12,2) as max_depth_m
  FROM {t_bioscore} b
  INNER JOIN {t_flood} o on st_intersects(b.{f_geom_t_bioscore},o.{f_geom_t_flood})
  WHERE o.{f_depth_t_flood} >= {Minimum vanddybde (meter)}
  group by b.{f_bioscore_t_bioscore};| |
|Queries|q_bioscore_spatial|SELECT
  b.{f_pkey_t_bioscore} as pkey,
  b.{f_bioscore_t_bioscore} as bioscore,
  st_area(st_intersection(b.{f_geom_t_bioscore},o.{f_geom_t_flood}))::NUMERIC(12,2) AS areal_m2,
  st_intersection(b.{f_geom_t_bioscore},o.{f_geom_t_flood}) as geom_clip,
  o.{f_depth_t_flood} as depth
  FROM {t_bioscore} b
  INNER JOIN {t_flood} o on st_intersects(b.{f_geom_t_bioscore},o.{f_geom_t_flood})
  WHERE o.{f_depth_t_flood} >= {Minimum vanddybde (meter)}
  | |
|Queries|q_building|WITH ob AS
(
    SELECT
        b.{f_pkey_t_building} as {f_pkey_q_building},
        b.{f_muncode_t_building} AS kom_kode,
		b.{f_usage_code_t_building} AS bbr_kode,
        b.{f_cellar_area_t_building}::NUMERIC(12,2) AS areal_kaelder_m2,
        st_area(b.{f_geom_t_building})::NUMERIC(12,2) AS areal_byg_m2,
        {Værditab, skaderamte bygninger (%)}::NUMERIC(12,2) as tab_procent,
        st_force2d(b.{f_geom_t_building}) AS {f_geom_q_building},
        COUNT (*) AS cnt_oversvoem,
        (SUM(st_area(st_intersection(b.{f_geom_t_building},o.{f_geom_t_flood}))))::NUMERIC(12,2) AS areal_oversvoem_m2,
        (MIN(o.{f_depth_t_flood}) * 100.00)::NUMERIC(12,2) AS min_vanddybde_cm,
        (MAX(o.{f_depth_t_flood}) * 100.00)::NUMERIC(12,2) AS max_vanddybde_cm,
        (AVG(o.{f_depth_t_flood}) * 100.00)::NUMERIC(12,2) AS avg_vanddybde_cm
        --(SUM(o.{f_depth_t_flood}*st_area(st_intersection(b.{f_geom_t_building},o.{f_geom_t_flood}))) * 100.0 / SUM(st_area(st_intersection(b.{f_geom_t_building},o.{f_geom_t_flood}))))::NUMERIC(12,2) AS avg_vanddybde_cm
    FROM {t_building} b
    INNER JOIN {t_flood} o on st_intersects(b.{f_geom_t_building},o.{f_geom_t_flood})
    WHERE o.{f_depth_t_flood} >= {Minimum vanddybde (meter)}
    GROUP BY b.{f_pkey_t_building}, b.{f_muncode_t_building}, b.{f_usage_code_t_building}, b.{f_cellar_area_t_building}, b.{f_geom_t_building}
),
os AS
(
    SELECT ob.*, 
        u.{f_pkey_t_build_usage} AS bbr_anv_kode,
        u.{f_usage_text_t_build_usage} AS bbr_anv_tekst,
        d.{f_category_t_damage} AS skade_kategori,
        d.{f_type_t_damage} AS skade_type,
        k.{f_sqmprice_t_sqmprice}::NUMERIC(12,2) as kvm_pris_kr,
        (k.{f_sqmprice_t_sqmprice} * ob.areal_byg_m2 * {Værditab, skaderamte bygninger (%)}/100.0)::NUMERIC(12,2) as {f_loss_q_building},
        (d.b0 + ob.areal_byg_m2 * (d.b1 * ln(ob.max_vanddybde_cm) + d.b2))::NUMERIC(12,2) AS {f_damage_q_building},
        CASE
	        WHEN '{Skadeberegning for kælder}' = 'Medtages ikke' THEN 0.0
	        WHEN '{Skadeberegning for kælder}' = 'Medtages' THEN ob.areal_kaelder_m2 * d.c0 
        END::NUMERIC(12,2) as {f_cellar_damage_q_building}

    FROM ob
    LEFT JOIN {t_build_usage} u on ob.bbr_kode = u.{f_pkey_t_build_usage}
    LEFT JOIN {t_damage} d on u.{f_category_t_build_usage} = d.{f_category_t_damage} AND d.{f_type_t_damage} = '{Skadetype}'   
    LEFT JOIN {t_sqmprice} k on (ob.kom_kode = k.{f_muncode_t_sqmprice})

)
SELECT 
    os.*,
	(
	CASE
	    WHEN '{Medtag i risikoberegninger}' = 'Intet (0 kr.)' THEN 0.0
	    WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb' THEN os.{f_damage_q_building} + os.{f_cellar_damage_q_building}
	    WHEN '{Medtag i risikoberegninger}' = 'Værditab' THEN os.{f_loss_q_building}
	    WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb og værditab' THEN os.{f_damage_q_building} + os.{f_cellar_damage_q_building} + os.{f_loss_q_building} 
	END * (0.089925/{Returperiode for hændelse i fremtiden (år)} + 0.21905/{Returperiode for hændelse i dag (år)}))::NUMERIC(12,2) AS {f_risk_q_building}
FROM os
|SQL template for "Bygninger" model |
|Queries|q_comp_build|WITH fb AS
(
  SELECT DISTINCT
    b.{f_pkey_t_building} as {f_pkey_q_comp_build},
    b.{f_muncode_t_building} AS kom_kode,
    b.{f_usage_code_t_building} AS bbr_anv_kode,
    b.{f_usage_text_t_building} AS bbr_anv_tekst,
    st_force2d(b.{f_geom_t_building}) AS {f_geom_q_comp_build}
  FROM {t_building} b
  INNER JOIN {t_flood} o on st_intersects(b.{f_geom_t_building},o.{f_geom_t_flood})
  WHERE o.{f_depth_t_flood} >= {Minimum vanddybde (meter)}
)
SELECT 
  fb.{f_pkey_q_comp_build},
  fb.kom_kode,
  fb.bbr_anv_kode,
  fb.bbr_anv_tekst,
  fb.{f_geom_q_comp_build},
  SUM(c.{f_empcount_t_company})::bigint AS empl_cnt, 
  COUNT(*) AS comp_cnt,
  COUNT(*) FILTER (WHERE c.{f_empcount_t_company} = 0) AS empcnt_zero,
  COUNT(*) FILTER (WHERE c.{f_empcount_t_company} > 0) AS empcnt_nonzero
FROM fb INNER JOIN {t_company} c on st_within(c.{f_geom_t_company},fb.{f_geom_q_comp_build})
GROUP BY fb.{f_pkey_q_comp_build}, fb.kom_kode, fb.bbr_anv_kode, fb.bbr_anv_tekst, fb.{f_geom_q_comp_build}|Query for compagny-buiding-emplyee|
|Queries|q_comp_nobuild|SELECT 
  c.{f_pkey_t_company} AS {f_pkey_q_comp_nobuild}, 
  c."Virk_cvrnr",
  c.pnr,
  c.modifikati,
  c.ajourfoeri,
  c."Virk_gyldi",
  c.{f_geom_t_company} AS {f_geom_q_comp_nobuild}, 
  o.{f_depth_t_flood} 
  FROM {t_company} c 
  LEFT JOIN {t_flood} o ON st_within(c.{f_geom_t_company},o.{f_geom_t_flood})
  LEFT JOIN {t_building} b ON st_within(c.{f_geom_t_company},b.{f_geom_t_building})
  WHERE o.{f_depth_t_flood} >= {Minimum vanddybde (meter)} AND b.{f_pkey_t_building} IS NULL
| |
|Queries|q_human_health|WITH ob AS (
  SELECT
    b.{f_pkey_t_building} as {f_pkey_q_human_health},
    b.{f_muncode_t_building} AS kom_kode,
    b.{f_usage_code_t_building} AS bbr_anv_kode,
    b.{f_usage_text_t_building} AS bbr_anv_tekst,
    st_area(b.{f_geom_t_building})::NUMERIC(12,2) AS areal_byg_m2,
    (SUM(st_area(st_intersection(b.{f_geom_t_building},o.{f_geom_t_flood}))))::NUMERIC(12,2) AS areal_oversvoem_m2,
    (MIN(o.{f_depth_t_flood}) * 100.00)::NUMERIC(12,2) AS min_vanddybde_cm,
    (MAX(o.{f_depth_t_flood}) * 100.00)::NUMERIC(12,2) AS max_vanddybde_cm,
    (AVG(o.{f_depth_t_flood}) * 100.00)::NUMERIC(12,2) AS avg_vanddybde_cm,
    --(SUM(o.{f_depth_t_flood}*st_area(st_intersection(b.{f_geom_t_building},o.{f_geom_t_flood}))) * 100.0 / SUM(st_area(st_intersection(b.{f_geom_t_building},o.{f_geom_t_flood}))))::NUMERIC(12,2) AS avg_vanddybde_cm,
    st_force2d(b.{f_geom_t_building}) AS {f_geom_q_human_health}
    FROM {t_building} b
    INNER JOIN {t_flood} o on st_intersects(b.{f_geom_t_building},o.{f_geom_t_flood})
    WHERE o.{f_depth_t_flood} >= {Minimum vanddybde (meter)}
    GROUP BY b.{f_pkey_t_building}, b.{f_muncode_t_building}, b.{f_usage_code_t_building}, b.{f_usage_text_t_building}, b.{f_geom_t_building}
),
om AS ( 
  SELECT 
    ob.{f_pkey_q_human_health},
    ob.kom_kode,
    ob.bbr_anv_kode,
    ob.bbr_anv_tekst,
	ob.areal_byg_m2,
	ob.areal_oversvoem_m2, 
	ob.min_vanddybde_cm, 
	ob.max_vanddybde_cm, 
	ob.avg_vanddybde_cm,
    ob.{f_geom_q_human_health},
    COUNT(*) as mennesker_total,
    COUNT(*) FILTER (WHERE m.{f_age_t_human_health} BETWEEN 0 AND 6) AS mennesker_0_6,
    COUNT(*) FILTER (WHERE m.{f_age_t_human_health} BETWEEN 7 AND 17) AS mennesker_7_17,
    COUNT(*) FILTER (WHERE m.{f_age_t_human_health} BETWEEN 18 AND 70) AS mennesker_18_70,
    COUNT(*) FILTER (WHERE m.{f_age_t_human_health} > 70) AS mennesker_71plus,
    COUNT(*) FILTER (WHERE m.{f_age_t_human_health} BETWEEN 18 AND 70) * (55 * 301 * 0.5)::integer AS rejsetid_kr,
    COUNT(*) FILTER (WHERE m.{f_age_t_human_health} BETWEEN 18 AND 70) * (17 * 7.4 * 301 * 0.5)::integer AS sygedage_kr, 
    COUNT(*) FILTER (WHERE m.{f_age_t_human_health} BETWEEN 18 AND 70) * (7 * 7.4 * 301 * 0.5)::integer AS feriedage_kr 
  FROM ob 
  INNER JOIN {t_human_health} m ON st_intersects (ob.{f_geom_q_human_health}, m.{f_geom_t_human_health})
  GROUP BY ob.{f_pkey_q_human_health}, ob.kom_kode, ob.bbr_anv_kode, ob.bbr_anv_tekst, ob.areal_byg_m2, ob.areal_oversvoem_m2, ob.min_vanddybde_cm, ob.max_vanddybde_cm, ob.avg_vanddybde_cm, ob.{f_geom_q_human_health}
)

SELECT 
    om.*,
    om.rejsetid_kr + om.sygedage_kr + om.feriedage_kr AS omkostning_kr,
	(
	CASE
	    WHEN '{Medtag i risikoberegninger}' = 'Intet (0 kr.)' THEN 0.0
	    WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb' THEN om.rejsetid_kr + om.sygedage_kr + om.feriedage_kr
	    WHEN '{Medtag i risikoberegninger}' = 'Værditab' THEN 0.0
	    WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb og værditab' THEN 0.0 + om.rejsetid_kr + om.sygedage_kr + om.feriedage_kr 
	END * (0.089925/{Returperiode for hændelse i fremtiden (år)} + 0.21905/{Returperiode for hændelse i dag (år)}))::NUMERIC(12,2) AS risiko_kr
FROM om
| |
|Queries|q_recreative|WITH fr AS
(
  SELECT 
    r.{f_pkey_t_recreative} as {f_pkey_q_recreative},
    st_force2d(MIN(r.{f_geom_t_recreative})) AS {f_geom_q_recreative},
    MIN(r.gridcode) AS gridcode, 
    MIN(r.valuationk) AS valuation_kr,
    st_area(MIN(r.{f_geom_t_recreative}))::NUMERIC(12,2)  AS areal_org_m2,
    (SUM(st_area(st_intersection(r.{f_geom_t_recreative},o.{f_geom_t_flood}))))::NUMERIC(12,2) AS areal_oversvoem_m2,
    (MIN(o.{f_depth_t_flood}) * 100.00)::NUMERIC(12,2) AS min_vanddybde_cm,
    (MAX(o.{f_depth_t_flood}) * 100.00)::NUMERIC(12,2) AS max_vanddybde_cm,
    (AVG(o.{f_depth_t_flood}) * 100.00)::NUMERIC(12,2) AS avg_vanddybde_cm
    --(SUM(o.{f_depth_t_flood}*st_area(st_intersection(r.{f_geom_t_recreative},o.{f_geom_t_flood}))) * 100.0 / SUM(st_area(st_intersection(r.{f_geom_t_recreative},o.{f_geom_t_flood}))))::NUMERIC(12,2) AS avg_vanddybde_cm
  FROM {t_recreative} r
  INNER JOIN {t_flood} o on st_intersects(r.{f_geom_t_recreative},o.{f_geom_t_flood})
  WHERE o.{f_depth_t_flood} >= {Minimum vanddybde (meter)}
  GROUP BY r.{f_pkey_t_recreative}
),
qr AS (
  SELECT fr.*,   
    {Antal dage med oversvømmelse} AS periode_dage, 
    (100.0 * fr.areal_oversvoem_m2 / fr.areal_org_m2)::NUMERIC(12,2) AS oversvoem_pct,
    (({Antal dage med oversvømmelse}/365.0) * (fr.areal_oversvoem_m2 / fr.areal_org_m2) * fr.valuation_kr)::NUMERIC(12,2)  AS total_omkost_kr
  FROM fr
)

SELECT 
  qr.*,
  (
    CASE
      WHEN '{Medtag i risikoberegninger}' = 'Intet (0 kr.)' THEN 0.0
      WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb' THEN qr.total_omkost_kr
      WHEN '{Medtag i risikoberegninger}' = 'Værditab' THEN 0.0
      WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb og værditab' THEN 0.0 + qr.total_omkost_kr 
    END * (0.089925/{Returperiode for hændelse i fremtiden (år)} + 0.21905/{Returperiode for hændelse i dag (år)}))::NUMERIC(12,2) AS risiko_kr
FROM qr
| |
|Queries|q_road_traffic|WITH
    tr AS (
        SELECT
            v.{f_pkey_t_road_traffic} as {f_pkey_q_road_traffic},
            st_force2d(v.{f_geom_t_road_traffic}) AS {f_geom_q_road_traffic},
            st_length(v.{f_geom_t_road_traffic})::NUMERIC(12,2) AS laengde_org_m,
            v.{f_number_cars_t_road_traffic} AS gennemsnit_biler_pr_dag,
            COUNT(*) AS ant_oversvoem,
            (SUM(st_length(st_intersection(v.{f_geom_t_road_traffic},o.{f_geom_t_flood}))))::NUMERIC(12,2) AS laengde_oversvoem_m,
            (MIN(o.{f_depth_t_flood}) * 100.00)::NUMERIC(12,2) AS min_vanddybde_cm,
            (MAX(o.{f_depth_t_flood}) * 100.00)::NUMERIC(12,2) AS max_vanddybde_cm,
            (SUM(o.{f_depth_t_flood}*st_length(st_intersection(v.{f_geom_t_road_traffic},o.{f_geom_t_flood}))) * 100.0 / SUM(st_length(st_intersection(v.{f_geom_t_road_traffic},o.{f_geom_t_flood}))))::NUMERIC(12,2) AS avg_vanddybde_cm
        FROM {t_road_traffic} v
        INNER JOIN {t_flood} o on st_intersects(v.{f_geom_t_road_traffic},o.{f_geom_t_flood})
        WHERE o.{f_depth_t_flood} >= 0.075
    GROUP BY v.{f_pkey_t_road_traffic}, v.{f_geom_t_road_traffic}, v.{f_number_cars_t_road_traffic}
    ),
    tr2 AS (
        SELECT
            tr.*,
            {Oversvømmelsesperiode (timer)} AS blokering_dage,
            0.3 AS vanddybde_bloker_m,
            0.075 AS vanddybde_min_m,
            {Renovationspris pr meter vej (DKK)} AS pris_renovation_kr_m,
            (CASE
                WHEN tr.avg_vanddybde_cm >= 30.0 THEN 0.0
                ELSE 0.0009 * (tr.avg_vanddybde_cm*10.0)^2.0 - 0.5529 * tr.avg_vanddybde_cm*10.0 + 86.9448
            END)::NUMERIC(12,2) AS hastighed_red_km_time,
            tr.laengde_oversvoem_m * {Renovationspris pr meter vej (DKK)} AS renovationspris_total_kr
        FROM tr
    ),
    tr3 AS (
        SELECT
            tr2.*,
            (CASE
                WHEN tr2.hastighed_red_km_time > 50.0 THEN 0.0
                ELSE (68.8 - 1.376 * tr2.hastighed_red_km_time) * (tr2.blokering_dage / 24.0) * tr2.laengde_org_m * (tr2.gennemsnit_biler_pr_dag/6200.00)*2.0
                
            END)::NUMERIC(12,2) AS transportpris_total_kr,
            (CASE
                WHEN tr2.hastighed_red_km_time > 50.0 THEN 0.0
                ELSE (68.8 - 1.376 * tr2.hastighed_red_km_time) * (tr2.blokering_dage / 24.0) * tr2.laengde_org_m * (tr2.gennemsnit_biler_pr_dag/6200.00)*2.0
            END + tr2.renovationspris_total_kr)::NUMERIC(12,2) AS pris_total_kr
        FROM tr2
)
SELECT 
    tr3.*,
	(
	CASE
	    WHEN '{Medtag i risikoberegninger}' = 'Intet (0 kr.)' THEN 0.0
	    WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb' THEN tr3.pris_total_kr
	    WHEN '{Medtag i risikoberegninger}' = 'Værditab' THEN 0.0
	    WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb og værditab' THEN 0.0 + tr3.pris_total_kr 
	END * (0.089925/{Returperiode for hændelse i fremtiden (år)} + 0.21905/{Returperiode for hændelse i dag (år)}))::NUMERIC(12,2) AS risiko_kr
FROM tr3
| |
|Queries|q_surrounding_loss|WITH 
  vb AS (
    SELECT
      st_union(b.{f_geom_t_building}) AS geom 
    FROM {t_building} b 
    JOIN {t_flood} o 
    ON st_intersects(o.{f_geom_t_flood}, b.{f_geom_t_building}) 
    WHERE o.{f_depth_t_flood} >= {Minimum vanddybde (meter)}),
    
  ob AS (
	SELECT
      distinct b.{f_pkey_t_building} as fid
    FROM {t_building} b 
    JOIN {t_flood} o 
    ON st_intersects(o.{f_geom_t_flood}, b.{f_geom_t_building}) 
    WHERE o.{f_depth_t_flood} >= {Minimum vanddybde (meter)}),

  fb AS (
    SELECT
      b.{f_pkey_t_building} AS {f_pkey_q_building},
      b.{f_muncode_t_building} AS kom_kode,
      b.{f_usage_code_t_building} AS bbr_anv_kode,
      b.{f_usage_text_t_building} AS bbr_anv_tekst,
      st_area(b.{f_geom_t_building})::NUMERIC(12,2) AS areal_byg_m2,
      k.{f_sqmprice_t_sqmprice}::NUMERIC(12,2) AS kvm_pris_kr,
      ({Værditab, skaderamte bygninger (%)}*{Faktor for værditab})::NUMERIC(12,2) AS tab_procent,
      (k.{f_sqmprice_t_sqmprice} * st_area(b.{f_geom_t_building}) * {Værditab, skaderamte bygninger (%)} * {Faktor for værditab} / 100.0)::NUMERIC(12,2) AS vaerditab_kr,
      st_force2d(b.{f_geom_t_building}) AS {f_geom_q_building}
        
    FROM {t_building} b
    LEFT JOIN ob ON ob.fid=b.{f_pkey_t_building} 
    LEFT JOIN {t_sqmprice} k ON b.{f_muncode_t_building}=k.{f_muncode_t_sqmprice}
    INNER JOIN vb ON st_dwithin(vb.geom, b.{f_geom_t_building}, {Bredde af nabozone (meter)})
    WHERE ob.fid IS NULL)

SELECT 
    fb.*,
	(
	CASE
	    WHEN '{Medtag i risikoberegninger}' = 'Intet (0 kr.)' THEN 0.0
	    WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb' THEN 0.0
	    WHEN '{Medtag i risikoberegninger}' = 'Værditab' THEN fb.vaerditab_kr
	    WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb og værditab' THEN 0.0 + fb.vaerditab_kr 
	END * (0.089925/{Returperiode for hændelse i fremtiden (år)} + 0.21905/{Returperiode for hændelse i dag (år)}))::NUMERIC(12,2) AS risiko_kr
FROM fb
|SQL template for "Nabobygninge værditabar" model |
|Queries|q_tourism_alphanumeric|WITH pn AS (
  SELECT
    row_number() over () as {f_pkey_q_tourism_alphanumeric},
    b.{f_muncode_t_building} AS kom_kode,
    b.{f_usage_code_t_building} AS bbr_kode,
    t.bbr_anv_tekst AS bbr_tekst,
    t.kapacitet AS kapacitet_bygning,
    t.omkostning AS omkostning_overnatning,
    {Antal tabte døgn} AS tabte_dage,
    SUM(t.kapacitet * {Antal tabte døgn}) AS sum_overnatninger,
    SUM(t.omkostning * t.kapacitet* {Antal tabte døgn} / 1000000.0)::NUMERIC(12,3)  AS sum_tot_kr,
    SUM(t.omkostning * t.kapacitet* {Antal tabte døgn} / 1000000.0)::NUMERIC(12,3)  AS sum_tot_mill_kr
    FROM {t_building} b
    INNER JOIN {t_flood} o on st_intersects(b.{f_geom_t_building},o.{f_geom_t_flood})
    JOIN {t_tourism} t  ON t.{f_pkey_t_tourism} = b.{f_usage_code_t_building}  
    WHERE o.{f_depth_t_flood} >= {Minimum vanddybde (meter)}
    GROUP BY b.komkode, b.{f_usage_code_t_building}, t.bbr_anv_tekst, t.kapacitet, t.omkostning
)
SELECT 
    pn.*,
	(
	CASE
	    WHEN '{Medtag i risikoberegninger}' = 'Intet (0 kr.)' THEN 0.0
	    WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb' THEN pn.sum_tot_mill_kr
	    WHEN '{Medtag i risikoberegninger}' = 'Værditab' THEN 0.0
	    WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb og værditab' THEN 0.0 + pn.sum_tot_mill_kr 
	END * (0.089925/{Returperiode for hændelse i fremtiden (år)} + 0.21905/{Returperiode for hændelse i dag (år)}))::NUMERIC(12,2) AS risiko_mill_kr
FROM pn
| |
|Queries|q_tourism_spatial|WITH pn AS (
  SELECT DISTINCT
    b.{f_pkey_t_building} as {f_pkey_q_tourism_spatial},
    b.{f_muncode_t_building} AS kom_kode,
    b.{f_usage_code_t_building} AS bbr_kode,
    t.bbr_anv_tekst AS bbr_tekst,
    t.kapacitet AS kapacitet,
    t.omkostning AS omkostninger,
    {Antal tabte døgn} AS tabte_dage,
    {Antal tabte døgn} * t.omkostning AS tabte_overnat,
    {Antal tabte døgn} * t.omkostning * t.kapacitet AS tot_kr,
    st_force2d(b.{f_geom_t_building}) AS {f_geom_q_tourism_spatial}
    FROM {t_building} b
    INNER JOIN {t_flood} o on st_intersects(b.{f_geom_t_building},o.{f_geom_t_flood})
    INNER JOIN {t_tourism} t  ON t.{f_pkey_t_tourism} = b.{f_usage_code_t_building}  
    WHERE o.{f_depth_t_flood} >= {Minimum vanddybde (meter)}
)
SELECT 
    pn.*,
	(
	CASE
	    WHEN '{Medtag i risikoberegninger}' = 'Intet (0 kr.)' THEN 0.0
	    WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb' THEN pn.tot_kr
	    WHEN '{Medtag i risikoberegninger}' = 'Værditab' THEN 0.0
	    WHEN '{Medtag i risikoberegninger}' = 'Skadebeløb og værditab' THEN 0.0 + pn.tot_kr 
	END * (0.089925/{Returperiode for hændelse i fremtiden (år)} + 0.21905/{Returperiode for hændelse i dag (år)}))::NUMERIC(12,2) AS risiko_kr
FROM pn
| |
|Rekreative områder|Skadeberegning, Rekreative områder| |Sæt hak såfremt der skal beregnes økonomiske tab i forbindelse med reduceret adgang til rekreative områder som bliver berørt at den pågældende oversvømmelse.|
|Skadeberegning, Rekreative områder|Antal dage med oversvømmelse|24|Angiv antallet af dage hvor de rekreative områder ikke kan anvendes som en konsekvens af den pågældende oversvømmelse.|
|Skadeberegning, vej og trafik|Oversvømmelsesperiode (timer)|24|Her angives det antal dage, hvor vejene ikke kan benyttes pga. oversvømmelsen.|
|Skadeberegning, vej og trafik|Renovationspris pr meter vej (DKK)|20|Her angives den økonomiske omkostning til oprydning per meter vej som bliver oversvømmet. Omkostningen angives i DKK per meter.|
|Skadeberegninger, Bygninger|Skadeberegning for kælder|Medtages ikke|Bestemmer skadeberegning for kælder medtages i udregningen|
|Skadeberegninger, Bygninger|Skadetype|Stormflod|Valg af økonomisk skademodel|
|Skadeberegninger, kælder|Minimum vanddybde (meter), kælder|0.1|Mindste værdi for vandybde for kælder, som medtages i beregningerne |
|SQL command templates|Clear cell layer template|UPDATE "{schema}"."{table}" SET val_intersect = 0.0, num_intersect = 0
| |
|SQL command templates|Create cell layer template|CREATE TABLE IF NOT EXISTS {celltable} AS
  WITH g AS (
    SELECT (
      st_squaregrid({cellsize}, st_geomfromewkt('SRID={epsg}; POLYGON(({xmin} {ymin},{xmax} {ymin},{xmax} {ymax},{xmin} {ymax},{xmin} {ymin}))'))
	).*
  )
  SELECT
    row_number() OVER () AS fid,
    *,
    0.0::NUMERIC(12,2) AS val_intersect, 
    0 AS num_intersect 
  FROM g;
ALTER TABLE {celltable} ADD PRIMARY KEY(fid);
CREATE INDEX ON {celltable} USING GIST(geom);| |
|SQL command templates|Delete parameter table|DELETE FROM {parametertable}| |
|SQL command templates|Fetch parameter table|WITH RECURSIVE tree_search AS (SELECT *, 0 AS "level" FROM {parametertable} WHERE "parent" = '' AND name <> '' UNION ALL SELECT t.*, ts."level"+1 AS "level" FROM {parametertable} t, tree_search ts WHERE t."parent" = ts."name") SELECT * FROM tree_search ORDER BY "level", "sort";| |
|SQL command templates|Insert parameter table|INSERT INTO {parametertable} ({parametercolumns}) VALUES ({parametervalues}| |
|SQL command templates|update cell layer|WITH cte AS (
  SELECT
    a.fid AS fid,
    a.i AS i,
    a.j AS j,
    sum(
      CASE 				  
        WHEN ST_GeometryType(b.{geom_value}) ILIKE '%POINT%' THEN b.{value_value} 				  
        WHEN ST_GeometryType(b.{geom_value}) ILIKE '%LINE%' THEN b.{value_value} * st_length(st_intersection(a.{geom_cell},b.{geom_value}))/st_length(b.{geom_value})				  
        WHEN ST_GeometryType(b.{geom_value}) ILIKE '%POLYGON%' THEN b.{value_value} * st_area(st_intersection(a.{geom_cell},b.{geom_value}))/st_area(b.{geom_value})	  
        ELSE -10000000.00      
      END
    ) as sum_value,
    count(*) as count_number
  FROM {cell_table} a JOIN {value_table} b ON st_intersects (a.{geom_cell},b.{geom_value})
  GROUP BY a.fid, a.i, a.j
)
UPDATE {cell_table} SET 
  val_intersect = val_intersect + sum_value, 
  num_intersect = num_intersect + count_number 
FROM cte
WHERE {cell_table}.fid = cte.fid; | |
|SQL command templates|Create comment command|COMMENT ON {} {} IS {}| |
|SQL command templates|Create schema command|CREATE SCHEMA {}| |
|SQL command templates|Create_result_index|CREATE INDEX ON "{Result_schema}"."{tablename_ts}" USING GIST ({geom_column})| |
|SQL command templates|Create_result_pkey|ALTER TABLE "{Result_schema}"."{tablename_ts}" ADD PRIMARY KEY ({pkey_column});  | |
|SQL command templates|Create_result_table|CREATE TABLE IF NOT EXISTS "{Result_schema}"."{tablename_ts}" AS {sqlquery}|SQL template for creating result tables|
|SQL command templates|Drop schema command|DROP SCHEMA {} /CASCADE| |
|t_bioscore|f_bioscore_t_bioscore|"Bioscore"| |
|t_bioscore|f_geom_t_bioscore|geom| |
|t_bioscore|f_pkey_t_bioscore|id| |
|t_build_usage|f_category_t_build_usage|skade_kategori|Field name for keyfield in Building table |
|t_build_usage|f_pkey_t_build_usage|bbr_anv_kode|Field name for keyfield in Building table |
|t_build_usage|f_usage_text_t_build_usage|bbr_anv_tekst| |
|t_building|f_cellar_area_t_building|kaelder123| |
|t_building|f_geom_t_building|geom|Field name for geometry field in building table|
|t_building|f_muncode_t_building|komkode|Fieldname for municipality code for building table|
|t_building|f_pkey_t_building|"OBJECTID"|Field name for keyfield in Building table |
|t_building|f_usage_code_t_building|"BYG_ANVEND"|Fieldname for usage code for building table|
|t_building|f_usage_text_t_building|"BYG_ANVE_1"|Fieldname for usage code for building table|
|t_company|f_empcount_t_company|aarsbes_an| |
|t_company|f_geom_t_company|geom| |
|t_company|f_pkey_t_company|"OBJECTID"| |
|t_damage|f_category_t_damage|skade_kategori|Field name for keyfield in damage function table |
|t_damage|f_pkey_t_damage|skade_type, skade_kategori|Field name for keyfield in damage function table |
|t_damage|f_type_t_damage|skade_type|Field name for keyfield in damage function table |
|t_flood|f_depth_t_flood|"Vanddybde"|Field name for detph field in flood table |
|t_flood|f_geom_t_flood|geom|Field name for geometry field in flood table|
|t_human_health|f_age_t_human_health|alder_rand|Field name for age field in Human health table |
|t_human_health|f_geom_t_human_health|geom|Field name for geometry field in Human health table |
|t_human_health|f_pkey_t_human_health|objectid|Field name for keyfield in Human health table |
|t_infrastructure|f_geom_t_infrastructure|geom| |
|t_infrastructure|f_pkey_t_infrastructure|objectid| |
|t_publicservice|f_geom_t_publicservice|geom| |
|t_publicservice|f_pkey_t_publicservice|objectid| |
|t_recreative|f_geom_t_recreative|geom| |
|t_recreative|f_pkey_t_recreative|objectid| |
|t_road_traffic|f_geom_t_road_traffic|geom| |
|t_road_traffic|f_number_cars_t_road_traffic|trafik_tal| |
|t_road_traffic|f_pkey_t_road_traffic|objectid| |
|t_sqmprice|f_muncode_t_sqmprice|kom_kode|Fieldname for municipalitycode|
|t_sqmprice|f_sqmprice_t_sqmprice|kvm_pris|Fieldname for sqm price|
|t_tourism|f_pkey_t_tourism|bbr_anv_kode| |
|Turisme|Antal tabte døgn|20|Her angives antallet af dage hvor bygningerne som bliver berørt af den pågældende oversvømmelse ikke kan anvendes til turistformål pga. skader eller oprydning efter oversvømmelsen.  |
|Turisme|Turisme, Kort| |Sæt hak såfremt der skal beregnes økonomiske tab for overnatningssteder som anvendes til turistformål. De berørte bygninger vises geografisk på et kort.  |
|Turisme|Turisme, Opsummering| |Sæt hak såfremt der skal beregnes økonomiske tab for overnatningssteder som anvendes til turistformål. De økonomiske tab opsummeres i en tabel.|
|Værditab nabobygninger|Bredde af nabozone (meter)|300.0|Maks. afstand for nabobygninger fra skaderamte bygningerder som medtages i beregningen|
|Værditab nabobygninger|Faktor for værditab|0.50|Faktor værdi til beregning af værditab for nabobygninger ud fra værditab for skaderamte bygninger|
|Vej og trafik|Skadeberegning, vej og trafik| |Sæt hak såfremt der skal beregnes økonomiske tab for vej og trafik i forbindelse med den pågældende oversvømmelseshændelse.|
