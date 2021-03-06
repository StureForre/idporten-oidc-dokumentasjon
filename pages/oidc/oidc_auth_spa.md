---
title: Autentisering til SPA'er
description: Bruk av Idporten sin OpenID Connect provider til autentisering til Single  Page Applikasjoner
summary: "Ved innlogging til en SPA, er det anbefalt å bruke implicit flow, siden en SPA ikke kan beskytte klient-hemmelighet/virksomhetssertifikater på en trygg måte."
permalink: oidc_auth_spa.html

layout: page
sidebar: oidc
---

## Overordna beskrivelse av bruksområdet

Single-page applikasjoner (SPA) har økende popularitet. Disse skiller seg fra tradisjonelle nettjenester ved at SPAen er realisert som en ren javascript-applikasjon i brukers browser, kontra tradisjonelle nettjtenester der en sentral applikasjonserver generer HTML som blir vist i browseren.

En utfordring med SPAer er at de ikke klarer å beskytte klient-hemmeligheten (evt. virksomhetssertifikatets privatnøkkel) siden hele klienten lever i brukers nettleser. SPAer er altså det som i Oauth2-verdenen kalles **public klienter**. For slike klienter er det anbefalt å bruke _implicit flow_, og ID-portens OpenID Connect provider tilbyr slik funksjonalitet.

Sidan det ikke er klient-autentisering ved bruk av implicit flow, kan i prinsippet hvem som helst lage en falsk SPA som bruker samme client-id, og da fremstår som tjenesteeiers tjeneste ovenfor ID-porten. Sikkerheten ved bruk av implisitt-flyten er derfor helt avhengig av at brukeren er tilstede og forstår om han har blitt lurt til å bruke en falsk applikasjon.

Man bør også vurdere om SPAen i det hele tatt har behov for å et id_token fra ID-porten.  En SPA vil alltid måtte aksessere bakenforliggende APIer, enten hos tjenesteeier selv eller hos andre, og kanskje kan man klare seg med "ren" Oauth2, siden autorisasjonsserveren ved utstedelse av access_token uansett vil gjennomføre en autentisering av brukeren.

## Anbefalinger / krav til bruk av SPAer

Trusselbildet er forskjellig ved bruk av implisittflyt kontra tjenester som bruker ordinær autorisasjonskodeflyt.  Siden access_token blir eksporert ut i brukers browser, er det øka risiko for at token lettere kan komme på avveie eller byttes ut/manipuleres.


Tjenesteeiere må:
 * Lese [Oauth2 threat model](https://tools.ietf.org/html/rfc6819) og følge anbefalingene i denne
 * Gjennomføre en risikovurdering av de dataene som blir eksport av APIet og vurdere om de sikringsmekanismer som implisittflyten tilbyr,gir tilstrekkelig beskyttelse.


## Beskrivelse av implisitt-flyten

Implisittflyt skiller seg fra autorisasjonskode-flyten ved at token utleveres direkte fra autorisasjonsendepunktet.  
Sjå [Oauth2 kapittel 4.2](https://tools.ietf.org/html/rfc6749#section-4.2) og Se [OIDC kapittel  3.2.2.1](http://openid.net/specs/openid-connect-core-1_0.html#ImplicitAuthRequest)


<div class="mermaid">
sequenceDiagram
  Sluttbruker  ->> SPA: Klikker login-knapp
  SPA ->> Browser : Redirect med autentiseringsforespørsel
  Browser ->> OpenID Provider: følg redirect...
  note over Sluttbruker,OpenID Provider: Sluttbruker autentiserer seg (og evt. samtykker til førespurte scopes)
  OpenID Provider ->> Browser: Redirect med id_token (og evt. access_token)
  Browser ->> SPA: uthenting av tokens
  note over Sluttbruker,SPA: Innlogget i tjenesten
  opt Eventuell videre bruk
    SPA ->> Eventuelle APIer: API-operasjon med access_token
  end
</div>

* Klienten sender en **autentiseringsforespørsel** til OpenID Connect provideren med _response_type_ lik `id_token` eller `id_token token` for å få utlevere id_token med OIDC.  Fortrinnsvis kan kun `token` benyttes for å bruke Oauth2 istedenfor, gjerne kombinert med egne scopes.  Bruk av _nonce_ er påkrevd.   
* Providerens **autorisasjonsendepunkt** validerer forespørselen (f.eks. gyldig tjeneste og gyldig redirect_uri tilbake til tjenesten).
* Brukeren gjennomfører **innlogging i provideren**
* Provideren redirect'er brukeren tilbake til tjenesten. Redirect url'en inneholder **id_token** og/eller et **access_token** direkte som et URI fragment.
* Brukeren er nå autentisert for tjenesten og ønsket handling kan utføres




## Autentiseringsforespørsel til autorisasjons-endepunktet

Denne er tilsvarende som ved [autorisasjonskodeflyten](oidc_auth_codeflow.html), med følgende forskjeller:

| Parameter  | Verdi |
| --- | --- |
| response_type | Må vere `id_token` eller `id_token token` for bruk av OIDC.<br/>Må vere `token` for bruk av Oauth2.|
| nonce | Påkrevd.|


Etter at brukeren har logget inn vil det sendes en redirect url tilbake til klienten. Denne url'en vil inneholde tokens direkte som et URI fragment.


### Eksempel på forespørsel
Eksempelet viser ikke OIDC, men en "ren" oauth2 implisitt flyt uten utlevering av id_token.


```
https://oidc-ver2.difi.no/idporten-oidc-provider/authorize?
  scope=profile&
  acr_values=Level3&
  client_id=test_rp_ver2&
  redirect_uri=https://eid-exttest.difi.no/idporten-oidc-client/authorize/response&
  response_type=token&
  nonce=nnnnnnnn&
  ui_locales=nb

```

### Eksempel på respons:
Her mottas kun access_token.
```

GET
http://eid-exttest.difi.no:80/idporten-oidc-client/authorize/response#

access_token=33uOLzudBXEHFkhZePVsEzX3OTL24Jg9f8JAF7RU_so=&
token_type=Bearer&
expires_in=599
```

### Eksempel på å hente fødselsnummer
API (ressursserver) som mottar access_token, kan kalle ID-portens /tokeninfo-endepunkt for å hente fødselsnummeret:

```
POST https://oidc-ver2.difi.no/idporten-oidc-provider/tokeninfo HTTP/1.1
Accept: application/json
Content-Type: application/x-www-form-urlencoded

token=33uOLzudBXEHFkhZePVsEzX3OTL24Jg9f8JAF7RU_so%3D
```
response
```
{
  "active" : true,
  "token_type" : "Bearer",
  "expires_in" : 570,
  "exp" : 1517492113,
  "iat" : 1517491513,
  "sub" : "4CD_nqnUhfff1868ccDjfh4PQs_oL13_PjG6hYYbMEU=",
  "pid" : "23079417653",
  "scope" : "profile",
  "client_id" : "test_rp_ver2",
  "client_orgno" : "991825827"
}
```
## Struktur på Id token

ID-tokenet er identisk som ved bruk av [autorisasjonskode-flyten](oidc_auth_codeflow#idtoken).


## Validering av Id token

Korrekt validering av Id token på klientsiden er kritisk for sikkerheten i løsningen. Tjenesteleverandører som tar i bruk tjenesten må utføre validering i henhold til kapittel *3.1.3.7 - ID Token Validation* i OpenID Connect Core 1.0 spesifikasjonen.




## Andre aspekt

Implicit flow tillater ikke bruk av refresh_token, siden klienten ikke kan beskytte dette på en god måte over lengre tid.

En alternativ løsning til implicit flow er at tjenesteeier setter en minimal  backend-tjeneste imellom ID-porten og SPAen og bruker autorisasjonskode-flyten til innlogging. ID-porten kan da være sikker på at sterk identitet kun bli utlevert til riktig tjenesteeier. id_token vil kun flyte til backend-tjenesten, som da må oversette til en lokal sesjon/tokens mellom SPAen og egen APIer.
