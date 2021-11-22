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
 værdi i felt "parent|

    Eksempel: I faneblad "Generelt" findes en parameterpost "Name templates". Parametrene "Cell layername" "Group name template", 
	"Main groupname" o.a. har alle fået tildelt "Name templates" som "parent". Det giver følgende resultat: 
	
	(Billede)
	
	
- "I": Parameter skal indeholde et heltal. Når man dobbeltklikker på paramternavnet eller værdi feltet i fanebladet, vil der
 vises en bruger dialog, hvor værdien kan ændres. Dialogen bliver udformet, således der kun kan indtastes et heltal. Minimu og maksimum 
 værdier bestemmes af værdierne indtastet i felterne "minval" og "maxval". Stepstørrelsen i det tilhørende rullefelt bestemmes af værdien 
 i felt "lookupvalues|
 
     Eksempel: Parameter "Cell size" har type "I", værdi 100, minval: 10, maxval: 1000, lookvalue: 50. I dialogen vise værdien 100, som kan ændres fra 10 til 1000. Og bruges scroll-pilene ændres værdier i skridt af 50. 

- "R": Parameter skal indeholde et reelt tal. Når man dobbeltklikker på paramternavnet eller værdi feltet i fanebladet, vil der
 vises en bruger dialog, hvor værdien kan ændres. Dialogen bliver udformet, således der kun kan indtastes et reelle tal. Minimum og 
 maksimum værdier bestemmes af værdierne indtastet i felterne "minval" og "maxval". Stepstørrelsen i det tilhørende rullefelt bestemmes 
 af værdien i felt "lookupvalues|

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








# Administrator vejledning for QGIS plugin "Skadesøkonomi|

Selve installationen af hhv. plugin og tilhørende database, inklusive opsætning af Postgres username/password, 
sikkerhedskopiering er gennegået i installationsvejledningen, så disse oplysning kan findes i denne vejledning.

### Opstart af plugin

I QGIS

WITH RECURSIVE tree_search AS (SELECT *, 0 AS "level" FROM fdc_admin.parametre WHERE "parent" = '' AND name <> '' UNION ALL SELECT t.*, ts."level"+1 AS "level" FROM fdc_admin.parametre t, tree_search ts WHERE t."parent" = ts."name") SELECT '|' || parent || '|' || name || '|' || type || '|' || left(left(value,30) || '...'), || '|' explanation FROM tree_search ORDER BY "level", "sort|


|Gruppe|Navn|Type|Værdi|Funktion|
|---|---|---|---|---|
||General|G||Hovedgrupper til administration af grundlæggende parametre for systemet|
||Queries|G||Hovedgrupper til administration af Forespørgsler|
||Data|G||Hovedgruppe for administration af Tabeller|
||Reports|G||Hovedgruppe til administration og kørsel af Rapporter|
||Models|G||Hovedgruppe for administration og kørsel af  Modeller|
|General|Delete parameter table|P|DELETE FROM {parametertable}...||
|General|Create cell layer template|P|CREATE TABLE IF NOT EXISTS {ce...||
|General|Fetch parameter table|P|WITH RECURSIVE tree_search AS ...||
|General|Cell size|I|100||
|General|Insert parameter table|P|INSERT INTO {parametertable} (...||
|General|Clear cell layer template|P|UPDATE ""{schema}"".""{table}"" SE...||
|General|Cell layername|T|celler||
|Models|Generelle modelværdier|G||Afsnit, hvor parametre, der bruges i mange forskellige modeller, kan værdisættes|
|General|Name of model value section|T|Generelle modelværdier||
|General|update cell layer|P|WITH cte AS (   SELECT     a.f...||
|Models|Kritisk infrastruktur|G||Skademodeller for Kritisk infrastruktur|
|Models|Offentlig service|G||Skademodeller for Offentlig service|
|Models|Vej og trafik|G||Skademodeller for Vej og trafik|
|Models|Bygninger|G||Skademodeller for Bygninger|
|Models|Biodiversitet|G||Skademodeller for Biodiversitet|
|Models|Rekreative områder|T||Skademodeller for Rekreative områder|
|Models|Turisme|G||Skademodeller for Turisme|
|Models|Industri|G| |Skademodeller for Industri|
|Queries|q_infrastructure|P|SELECT DISTINCT   k.*   FROM {...||
|Queries|q_publicservice|P|SELECT DISTINCT   k.*   FROM {...||
|Models|Mennesker og helbred|G||Skademodeller for Mennesker og helbred|
|Data|t_human_health|T|fdc_data.mennesker|Parametergruppe til tabel ""mennesker""|
|Data|t_recreative|T|fdc_data.rekreative_omr|Parametergruppe til tabel ""rekreative områder""|
|Data|t_flood|T|fdc_data.oversvoem|Parametergruppe til tabel ""oversvømmelser""|
|Data|t_infrastructure|T|fdc_data.kritisk_infrastruktur...|Parametergruppe til tabel ""kritisk infrastruktur""|
|Data|t_building|T|fdc_data.bygninger|Parametergruppe til tabel ""Bygninger""|
|Data|t_build_usage|T|fdc_admin.bbr_anvendelse|Parametergruppe til opslagstabel ""bbr-anvendelse""|
|Data|t_sqmprice|T|fdc_admin.kvm_pris|Parametergruppe til opslagstabel ""kommunal kvm. pris""|
|Data|t_damage|T|fdc_admin.skadefunktioner|Parametergruppe til opslagstabel ""skadesfunktioner""|
|General|Drop schema command|P|DROP SCHEMA {} /CASCADE||
|General|Create schema command|P|CREATE SCHEMA {}||
|General|Create_result_table|P|CREATE TABLE IF NOT EXISTS ""{R...|SQL template for creating result tables|
|General|Main groupname|T|Modeller||
|General|Group name template|T|Kørsel: {time_stamp}||
|General|Create_result_pkey|P|ALTER TABLE ""{Result_schema}""....||
|General|Create_result_index|P|CREATE INDEX ON ""{Result_schem...||
|General|Model_layergroup|T|Resultater fra modelkørsler||
|General|Create comment command|P|COMMENT ON {} {} IS {}||
|Queries|q_building|P|WITH ob AS (     SELECT       ...|SQL template for ""Bygninger"" model |
|Queries|q_bioscore_spatial|P|SELECT   b.{f_pkey_t_bioscore}...||
|Queries|q_comp_nobuild|P|SELECT    c.{f_pkey_t_company}...||
|Queries|q_bioscore_alphanumeric|P|  SELECT   row_number() over (...||
|Queries|q_surrounding_loss|P|WITH    vb AS (     SELECT    ...|SQL template for ""Nabobygninge værditabar"" model |
|Queries|q_comp_build|P|WITH fb AS (   SELECT DISTINCT...|Query for compagny-buiding-emplyee|
|Queries|q_recreative|P|WITH fr AS (   SELECT      r.{...||
|Queries|q_road_traffic|P|WITH     tr AS (         SELEC...||
|Queries|q_human_health|P|WITH ob AS (   SELECT     b.{f...||
|Queries|q_tourism_spatial|P|WITH pn AS (   SELECT DISTINCT...||
|Queries|q_tourism_alphanumeric|P|WITH pn AS (   SELECT     row_...||
|General|Result_schema|T|fdc_results|Name of schema to place result tables in|
|Data|t_company|T|fdc_data.industri|Parametergruppe til tabel ""industri""|
|Data|t_publicservice|T|fdc_data.offentlig_service||
|Data|t_road_traffic|T|fdc_data.vejnet|Parametergruppe til tabel ""vejnet""|
|Data|t_tourism|T|fdc_admin.turisme|Parametergruppe til opslagstabel ""turisme""|
|Data|t_bioscore|T|fdc_data.biodiversitet|Parametergruppe til tabel ""biodiversitet""|
|q_human_health|f_risk_q_human_health|T|risiko_kr||
|q_building|f_cellar_damage_q_building|T|skadebeloeb_kaelder_kr||
|q_human_health|f_damage_q_human_health|T|omkostning_kr||
|Bygninger|Værditab, skaderamte bygninger (%)|R|10.0|Her angives størrelsen på reduktionen i salgspris for de bygninger som bliver berørt af den pågældende oversvømmelse. Tabet beregnes som en procentsats, som angives af brugeren, af den gennemsnitlige kommunale m2 pris for solgte boliger i løbet af de seneste år. Det anbefales at anvende værdien 10% såfremt man ikke har bedre data.|
|Turisme|Antal tabte døgn|I|20|Her angives antallet af dage hvor bygningerne som bliver berørt af den pågældende oversvømmelse ikke kan anvendes til turistformål pga. skader eller oprydning efter oversvømmelsen.  |
|q_infrastructure|f_pkey_q_infrastructure|T|objectid||
|q_infrastructure|f_geom_q_infrastructure|T|geom||
|q_publicservice|f_pkey_q_publicservice|T|objectid||
|q_publicservice|f_geom_q_publicservice|T|geom||
|q_building|f_damage_q_building|T|skadebeloeb_kr||
|q_building|f_loss_q_building|T|vaerditab_kr||
|q_building|f_risk_q_building|T|risiko_kr||
|Generelle modelværdier|Returperiode for hændelse i dag (år)|I|20|Indtast returperioden under nuværende klima for den oversvømmelseshændelse som der beregnes skader og risiko for. Returperioden angives i år.|
|t_infrastructure|f_geom_t_infrastructure|T|geom||
|q_tourism_spatial|f_damage_q_tourism_spatial|T|tot_kr||
|q_tourism_spatial|f_risk_q_tourism_spatial|T|risiko_kr||
|t_infrastructure|f_pkey_t_infrastructure|T|objectid||
|t_recreative|f_geom_t_recreative|T|geom||
|t_publicservice|f_geom_t_publicservice|T|geom||
|q_surrounding_loss|f_risk_q_surrounding_loss|T|risiko_kr||
|q_surrounding_loss|f_loss_q_surrounding_loss|T|vaerditab_kr||
|t_publicservice|f_pkey_t_publicservice|T|objectid||
|q_road_traffic|f_risk_q_road_traffic|T|risiko_kr||
|q_road_traffic|f_damage_q_road_traffic|T|pris_total_kr||
|q_recreative|f_damage_q_recreative|T|total_omkost_kr||
|q_recreative|f_risk_q_recreative|T|risiko_kr||
|Generelle modelværdier|Returperiode for hændelse i fremtiden (år)|I|5|Indtast returperioden om 100 år under fremtidigt klima for den oversvømmelseshændelse som der beregnes skader og risiko for. Dette kan f.eks. være under klimascenarie RCP4.5 eller RCP8.5 Returperioden angives i år.|
|Generelle modelværdier|Medtag i risikoberegninger|O|Skadebeløb|Her vælger man om det kun er skadeomkostningen eller skadeomkostning inkl. værditab for bygninger som inkluderes i risikoberegningen.|
|Biodiversitet|Biodiversitet, kort|T||Sæt hak såfremt modellen skal identificere særlige levesteder for rødlistede arter som bliver berørt i forbindelse med den pågældende oversvømmelseshændelse. Her vises levestederne geografisk på et kort.|
|q_bioscore_alphanumeric|f_pkey_q_bioscore_alphanumeric|T|fid||
|Industri|Industri, personale i bygninger|T| |Sæt hak såfremt modellen skal identificere de virksomheder som bliver berørt af den pågældende oversvømmelse, og angive antallet af medarbejdere per virksomhed.|
|q_building|f_pkey_q_building|T|fid|Name of primary keyfield for query|
|q_building|f_geom_q_building|T|geom_byg|Field name for geometry column|
|q_tourism_spatial|f_geom_q_tourism_spatial|T|geom|Field name for geometry field in tourism query|
|q_tourism_spatial|f_pkey_q_tourism_spatial|T|id||
|q_tourism_alphanumeric|f_pkey_q_tourism_alphanumeric|T|fid||
|q_surrounding_loss|f_pkey_q_surrounding_loss|T|fid||
|q_surrounding_loss|f_geom_q_surrounding_loss|T|geom_byg||
|q_bioscore_spatial|f_geom_q_bioscore_spatial|T|geom_clip||
|q_comp_build|f_pkey_q_comp_build|T|fid||
|q_comp_build|f_geom_q_comp_build|T|geom_byg|Field name for building geometry|
|q_comp_nobuild|f_geom_q_comp_nobuild|T|geom||
|q_comp_nobuild|f_pkey_q_comp_nobuild|T|fid||
|q_human_health|f_pkey_q_human_health|T|fid||
|q_human_health|f_geom_q_human_health|T|geom||
|q_recreative|f_pkey_q_recreative|T|id||
|q_recreative|f_geom_q_recreative|T|geom||
|q_road_traffic|f_pkey_q_road_traffic|T|id||
|q_road_traffic|f_geom_q_road_traffic|T|geom||
|t_road_traffic|f_number_cars_t_road_traffic|I|trafik_tal||
|t_road_traffic|f_geom_t_road_traffic|T|geom||
|t_road_traffic|f_pkey_t_road_traffic|T|objectid||
|t_company|f_pkey_t_company|T|""OBJECTID""||
|t_company|f_empcount_t_company|I|aarsbes_an||
|t_company|f_geom_t_company|T|geom||
|t_bioscore|f_pkey_t_bioscore|T|id||
|t_bioscore|f_geom_t_bioscore|T|geom||
|t_bioscore|f_bioscore_t_bioscore|T|""Bioscore""||
|t_human_health|f_pkey_t_human_health|T|objectid|Field name for keyfield in Human health table |
|t_human_health|f_age_t_human_health|T|alder_rand|Field name for age field in Human health table |
|t_human_health|f_geom_t_human_health|T|geom|Field name for geometry field in Human health table |
|t_recreative|f_pkey_t_recreative|T|objectid||
|t_flood|f_geom_t_flood|T|geom|Field name for geometry field in flood table|
|t_flood|f_depth_t_flood|T|""Vanddybde""|Field name for detph field in flood table |
|t_building|f_muncode_t_building|T|komkode|Fieldname for municipality code for building table|
|t_building|f_geom_t_building|T|geom|Field name for geometry field in building table|
|t_building|f_pkey_t_building|T|""OBJECTID""|Field name for keyfield in Building table |
|Generelle modelværdier|Minimum vanddybde (meter)|R|0.3|Her angives den minimale vanddybde på terræn som der skal til for at der opstår økonomiske tab i forbindelse med oversvømmelsen. Denne værdi angives i m, og anvendes kun for de sektorer hvor der ikke er angivet en alternativ minimum vanddybde.|
|Offentlig service|Oversvømmet offentlig service|T||Sæt hak såfremt modellen skal identificere offentlig service som bliver berørt i forbindelse med den pågældende oversvømmelseshændelse.|
|Vej og trafik|Skadeberegning, vej og trafik|T||Sæt hak såfremt der skal beregnes økonomiske tab for vej og trafik i forbindelse med den pågældende oversvømmelseshændelse.|
|Kritisk infrastruktur|Oversvømmet infrastruktur|T||Sæt hak såfremt modellen skal identificere kritisk infrastruktur som bliver berørt i forbindelse med den pågældende oversvømmelseshændelse.|
|Bygninger|Værditab nabobygninger|T|||
|t_tourism|f_pkey_t_tourism|T|bbr_anv_kode||
|t_sqmprice|f_muncode_t_sqmprice|T|kom_kode|Fieldname for municipalitycode|
|t_sqmprice|f_sqmprice_t_sqmprice|T|kvm_pris|Fieldname for sqm price|
|Bygninger|Skadeberegninger, Bygninger|T||Skadeberegning for bygninger, forskellige skademodeller, med eller uden kælderberegning|
|t_building|f_cellar_area_t_building|T|kaelder123||
|Biodiversitet|Biodiversitet, opsummering|T||Sæt hak såfremt modellen skal identificere særlige levesteder for rødlistede arter som bliver berørt i forbindelse med den pågældende oversvømmelseshændelse. Her opsummeres de berørte levesteder i en tabel.|
|Turisme|Turisme, Opsummering|T||Sæt hak såfremt der skal beregnes økonomiske tab for overnatningssteder som anvendes til turistformål. De økonomiske tab opsummeres i en tabel.|
|Turisme|Turisme, Kort|T||Sæt hak såfremt der skal beregnes økonomiske tab for overnatningssteder som anvendes til turistformål. De berørte bygninger vises geografisk på et kort.  |
|t_building|f_usage_code_t_building|T|""BYG_ANVEND""|Fieldname for usage code for building table|
|t_build_usage|f_pkey_t_build_usage|T|bbr_anv_kode|Field name for keyfield in Building table |
|t_build_usage|f_usage_text_t_build_usage|T|bbr_anv_tekst||
|t_damage|f_pkey_t_damage|T|skade_type, skade_kategori|Field name for keyfield in damage function table |
|t_damage|f_category_t_damage|T|skade_kategori|Field name for keyfield in damage function table |
|t_damage|f_type_t_damage|T|skade_type|Field name for keyfield in damage function table |
|t_build_usage|f_category_t_build_usage|T|skade_kategori|Field name for keyfield in Building table |
|t_building|f_usage_text_t_building|T|""BYG_ANVE_1""|Fieldname for usage code for building table|
|Industri|Industri, punkter fejlplaceret|T|||
|Mennesker og helbred|Humane omkostninger |T||Sæt hak såfremt der skal beregnes økonomiske tab ifm. oprydning, sygedage og feriedage som en konsekvens af ens ejendom har været oversvømmet.|
|Rekreative områder|Skadeberegning, Rekreative områder|T||Sæt hak såfremt der skal beregnes økonomiske tab i forbindelse med reduceret adgang til rekreative områder som bliver berørt at den pågældende oversvømmelse.|
|Skadeberegning, vej og trafik|Oversvømmelsesperiode (dage)|I|24|Her angives det antal dage, hvor vejene ikke kan benyttes pga. oversvømmelsen.|
|Skadeberegning, vej og trafik|Renovationspris pr meter vej (DKK)|I|20|Her angives den økonomiske omkostning til oprydning per meter vej som bliver oversvømmet. Omkostningen angives i DKK per meter.|
|Skadeberegninger, Bygninger|Skadeberegning for kælder|O|Medtages ikke|Bestemmer skadeberegning for kælder medtages i udregningen|
|Skadeberegninger, Bygninger|Skadetype|O|Stormflod|Valg af økonomisk skademodel|
|Værditab nabobygninger|Bredde af nabozone (meter)|R|300.0|Maks. afstand for nabobygninger fra skaderamte bygningerder som medtages i beregningen|
|Værditab nabobygninger|Faktor for værditab|R|0.50|Faktor værdi til beregning af værditab for nabobygninger ud fra værditab for skaderamte bygninger|
|Skadeberegning, Rekreative områder|Antal dage med oversvømmelse|I|24|Angiv antallet af dage hvor de rekreative områder ikke kan anvendes som en konsekvens af den pågældende oversvømmelse.|
