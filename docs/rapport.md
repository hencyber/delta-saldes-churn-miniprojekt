# Mini-projekt: utrullning av Saldes churnmodell i Kubernetes

Henrik Göransson (Delta Team) · repo: https://github.com/hencyber/delta-saldes-churn-miniprojekt

## Upplägg

Tjänsten är teamets FastAPI-image med churnmodellen. Den körs i två miljöer: lokalt i kind och i
kursens kluster, där den nås på https://henrik-goransson.89-167-53-206.sslip.io. Manifesten ligger
i `k8s/`: en deployment med två repliker, en service och en ingress med certifikat från Let's
Encrypt. Imagen pekas ut med commitens SHA, aldrig `latest`, så att det alltid går att säga vilken
commit och vilken modell som körs. `/health` svarar dessutom med commit, ROC AUC och datans sha256.

Deploymenten har resursgränser (50m CPU och 128 Mi minne reserverat, tak 500m och 256 Mi) och två
probes mot `/health`. Readiness styr när en pod får trafik: `/health` svarar 503 tills modellen är
laddad, så ingen trafik skickas till en pod som inte kan svara. Liveness startar om containern om
den slutar svara.

Pipelinen i `.github/workflows/ci.yml` testar, validerar datan, tränar och utvärderar modellen,
bygger imagen för amd64 och arm64 och smoke-testar den. Från `main` pushas imagen till ghcr.io och
jobbet `deploy` rullar ut den. `deploy` körs i miljön `produktion`, som kräver mitt godkännande innan
jobbet startar. Klusterfilen ligger som hemlighet i miljön, så bara ett godkänt jobb kan läsa den. Efter
utrullningen kontrollerar jobbet att `/health` svarar med den nya commiten, annars rullar det tillbaka.

## Utrullningsstrategi: rolling update

Jag valde rolling update med `maxSurge: 1` och `maxUnavailable: 0`. Kubernetes startar en ny pod,
väntar tills readiness-proben säger att den är redo, och tar först då bort en gammal. Tjänsten har
alltså hela tiden minst två pods som kan svara.

Varför inte de andra:

- **Blue/green** kräver två fulla uppsättningar pods samtidigt. Min kvot i kursens kluster är 384 Mi
  reserverat minne, och tjänsten använder runt 123 Mi per pod i vila. Fyra pods får inte plats. Fördelen, att kunna
  byta tillbaka direkt, får jag ändå genom `kubectl rollout undo`.
- **Canary** passar när man vill pröva en ny modell på en del av trafiken och jämföra resultat. Det
  kräver att man kan styra hur stor andel som går till den nya versionen och mäta utfallet. Ringlistan
  har få användare och utfallet (om kunden slutar) syns först efter flera veckor, så en canary skulle
  inte ge något svar under själva utrullningen.

Rolling update ryms i kvoten (tre pods under uppdateringen, 3 × 128 = 384 Mi) och ger ingen nertid,
vilket jag mätte. Först reserverade jag 96 Mi, men `kubectl top pods` visade att en pod använder
123 Mi i vila. Reservationen ska täcka det, annars kan en pod hamna på en nod som inte har plats för
den, så jag höjde till 128 Mi. Mätningarna nedan gjordes med 96 Mi.

### Mätning

En pod i klustret anropade `/health` var 0,2 sekund medan en uppdatering pågick (300 anrop).

| Mätning | Resultat |
| --- | --- |
| kind, utan paus före nedstängning | 297 av 300 lyckades |
| kind, med `preStop: sleep 5` | 300 av 300 |
| Kursens kluster, inifrån | 300 av 300 |
| Kursens kluster, utifrån via internet | 299 av 300 (utan uppdatering: 296 av 300) |

De tre felen i den första mätningen kom från att en pod som stängs fortfarande fick trafik en kort
stund. En paus på 5 sekunder innan containern stängs löste det. De enstaka felen utifrån var
nätverksfel mellan min dator och servern, eftersom de kom lika ofta utan någon uppdatering.

### Uppdatering och tillbakarullning

Jag har rullat ut, uppdaterat och rullat tillbaka i båda miljöerna (utdrag i `docs/utdrag/`):
version 1 (`1bd2a1c`) → version 2 (`f50f5f1`) → `kubectl rollout undo` tillbaka till version 1.
Från pipelinen rullades sedan `3bf6bf5` ut efter godkännande. Tillbakarullning görs med
`kubectl rollout undo deployment/churn`, och `kubectl rollout history` visar vilken version som är vilken.

## Plan för drift

**Hur jag upptäcker att en ny modell är sämre**

- *Innan den rullas ut:* `utvardera` stoppar en modell med ROC AUC under 0,70, och `test_slar_dummy`
  stoppar en modell som inte är bättre än att gissa. En ny modell bör också jämföras med den som körs:
  är den sämre än nuvarande version ska den inte rullas ut även om den klarar 0,70.
- *Efter utrullningen:* om en kund på ringlistan slutar syns först efter några veckor. Varje månad
  räknar man om ROC AUC på de kunder som faktiskt slutade eller stannade, och jämför med värdet vid
  träningen. Sjunker det tydligt, till exempel under 0,70, är modellen sämre i verkligheten.
- *Direkt i tjänsten:* fördelningen av riskerna som `/predict` ger. Om andelen högriskkunder plötsligt
  ändras mycket är något fel, antingen i modellen eller i datan som skickas in.

**Hur jag upptäcker att datan har förändrats**

- Schemafel (fel kolumner, saknade värden, orimliga åldrar, dubbletter) stoppas redan av datagrinden.
- Förändrad men giltig data stoppas inte, till exempel att dataförbrukningen i den nya leveransen var
  40 % högre. Den fångas genom att varje ny leverans jämförs med träningsdatan: medelvärde och fördelning
  per kolumn, och andelen kunder per region och abonnemang. Avviker en kolumn mer än en gräns (t.ex.
  20 % i medelvärde) är det drift.

**Vad som ska hända**

| Händelse | Åtgärd |
| --- | --- |
| Tjänsten svarar inte | Liveness startar om poden. Händer det igen: larm och tillbakarullning |
| Ny version beter sig fel efter utrullning | `kubectl rollout undo` till förra versionen, sedan felsökning |
| Datadrift i en ny leverans | Larm till analysteamet. Modellen tränas om på den nya datan |
| Modellen har blivit sämre i verkligheten | Omträning. Den nya modellen måste klara tröskeln och slå den nuvarande |
| Omtränad modell klarar inte kraven | Den gamla får köra vidare, inget rullas ut |

Omträning går genom samma pipeline som allt annat: ny data i en pull request, datagrinden, tester,
träning, utvärdering och godkännande innan utrullning. Då är också den nya modellen spårbar till commit
och data.
