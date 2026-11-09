# nginx + Let's Encrypt i Docker

nginx og certbot er to tjenester i samme compose-fil. nginx kjører hele tiden; certbot kjøres med `docker compose run` når dere trenger et sertifikat, og avslutter etterpå. De deler to mapper:

- `certbot/www` – certbot skriver valideringsfilen hit, nginx serverer den på `/.well-known/acme-challenge/`
- `certbot/conf` – certbot lagrer sertifikatene her, nginx leser dem

```text
nginx-letsencrypt/
├── docker-compose.yml
├── nginx/
│   └── default.conf      <- aktiv konfig (starter som steg 1)
├── steg2-https.conf      <- kopieres inn når sertifikatet finnes
└── certbot/
    ├── www/
    └── conf/
```

## Før dere starter

- Domenet har en A-post som peker på serverens offentlige IP. Har det en AAAA-post (IPv6), må den også peke riktig – ellers fjern den.
- Port 80 **og** 443 er åpne inn fra internett.
- Ingen annen container bruker port 80 eller 443 (`docker ps`). Stopp den gamle nginx-containeren først.
- Bytt `elev.eksempel.no` med domenet i `nginx/default.conf` og `steg2-https.conf`.

## Steg 1: Start nginx med bare HTTP

```bash
mkdir -p nginx certbot/www certbot/conf
docker compose up -d nginx
```

## Steg 2: Hent sertifikatet

Prøv først med `--dry-run`. Den tester mot Let's Encrypts testserver og lagrer ingenting, så dere ikke bruker opp forsøkene.

```bash
docker compose run --rm certbot certonly \
  --webroot -w /var/www/certbot \
  -d elev.eksempel.no \
  --email elev@eksempel.no --agree-tos --no-eff-email \
  --dry-run
```

Står det `The dry run was successful`, kjør samme kommando uten `--dry-run`.

## Steg 3: Slå på HTTPS

```bash
cp steg2-https.conf nginx/default.conf
docker compose exec nginx nginx -t          # sjekk konfigen
docker compose exec nginx nginx -s reload
```

Åpne `https://elev.eksempel.no`. `http://` skal sende videre til `https://`.

## Fornyelse

Sertifikatet varer 90 dager. Forny manuelt når det nærmer seg:

```bash
docker compose run --rm certbot renew
docker compose exec nginx nginx -s reload
```

## Feilsøking

Følg nginx-loggen mens certbot kjører: `docker compose logs -f nginx`

| Ser dere | Betyr |
| --- | --- |
| `GET /.well-known/acme-challenge/... 404` | certbot og nginx deler ikke `certbot/www` – sjekk volumene i compose-filen |
| `GET /.well-known/acme-challenge/... 301` | Redirect til HTTPS er på for tidlig – bruk steg 1-konfigen |
| Ingen forespørsel i loggen | Trafikken når ikke fram – sjekk DNS (A og AAAA), portforward og brannmur |
| nginx starter ikke etter steg 3 | Feil domenenavn i stien til sertifikatet – sjekk `ls certbot/conf/live/` |
