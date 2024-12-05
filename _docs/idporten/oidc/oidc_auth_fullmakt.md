---
title: Fullmaktspålogging
description: Innlogging på vegne av andre

sidebar: oidc
product: ID-porten
redirect_from: /oidc_auth_fullmakt
---



{% include note.html content="Innlogging på vegne av andre er beta-funksjonalitet .  Vi ønsker virksomheter velkommen til å hjelpe oss med å teste og forbedre funksjonaliteten gjennom en pilotfase i 2025. De som deltar, må være forberedt på at det kan bli løpende endringer i funksjonalitet og tekniske grensesnitt. %}


# Bakgrunn

Det er et stort udekka behov

Basert på denne bakgrunn ønsker Digdir å teste hvordan ID-porten kan understøtte en nasjonal fullmaktsløsning.

I første rekke er det


# Brukerreise

ID-porten ønsker at virksomheten implementerer fullmaktspålogging på følgende måte: 

1. Bruker logger inn som seg selv hos en tjeneste  
1. Tjenesten tilbyr brukeren en egen knapp for å representere noen andre
1. Tjenesten sender en ny innloggingsforespørsel til ID-porten der den indikerer at dette er fullmaktspålogging
1. Bruker velger hvem hen vil representere i ID-portens "fullmaktsvelger"
1. Bruker er innlogget i tjenesten på vegne av en annen

Vi mener det blir best brukeropplevelse at innlogget bruker må "tvinges" gjennom to steg: først logge inn som seg selv, deretter logge inn på vegne av noen andre.  Dette gjør det mye tydligere for brukeren at man representerer noen andre. Det medfører også at eksisterende tjenester må tilpasses dersom de ønsker å utvides med fullmaktspålogging.

ID-porten har bevisst valgt å *ikke* realisere fullmaktspålogging ved impersonifisering (dvs. bare "bytte fødselsnummer"), selv om det kanskje kunne gitt raskere utbredelse og mindre behov for tilpassing i eksisterende tjenester. Vi mener relevant regelverk pålegger tjenester å spore at handlinger har blitt utført av fullmektig og hvem fullmektig er, og da vil ikke impersonifisering i ID-porten være hensiktsmessig.



# Hvem kan innbygger representere ?

Innlogget bruker kan ikke velge fritt, men må velge et eksisterende fullmaktsforhold.  Dette fullmaktsforholdet må være registrert i en ekstern, autorativ kilde.

I pilotperioden 2025 så er det Vergemålsregisteret til Statens Sivilrettsforvaltning som er den tilgjengelige autorative kilden.  

ID-porten har ikke - og vil aldri få - en egen database over fullmakter.  Dette betyr at administrasjon av fullmakter (å gi, endre, eller trekke tilbake en fullmakt) skal og må skje i hos den autorative kilden.

# Virksomhetens plikter ved fullmaktspålogging

TBD - vil bli avklart gjennom pilot-perioden.

Vi antar at virksomheten må: 
- selv avklare nødvendig hjemmel til å behandle personopplysninger knyttet til fullmaktsforhold
- selv avklare hvilken av de strukturerte fullmaktsgruppene som passer deres tjeneste
- implementere nødvendig sporing i egne logger (tekniske og juridiske) at handlinger skjer i fullmaktsforhold



# Protokoll-flyt

<div class="mermaid">
sequenceDiagram

participant B as Bruker
participant C as Tjeneste
participant A as ID-porten
participant S as Najonal Fullmaktsløsning

B->>C: Klikker "login" på tjeneste
C-->>A: /authorize (redirect)
note over A: sluttbruker autentiserer seg
A-->>C: redirect med code
C->>+A: /token
A->>-C: id_token
note over B,C: innlogget som seg selv

B->>C: Klikker "representér noen andre" på tjeneste
C-->>A: /authorize med <br/>påkrevd fullmaktstype (redirect)
A->>+S: Har innlogga bruker <br/>forespurt fullmaktstype(r) ?
S->>-A: Liste med fullmaktsgivere
note over A: sluttbruker velger en fullmaktsgiver
A-->>C: redirect med code
C->>+A: /token
A->>-C: id_token
note over B,C: innlogget på vegne av valgt fullmaktsgiver
</div>


Den første delen av flyten er altså en helt normal innlogging i ID-porten.  Dette er dokumentert [her](oidc_auht_codeflow.html)

Den andre delen av flyten, dvs. selve fullmaktspålogginga, er realisert ved bruk av Oauth2-utvidelsen [Rich Authorization Requests (RAR)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-rar), og er dokumentert nedenfor i 3 steg:

### Steg 1: påloggingsforespørsel

Klienten må inkludere claimet `authorization_details` i autorisasjonsforespøreselen for å trigge fullmaktspålogging.  Et eksempel er vist her:

```
https://login.test.idporten.no/authorize?
  scope=openid&
  client_id=9a99e96d-b56c-4f74-a689-f936f71c8819&
... 
  authorization_details= [
    {
      "type": "idporten:fullmakt",
      "permission_roles": [ "skatt"] 
    }
  ]
```
(merk at eksempelet er forenklet)

`authorization_details`-arrayet inneholdet et JSON-objekt der claimet `type` forteller hvilken type representasjon som tjenesten ønsker å benytte. Ulike `type` vil ha egne datamodeller for hvilke andre claims som inngår i request og respons.  Datamodellene er beskrevet [her](oidc_protocol_rar.html).




### Steg 2: fullmaktsvelger

I normale tilfeller så har brukeren allerede en aktiv SSO-sesjon i ID-porten som følge av den første pålogginga som seg selv.  Derfor vil bruken nå slippe å autentisere seg en gang til og hoppe direkte til fullmaktsvelgeren.  Her må brukeren velge hvem en vil representere. 

ID-porten vil kontrollere, ved kall mot autorativ kilde, at fullmaktsforholdet eksisterer og er gjeldende, slik at innlogget bruker ikke får valgt eller injisert ikke-eksisterende fullmaktsforhold.

Dersom innlogget bruker ikke har noen fullmaktsforhold vil ID-porten vise en feilmelding, og bruker må trykke "back" i browser for å komme tilbake til tjenesten.

Dersom innlogget bruker angrer seg, kan hen velge å representere seg selv.

Når bruker har gjort sitt valgt, vil browser bli redirecta tilbake til tjeneste på vanlig måte, med en autorisasjonskode (`code`).


### Steg 3: hente fullmaktstoken

Når brukeren blir redirecta tilbakt til klient, [henter klienten tokens på vanlig måte](../../docs/idporten/oidc/oidc_protocol_token.html), og bruker dette til å oppdatere / endre eksisterende lokale brukersesjon i egen tjeneste.

Klienten finner opplysninger om valgt representasjonsforhold i claimet `authorization_details`. Claimet er både returnert direkte som del av selve token-responsen, men er også inkludert i selve id_tokenet, for fleksibiltet.


*Eksempel på token-response:*
```
200 OK

{
  "id_token"      : "eyJraWQiO...",

  "access_token"  : "eyJraWQiO..."
  "token_type" : "Bearer",
  "expires_in" : 600

  "scope" : "openid profile",

  "authorization_details" : [ {
    "resource" : "idportenurn:altinn:resource:2480:40",
    "type" : "ansattporten:altinn:service",
    "resource_name" : "Produkter og tjenester fra Brønnøysundregistrene",
    "reportees" : [ {
      "Rights" : [ "Read", "ArchiveDelete", "ArchiveRead" ],
      "Authority" : "iso6523-actorid-upis",
      "ID" : "0192:987464291",
      "Name" : "DIGITALISERINGSDIREKTORATET AVD LEIKANGER"
    } ]
  } ],

  "refresh_token" : "eyJlbmMiO...",
  "refresh_token_expires_in" : 7200,
}
```

*Eksempel på id_token med representasjons-informasjon: *
```
{
  "sub" : "z9RuQiLefXmJOBnywa_c75YQMH05nDsHjw0RFzuJC8M",
  "amr" : [ "TestID" ],
  "iss" : "https://test.ansattporten.no",
  "pid" : "45840375084",
...
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
  } ]
}

```

Merk at bruken av `authorization_details` inne i et id_token ikke er beskrevet i RAR-spesifikasjonen. Klienten skal fortrinnsvis bruke token-responsen til å utlede hvilke rettigheter sluttbruker gav til klienten. Vi har valgt å inkludere det i id_token fordi vi tror det kanskjer er litt lettere for kundene våre.  Det er også enklere å forklare at "fullmaktsinformasjon finner du i authorization_details-feltet, og det finner du i id_token dersom du lager en tjeneste eller du finner det i access_token dersom du tilbyr et api"



# Om scopes, rar og sesjoner

Den sentrale SSO-sesjonen i ID-porten er upåvirket av fullmaktspålogging.  RAR-elementet som blir lagt ved fullmaktspålogginga gjelder kun for denne ene autorisasjonsforespørselen.  Sentral SSO-sesjon blir altså ikke endret til å være en "på-vegne-av-sesjon".  Dersom innlogget bruker forsøker å sende en ny fullmaktspålogging, enten det er til en ny tjeneste, eller til samme tjeneste, vil hen bli vist fullmaktsvelger på ny.

Videre så er det ingen sammenheng eller teknisk avhengighet mellom [scopes i brukerstyrt datadeling](oidc_auth_oauth2.html) og RAR.   


 Teknisk er fullmaktspålogginga (RAR-elementene) bare en beriket innlogging, og ihht oauth2-standardene er mekanismene for scopes og RAR valgfri "tilleggsinformasjon".
