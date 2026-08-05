# STUPA Inte — arkitektur

Ett klubbcentrerat skal ovanpå STUPA. Användaren väljer sin klubb och ser allt
aktuellt på en sida: kommande matcher, tabeller, spelade resultat och turneringar.

## Grundproblemet — och lösningen

STUPA (`*.stupaevents.com`) är en **klientrenderad SPA**. Ett anrop mot
`https://sbtfeventsott.stupaevents.com/events/435/1186/2/7/7` returnerar ett tomt
HTML-skal — allt innehåll hämtas av JavaScript efter att sidan laddats.
HTML-skrapning fungerar därför inte.

**Men det behövs inte.** SPA:n hämtar all sin data från ett öppet JSON-API, och
det API:et kan vi anropa direkt. Ingen Playwright, ingen webbläsare, inga
CSS-selektorer som går sönder när STUPA byter design.

Kvar finns dock ett hinder: **CORS**. API:et accepterar bara anrop från
`*.stupaevents.com`. Ett anrop från `stupainte.github.io` blockeras av
webbläsaren (verifierat: `Failed to fetch` från främmande origin, både med
`tenant` som header och som query-parameter). Frontenden kan alltså inte prata
med STUPA direkt — datan måste hämtas i förväg, serverside. Därav GitHub Actions.

## Vald arkitektur

```
  ┌────────────────────────┐
  │ GitHub Actions         │   schemalagt, t.ex. 04:00 varje natt
  │  scraper/hamta.py      │   + manuell körning via workflow_dispatch
  └───────────┬────────────┘
              │ läser
              ▼
  ┌────────────────────────┐
  │ STUPA                  │
  │ stupaevents.com        │
  └───────────┬────────────┘
              │ skriver JSON
              ▼
  ┌────────────────────────┐
  │ data/ i repot          │   git commit + push
  │  index.json            │
  │  klubb/<slug>.json     │
  └───────────┬────────────┘
              │ serveras statiskt
              ▼
  ┌────────────────────────┐
  │ stupainte.github.io    │   index.html, vanilla JS, ingen backend
  └────────────────────────┘
```

**Varför den här lösningen:**

- Gratis och driftfri. Ingen server, inget konto utöver GitHub.
- Snabb. Frontenden läser statiska JSON-filer från GitHub Pages CDN.
- Robust. Om STUPA ändrar sidstruktur går skrapningen sönder, men den senast
  fungerande datan ligger kvar i repot och sidan fortsätter fungera.
- Versionshanterad. Varje commit är en ögonblicksbild — man kan se exakt när och
  hur datan ändrades, vilket gör felsökning enkel.

**Priset:** datan är upp till ett dygn gammal. För seriespel är det oftast
oproblematiskt (matchprogram ändras sällan med kort varsel). Vill man ha färska
resultat under en pågående seriehelg kan man köra workflowet oftare, t.ex.
varje timme lördag–söndag.

## Datamodell

### `data/index.json`

Laddas först. Driver klubbsökningen.

```json
{
  "genererad": "2026-08-05T04:00:00Z",
  "sasong": "2026/2027",
  "klubbar": [
    { "slug": "hammarby", "namn": "Hammarby IF BTF", "antal_lag": 8 }
  ]
}
```

### `data/klubb/<slug>.json`

Laddas när användaren valt klubb. Innehåller allt klubben behöver.

```json
{
  "klubb":     { "slug": "hammarby", "namn": "Hammarby IF BTF" },
  "sasong":    "2026/2027",
  "uppdaterad": "2026-08-05T04:00:00Z",

  "lag": [
    {
      "namn": "Hammarby IF BTF 1",
      "serie": { "namn": "Division 2 Östra", "niva": "regional",
                 "region": "Nordöstra Svealand" },
      "stupa_url": "https://..."
    }
  ],

  "kommande": [
    {
      "datum": "2026-09-13", "tid": "13:00",
      "serie": "Division 2 Östra", "omgang": "Omgång 1",
      "hemma": "Hammarby IF BTF 1", "borta": "Spårvägens BTK",
      "arrangor": "Hammarby IF BTF", "plats": "Eriksdalshallen",
      "stupa_url": "https://..."
    }
  ],

  "resultat":  [ "samma form som kommande, plus \"hemma_poang\" och \"borta_poang\"" ],

  "tabeller": [
    {
      "serie": "Division 2 Östra",
      "rader": [
        { "placering": 1, "lag": "Hammarby IF BTF 1", "spelade": 5,
          "vunna": 4, "oavgjorda": 0, "forlorade": 1,
          "matchpoang": 8, "setdiff": "+12" }
      ]
    }
  ],

  "turneringar": [
    { "namn": "Stockholm Open", "datum": "2026-10-11", "ort": "Stockholm",
      "status": "Anmälan öppen", "sista_anmalan": "2026-09-27",
      "stupa_url": "https://..." }
  ]
}
```

Datumen är alltid `YYYY-MM-DD`. Tider är `HH:MM` i 24-timmarsformat.

Fältet `stupa_url` finns överallt så att användaren alltid kan klicka sig
vidare till originalsidan i STUPA när något saknas eller behöver verifieras.
Skalet ersätter inte STUPA, det gör det navigerbart.

## API-referens

Allt nedan är verifierat mot skarp data i augusti 2026.

**Bas-URL:** `https://testbackend.stupaevents.com/ott/v1`

Namnet till trots är detta produktionsbackenden — det är den värd som den
publika sajten `sbtfeventsott.stupaevents.com` själv anropar. Innehållet
stämmer exakt med vad sidan renderar.

**Autentisering:** ingen. Enda kravet är headern `tenant: sbtf`.
Utan den svarar API:et `422 field required`. Fel värde ger
`400 Ogiltig organisation`.

**Dokumentation:** hela OpenAPI-specifikationen ligger öppen på
`https://testbackend.stupaevents.com/openapi.json` (1,9 MB, 893 endpoints,
varav 71 under `/ott/`). `/docs` kräver inloggning, men specen gör inte det.

### Datakedjan

```
get_events                            → 278 evenemang
  event_type "L" = seriespel (14 st)  ·  "T" = turneringar (264 st)
  │
  ├── get_stages?event_id=435         → en stage per division
  │     stage.event_category.category.category_description → "Division 4A"
  │     stage.event_category.category_id                   → 1186
  │
  └── get_group_matches?stage_id=6137&view=standard
                       &show_matrix=true&fetch_point_system=true
        ├── group_matrix[]  → tabellen (rank, matches_won/lost, group_points)
        └── matches[]       → matcherna
              start_time, round.name, venue.name
              participants[].participant_name  → "Heby AIF B"
              participants[].selected_parents  → klubbtillhörighet
```

Använd **inte** `get_events_categories` för divisionsnamn — den returnerar bara
toppnivåerna (`Division 4`) och saknar undergrupperna A/B/C. Namnet finns
inbäddat i varje stage enligt ovan.

### Klubbtillhörighet är löst i API:et

Varje lag bär sin kanoniska förening i `selected_parents`:

```json
"selected_parents": [
  { "parent_role": "Member Association",
    "parent_name": "Nordöstra Svealands Bordtennisförbund", "parent_abbr": "Nordö" },
  { "parent_role": "Club",
    "parent_name": "Vassunda Idrottsförening", "parent_abbr": "Vassunda IF" }
]
```

Detta är värt att uppmärksamma: det problem som `thelinkan/bt-serier` lägger
mest möda på — att gissa vilken förening ett lagnamn tillhör genom
normaliseringsregler — **existerar inte** när man går via API:et. Föreningen
följer med som strukturerad data. `get_role_parents?role_id=590` returnerar
dessutom hela det svenska klubbregistret.

### URL-strukturen, dekodad

`https://sbtfeventsott.stupaevents.com/events/435/1186/2/7/7`

| Segment | Betydelse | I exemplet |
| ------- | --------- | ---------- |
| `435`   | `event_id` — ett distrikts seriespel för en säsong | Nordöstra Svealands BTF 26/27 |
| `1186`  | `category_id` — divisionen | Division 4A |
| `2/7/7` | vy-index i gränssnittet (tabell/matcher/lag) | — |

Djuplänkar kan alltså konstrueras som
`/events/{event_id}/{category_id}/2/7/7`.

### Omfattning

Ett enda anrop till `get_events` ger hela Sverige: 14 seriespel (13 distrikt
plus *Nationellt seriespel 2026-2027*, id 417) och 264 turneringar. För
Nordöstra Svealand (id 435) ger kedjan **560 matcher, 10 divisioner och
24 klubbar** på ungefär tio HTTP-anrop.

Notera att evenemangslistan även innehåller uppenbara testposter
(*"Malin gör en till serie"*, *"TEST TEST NBTF distriktsserier"*). Dessa bör
filtreras bort innan publicering.

## Förhållande till `thelinkan/bt-serier`

Det projektet är ett **desktopprogram** (PySide6 + SQLite) för den som vill
analysera matchprogram lokalt. Det löser en angränsande men annan uppgift.

Skillnaden i ansats: bt-serier är arrangörscentrerat (vem arrangerar vilken
seriehelg), medan detta skal är spelarcentrerat (vad händer i min klubb).

Den tekniska skillnaden är dock större än så. bt-serier skrapar den renderade
webbsidan med Playwright, vilket medför att:

- hämtningen tar minuter istället för sekunder,
- den går sönder varje gång STUPA ändrar sin layout,
- föreningsnamn måste gissas fram med normaliseringsregler,
- man måste manuellt registrera en fungerande startadress per distrikt.

Ingen av dessa svårigheter finns när man går via API:et. Det är värt att höra
av sig till honom — han har lagt ner rejält med arbete på ett problem som
`openapi.json` löser gratis, och hans domänkunskap om seriespelet vore
värdefull i det här projektet.

## Nästa steg

1. Skapa repot `stupainte/stupainte.github.io` med följande struktur:

   ```
   index.html                       ← frontenden
   scraper/hamta.py                 ← hämtaren
   .github/workflows/uppdatera-data.yml
   data/                            ← genereras av hämtaren
   ```

2. Kör `python3 scraper/hamta.py --ut data` lokalt och kontrollera resultatet.
   Börja gärna med ett distrikt: `--serie-id 435`.
3. Starta en lokal server (`python3 -m http.server`) och testa frontenden.
   `fetch` fungerar inte mot `file://`.
4. Slå på GitHub Pages för repot, committa, och låt workflowet ta över.
5. Filtrera bort testevenemangen ur `get_events`.
6. Fyll ut turneringsdelen — `get_events` ger namn och datum, men
   anmälningsstatus och deltagarlistor kräver ytterligare endpoints
   (`get_event_participants`, `bookings/{event_id}/get_registered_players`).
