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

## Tools

- [n8n](https://n8n.io): Form Trigger, Information Extractor, Postgres en Send Email
- Anthropic Claude (Haiku) als taalmodel
- PostgreSQL (Supabase)
- SMTP voor de mail

## Database

De functie `intake_aanvraag` en de rol voor n8n staan in migratie **0005** in de repo [ArneGoesAI/verbeke-data](https://github.com/ArneGoesAI/verbeke-data).

## Importeren

Importeer `workflows/intake.json` in n8n. Het bestand bevat geen wachtwoorden of sleutels, alleen verwijzingen naar credentials. Koppel na het importeren je eigen credentials voor Anthropic, Postgres en SMTP, en pas de afzender en ontvanger van de mail aan.

## Beperkingen

- Een aanvraag krijgt maar één onderwerp. Een klant die bijvoorbeeld een warmtepomp én nieuw sanitair vraagt, komt maar in één categorie terecht.
