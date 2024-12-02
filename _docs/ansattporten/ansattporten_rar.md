---
title: RAR-typer i Ansattporten
description: Ansattporten bruker RAR til kommunisere representasjonsforhold

sidebar: ansattporten
product: Ansattporten
redirect_from: /ansattporten_rar
---


Ansattporten bruker standarden [Rich Authorization Requests (RAR)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-rar) til å strukturere informasjon om representasjonsforhold, både i forespørsler og tokens. 


RAR er en ny Oauth2-utvidelse for transaksjonsspesifikke autorisasjoner. Der “basic” Oauth2 kun gir tilgang til et såkalt “scope” (tekst-streng), åpner RAR for tilgang til mer utvidede datamodeller i form av autorisasjonstyper. Autorisasjonstypen(e) blir utlevert i token som et nytt hierarkisk claim kalla `authorization_details` som igjen er ein array av autorisasjonsobjekter, der hvert objekt består av:

* [Standardiserte felt](https://www.rfc-editor.org/rfc/rfc9396.html#name-common-data-fields):
    * `type` (påkrevd felt, definerer den aktuelle autorisasjonstypen)
    * `actions`
    * `locations` (tiltenkt mottakar, aka =audience for tokenet)
    * `identifier` (kan peike på ein konkret ressurs hjå APIet)
    * `privileges`: 
    * `datatypes` (ein array med datatyper klient ønsker å få frå APIet)

* Eigendefinert datamodell
    * til ein gitt type vil det normalt vere definert og dokumentert ein tilhøyrande gyldig datamodell


# RAR-typer støttet i Ansattporten

Følgende authorization_type er støttet i Ansattporten:

| `authorization_type` | 	 Skildring |
|-|-|
| `ansattporten:altinn:service`  | Bruker lenketjenester (ServiceCode) fra Altinn 2 som autorativ kilde for representasjonsforhold |
| `ansattporten:altinn:resource` | IKKE I BRUK.  Skal støtte bruk av Altinn3-ressurser som autorattiv kilde for representasjon. [Se backlog-sak](https://github.com/orgs/digdir/projects/8/views/38?pane=issue&itemId=75426143&issue=digdir%7Croadmap%7C400) |
| `ansattporten:entra` | IKKE I BRUK.  Skal støtte bruk av arbeidsgivers pålogging som autorattiv kilde for representasjon. [Se backlog-sak](https://github.com/orgs/digdir/projects/8/views/38?pane=issue&itemId=87373562&issue=digdir%7Croadmap%7C438) |


Det er p.t. ikke mulig å be om ulike RAR-type i samme påloggingsforespørsel. Klienten må istedet implementere flere login-knapper i sin egen løsning.



# Datamodell for Altinn 2 Lenketjenester (`ansattporten:altinn:service`)

Bare kunder som er tjenesteeier i Altinn kan benytte Altinn Autorisasjon som autorativ kilde.

Dersom kunden ønsker å bruke Altinn 2 lenketjenester (ServiceCode) som autorativ kilde for representasjonsforhold, må klienten oppgi  `ansattporten:altinn:service` som RAR-type. 

Følgende claims kan sendes inn i request: 

| claim | kardinalitet|beskrivelse |
|-|-|-|
|resource | påkrevd|Hvilken lenketjeneste i Altinn som etterspørres. Må formatteres slik: `urn:altinn:resource:{tjenestekode}:{tjenesteugave} `|
|organizationform | Valgfri| Begrense organisasjonsvelger til at sluttbruker bare kan velge hovedenheter (`enterprise`) eller underenheter (`business`).|
|allow_multiple_organizations| Valgfri| Dersom `true` så kan sluttbruker velge flere organisasjoner i organisasjonsvelgeren. Default er false.|
|allow_deleted_organizations | Valgfri| Dersom `true` så vil organisasjonsvelger vise sletta verksemder. Default er false.|

[Her finner en liste over alle tjenestekoder i Altinn 2](https://www.altinn.no/api/metadata?language=1044) 

> **Mange av dagens standard Altinn-roller gir veldig breie tilganger ("Post/arkiv", "Utfyller/innsender").**  Dette er problematisert med at de ikke følger gode dataminimeringsprinsipp, og vanskeliggjør det å skulle holde oversikt over hva en gitt rolle faktisk gir tilgang til.  Derfor tilbyr vi ikke innlogging på vegne av Altinn-roller i Ansattporten, tjenesten må spesifisere en lenketjeneste. 


Datamodellen for respons inneholder de samme claimene som i request, men i tillegg vil det utleveres:

| claim | beskrivelse |
|-|-|-|
| reportees | Array med valgte virksomheter. |
| Rights | For hver virksomhet, et array med rettigheter som innlogget bruker har for aktuell tjenestekode.  |
| Name | For hver virksomhet, navnet på valgt virksomhet|

Dersom det er forespurt flere representasjonsforhold, så vil `authorization_details` inneholde ett json-objekt per lenketjeneste som brukeren har rettighet til. 


#### Testbrukere

Man kan teste løsningen uten å lage en integrasjon ved å bruke vår demo-tjeneste [https://demo-client.test.ansattporten.no/](https://demo-client.test.ansattporten.no/).  Her kan man også studere protokoll-flyten i detalj.   Dersom man ønsker å teste organisasjonsvelger, så kan man bruke `[{"type":"ansattporten:altinn:service","resource": "urn:altinn:resource:2480:40"}]` i authorization_details-feltet (denne tjenestekoden gir ut nøkkelroller).

Vi anbefaler å bruke [Tenor testdata-søk](https://www.skatteetaten.no/skjema/testdata/) til å finne test-brukere. Tenor har mulighet til å filtrere slik at man får bare **daglig leder** fra test-Enhetsregisteret. En annen fordel med Tenor er at det kun er syntetiske testdata her, så man slipper å risikere å blande produksjons- og test-data.

> **MERK:** Dersom testbrukeren ikke finnes fra før i Altinn sitt testmiljø (typisk for syntetiske fødselsnummer), vil ikke organisasjonsvelger fungere. Dette løses enkelt ved å logge inn i TT02 en gang.


# Datamodell for Altinn 3 ressurser (`ansattporten:altinn:resource`)

TBD

# Datamodell for Entra ID  (`ansattporten:entra`)

TBD