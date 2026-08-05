# STUPA Inte

Ett klubbcentrerat skal ovanpå STUPA. Välj din klubb och se allt aktuellt på
en sida: kommande matcher, tabeller, spelade resultat och turneringar.

## Repostruktur

Lägg filerna så här i `stupainte/stupainte.github.io`:

```
index.html                            ← frontenden (en fil, inget byggsteg)
scraper/hamta.py                      ← hämtar från STUPA:s API
.github/workflows/uppdatera-data.yml  ← kör hämtaren varje natt
data/                                 ← genereras, committas av workflowet
  index.json
  klubb/<slug>.json
ARKITEKTUR.md                         ← hur alltihop hänger ihop, plus API-referens
```

## Kom igång

```bash
pip install requests

# Hämta ett distrikt först — går på några sekunder
python3 scraper/hamta.py --ut data --serie-id 435

# Eller allt: 14 seriespel, hela landet
python3 scraper/hamta.py --ut data

# Testa frontenden. fetch fungerar inte mot file://, så en server behövs.
python3 -m http.server
# → http://localhost:8000
```

## Så fungerar det

STUPA:s webb är en klientrenderad SPA, men den hämtar sin data från ett öppet
JSON-API som vi kan anropa direkt. API:et tillåter dock bara anrop från
`*.stupaevents.com`, så frontenden kan inte prata med det. Därför hämtar ett
schemalagt GitHub Actions-jobb datan, committar den som JSON, och den statiska
sidan läser filerna.

```
GitHub Actions  →  STUPA:s API  →  data/*.json i repot  →  GitHub Pages
   varje natt                          git commit            index.html
```

Ingen server, inga kostnader, inget att underhålla utöver hämtaren.

Se `ARKITEKTUR.md` för API-referensen: endpoints, datakedjan, hur URL-segmenten
i `/events/435/1186/2/7/7` ska tolkas, och varför klubbtillhörighet inte behöver
gissas fram.

## Medföljande exempeldata

`data/` innehåller ett litet urval verklig data från Nordöstra Svealands
BTF 26/27 (24 klubbar i indexet, en komplett klubbfil för Heby AIF) så att
frontenden går att köra direkt. Kör hämtaren för att ersätta den med allt.

## Status

Klart:

- Frontenden — klubbsökning, fyra flikar, mobilanpassad, mörkt läge, sparar
  klubbval lokalt. 18 automatiska kontroller passerar.
- Hämtaren — går via API:et, ingen Playwright.
- Workflowet — schemalagt, med skyddsnät mot att skriva över fungerande data
  om STUPA ändrar sitt API.

Kvar:

- Filtrera bort testevenemang ur `get_events` (`"Malin gör en till serie"`,
  `"TEST TEST NBTF distriktsserier"`).
- Turneringsdelen hämtar bara namn och datum. Anmälningsstatus och
  deltagarlistor kräver `get_event_participants` respektive
  `bookings/{event_id}/get_registered_players`.
- Spelade resultat är obeprövade — säsongen 26/27 har inte börjat, så inga
  avgjorda matcher finns att testa mot ännu.
- Rimlighetskontrollen i workflowet är satt till minst 20 klubbar. Justera
  när du vet hur många det blir när hela landet hämtas.
