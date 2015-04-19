# Hent java løsningen

`git clone something something`

### Bygg og start løsningen lokalt for å se at det fungerer

`mvn clean install jetty:run`

# TeamCity Login

### Url: [http://nerdschool.cloudapp.net/](http://nerdschool.cloudapp.net/) 
### Bruker: public
### Passord: Programming is fun

## Plan

Målet er at når man sjekker inn kode på en git branch i GitHub, så skal et bygg utføres i TeamCity. Bygget vi skal jobbe med her består av følgende steg:

1. Kompiler koden og kjør tester
	- TeamCity poller git repositoriet hvert 30 sekund
	- Når en endringer er funnet, kopierer TeamCity alle filene i git repositoriet over til byggeserveren til en mappe på disk
	- Når koden er kopiert, utføres byggetrinnene vi angir, i dette tilfellet vil det være å kjøre en Mavn kommando `mvn clean install` som vil kompilere og kjøre tester.
3. Pakk løsningen (for hvert miljø appen skal ut til)
	- Når vi har bekreftet at løsningen er gyldig (kompilerer + kjører tester), pakker vi hele løsningen sammen i en `.war` fil, som skal deployest til web serveren.
4. Kjør løsningen (for å minske kompleksiteten kjører vi appen på byggeserveren her)
	- I en virkelig løsning vil vi kopiere `.war`-filen til en annen server for å bli kjørt der, men for enkelhets skyld kjører vi løsningen på byggeserveren her.
	- For å kjøre løsningen bruker vi webserveren Jetty som vi gir `.war` filen til.  

## Forberedelser

### Velg et gruppenavn

Siden vi kommer til å ha flere like konfigurasjoner side ved side må vi ha et unikt navn en del steder. Gruppenavnet bør være kort og ikke ha mellomrom. Vi kommer til å referere til gruppenavnet en del steder videre.

### TeamCity 101

![TeamCity oversikt](https://github.com/eaardal/eaardal.github.io/tree/master/nerdschool-ci/TeamCityOverview.jpeg)

### Opprett prosjekt
Hver gruppe må sette opp sitt eget sub-prosjekt (se bildet over)

**Merk: Vi refererer til dette prosjektet som gruppeprosjektet videre**

1. `Administration` 
2. Trykk på `Nerdschool Training` prosjektet
3. `Create subproject`
4. Name: `Unikt navn på gruppe eller person`

### Opprett parametere
Parametere er systemvariabler som referer til verdier, navn eller mapper på byggeserveren som vi kan bruke under oppsettet. F.eks så referer variabelen `%teamcity.agent.work.dir%` til mappen `C:\TeamCity\buildAgent\work` på byggeserveren, som er rotmappen for der filer TeamCity henter fra git blir lagt. 

Vi må opprette noen slike variabler som er unike for *gruppeprosjektet*.

1. På Admin-siden til gruppeprosjektet du nettopp opprettet, velg `Parameters` i menyen til venstre.
2. `Add new parameter`
3. Name: `<gruppenavn>.app.target`
4. Value: Plassering til `target`-mappen: `app\target`
5. `Add new parameter`
6. Name: `<gruppenavn>.dropfolder`
7. Value: `%teamcity.agent.work.dir%\<gruppenavn>\`
8. `Add new parameter`
9. Name: `<gruppenavn>.dropfolder.app`
10. Value: `app`
11. `Add new parameter`
12. Name: `<gruppenavn>.dropfolder.app.target`
13. Value: `%system.dropfolder%\app\target`

### Opprett VCS Root (Version Control System = Git)

1. `Create VCS root`
2. Type of VCS: Git
3. VCS root name: Valgfritt, anbefaler å bruker GitHub repository navn
4. Fetch URL: `.git` url fra GitHub (f.eks `https://github.com/eaardal/eaardal.github.io.git` - du finner den nede til høyre på GitHub repoet's fremside)
6. Test connection

## Build Configuration: Compile & Test

Ofte vil man skille mellom Compile og Test i en slik prosess, dvs ha disse i to separate trinn, der Test-trinnet bare kjører om Compile-trinnet er ok. I dette tilfellet bruker vi Maven for å compile og teste i samme steg.

Maven (og Java) er installert på byggeserveren, så alle kommandolinjefunksjoner vi kan gjøre med Maven og Java på vår utviklermaskin kan gjøres på byggeserveren, via TeamCity. (Det vil også si at om vi tar inn andre avhengigheter i et prosjekt senere, som f.eks Node/npm, så må disse også installeres på byggeserveren).  

Kommandoen for å bygge med Maven er `mvn clean install`, gitt at man står i mappen med `POM.xml` filen, som er Maven sitt register over avhengigheter og konfigurasjon.

1. På admin-fremsiden til *gruppeprosjektet*, velg `General Settings` og `Create build configuration`
2. Name: `Compile & Test` -> `Save`

### Build Step 1: Compile & Test

1. I menyen til venstre velg `Build Steps` -> `Add build step`
2. Runner type: `Maven`
3. Step name: `Compile & Test`
4. Goals: `clean install`
5. Path to POM file: Avhenger av path til POM.xml i ditt GitHub repo. Trykk på "tree-node" ikonet ved siden av tekstboksen for å velge manuelt. Merk: Pathen inkluderer POM.xml filen, ikke bare mappen til der POM.xml er. Om du bruker repoet fra nerdschool er pathen trolig `app\POM.xml`.
6. `Save`

### Build Step 2: Copy to dropfolder

1. `Add build step`
2. Runner type: `Command line`
3. Step name: `Copy to dropfolder`
4. Execute step: `If all previous steps finished successfully`
5. Working dir: `%teamcity.agent.work.dir%`
6. Custom script: `xcopy %teamcity.build.default.checkoutDir% %<gruppenavn>.dropfolder% /s /e /Y`. Her bruker vi en av *Parameterene* vi lagde i forberedelsene. Dette steget er nødvendig fordi de forskjellige byggekonfigurasjonene i TeamCity befinner seg i ulike mapper på byggeserveren, så for at `Package`-steget (som vi ikke har laget enda, men som blir neste steg i rekken) skal kunne pakke filene fra `Compile & Test`-steget (som vi lager nå), så må vi kopiere filene til en felles mappe der begge byggekonfigurasjonene kan nå filene. Derfor, når vi har kompilert og testet løsningen, så kopierer vi rubbel og bit til en "dropfolder" slik at neste byggesteg kan fortsette.
7. `Save`

### Trigger: VCS Trigger

1. I menyen til venstre velg `Triggers` -> `Add new trigger`
2. `VCS Trigger`
3. `Save`

Nå skal TeamCity automatisk se etter endringer i git repoet hvert 60. sekund og sette i gang `Compile & Test` byggekonfigurasjonen. Gå til `Projects`-siden og se at bygget starter og fullfører ok. Når det har fullført ok, prøv å  start det manuelt med `Run` knappen.

18. Choose coverage runner: No coverage
19. Triggers: Add new trigger -> VCS Trigger
20. Sjekk inn en endring i git, push til GitHub og vent på at TeamCity skal trigge et bygg. TeamCity poller på ca 30 sek intervaller (kan stilles på per prosjekt).
21. Project administration -> Create build configuration
22. Name: Package QA
23. Triggers -> Add new trigger -> Finish Build Trigger
24. Build configuration: Compile & Test
25. Trigger after successful build only -> True
26. Build Steps -> 
27. Project administration -> Create build configuration
28. Name: Deploy QA
29. Triggers -> Add new trigger
30. Finish Build Trigger
31. Build configuration: Package QA
32. Build Steps -> 

`mvn compile war:war`

### 1.2 Test
### 1.3 Deploy