# Politiloggen API → Supabase: Komplett oppsett

Dette dokumentet beskriver hele oppsettet for å hente politihendelser fra det
offisielle, åpne Politiloggen-API-et og lagre dem i Supabase, klar til bruk
på en nettside. Skrevet for å kunne overføres til et nytt prosjekt.

**Status:** Verifisert fungerende 2026-06-30. API-et var "under utvikling" pr.
11. mai 2026 ifølge politiet selv — feltnavn kan endre seg uten varsel.

---

## 1. Bakgrunn / hvorfor dette finnes

Det gamle, mye omtalte endepunktet `api.politiet.no/politiloggen/v1/atom`
**blokkerer all automatisert tilgang globalt** (testet fra hjemme-IP, RPi,
Supabase Edge Function, mobil 4G, rss2json-proxy — alt feiler med
`Connection reset by peer`, sannsynligvis Cloudflare/TLS-fingerprinting).

Det finnes derimot et **annet, nyere offisielt API** som ikke er blokkert:

```
https://api.politiloggen.politiet.no
```

Dette er IKKE lenket fra den vanlige `politiet.no/politiloggen`-siden (ingen
"API for utviklere"-lenke der), men det er bekreftet ekte og offisielt:
Swagger-spec'ens `info`-felt sier:

> "11. mai 2026: Dette APIet er fortsatt under utvikling. Formatet kan
> endres uten forvarsel. Et API utviklet av Politiet for å hente ut
> operative meldinger fra operasjonssentralene i Norge."
> Kontakt: politiloggen@politiet.no. Lisens: NLOD 2.0.

Testet med vanlig `curl` (ikke bare nettleser) — fungerer uten auth, uten
blokkering.

---

## 2. API-referanse

**Swagger/OpenAPI-spec:** `https://api.politiloggen.politiet.no/swagger/v1/swagger.json`

### Hovedendepunkt: hendelser

```
GET https://api.politiloggen.politiet.no/messages
```

**Query-parametere (alle valgfrie):**

| Parameter | Type | Beskrivelse |
|---|---|---|
| `Districts` | array | Filtrer på politidistrikt-navn (f.eks. `Sør-Øst Politidistrikt`) |
| `Municipalities` | array | Filtrer på kommunenavn (f.eks. `Siljan`) — mer presist enn Districts |
| `Categories` | array | Filtrer på kategori (f.eks. `Trafikk`, `Brann`, `Ulykke`) |
| `Ids` | array | Filtrer på spesifikke melding-IDer |
| `IsActive` | boolean | Filtrer på om hendelsen er aktiv |
| `DateFrom` / `DateTo` | string | Datoformat `yyyy-mm-dd` |
| `SortBy` | enum | `None`, `Date`, `Update`, `LastMessageOn`, `District`, `Category`, `Area`, `Municipality`, `IsActive` |
| `SortOrder` | enum | `Ascending`, `Descending` |
| `Fields` | array | Begrens hvilke felt som returneres |
| `Skip` | integer | Pagineringsoffset (default 0) |
| `Take` | integer | Antall rader (default 10, **maks 50**) |

**Eksempel (testet OK):**
```bash
curl -s -G "https://api.politiloggen.politiet.no/messages" \
  --data-urlencode "Municipalities=Siljan" \
  --data-urlencode "Take=10" \
  --data-urlencode "SortBy=Date" \
  --data-urlencode "SortOrder=Descending"
```

**Responsformat:**
```json
{
  "messages": [
    {
      "id": "261hcl-0",
      "threadId": "261hcl",
      "category": "Trafikk",
      "district": "Sør-Øst Politidistrikt",
      "municipality": "Siljan",
      "area": "Siljanvegen",
      "isActive": false,
      "text": "Melding om trafikkuhell...",
      "createdOn": "2026-06-17T05:29:02.1165847+00:00",
      "updatedOn": "2026-06-17T05:29:02.173688+00:00",
      "imageUrl": null,
      "previouslyIncludedImage": false,
      "isEdited": false
    }
  ],
  "totalCount": 7
}
```

**Viktig om `id`/`threadId`:** Flere meldinger kan tilhøre samme hendelse
("tråd"). `id` er formatert som `{threadId}-{sekvensnummer}`, f.eks.
`261hcl-0`, `261hcl-1`, `261hcl-2` for tre oppdateringer av samme sak.
Bruk `threadId` (eller parse `id` på siste `-tall`) for å gruppere
oppdateringer som hører sammen i UI.

### Andre endepunkter
- `GET /districts` — liste over politidistrikt (id + navn)
- `GET /districts/extended` — distrikt med tilhørende kommuner
- `GET /categories` — kategoriliste (støtter `nb`/`nn`)
- `GET /feeds/atom` / `GET /feeds/rss` — feed-format
- `GET /messages/{id}` — enkelt-melding
- `GET /messages/{id}/image` — bilde tilknyttet melding
- `GET /messages/export` — CSV-eksport (genereres én gang daglig)
- `GET /messagethreads` / `GET /messagethreads/{id}` — trådvisning (maks 1000/side)

### Rate limiting
**Ingen dokumentert eller observert rate limit.** Eneste begrensning sett i
praksis: `Cache-Control: public, max-age=15` — en CDN-cache på 15 sekunder
foran API-et (header `X-Cache-Status: MISS/HIT`). Hyppigere kall enn det gir
uansett ikke ferskere data. Helt uproblematisk for en cron som kjører hvert
5. minutt eller sjeldnere.

### Distrikts-IDer (fra `/districts`)
```json
[
  {"id":205,"name":"Agder politidistrikt"},
  {"id":212,"name":"Finnmark politidistrikt"},
  {"id":203,"name":"Innlandet politidistrikt"},
  {"id":208,"name":"Møre og Romsdal politidistrikt"},
  {"id":210,"name":"Nordland politidistrikt"},
  {"id":201,"name":"Oslo politidistrikt"},
  {"id":206,"name":"Sør-Vest politidistrikt"},
  {"id":204,"name":"Sør-Øst politidistrikt"},
  {"id":211,"name":"Troms politidistrikt"},
  {"id":209,"name":"Trøndelag politidistrikt"},
  {"id":207,"name":"Vest politidistrikt"},
  {"id":202,"name":"Øst politidistrikt"}
]
```

---

## 3. Supabase-tabell

```sql
CREATE TABLE police_events (
  id SERIAL PRIMARY KEY,
  event_id TEXT UNIQUE,        -- = API sin "id" (threadId-sekvensnummer)
  title TEXT,
  description TEXT,            -- = API sin "text"
  municipality TEXT,
  category TEXT,
  event_time TIMESTAMPTZ,      -- = API sin "createdOn"
  created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE police_events ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Public read access" ON police_events
  FOR SELECT USING (true);

-- Skriving kun via service_role (Edge Function bruker service key, omgår RLS)
```

**Regel:** RLS alltid på (public read, service write), aldri unrestricted.

---

## 4. Supabase Edge Function: `politilogg-fetch`

Henter siste hendelser fra API-et og upserter til `police_events`.
Deno/TypeScript, kjører på Supabase sin Edge Runtime.

`index.ts`:
```ts
import "jsr:@supabase/functions-js/edge-runtime.d.ts";

const SUPABASE_URL = Deno.env.get('SUPABASE_URL') ?? '';
const SUPABASE_SERVICE_KEY = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '';
const POLITILOGGEN_URL = 'https://api.politiloggen.politiet.no/messages?Municipalities=Siljan&Take=50&SortBy=Date&SortOrder=Descending';

interface PolitiMessage {
  id: string;
  threadId: string;
  category: string;
  district: string;
  municipality: string;
  area: string;
  isActive: boolean;
  text: string;
  createdOn: string;
  updatedOn: string;
}

Deno.serve(async (_req: Request) => {
  try {
    const response = await fetch(POLITILOGGEN_URL, {
      headers: { 'Accept': 'application/json' }
    });

    if (!response.ok) {
      return new Response(JSON.stringify({ error: `HTTP ${response.status}` }), { status: 500 });
    }

    const data = await response.json();
    const messages: PolitiMessage[] = data.messages ?? [];

    if (messages.length === 0) {
      return new Response(JSON.stringify({ message: 'No entries found' }), { status: 200 });
    }

    let inserted = 0;
    let skipped = 0;

    for (const msg of messages) {
      const locationPart = [msg.municipality, msg.area].filter(Boolean).join(', ');
      const title = [msg.category, locationPart].filter(Boolean).join(': ');

      const entry = {
        event_id: msg.id,
        title,
        category: msg.category,
        description: msg.text,
        municipality: msg.municipality,
        event_time: msg.createdOn,
      };

      const res = await fetch(`${SUPABASE_URL}/rest/v1/police_events`, {
        method: 'POST',
        headers: {
          'apikey': SUPABASE_SERVICE_KEY,
          'Authorization': `Bearer ${SUPABASE_SERVICE_KEY}`,
          'Content-Type': 'application/json',
          'Prefer': 'return=minimal,resolution=ignore-duplicates',
        },
        body: JSON.stringify(entry)
      });

      if (res.status === 201 || res.status === 200) {
        inserted++;
      } else {
        skipped++;
      }
    }

    return new Response(JSON.stringify({
      success: true,
      entries_found: messages.length,
      inserted,
      skipped
    }), {
      headers: { 'Content-Type': 'application/json' }
    });

  } catch (err) {
    return new Response(JSON.stringify({ error: String(err) }), { status: 500 });
  }
});
```

**Deploy (med Supabase MCP-verktøy eller `supabase functions deploy`):**
- Navn: `politilogg-fetch`
- Entrypoint: `index.ts`
- `verify_jwt: false` (funksjonen kalles fra cron uten JWT — den skriver med
  service role-nøkkel internt, så ingen ekstern auth kreves)
- Bytt `Municipalities=Siljan` til ønsket kommune/distrikt ved gjenbruk i
  annet prosjekt

**Manuelt testkall:**
```bash
curl -s -X POST "https://<project-ref>.supabase.co/functions/v1/politilogg-fetch"
# -> {"success":true,"entries_found":7,"inserted":7,"skipped":0}
```

---

## 5. Automatisk kjøring: pg_cron + pg_net (hvert 5. minutt)

Kjøres direkte i Postgres-databasen via Supabase sine innebygde extensions
`pg_cron` (scheduler) og `pg_net` (async HTTP-kall). Ingen ekstern
cron-tjeneste eller HA-automasjon nødvendig.

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
CREATE EXTENSION IF NOT EXISTS pg_net;

SELECT cron.schedule(
  'politilogg-fetch-5min',
  '*/5 * * * *',
  $$
  SELECT net.http_post(
    url := 'https://<project-ref>.supabase.co/functions/v1/politilogg-fetch',
    headers := '{"Content-Type": "application/json"}'::jsonb
  );
  $$
);
```

**Sjekk at jobben kjører:**
```sql
SELECT jobid, jobname, schedule, active FROM cron.job WHERE jobname = 'politilogg-fetch-5min';

-- Kjørehistorikk:
SELECT * FROM cron.job_run_details
WHERE jobid = (SELECT jobid FROM cron.job WHERE jobname = 'politilogg-fetch-5min')
ORDER BY start_time DESC LIMIT 10;
```

**Stoppe/fjerne jobben:**
```sql
SELECT cron.unschedule('politilogg-fetch-5min');
```

---

## 6. Frontend: hente og gruppere data

```js
const SUPABASE_URL = 'https://<project-ref>.supabase.co';
const SUPABASE_KEY = '<anon/publishable key>';

const res = await fetch(
  `${SUPABASE_URL}/rest/v1/police_events?select=*&order=event_time.desc&limit=10`,
  { headers: { 'apikey': SUPABASE_KEY } }
);
const data = await res.json();
```

**Gruppering av oppdateringer per sak** (samme `threadId` → flere `event_id`
som `261hcl-0`, `261hcl-1`, `261hcl-2`):

```js
const match = eventId.match(/([a-z0-9]+)[\/-]\d+$/i);
const caseId = match ? match[1] : eventId;
```

> Regex matcher BÅDE gammelt format (skråstrek, fra det utgåtte
> atom-feedet, f.eks. `.../26n1bh/0`) OG nytt format (bindestrek, fra dette
> API-et, f.eks. `26n1bh-0`) — viktig hvis tabellen inneholder historiske
> rader fra før byttet.

---

## 7. Overføring til nytt prosjekt — sjekkliste

1. Opprett `police_events`-tabell med RLS (seksjon 3)
2. Deploy Edge Function `politilogg-fetch` (seksjon 4), juster
   `Municipalities=`/`Districts=` i URL-en til ønsket sted
3. Aktiver `pg_cron` + `pg_net`, planlegg jobb (seksjon 5)
4. Bytt `<project-ref>` og API-nøkkel i frontend-koden (seksjon 6)
5. Test manuelt med `curl -X POST .../functions/v1/politilogg-fetch` før du
   stoler på cron
