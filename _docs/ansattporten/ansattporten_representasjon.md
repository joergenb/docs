---
title: Integrasjonsguide for Ansattporten
description: Ansattporten er en kopi av ID-porten men der funksjonaliteten er tilpasset innlogging i ansatt/representasjonskontekst.

sidebar: ansattporten
product: Ansattporten
redirect_from: /ansattporten_guide
---

Ansattporten er en egen innloggingtjeneste med funksjonalitet som skiller seg noe fra ID-porten, slik at den skal være mer hensiktmessig å bruke i ansattkontekst eller i andre situasjoner der det er ønskelig å opptre i et representasjonsforhold på vegne av andre virksomheter eller personer.

Du finner mer overordned informasjon om Ansattporten ved å klikke [her](ansattporten_om.html)

# Beskrivelse av bruksscenarioet

På denne siden beskriver vi hvordan du setter opp en tjeneste som bruker Ansattporten til punkt-autentisering.   Dersom du heller vil lese om hvordan tjenesten din kan kreve innlogging på vegne av en virksomhet, må du klikke [her](ansattporten_representasjon.html). 

# Brukerreise

Punkt-autentisering er den enkleste brukerreisen.  I dette scenariet utfører brukeren en innlogging til en tjeneste, og får etablert en isolert SSO-sesjon kun til denne tjenesten:

1. Bruker klikker login-knapp hos tjeneste.  
2. Bruker autentiserer seg med eID gjennom Ansattporten.
4. Bruker blir sendt tilbake til tjenesten.

Teknisk er dette løst som en helt standard [OpenID Connect code flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth), som vist i sekvensdiagrammet nedenfor:

<div class="mermaid">
sequenceDiagram

participant B as Bruker
participant C as Tjeneste
participant A as Ansattporten

B->>C: Klikker "login" på tjeneste
C->>A: /authorize (redirect)
note over B, A: sluttbruker autentiserer seg
A->>C: redirect med code
C->>A: /token
A->>C: id_token
note over B,C: innlogget i tjenesten

</div>

Ulikt ID-porten så vil ikke brukeren få opprettet en felles SSO-sesjon i Ansattporten.  Dersom brukeren forsøker å logge på en annen tjeneste rett etterpå, med samme browser, så må brukeren autentisere seg på nytt.  Men dersom brukeren forsøker å logge på samme tjeneste på nytt, så vil vil hen bli logga inn automatisk. 

Denne SSO-oppførselen er realisert vha funksjonaliteten [isolert SSO-sesjon](oidc_func_nosso.html), der Ansattporten overstyrer flagget `sso_disabled` til true uavhengig av hva kunden selv har satt i selvbetjening.   Merk at dette også betyr at Ansattporten ikke tilbyr individuelle sesjonslevetider per tjeneste, men isteden så deler alle tjenestene en felles underliggende http-sesjonscookie der max-levetid på 120 min starter ved innlogging til første tjeneste, og inaktivitetstimer på 30 min gjelder for unionen av de innloggede tjenestene. 

Dette betyr også at kunder må støtte utlogging både fra egen tjeneste, men også kunne håndtere utloggingsforespørsler (front_channel_logout) fra Ansattporten initiert av en annen tjeneste.


### Brukerreise 2: Innlogging på vegne av virksomhet

Ansattporten tilbyr *beriket* autentisering, altså at informasjon om innlogget bruker blir beriket med et representasjonsforhold/autorisasjonsinformasjon fra en ekstern autorativ kilde.  I første versjon av løsningen er det Altinn Autorisasjon som tilbys som autorativ kilde.

En tjeneste aktiverer støtte for beriket autentisering ved å inkludere informasjon om ønsket representasjonsforhold (="avgiver") i autentiseringforespørselen.  Ansattporten vil da vise en organisasjonsvelger etter autentisering, der sluttbruker må velge hvilke(n) organisasjon hen vil representere:

![organsisasjonsvelger](/images/idporten/oidc/ansattporten_orgvelger2.png)

Brukerreise blir da som følger:

1. Bruker klikker login-knapp hos tjeneste.  Kallet til ansattporten inneholder informasjon om hvilket representasjonsforhold som tjenesten trenger
2. Bruker autentiserer seg med sterk eID.  Det opprettes en isolert SSO-sesjon i Ansattporten.
3. Dersom bruker har en eller flere representasjonsforhold av de forespurte typene, vil Ansattporten vise en organisasjonsvelger, der hen kan velge hvilken organisasjon (avgiver) som denne innloggingen skal være på vegne av
4. Bruker blir sendt tilbake til tjenesten, med informasjon om valgt organisasjon i id_tokenet

Dette er detaljert i sekvensdiagrammet under:


<div class="mermaid">
sequenceDiagram

participant B as Bruker
participant C as Tjeneste
participant A as Ansattporten
participant AA as Altinn Autorisasjon

B->>C: Klikker "login" på tjeneste
C->>A: /authorize inkl forespurte representasjonstyper  (redirect)
note over B, A: sluttbruker autentiserer seg
  A->>AA: hente avgivere for bruker for ønska representasjon
  A->>A: vise organisasjonsvelger
  note over B, A: Bruker velger en  organisasjon
A->>C: redirect med code
C->>A: /token  (id_token)
note over B,C: innlogget i tjenesten

</div>


**endring Q1-2024:** Dersom brukeren mangler det forespurte representasjonsforholdet, så vil brukeren likevel bli logga inn, men det vil ikke inkluderes noen representasjons-informasjon.

### Brukerreise 3:  datadeling på vegne av virksomhet

Denne brukerreisen er nært beslektet med #2, men med et ekstra steg: Tjenesten som brukeren har logget inn til, vil i tilegg utveksle data på vegne av valgt organisasjon mot et API hos en 3dje-part.  

<div class="mermaid">
sequenceDiagram

participant B as Bruker
participant C as Tjeneste
participant A as Ansattporten
participant AA as Altinn Autorisasjon
participant D as API

B->>C: Klikker "login" på tjeneste
C->>A: /authorize inkl forespurte representasjonstyper  (redirect)
note over B, A: sluttbruker autentiserer seg
  A->>AA: hente avgivere for bruker for ønska representasjon
  A->>A: vise organisasjonsvelger
  note over B, A: Bruker velger en  organisasjon
A->>C: redirect med code
C->>A: /token  
note over B,C: innlogget i tjenesten
C->>D: API-kall (access_token)
D->>C: data

</div>

API-tilbydere bør merke seg at den organisasjonen som brukeren velger i organisasjonsvelgeren ikke trenger å samsvare med den organisasjonen som eier tjenesten.  

> Eksempel: Ola logger inn på vegne av Oslo Kommune til en tjeneste fra Visma som konsumerer et API hos Skatteetaten.


Teknisk sett benytter tjenesten (oauth2-klienten) Ansattporten sine access_token for å få denne API-tilgangen. 

## Metadata

Følgende miljøer er tilgjengelige for kunder:

|Miljø|Metadata |
|-|-|
|TEST|[https://test.ansattporten.no/.well-known/openid-configuration](https://test.ansattporten.no/.well-known/openid-configuration)| 
|PROD|[https://ansattporten.no/.well-known/openid-configuration](https://ansattporten.no/.well-known/openid-configuration) | 



## Test

Man kan teste løsningen uten å lage en integrasjon ved å bruke vår demo-tjeneste [https://demo-client.test.ansattporten.no/](https://demo-client.test.ansattporten.no/).  Her kan man også studere protokoll-flyten i detalj.   Dersom man ønsker å teste organisasjonsvelger, så kan man bruke `[{"type":"ansattporten:altinn:service","resource": "urn:altinn:resource:2480:40"}]` i authorization_details-feltet (denne tjenestekoden gir ut nøkkelroller).

Vi anbefaler å bruke [Tenor testdata-søk](https://www.skatteetaten.no/skjema/testdata/) til å finne test-brukere. Tenor har mulighet til å filtrere slik at man får bare **daglig leder** fra test-Enhetsregisteret. En annen fordel med Tenor er at det kun er syntetiske testdata her, så man slipper å risikere å blande produksjons- og test-data.

> Ved å bruke **TestID** som innloggingsmetode slipper man å kontakte Digdir for å få opprettet og resatt testbrukere.  TestID har også integrasjon mot Tenor, så du kan hente tilfeldige test-personer derifra.

**MERK:** Dersom testbrukeren ikke finnes fra før i Altinn sitt testmiljø (typisk for syntetiske fødselsnummer), vil ikke organisasjonsvelger fungere. Dette løses enkelt ved å logge inn i TT02 en gang.


## Representasjonsforhold og RAR

Ansattporten bruker [Rich Authorization Requests (RAR)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-rar) til å strukturere informasjon om representasjonsforhold, både i forespørsler og tokens.  Dette er nærmere forklart under *protokollflyt* nedenfor.

Følgende `authorization_type` er støttet i Ansattporten:

| `authorization_type` | Skildring |
|-|-|
|ansattporten:altinn:service| Bruker lenketjenester (ServiceCode) fra Altinn 2 som autorativ kilde for representasjonsforhold |

Det er mulig å be om flere representasjonsforhold i samme påloggingsforespørsel.  Da vil organisasjonsvelger vise unionen av alle organisasjoner som brukeren har ett eller flere av de forespurte representasjonsforholdene for.  Tokenet vil innehold en liste med de faktiske representasjonsforholdene brukere har for valgt(e) organisasjon(er).

#### Datamodell for `ansattporten:altinn:service`

Datamodell for request:

| claim | kardinalitet|beskrivelse |
|-|-|-|
|resource | påkrevd|Hvilken ressurs i Altinn som etterspørres. Se kodeverk nedenfor. |
|organizationform | Valgfri| Begrense organisasjonsvelger til at sluttbruker bare kan velge hovedenheter (`enterprise`) eller underenheter (`business`)|
|allow_multiple_organizations| Valgfri| Dersom `true` så kan sluttbruker velge flere organisasjoner i organisasjonsvelgeren. |
|allow_deleted_organizations | Valgfri| Dersom `true` så vil organisasjonsvelger vise sletta verksemder|


der `resource` må følgje desse reglane:

|Ressurs-identifikator| Beskrivelse|Eksempel|
|-|-|-|
|urn:altinn:resource:{tjenestekode}:{tjenesteutgave} | Altinn 2 [tenestekode/utgåve](https://www.altinn.no/api/metadata?language=1044) | altinn:resource:3906:141205

De konkrete ressurs-definisjonene kan finnes på [metadata-endepunktet til Altinn](https://tt02.altinn.no/api/metadata).

> **Mange av dagens standard Altinn-roller gir veldig breie tilganger ("Post/arkiv", "Utfyller/innsender").**  Dette er problematisert med at de ikke følger gode dataminimeringsprinsipp, og vanskeliggjør det å skulle holde oversikt over hva en gitt rolle faktisk gir tilgang til.  Derfor er ikke rolle tilbudt som mulig ressurs i Ansattporten i første runde. 


##### Respons

Datamodellen for respons er lik datamodellen for request, men i tillegg vil det utleveres:


| claim | beskrivelse |
|-|-|-|
| reportees | Array med valgte virksomheter. |
| Rights | Array med rettigheter som innlogget bruker har for aktuell tjenestekode.  |
| Name | Namnet på valgt organisasjon|

Dersom det er forespurt flere representasjonsforhold, så vil responsen være en liste, med ett respons-objekt per lenketjeneste som brukeren har rettighet til. 



## Protokoll-flyt

Ansattporten følger en helt normal autorisasjons-kodeflyt ihht Oauth2/OIDC.

### 1: Autentiseringsforespørsel til autorisasjons-endepunktet

Klienten sender en autentiseringsforespørsel ved å redirecte sluttbrukeren til autorisasjonsendepunktet.

Se [detaljert dokumentasjon for autorisasjonsendepunktet](oidc_protocol_authorize.html) for valgmuligheter.  Merk at i Ansattporten følger vi Oauth2.1, slik at bruk av PKCE, state og nonce er påkrevd for alle klienter.

Klienten må være forhåndsregistrert i Ansattporten, se [klient-registrering](oidc_func_clientreg.html).

For tjenester med høye krav til sikkerhet bør en i tillegg vurdere å bruke [PAR](oidc_protocol_par.html) til å først POSTe autentiseringsparametrene direkte til ID-porten før en redirecter, slik at disse parametrene ikke blir eksponert i brukers browser.

Dersom klienten ønsker å vise organisasjonsvelger, må forespørselen inkludere et eller flere RAR-objekter som ytterligere detaljerer forespørselen. 

Eksempel på request:
```
https://login.test.ansattporten.no/authorize?
  scope=openid&
  client_id=9a99e96d-b56c-4f74-a689-f936f71c8819&
  acr_values=low&
  response_type=code&
  redirect_uri=https%3A%2F%2Ftest-client.test.ansattporten.no%2Fcallback&
  state=Hocd3Rs77Jw1BYOFJ_PP87XPza-MdrC0M9MeL33cmqE&
  nonce=KUXk5WlVwgz-YYf0UkhLuquqaJSRr7BcmwwPC22IC1o&
  code_challenge_method=S256&
  code_challenge=YhKJpC67w6qB2KupfDuKocVarvxL8vb9WSmSB6-p-Zc&
  authorization_details= [
    {
      "type": "ansattporten:altinn:service",
      "ressurs": "urn:altinn:resource:3906:141205",
      "allow_multiple_organizations": "true",   
      "organizationform": "business"
    },
    {
      // eventuelt flere representasjonsforhold
    }
]
```
(merk at eksempelet er vist i klartekst for lesbarhet og ikke riktig enkoda)



### 2: Redirect tilbake til tjenesten

Etter at brukeren har logget inn vil det sendes en redirect tilbake til klienten til den forhåndsregistrerte `redirect_uri`.  Redirecten vil vil inneholde et autorisasjonskode-parameter `code` som  brukes til oppslag for å hente tokens.  Koden er base64-enkoda og URL-safe.

Redirecten vil også inneholde `state`.  Klienten MÅ validere at mottatt state-verdi stemmer med det den sendte i forespørsel, for å detektere replay attacks.  Klienten MÅ også validere at `iss` stemmer med forventa utsteder.

Merk at Ansattporten vil redirecte tilbake selv om brukeren ikke hadde noe representasjonsforhold (evt. valgt selv å ikke representere noen organisasjon for akkurat denne innlogginga.) 

Eksempel på respons:

```
https://test-client.test.ansattporten.no/callback?
 code=ERnHqwptnS9t3nGyad09Jw.og36LcOit1CaoxHSASE4_w&
 iss=https%253A%252F%252Ftest.ansattporten.no&
 state=Hocd3Rs77Jw1BYOFJ_PP87XPza-MdrC0M9MeL33cmqE
```




### 3: Utstedelse av token fra token-endepunktet

Token-endepunktet brukes for utstedelse av tokens.


Bruk av endepunktet varierer litt med hvilken klient-autentiseringsmetode som benyttes. Følgende autentiseringsmetoder fra [OIDC kap. 9](http://openid.net/specs/openid-connect-core-1_0.html#ClientAuthentication) støttes:

* **client_secret_basic** / **client_secret_post** - Klientautentisering basert på client_secret
* **private_key_jwt** - Klientautentisering basert på JWT'er signert med virksomhetssertifikater eller forhåndsregistrerte nøkkel-par.

Sistnevnte metode er anbefalt for klienter med høye krav til sikkerhet.

Eksempel på forespørsel:

```
POST https://test.ansattporten.no/token
Authorization: Basic ***
Content-Type: application/x-www-form-urlencoded; charset=UTF-8

code=ERnHqwptnS9t3nGyad09Jw.og36LcOit1CaoxHSASE4_w&
grant_type=authorization_code&
redirect_uri=https://test-client.test.ansattporten.no/callback&
code_verifier=oQEG5SwL-dQlUL2ZkteJV8v0Fxz9z6j4Y1Q_86gEq78
```

Se [detaljert dokumentasjon for token-endepunktet](oidc_protocol_token.html) for alle valgmuligheter.  

Dersom forespørselen blir validert som gyldig, vil det returneres et eller flere token:

* **id_token**: Autentiseringsbevis,  "hvem brukeren er"
* **access_token**: Tilgangs-token, forteller "hva brukeren kan få tilgang til"
* **refresh_token**: Brukes av klienten til å fornye access_token uten brukerinteraksjon (så lenge som autorisasjonen er gyldig)


```

{
  "access_token" : "eyJ4NWMiOlsiTUlJRktqQ0NC....",
  "refresh_token" : "eyJlbmMiOiJBMTI4Q0JDLUhTMjU....",
  "scope" : "openid",
  "id_token" : "eyJ4NWMiOlsiTUlJRktqQ0NCQktnQXdJQkFnSUxBbGx....",
  "token_type" : "Bearer",
  "expires_in" : 600
}
```



#### id_token

id_tokenet inneholder identiteten til den autentiserte brukeren - det forteller det hvem brukeren er og hvilken organisasjon hen har valgt å representere.

Normal bruker tjenesten id_tokenet kun til å opprette en egen, lokal sesjon.  id_tokenet har derfor en ganske kort gyldighetsperiode.

Eksempel:
```

{
  "sub" : "z9RuQiLefXmJOBnywa_c75YQMH05nDsHjw0RFzuJC8M",
  "amr" : [ "TestID" ],
  "iss" : "https://test.ansattporten.no",
  "pid" : "45840375084",
  "locale" : "en",
  "nonce" : "h3DeaRXe4pVXEdYAwR4hxeOtG2DHwF_zIDvjAAWf-x0",
  "sid" : "-EuMkNN8AuLgMhbXgID7ewtpC4Kuw-HS1dXWi7zo8Dc",
  "aud" : "ansattporten_demo_client_prod",
  "acr" : "high",
  "authorization_details" : [ {
    "resource" : "urn:altinn:resource:2480:40",
    "type" : "ansattporten:altinn:service",
    "resource_name" : "Produkter og tjenester fra Brønnøysundregistrene",
    "reportees" : [ {
      "Rights" : [ "Read", "ArchiveDelete", "ArchiveRead" ],
      "Authority" : "iso6523-actorid-upis",
      "ID" : "0192:987464291",
      "Name" : "DIGITALISERINGSDIREKTORATET AVD LEIKANGER"
    } ]
  } ],
  "auth_time" : 1671033189,
  "name" : "NAMNET TIL SLUTTBRUKER",
  "exp" : 1671033331,
  "iat" : 1671033211,
  "jti" : "raWcGKMobFU"
}


```


**Korrekt validering av id_token** av klienten er kritisk for sikkerheten i løsningen. Tjenesteleverandører som tar i bruk tjenesten må utføre validering i henhold til kapittel [3.1.3.7 - ID Token Validation i OpenID Connect Core 1.0 spesifikasjonen](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation).

I utgangspunktet er id_token frå Ansattporten identiske som [id_token fra ID-porten](oidc_protocol_id_token.html), men det kan være verdt å merke seg følgende forskjeller:

|claim|beskrivelse
|-|-|
|acr| Ansattporten bruker de nye verdiene: `substantial` og `high`|
|authorization_details| Informasjon om representasjonsforhold. Se detaljer ovenfor.|




#### access_token

Access_tokenet (tilgangstoken) gir klienten [tilgang til APIer hos tredjepart](oidc_auth_oauth2.html) på vegne av kombinasjonen av autentiserte brukeren og valgt organisasjon(er).

Levetiden på aksess_tokenet er som oftest relativt kort (typisk 120 sekunder). Dersom tokenet er utløpt, kan klienten forespørre nytt acess_token ved å bruke refresh_tokenet. Det gjennomføres da en klient-autentisering, for å sikre at tokens ikke blir utlevert til feil part.

Levetider kan også tilpasses per klient. Men merk at dette kan overstyres alt etter [hvilke oauth2 scopes](oidc_protocol_scope.html) som er i tokenet.

Ansattporten sine access_token er svært like [ID-porten sine access token](oidc_protocol_access_token.html), men med samme unntakene som i avsnittet over.

API-tilbydere bør merke seg at sluttbruker kan velge en annen organisasjon enn den som har 



### 4: userinfo og utlogging
Ansattporten tilbyr ikke et /userinfo-endepunkt.

Siden Ansattporten er basert på [isolerte SSO-sesjoner](oidc_func_nosso.html), så må tjenesten kunne håndtere [utlogging på samme måten som ID-porten](oidc_protocol_logout.html).  Dvs. både tilby brukeren å kunne logge ut, samt å måtte håndtere utloggingsforsepørsler initiert fra andre tjenester i Ansattporten.




### Litt mer om RAR

RAR er en ny Oauth2-utvidelse for transaksjonsspesifikke autorisasjoner.  Der "basic" Oauth2 kun gir tilgang til et såkalt "scope" (tekst-streng), åpner RAR for tilgang til mer utvidede datamodeller i form av **autorisasjonstyper**.  Autorisasjonstypen(e) blir utlevert i token som et nytt hierarkisk claim kalla `authorization_details` som igjen er ein array av autorisasjonsobjekter, der hvert objekt består av:
- standardiserte felt:
  - type (påkrevd felt, definerer den aktuelle autorisasjonstypen)
  - action
  - locations (tiltenkt mottakar, aka =audience for tokenet)
  - identifier  (kan peike på ein konkret ressurs hjå APIet)
  - datatypes (ein array med datatyper klient ønsker å få frå APIet)
- eigendefinerte felt,
  - til ein gitt `type` vil det normalt vere definert og dokumentert ein tilhøyrande gyldig datamodell

