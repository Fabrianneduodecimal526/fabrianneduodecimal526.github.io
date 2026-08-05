# Tester

Frontendtester som kör `index.html` i en riktig DOM (jsdom) mot datan i
`data/`, med `fetch` ersatt av läsning från disk.

```bash
npm install jsdom
node test/frontend.test.mjs
```

Testet kontrollerar bland annat att klubbsökningen hanterar å/ä/ö, att egna
lag markeras i både matchlistor och tabeller, att ospelade serier visas som
deltagarlista i stället för tabell, att arrangörsvyn grupperar rätt antal
matcher per speldag, och att djuplänkarna till STUPA innehåller category_id.

Siffrorna i testet är bundna till Hammarby IF och nuvarande data. När
säsongen fortskrider behöver de justeras.
