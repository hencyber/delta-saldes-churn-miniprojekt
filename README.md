# Churnmodellen – Saldes Mobil AB

> *Saldes Mobil AB är ett påhittat företag och all kunddata är fiktiv. Repot är en övning i
> kursen Kontinuerlig integration och leverans (MLOps25, Nackademin).*

**Det här är mitt mini-projekt** (Henrik Göransson, Delta Team). Repot är en kopia av teamets
labbrepo [hencyber/delta-saldes-churn](https://github.com/hencyber/delta-saldes-churn), och
jag har byggt vidare med utrullning till Kubernetes. Det nya beskrivs under
[Mini-projekt: utrullning till Kubernetes](#mini-projekt-utrullning-till-kubernetes) längst ner.
Texten fram till dess är lärarens mall och teamets beskrivning av pipelinen.

## Välkommen till teamet

Hej och välkommen!

Kul att du börjar hos oss som MLOps-ingenjör. Här är bakgrunden till ditt första uppdrag.

Vi tappar kunder. Analysteamet har byggt en modell som förutsäger vilka kunder som riskerar
att säga upp sitt abonnemang. Varje måndag får kundtjänst en **ringlista** med de kunder som
har högst risk, och ringer dem med ett erbjudande.

I dag sköts det mesta för hand. Modellen tränas på en dator här på kontoret, och modellfilen
ligger incheckad i `models/`. Ringlistan gör någon varje måndag morgon med kommandona nedan.
Vi har ett CI-workflow, men det kör allt i ett enda jobb, installerar om alla paket vid
varje push och testar på en annan Python-version än servern som gör ringlistan (3.11).
Sedan förra veckan är det dessutom rött, och ingen har hunnit ta reda på varför.

För två veckor sedan gick det riktigt fel. Kundtjänst fick en ringlista från en modell som
var sämre än den förra. Ingen märkte det förrän säljarna klagade, och ingen kunde säga vilken
commit eller vilken data modellen kom från.

**Ditt uppdrag:** se till att bara en testad modell med känd härkomst kan bli en ringlista.
Jag har skrivit ärenden åt dig, se *Kom igång* nedan.

Två regler:

- Ändra inte modellen, datan eller feature-koden i `src/churn/data.py`. Den sköter
  analysteamet. Ditt arbete är workflows, tester och inställningar i repot.
- Ingen pushar direkt till main när vi väl har skydd på plats. Allt går via pull requests.

Lycka till, och fråga hellre en gång för mycket.

*Teamleaden*

## Kom igång

1. Skapa ditt eget repo från mallen: **Use this template → Create a new repository**.
   Välj ditt eget konto och **Public**.
2. Klona det och skapa ärendena som issues (kräver `gh auth login`):

   ```bash
   git clone https://github.com/<ditt-användarnamn>/<ditt-repo>.git
   cd <ditt-repo>
   bash arenden/skapa.sh
   ```

3. Öppna fliken **Actions** och titta på den första körningen. Börja med ärende F1.

## Så kör man det lokalt

```bash
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -e ".[dev]"
pytest                          # testerna
python -m churn.model           # tränar, skriver outputs/modell.joblib och outputs/matvarden.json
python -m churn.evaluate        # kvalitetsgrind: felkod om ROC AUC är under tröskeln
python -m churn.score           # skriver outputs/ringlista.csv
```

## Vad som finns här

| Sökväg | Innehåll |
| --- | --- |
| `data/kundbas.csv` | 2 000 kunder: ålder, region, abonnemang, kostnad, dataförbrukning, supportärenden och om kunden avslutat (`avslutat`) |
| `src/churn/data.py` | Inläsning och features |
| `src/churn/model.py` | Träning. Skriver modell och mätvärden till `outputs/` |
| `src/churn/evaluate.py` | Kvalitetsgrind på mätvärdena |
| `src/churn/score.py` | Ringlistan |
| `tests/` | Tester på tre nivåer: kod, data och modell |
| `kontroller/schema.py` | Datakontrollen (pandera-schemat) |
| `src/churn/app.py`, `Dockerfile` | API:t och imagen som serverar modellen |
| `arenden/` | Ärendena, och `skapa.sh` som gör dem till issues |
| `.github/workflows/ci.yml` | Vårt CI |
| `k8s/` | Deployment, service och ingress för tjänsten (mini-projektet) |
| `docs/rapport.md` | Utrullningsstrategi och plan för drift (mini-projektet) |
| `docs/utdrag/` | Terminalutdrag från uppdatering och tillbakarullning (mini-projektet) |

## Så fungerar pipelinen

Workflowet `.github/workflows/ci.yml` körs vid varje push, vid pull requests mot main och
manuellt (Actions → CI → Run workflow, där man kan välja vilken datafil som ska kontrolleras).
Jobben körs i den här ordningen:

1. `datakontroll` kontrollerar datafilen mot pandera-schemat i `kontroller/schema.py`, med alla
   fel i en rapport. Schemafel stoppar pipelinen. Högre värden än vanligt, som 40 % högre
   dataförbrukning, är inget schemafel utan en signal om att modellen kan behöva tränas om.
2. `test` kör testerna på kod, data och modell, parallellt på Python 3.11, 3.12 och 3.13.
3. `train` startar bara om `datakontroll` och alla testjobb är gröna. Det tränar på 3.11, samma
   version som servern, och sparar modellen som artefakten `modell-<commit-SHA>`.
4. `utvardera` laddar ner samma artefakt, visar mätvärdena i körningens Summary och stoppar
   pipelinen om ROC AUC är under 0,70.
5. `docker` bygger en image med samma modell, smoke-testar `/health` och `/predict` och pushar
   imagen för amd64 och arm64 till `ghcr.io/hencyber/delta-saldes-churn-miniprojekt:<commit-SHA>`.
   Push görs bara från main.
6. `deploy` (mini-projektet) rullar ut imagen till kursens Kubernetes-kluster. Se nedan.

Om ett jobb fallerar hoppas alla jobb efter det över. I teamets repo är `main` skyddad: varje
ändring går via en pull request som en teamkamrat godkänner, och `datakontroll`, alla tre
`test`-jobb, `utvardera` och `docker` måste vara gröna innan den kan slås ihop. I mitt mini-repo
jobbar jag ensam, så där går ändringar via pull requests utan granskning från någon annan.
Godkännandet som krävs är det före produktion.

### Kvalitetströskeln

Vi använder ROC AUC och inte accuracy. Ungefär 81 % av kunderna stannar, så en modell som alltid
gissar att ingen slutar får redan 0,81 i accuracy utan att vara till någon nytta. ROC AUC mäter om
modellen sätter kunder som slutar högre upp i ringlistan än kunder som stannar. 0,5 är slump, och
vi kräver minst 0,70. Tröskeln kan ändras med miljövariabeln `MIN_ROC_AUC`.

### Vilken modell kör vi?

Imagen är taggad med commitens SHA och innehåller modellen från artefakten med samma SHA, alltså
exakt den modell som klarade utvärderingen. `matvarden.json` i artefakten innehåller commit,
datafil och datans sha256. `/health` i den körande tjänsten visar commit, ROC AUC och datans
sha256, så man kan se vilken modell som körs utan att ladda ner artefakten.

### Köra tjänsten lokalt

```bash
pip install -e ".[api]"
python -m churn.model
docker build -t churn .
docker run -p 8000:8000 churn      # http://localhost:8000/health
```

## Mini-projekt: utrullning till Kubernetes

Tjänsten körs i två miljöer: lokalt i kind och i kursens kluster, där den svarar på
<https://henrik-goransson.89-167-53-206.sslip.io/health>. `/health` visar vilken commit som körs.

**Manifesten** i `k8s/`:

- `deployment.yaml`: två repliker, imagen utpekad med commitens SHA (aldrig `latest`),
  readiness och liveness mot `/health`, resursgränser och `preStop` som väntar 5 sekunder innan
  en pod stängs.
- `service.yaml`: ClusterIP på port 80 till containerns port 8000.
- `ingress.yaml`: Traefik och ett certifikat från Let's Encrypt via cert-manager.

**Utrullningsstrategi:** rolling update med `maxSurge: 1` och `maxUnavailable: 0`. En ny pod
startar och måste vara redo innan en gammal tas bort, så tjänsten svarar under hela bytet. Varför
jag valde den och inte blue/green eller canary står i [docs/rapport.md](docs/rapport.md).

**Utrullning från pipelinen:** jobbet `deploy` körs bara från main och bara efter att `docker`
har byggt, smoke-testat och pushat imagen. Det körs i miljön `produktion`, som kräver mitt
godkännande innan jobbet startar. Nyckeln till klustret ligger som hemligheten `KUBECONFIG` i
miljön, så den kan bara läsas av ett godkänt jobb. Jobbet byter image i manifestet till den här
körningens SHA, väntar på `kubectl rollout status` och kontrollerar att `/health` svarar med den
nya commiten. Annars rullar det tillbaka.

**Rulla tillbaka för hand:**

```bash
kubectl rollout history deployment/churn   # vilka versioner som finns
kubectl rollout undo deployment/churn      # tillbaka till förra versionen
kubectl rollout status deployment/churn
curl https://henrik-goransson.89-167-53-206.sslip.io/health
```

**Köra i kind:**

```bash
kind create cluster --name mini
kubectl apply -f k8s/deployment.yaml -f k8s/service.yaml
kubectl port-forward service/churn 8000:80   # http://localhost:8000/health
```

Utdrag från uppdatering, tillbakarullning och mätning av tillgänglighet i båda miljöerna ligger i
`docs/utdrag/`. Plan för drift (hur jag upptäcker en sämre modell eller förändrad data, och vad som
händer då) står i [docs/rapport.md](docs/rapport.md).
