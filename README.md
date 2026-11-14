# verbeke-n8n

n8n-workflows voor Verbeke (project 3: intake en opvolging).

## Workflow: intake

Bestand: [`workflows/intake.json`](workflows/intake.json)

1. **Formulier**: de klant vult een webformulier in (naam, e-mail, telefoon, gemeente en een vrije omschrijving).
2. **AI-classificatie**: een Information Extractor met Claude Haiku leest de omschrijving en geeft gestructureerde output volgens een JSON Schema:
   - `onderwerp`: één waarde uit een vaste enum (`warmtepomp`, `ventilatie`, `sanitair`, `renovatie`, `herstelling`)
   - `dringend`: `true` als iets kapot is of de klant zonder verwarming of water zit
   - `samenvatting`: hoogstens 15 woorden, enkel wat de klant zelf zegt
3. **Database**: de aanvraag wordt opgeslagen via de databasefunctie `public.intake_aanvraag(...)`, met geparametriseerde query's. n8n verbindt met een aparte rol met least privilege (`n8n_intake`), die alleen deze functie mag uitvoeren en geen rechten op de tabellen heeft. De functie draait als `security definer` en controleert zelf haar invoer.
4. **Mail**: de werfleider krijgt een mail met het aanvraagnummer, het onderwerp, de AI-samenvatting en de originele tekst van de klant. Bij dringende aanvragen begint het onderwerp met `DRINGEND`.

## Workflow 2: opvolging

Bestand: [`workflows/opvolging.json`](workflows/opvolging.json)

1. **Planning**: de workflow start elke dag om 8u (tijdzone Europe/Brussels).
2. **Herinnering aan de klant**: `public.offertes_voor_herinnering($1)` geeft de offertes terug die 7 of meer dagen geleden verstuurd zijn, nog niet opgevolgd werden en een klant met een e-mailadres hebben. Oudste eerst, standaard hoogstens 20 per run (de functie weigert meer dan 50). Let op: de geëxporteerde workflow geeft nu als test `2` mee als limiet. Zet die waarde op `20` voor het echte gebruik.
3. **Eerst mailen, dan registreren**: per offerte gaat eerst de herinnering weg, daarna roept de workflow `public.registreer_opvolging($1)` aan. Dat is *at-least-once*: mislukt het registreren, dan komt de offerte de volgende run terug en krijgt de klant in het slechtste geval twee keer een mail, maar nooit geen. De registratie is idempotent: een tweede aanroep op dezelfde dag verandert niets.
4. **Verzamelmail per werfleider**: `public.offertes_voor_werfleider()` geeft de offertes die al een herinnering kregen maar nog open staan. De `group by` per werfleider gebeurt in SQL (met `count(*)` en `string_agg`), zodat n8n per werfleider één rij krijgt en één mail stuurt.
5. **Database-rol**: n8n verbindt met een aparte rol `n8n_opvolging`, los van `n8n_intake` (scheiding van taken). De rol mag alleen deze drie functies uitvoeren en heeft geen rechten op de tabellen. De functies draaien als `security definer`.
6. **Testinbox**: alle mails gaan naar een vaste testinbox, nooit naar de klant of de werfleider. Het onderwerp begint met `[TEST]` en de mail vermeldt voor wie ze bedoeld was. De adressen in de data zijn fictief.

## Tools

- [n8n](https://n8n.io): Form Trigger, Schedule Trigger, Information Extractor, Postgres en Send Email
- Anthropic Claude (Haiku) als taalmodel
- PostgreSQL (Supabase)
- SMTP voor de mail

## Database

In de repo [ArneGoesAI/verbeke-data](https://github.com/ArneGoesAI/verbeke-data):

- migratie **0005**: de functie `intake_aanvraag` en de rol `n8n_intake` (workflow intake)
- migratie **0006**: de functies `offertes_voor_herinnering`, `registreer_opvolging` en `offertes_voor_werfleider` en de rol `n8n_opvolging` (workflow 2: opvolging)

## Importeren

Importeer `workflows/intake.json` en `workflows/opvolging.json` in n8n. De bestanden bevatten geen wachtwoorden, tokens of sleutels, alleen verwijzingen naar credentials. Koppel na het importeren je eigen credentials voor Anthropic, Postgres (per workflow een eigen rol) en SMTP, en pas de afzender en ontvanger (de testinbox) van de mails aan.

## Beperkingen

- Een aanvraag krijgt maar één onderwerp. Een klant die bijvoorbeeld een warmtepomp én nieuw sanitair vraagt, komt maar in één categorie terecht.
- n8n draait lokaal. De workflow opvolging is daarom niet gepubliceerd (niet actief): het dagelijkse schema van 8u loopt pas als n8n op een server draait en de workflow daar geactiveerd wordt.
