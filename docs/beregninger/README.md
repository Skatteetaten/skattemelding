---
icon: "cloud"
title: "Næringsberegninger"
description: ""
---

# Næringsberegninger

Denne seksjonen beskriver de beregningene som vi gjør på dokumentet med næringsopplysninger. Det er disse beregningene som Skatteetaten kjører
når et dokument med næringsopplysninger blir mottatt. Beregningene blir fra Skatteetatens side benyttet for å sjekke at dokumentet er korrekt utfylt,
og at alle verdiene er i henhold til de beregningene som Skatteetaten har spesifisert. Er det mangler eller avvik i de beregnede verdiene å vil valideringstjensten
bygge opp et valideringsresulat som sier noe om hva som mangler i det beregnede grunnlaget, men ikke nødvendigvis referere til hvilken beregning som
er berørt, den vil kun referere til de beregnede verdiene som har mangler eller avvik. Beregningene vil fremover i teksten bli betegnet som
_Kalkyle_ eller _Kalkyler_.

## Kalkyler

Skatteetaten har etablert en DSL for Kalkylene som henviser til elementer og felt i XML-strukturen. Tanken er at denne skal kunne
brukes som en spesifikasjon av Kalkylene som sluttbrukersystemene må implementere for å kunne sende inn de riktige dataene. Denne DSL'en er laget
som [Kotlin](https://kotlinlang.org/) kode og Skatteetaten bruker selv denne DSL'en som en kjørbar spesifikasjon av Kalkylene. Kalkylene vil derfor
bruke reserverte ord i Kotlin som en del av Kallkylespråket.

Denne spesifikasjonen har navngivning av del konsepter som man må gjøre seg kjent med for å kunne forstå hvordan Kalkylene er bygget opp.

Eksempel på en Kalkyle:

```kotlin
val annenDriftsinntektKalkyle =
            summer forekomsterAv annenDriftsinntekt forVerdi { it.beloep }
```

Denne kalkylen summerer alle feltene med navn `beloep` for alle forekomster av `annenDriftsinntekt`. En forekomst i dokumentet med Næringsopplysninger er elementer
som er av typen `EntitetMedGenerelleEgenskaper` i xsd. Man vil finne igjen elementene fra kalkylen gjennom de navnene som er bruk
gjennom å slå opp i tilhørende XDS for næringsopplysninger.

Alle elementer som har `EntitetMedGenerelleEgenskaper` som en supertype kan refereres gjennom `forekomsterAv` i kalkylen. Den fulle xpath er ikke gjengitt i uttrykket
men alle navn som gjengis på denne måten i en kalkyle kan finnes unikt i pågjeldene dokument og referert xsd slik at full sti ikke benyttes for å
gjøre definisjonen mer kompakt.
Konstruksjonen `forVerdi` i uttrykket over representerer navnet på det feltet i forekomsten som skal summeres. Flere felt kan refereres i kombinasjon, eksempler på dette kommer
lengre ned i beskrivelsen. Referansen `it` er det implisitte navnet på forekomsten som man henter feltet på. Det er et standard navn som kotlin bruke for
en implisitt iterator og kommer til å bli benyttet mange steder i kalkylene.

Feltet `it.beloep` som bli referert under forekomsten `annenDriftsinntekt` kan gjenfinnes i XSD for typen av forekomst som
`annenDriftsinntekt` representerer:

```xml
<xsd:complexType name="AnnenDriftsinntekt">
        <xsd:complexContent>
            <xsd:extension base="EntitetMedGenerelleEgenskaper">
                <xsd:sequence>
                    <xsd:element name="annenDriftsinntektstype" type="AnnenDriftsinntektstypeMedInnkapsling"/>
                    <xsd:element name="beloep" type="BeloepMedSkattemessigeEgenskaper"/>
                </xsd:sequence>
            </xsd:extension>
        </xsd:complexContent>
    </xsd:complexType>
```

Vi ser at feltet `beloep` er definert som en kompleks type: `BeloepMedSkattemessigeEgenskaper`. Selve beløpsfeltet er definert slik i denne typen:

```xml
<xsd:complexType name="BeloepMedSkattemessigeEgenskaper">
        <xsd:sequence>
            <xsd:element name="beloep" type="BeloepMedInnkapsling"/>
        </xsd:sequence>
    </xsd:complexType>

```

og

```xml
<xsd:complexType name="BeloepMedInnkapsling" skatt:begrepsreferanse="https://data.skatteetaten.no/begrep/20b2e146-9fe1-11e5-a9f8-e4115b280940">
        <xsd:sequence>
            <xsd:element name="beloep" type="BeloepMed2Desimaler"/>
        </xsd:sequence>
    </xsd:complexType>
```

Så en gyldig xpath referanse til dette feltet vil være: `beloep/beloep`. Men for typene `BeloepMedSkattemessigeEgenskaper`og `BeloepMedSkattemessigeEgenskaperOgOverstyring`
så har vi etablert en snarveiskonvenskjon som sier at dette kan refereres direkte når det dreier seg om beløpsverdien, og da
altså på formen `it.beloep` for eksempelet over.

### Kombinasjon av kalkyler

Kalkylene kan kombineres, der man ønsker å kombinere summer fra forskjellige kalkyler. I eksempelet under så kombinerer vi to kalkyler for å returnere en tredje verdi:

```kotlin
    val sumDriftsinntekterKalkyle: Kalkyle =
            (salgsinntekterKalkyle + annenDriftsinntektKalkyle) verdiSom sumDriftsinntekt
```

Denne kalkylen vil returnere en ny sum for driftsinntekt som en verdi navngitt `sumDriftinntekt`. Dette er en enkeltstående verdi som er
global i dokumentet. På samme måte som `annenDriftsinntekt` så kan dette feltet finnes i XSD her:

```xml
<xsd:element minOccurs="0" name="sumDriftsinntekt" type="BeloepMedSkattemessigeEgenskaperOgOverstyring"/>
```

Typen `BeloepMedSkattemessigeEgenskaperOgOverstyring` representerer et beløp i dokumentet og kan benyttes for å representere en verdi. Hvis dette feltet
hadde navngitt en forekomst så vill det ha vært en galt spesifisert kallyle. `verdiSom` kan kun benyttes som en global verdi for et enkeltstående felt.

### Kalkyler og beregnede forekomster

Mange av kalkylene skal beregne verdier som ligger inne som forekomster av typen `EntitetMedGenerelleEgenskaper`. Det betyr at kalkylen
returnerer en forekomst med gitte verdier og også en definert id. Det vil derfor, for noen av forekomstene i dokumentet med næringsopplysninger
være krav til hvordan en id er utfylt. Alle elementene av typen `EntitetMedGenerelleEgenskaper` har id som et påkrevd felt, og denne
implementasjonsguiden vil definere regler for disse id'en. Dette er viktig fordi id brukes som utgangspunkt for å sammenlikne innkommende verdi med det
som Skatteetaten selv beregner. To forekomster med ulik id vil i henhold til denne definisjonen være ulike. Disse reglene er ikke spesifisert enda.

Eksempel på kalkyle som returnerer en forekomst:

```kotlin
internal val annenDriftsinntektstypeInntektKalkyle = summer forekomsterAv gevinstOgTapskonto forVerdi { it.inntektFraGevinstOgTapskonto } verdiSom NyForekomst(forekomststTypeSpesifikasjon = annenDriftsinntekt, idVerdi = "3890", feltKoordinat = annenDriftsinntekt.beloep, feltMedFasteVerdier =
    {
        listOf(
                FeltOgVerdi(it.type, "3890")
        )
    }
    )
```

Denne kalkylen summerer verdien hentet fra beløpet i feltet `inntektFraGevinstOgTapskonto` for alle forekomster av `gevinstOgTapskonto`. Dette returneres gjennom konstruksjonen
`verdiSom NyForekomst`. Den nye forekomsten settes opp med de ønskede feltene, med angitte verdier for de navngitte feltene. Denne konstruksjonen vil etablere en forekomst som ser slik ut:

```xml
<annenDriftsinntekt>
                <id>3890</id>
                <annenDriftsinntektstype>
                    <annenDriftsinntektstype>3890</annenDriftsinntektstype>
                </annenDriftsinntektstype>
                <beloep>
                    <beloep>
                        <beloep>5000</beloep>
                    </beloep>
                </beloep>
            </annenDriftsinntekt>
```

gitt at summen for denne kalkylen returnerer 5000.

### Kalkyler med flere verdier i en forekomst

```kotlin
private val forekomsterGevinstOgTapskonto = itererForekomster forekomsterAv gevinstOgTapskonto
private val inntektFraGevinstOgTapskontoKalkyle = forekomsterGevinstOgTapskonto forVerdier (
            listOf(
                    { f -> f.grunnlagForAaretsInntektsOgFradragsfoering.der(derVerdiErStoerreEnn(15000)) * f.satsForInntektsfoeringOgInntektsfradrag.prosent() somFelt gevinstOgTapskonto.inntektFraGevinstOgTapskonto },
                    { f -> f.grunnlagForAaretsInntektsOgFradragsfoering.der(derVerdiErMellom(0, 15000)) somFelt gevinstOgTapskonto.inntektFraGevinstOgTapskonto },
                    { f -> f.grunnlagForAaretsInntektsOgFradragsfoering.der(derVerdiErMellom(-15000, -1)) somFelt gevinstOgTapskonto.inntektsfradragFraGevinstOgTapskonto.abs() },
                    { f -> f.grunnlagForAaretsInntektsOgFradragsfoering.der(derVerdiErMindreEnn(-15000)) * f.satsForInntektsfoeringOgInntektsfradrag.prosent() somFelt gevinstOgTapskonto.inntektsfradragFraGevinstOgTapskonto.abs() }
            ))
```

Over så har vi referert `forekomsterAvGevinstOgTapskonto` som en egen variabel fordi vi skal gjøre flere ting med disse forekomstene. Dette er en forenkling som
brukes i flere av kalkylene. Kalkylen `inntektFraGevinstOgTapskontoKalkyle` henvser til flere komvinasjoner av uttrykk og felt som skal returners gitt ulike betingelser. Alle
betingelsene i listen blir evaluert opp mot verdiene i dokumentet. Det første uttrykket i listen over vil evaluere fra venstre mot høyre gitt at det
finnes felt med angitte egenskaper på vedrdien i feltet som refereres. `f.grunnlagForAaretsInntektsOgFradragsfoering.der(derVerdiErStoerreEnn(15000))` krever at feltet `grunnlagForAaretsInntektsOgFradragsfoering` har en verdi som er
større enn `15000`, og gitt at det er sant så multipliserer vi denne verdien med feltet `satsForInntektsfoeringOgInntektsfradrag` Denne satsen er angitt
i prosent, feks 20, og vi legger derfor på `.prosent()` for å angi at dette er oppgitt i prosents lik at verdien blir ganget med 0.20 ikke 20. Gitt
at begge feltene finnes i forekomsten. Den siste delen av uttrykket `somFelt gevinstOgTapskonto.inntektFraGevinstOgTapskonto` vil legge til dette beregnede
feltet som et felt i forekomsten med den beregnede verdien og da i feltet med navn `inntektFraGevinstOgTapskonto`.

Det samme er gjeldende for de andre uttrykkene i listen. Andre begreper for felt som returneres er: `abs()` for å returnere en absoluttverdi for et felt.

### Rekkefølge på kalkyler

Noen av kalkylene bruker felt som er beregnet i andre kalkyler. Det er derfor viktig at kalkylene kjører i riktig rekkefølge. Vi spesifiserer rekkefølgen slik:

```kotlin
    internal val tre = Kalkyletre(
            driftskostnadstypeKalkyle,
            annenDriftsinntektstypeInntektKalkyle,
            annenDriftsinntektstypeFradragKalkyle,
            sumDriftsinntekterKalkyle,
            sumDriftskostnaderKalkyle,
            sumFinansinntektKalkyle,
            sumFinanskostnadKalkyle,
            sumEkstraordinaerePosterKalkyle,
            sumSkattekostnadKalkyle,
            aarsresultatKalkyle)
```

Kalkylene i dette kalkyletreet kjører i denne rekkefølgen. Det betyr at eventuelt beregnede vedier fra `driftskostnadstypeKalkyle` er tilgjengelige
for kalkylen `aarsresultatKalkyle` når denne kjører. Dette fordi kalkyler som kjøres tidlig kan _beregne verdier som brukes av andre kalkyler_. Det jobbes med å
lage dette slik at rekkefølgen kan utledes og ikke spesifiseres slik.

For at listene av kalkyler ikke skal bli for store så er de organisert i ulike områder, som feks resultatregnskapet. De ulike områdene har også avhengigheter. Feks så må
kalkylene for GevinstOgTapskonto kjøre før kalkylene for Resultatregnskapet. Dette settes opp slik:

```kotlin
object DefaultKalkyletre : Kalkyletre(
        GevinstOgTapskontoKalkyler,
        SpesifikasjonAvBalanse,
        Resultatregnskapet
)
```

Dette definerer rekkefølgen på kallkyleområdene innbyrdes.

# Kalkyler

En oversikt over kalkylene som er definert så langt finnes i [Oversikten over kalkyler](https://github.com/Skatteetaten/skattemeldingen/tree/master/docs/documentation/beregninger/kotlin)
Det er ikke laget DSL for kalkylen for personinntekt enda. Dette kommer senere. Kalkylen for personinntekt er beskrevet nedenfor.

## Beregning av personinntekt

Beregning av de ulike feltene av personinntekt er beskrevet i seksjonenen nedenfor

### sumSkjermingsgrunnlagFørGjeldsfradrag

Hvis selskapstype = enkeltpersonforetak

    For samme identifikatorForFordelingAvNæringsinntektOgPersoninntekt:

    (spesifikasjonAvSkjermingsgrunnlag/inngaaendeVerdi + spesifikasjonAvSkjermingsgrunnlag/utgaaendeVerdi) / 2

    for alle forekomster av skjermingsgrunnlagstype

    ( saldogruppeA, saldogruppeB, saldogruppeC, saldogruppeC2, saldogruppeD, saldogruppeE, saldogruppeF, saldogruppeG, saldogruppeH, saldogruppeI, saldogruppeJ, lineaertavskrevetAnleggsmiddel, ikkeAvskrivbartAnleggsmiddel, ervervetImmatriellRettighet, aktivertForskningsOgUtvklingskostnad, varelager, kundefordring)

    - (spesifikasjonAvSkjermingsgrunnlag/inngaaendeVerdi

    + spesifikasjonAvSkjermingsgrunnlag/utgaaendeVerdi)
      /2

    for alle forekomster av skjermingsgrunnlagstype = leverandoergjeld
    = sumSkjermingsgrunnlagFørGjeldsfradrag

    Sum skjermingsgrunnlag før gjeldsfradrag kan ikke bli negativt, dvs hvis leverandørgjeld er høyere enn sum av eiendelene skal skjermingsgrunnlag før gjeldsfradrag settes til null:

    sum av

    (spesifikasjonAvSkjermingsgrunnlag/inngaaendeVerdi

    + spesifikasjonAvSkjermingsgrunnlag/utgaaendeVerdi)
      /2

    for alle forekomster av skjermingsgrunnlagstype = leverandoergjeld >=
    sum av
    (spesifikasjonAvSkjermingsgrunnlag/inngaaendeVerdi

    + spesifikasjonAvSkjermingsgrunnlag/utgaaendeVerdi)
      /2

    for alle forekomster av skjermingsgrunnlagstyper
    (= saldogruppeA, saldogruppeB, saldogruppeC, saldogruppeC2, saldogruppeD, saldogruppeE, saldogruppeF, saldogruppeG, saldogruppeH, saldogruppeI, saldogruppeJ, lineaertavskrevetAnleggsmiddel, ikkeAvskrivbartAnleggsmiddel, ervervetImmatriellRettighet, aktivertForskningsOgUtvklingskostnad, varelager, kundefordring)
    skal sumSkjermingsgrunnlagFørGjeldsfradrag = 0

### sumSkjermingsgrunnlagEtterGjeldsfradrag

    sumSkjermingsgrunnlagFørGjeldsfradrag

    - (spesifikasjonAvSkjermingsgrunnlag/inngaaendeVerdi

    + spesifikasjonAvSkjermingsgrunnlag/utgaaendeVerdi)
      / 2

    for alle forekomster av skjermingsgrunnlagstype = foretaksgjeld
    = sumSkjermingsgrunnlagEtterGjeldsfradrag

Skjermingsgrunnlag etter gjeldsfradrag kan ikke være negativt, dvs at foretaksgjeld ikke kan være høyere enn skjermingsgrunnlag før gjeldsfradrag. I så fall skal skjermingsgrunnlag etter gjeldsfradrag settes til null:

    Hvis
    sum av (spesifikasjonAvSkjermingsgrunnlag/inngaaendeVerdi

    + spesifikasjonAvSkjermingsgrunnlag/utgaaendeVerdi)
      / 2

    for alle forekomster av skjermingsgrunnlagstype = foretaksgjeld >=
    sumSkjermingsgrunnlagFørGjeldsfradrag
    skal sumSkjermingsgrunnlagEtterGjeldsfradrag = 0

### skjermingsfradrag

    sumSkjermingsgrunnlagEtterGjeldsfradrag

    * antallMaanederDrevetIAar / 12
    * skjermingsrente
    = skjermingsfradrag

### åretsBeregnedePersoninntektFørFordelingOgSamordning

Hvis selskapstype = enkeltpersonforetak

    For samme
    identifikatorForFordelingAvNæringsinntektOgPersoninntekt:
    fordelingAvNaeringsinntekt/skattemessigResultatForNaeringEtterKorreksjon

    - rentekostnadPaaForetaksgjeld
    - kapitalinntekt

    + kapitalkostnad

    - reduksjonsbeloepForLeidEiendomMotInnskudd
    - gevinstVedRealisasjonAvAlminneligGaardsbrukEllerSkogbruk
    -  skjermingsfradrag
     åretsBeregnedePersoninntektFørFordelingOgSamordning

# Skattemeldingsberegninger

Skatteetaten har følgende beregning av verdier som er definert innenfor egenfastsettingsprinsippet i skatteforvaltningsloven:

- formuesverdi for annet privat innbo og løsøre
- årets sparebeløp knyttet til boligsparing for ungdom (norge og/eller utland)
- fradragsberettiget beløp som følge av betalt fagforeningskontingent
- formuesverdi for fast eiendom
- nettoinntekt eller underskudd som følge av utleie av fast eiendom
- foreldrefradrag
- formuesverdi for fritidsbåt med salgsverdi over salgsverdigrense
- fradrag for gaver til frivillige organisasjoner (norge og/eller utland)
- fradrag for individuell sparing til pensjon (ny og/eller gammel ordning)
- formuesverdi for kjøretøy
- fradrag for kostnader knyttet til arbeid og annen inntekt
- nettoinntekter som følge av krav om den alternative fordelingsmetoden
- nettoinntekter som følge av krav om kreditfradag
- skattepliktig fordel ved elektronisk kommunikasjon som naturalytelse fra arbeidsgiver
- skattepliktig lønn til barn under 13 år
- overskudd av ulike utgiftsgodtjørelser fra arbeidsgiver
- samlet pensjonsinntekt og øvrige grunnlag for skattefradrag for pensjonsinntekt
- reisefradrag
- merkostnader ved arbeidsopphold utenfor hjemmet (pendleropphold, opphold under yrkes- og tjenestereise, opphold under kjøreoppdrag)
- standardfradrag for utenlandske sjøfolk og sokkelarbeidere
- særfradrag for enslig forsørger
- særfradrag for lettere nedsatt ervervsevne
- særfradrag for store sykdomskostnader
- særskilt fradrag for sjøfolk
- særskilt inntektsfradrag i barns inntekt
- fradrag for pengetilskudd til vitenskapelig forskning og yrkesopplæring
- særskilt fradrag i næringsinntekt for jordbruk mv.
- særskilt fradrag for reindrift
- særskilt fradrag for skiferproduksjon
- særskilt fradrag for fiskere og fangstfolk
- lønnsomhetsvurdering minstefradrag vs. fradrag for kostnader knyttet til arbeid
- skattemessig avkorting av fradragsberettiget beløp ved kombinasjonsforhold særfradrag for lettere nedsatt ervervsevne vs. særfradrag for store sykdomskostnader
- skattemessig avkorting av fradragsberettiget beløp ved kombinasjonsforhold særskilt fradrag for sjøfolk og særskilt fradrag for fiskere og fangstfolk
- skattemessig avkorting av fradragsberettiget beløp ved kombinasjonsforhold diverse fradrag og krav om standardfradrag

# Validering av id på næringsopplysninger

For at vi skal kunne foreta sammenlikninger og vise differanse mellom inngående og beregnede - feltverdier på en
enhetlig måte så må vi kunne gjenfinne elementer i spesifikasjonen av næringsopplysnigner slik at vi kan sammenlikne 1:1.
Dette gjøres gjennom at vi har etablert et regelsett på at id på samme nivå må være lokalt unike. Id - attributtet finnes på flere typer i næringsspesifikasjonen og er da et påkrevd felt.
Det er et krav at id-felt på samme nivå må være unike, da typisk i en liste av flere elementer. De trnger ikke
å være unike på tvers av lister.

Det er flere av elementene der vi krever at id - verdien er den samme som type - feltet, og verdiene skal være det samme som den
tilhørende aksepterte typen for et gitt element. Verdiene hentes fra kodelisten fra elementet underkodeliste
for forekomsten med samme navn hentet. Kodelisten det hentes fra er 2020_resultatregnskapOgBalanse.xml

Dette gjelder følgende forekomster og type - felt:

- andreDriftsinntekter.annenDriftsinntekt.type underkodeliste:annenDriftsinntekt
- andreDriftskostnader.annenDriftskostnad.type underkodeliste:annenDriftskostnad
- salgsinntekter.salgsinntekt.type underkodeliste:salgsinntekt
- finansinntekter.finansinntekt.type underkodeliste:finansinntekt
- finanskostnader.finanskostnad.type finanskostnad
- varekostnader.varekostnad.type underkodeliste:varekostnad
- skattekostnader.skattekostnad.type underkodeliste:skattekostnad
- balanseverdiForAnleggsmidler.balanseverdiForAnleggsmiddel.anleggsmiddeltype underkodeliste:balanseverdiForAnleggsmiddel
- balanseverdiForOmloepsmidler.balanseverdiForOmloepsmiddel.omloepsmiddeltype underkodeliste:balanseverdiForOmloepsmiddel
- allKortsiktigGjeld.kortsiktigGjeld.kortsiktigGjeldtype underkodeliste:kortsiktigGjeld
- allLangsiktigGjeld.langsiktigGjeld.langsiktigGjeldtype underkodeliste:langsiktigGjeld
- allEgenkapital.egenkapital.egenkapitaltype underkodeliste:egenkapital

Det finnes også andre områder som krever fast id, med verdi 1, der det kun skal
være en forekomst. Da dette gjenspeiler elementer der beregningen også skal generere
opp en forekomst med verdi, så har vi sagt at denne må ha verdien 1.
Dette gjelder følgende elementer i næringsspesifikasjonen:

- andreDriftsinntekter,
- andreDriftskostnader,
- salgsinntekter,
- varekostnader,
- finanskostnader,
- finansinntekter,
- skattekostnader,
- skattekostnader,
- balanseverdiForOmloepsmidler,
- balanseverdiForAnleggsmidler,
- allKortsiktigGjeld,
- allLangsiktigGjeld,
- allEgenkapital,
- samletGjeldOgFormuesobjekter,

Dokumenter som ikke følger dette regelsettet blir avist med en eller flere av disse kodene:

- idAvvikerFraKrav

# Begreper

- DSL, Domain Specific Language, `https://en.wikipedia.org/wiki/Domain-specific_language`
- Forekomst, et element i xml dokumentet som har en identifikator på et element som kan forekommer flere ganger. Id-feltet skiller de ulike
  repeterende feltene fra hverandre gjennom at to forekomser må ha _ulik id_
- Beløp med egenskaper. Er et element som representerer et beløp, men som kan være knyttet til ulike egenskaper, feks trygdeEllerSkattemessigeUnntak eller valuta.
  Et slik felt refereres til som er _felt med egenkaper_
- Globalt felt, er et felt som det kun finnnes ett innslag av i xml - dokumentet, feks `sumFinansinntekt`
- Global forekomst, en gruppeing av felt som kun kan forekomme en gang i xml - dokumentet, feks `virksomhet`. Slike forekomster trenger ikke id.
