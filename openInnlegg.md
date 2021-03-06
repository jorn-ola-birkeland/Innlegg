
I et tidligere [blogginnlegg](http://open.bekk.no/jakten-pa-fem-tusen-skatteberegninger-i-sekundet) snakket vi om hvordan skatteetaten er i ferd med å utvikle en ny plattform for forskudd- og skattebergninger, som vil erstatte dagens løsning på stormaskin. Målet med den nye plattform er at den har en arkitektur som er fleksibel, skalerbar, robust, og ikke minst fremtidsrettet. En indikator på at denne løsningen er fremtidsrettet, er hvis den enkelt kan flyttes over til kommersielle skyleverandører. 

For å beregne skatten til alle skatteyterne i Norge kreves det bare to ting, *skattegrunnlag* og *skatteplikt*. *Skattegrunnlag* inneholder verdiene av alt du eier, tjener, skylder og har av avgifter og fradrag, mens *skatteplikt* er relevant informasjon om deg, som alder, bosted og familieforhold. Stegene for skatteberegning er som følger:
1. Hent *skattegrunnlag* og *skatteplikt*
2. Kall skattebergnerfunksjon med de to dokumentene
3. Lagre resultat

Ved å kjøre skatteberegningsfunksjonen for alle skatteytere i Norge vil vi kunne finne hvor mye hver enkelt skal betale i skatt. I teorien vil ikke en slik omfattende beregning kjøres ofte, men i praksis er det interessant å kjøre den mange ganger i forbindelse med testing, analyser og rapportering. Det er dermed kritisk at en slik beregning er rask. 

Løsningen er bygd med Amazon Web Services (AWS). AWS har lang fartstid og stiller med en rekke tjenester. Vår arkitektur tar i bruk følgende tjenester. 
- DynamoDB for datalagring
- Lambda for å kalle beregningsfunksjonen
- Kinesis som meldingstjeneste
- Key Manangement Service (KMS) for nøkkelhåndtering

# AWS-arkitektur

DynamoDB er en moderne NoSQL database som støtter et fleksibelt antall attributter, enkle og sammensatte primærnøkler og muligheten for å skrive og lese i grupper. DynamoDB har også støtte for triggere, som kaller en lambda-funksjon for endring i databasen. Lambda kjører et stykke kode når den mottar en hendelse, i vårt tilfelle, en skatteberegningsfunksjon. 

Kinesis ble brukt som hendelsekilde for Lambda. Kinesis jobber med datastrømmer (*stream*), som kan deles opp i delstrømmer (*shards*). Dokumentasjonen sier at Lambda vil starte like mange samtidige eksekveringer som det er delstrømmer i Kinesis-strømmen og vi har en tilsynelatende enkel skaleringsmekanisme.

![Kinesis shards og lambda][kinesis-lambda]

Hendelsesforløpet er som følger: 

1. Klienten laster opp *skattegrunnlag* og *skatteplikt* til DynamoDB.
2. Klienten mater Kinesis med fødselsnummer og fordeler de på delstrømmer.
3. Lambda-funksjonen mottar hendelser fra Kinesis-strømmer og henter *skattegrunnlag* og *skatteplikt* fra DynamoDB, beregner skatt og skriver resultat til DynamoDB. 

![AWS-arkitektur][AWS-arkitektur]

# Konsept

For å gjøre det mulig for skatteetaten å bruke en serverless løsing for skatteberegninger må skattedataene være tilstrekkelig sikret. For å gjøre skatteberegningsløsningen sikker bestemte vi oss for å kryptere de to tabellene, skatteplikt og skattegrunnlag på klientsiden før de lastes opp til DynamoDB. Dette burde gjøre opplasting og lagring av dataene tilstrekkelig sikkert. Lambda-funksjonen henter dokumenter på samme måte som tidligere. Forskjellen er at for å kunne gjøre beregninger må de nå dekrypteres. Når skattebergningen er fulført krypteres resultatet før det sendes tilbake til databasen. 

Dette var i enkle trekk den slagplanen vi la etter å blitt introdusert til prosjektet. Nå måtte vi finne ut hvordan dette kunne la seg gjøre. Det første vi trengte var en løsning for å håndtere nøkler og kryptering i AWS. AWS har en tjeneste for nøkkelhåndtering kalt Key Management Service (KMS).  Man kan opprette nøkler, kalt Customer Master Key (CMK), og sette hvilke personer og tjenester som skal ha tilgang til dem. Nøklene slipper aldri ut av KMS, for å bruke nøklene i kryptografiske operasjoner må data sendes til KMS og bli kryptert der. 

# Kryptering i KMS
For å komme i gang med KMS implementerte vi noen enkle tester for å utforske funksjonaliteten den tilbød. Vi opprettet en CMK og brukte den til å kryptere dokumentene i *skattegrunnlag* og *skatteplikt*. Dette ble gjort ved å sende en encryption request til KMS med klarteksten lagt ved for så å få tilbake en kryptert versjon. 

```java
String keyId = "dummy-key";
ByteBuffer plaintext = ByteBuffer.wrap(new byte[]{1,2,3,4,5,6,7,8,9,0});
EncryptRequest req = new EncryptRequest();
req.withKeyId(keyId).withPlaintext(plaintext);
ByteBuffer ciphertext = kms.encrypt(req).getCiphertextBlob();
```

![krypterer med customer master key][server-side-kms]
I testen vår prøvde vi å laste opp 10 000 dokumenter til både *skattegrunnlag* og *skatteplikt*. Det første vi oppdaget var at KMS ga feilmelding om at tjenesten bare godtar 100 forespørsler i sekundet. Et annet problem med denne løsningen var at for å kryptere et dokument måtte man sende det til KMS for så å få det krypterte dokumentet som svar. En slik kryptering tok i gjennomsnitt XXX ms. 

# Envelope Encryption

I KMS-dokumentasjon leste vi om Envelope Encryption (EE). EE går i korte trekk ut på følgende: 

**Kryptering:** 
- Bruk en unik nøkkel for å kryptere dokumenter. 
- Krypter den unike nøkkelen med hovednøkkel (CMK)
- Lagre det krypterte dokumentet sammen med den krypterte nøkkelen. 

**Dekryptering: **
- Hent dokumentet man skal dekryptere. 
- Bruk hovednøkkelen til å dekryptere den unike nøkkelen.
- Dekrypter dokumentet med den unike nøkkelen. 


I KMS realiserte vi dette på følgende måte. Send en forespørsel til KMS for å generere en unik datanøkkel, som returnerer en datanøkkel og en kryptert versjon av denne. 

```
// Generere ny datanøkkel
String keyId = "dummy-key";
AWSKMSClient kms = new AWSKMSClient();
GenerateDataKeyResult keyResult = kms.generateDataKey(
       new GenerateDataKeyRequest()
               .withKeySpec("AES_128")
               .withKeyId(keyId));
```

![hent-datakey-kms][hent-datakey-kms]


Datanøkkelen blir brukt til kryptering av dokumentet. Den krypterte datanøkkelen og det krypterte dokumentet kan da lagres sammen i DynamoDB. 

![kryptering-og-lagring][kryptering-og-lagring]

For å dekryptere dokumentet henter man først den krypterte nøkkelen fra dokumentet og sender denne til KMS for dekryptering. KMS returnerer den krypterte nøkkelen og man kan dekryptere dokumentet. 

![kryptere-datakey-kms][kryptere-datakey-kms]



Dette medførte at vi måtte ta hånd om kryptering/dekryptering selv. Vi bruker Java’s innebygde [Cipher](https://docs.oracle.com/javase/7/docs/api/javax/crypto/Cipher.html) bibliotek som er basert på Bouncy Castle for kryptering. Vi bruker AES-128, med GCM mode of operation uten padding. Cipher har ikke mulighet til å bruke AES-256, selv om det hadde vært å foretrekke er ikke dette noe stort tap. AES-128 burde tilby tilstrekkelig sikkerhet. Vi bruker samme metode for å kryptere både for opplasting fra klient og i lambda funksjonen. 

```
String keyId = "dummy-key";
byte [] dataKey = keyResult.getCiphertextBlob().array()
ByteBuffer plaintext = ByteBuffer.wrap(new byte[]{1,2,3,4,5,6,7,8,9,0});
Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
cipher.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(dataKey, "AES"));
byte [] encryptedFile = cipher.doFinal(fil)
```

For å begrense antallet kall til KMS bestemte vi oss for å bruke EE for å låse flere dokumenter med den samme master-nøkelen. På den måten trengte vi bare å hente en ny datanøkkel en gang i blant. Dette valget ledet til en diskusjon om hvor mange dokumenter som kan kryptere med samme nøkkel. Vi prøvde oss i begynnelsen med ny nøkkel per 25. dokument, det viste seg å fremteles produsere for mange kall til KMS. 
Vi innså at vi måtte ta høyde for at hver lambda-instans ville multiplisere antallet forespørsler til KMS både ved dekryptering og kryptering. For å redusere antallet forespørsler implementerte vi cashing av datanøkler ved dekryptering. Hver gang det ble sendt en forespørsel om å dekryptere en nøkkel ble både klartekst-nøkkelen og den krypterte nøkkelen lagret. Vi skrudde også antallet nøkler videre ned til å kunne bruke samme nøkkel på 10 prosent av dokumentene. Prosessen gikk fremdeles for sent, vi hadde fremdeles ikke tatt helt hensyn til hvor mange kall som ville bli generert med et høyt antall *shards*. 10 prosents grensen var heller ikke så godt gjennomtenkt da den harde grensen på 100 kall i sekundet lett kunne overskrides dersom vi brukte mer en 100 datanøkler. Etter å ha spurt mer erfarne kryptologer og google bestemte vi oss for å gå for en nøkkel per tabell i DynamoDB. Det virket ikke som om dette ville gå hardt utover sikkerheten så lenge nøklene blir rullert ofte nok. I tillegg, for å spare tid når lambda-funksjonen skal kryptere beregnet skatt, gjenbrukte vi nøklene som ble cashet under dekryptering av *skatteplikt* og *skattegrunnlag*. Dette gjorde vi da vi ikke fant en enkel måte å begrense antallet datanøkler for kryptering av *beregnet skatt* til å være lavere enn antall shards. Det sparte oss også mange kall til til KMS. 

![lambda-og-kms][lambda-og-kms]

Etter å ha gått over til en datanøkkel per tabell endte vi opp med hastigheter som kunne måle seg med dem vi hadde sett før vi implementerte kryptering. Krypteringsoperasjonene gikk som ventet raskt, det er et ganske enkele regnestykker. Det som virkelig krevde tid var dataoverføring, mer spesifikt henting av nøkler fra KMS. Ved å begrense oss til en nøkkel per tabell begrenset vi også kall til KMS slik at det kun var den første dekrypteringsoperasjonen i hver lambda-funksjon som sendte en forespørsel om ny nøkkel, etter det brukte den bare den nøkkelen den hadde fått sist. Når vi i tilleg gjenbrukte nøkler for kryptering av *beregnet skatt* ble antallet kall veldig lavt. 



# Sikkerhet
Ved å begrense nøkler og tilganger til “least privilege” er det mulig å oppnå ett høyt nivå av sikkerhet for løsningen vår. Kravene for dette er blandt annet at nøklene kun skal kunne aksesseres fra nettet til klienten, skatteetaten i vår case, og internt i AWS løsningen. Bortset fra klienten skal kun lambda-instansene kunne lese og skrive til DynamoDB tabellene. Tilgangen til nøklene må også ha policies som begrenser hvem som kan bruke dem og hva de kan brukes til. Dette gjelder hovedsaklig CMS-en. Nøklene må også begrenses slik at ingen menneskelig konto har tilgang til dem, kun lambda prosessene skal ha tilgang til nøklene. I tillegg må bruk av nøklene logges slik at man kan sette opp alarmer som triggres hvis unormal oppførsel oppdages.

Alle disse kravene kan oppnås ved å bruke tjenester og funksjoner i AWS og AWS KMS. For det første, det å begrense hvem som skal ha tilgang til å lese og skrive til DynamoDB settes ved å bruke AWS’ IAM tilganger. For å sette tilganger bruker man AWS’ terminal view på web, her kan man enkelt sette presise begrensninger på hvilke tilganger forskjellige instanser skal ha i systemet. Det samme gjelder for nøkler og tilgang til bruk av nøkler. 

AWS Identity and Access Management (IAM) er en tjeneste som hjelper deg å sikre tilganger til de forskjellige resursene man har i AWS. Man kan legge inn brukere, grupper og roller som man kan legge rettigheter til. Brukerrettigheter var nødvendig for at vi skulle laste opp data til DynamoDB, og roller var nødvendig for å gi Lambda rettigheter mot Kinesis og DynamoDB. På samme måte kan man sette hvilke brukere eller roller som har tilgang til å bruke CMK i KMS. 

![key users][key-users]

CloudTrail er en tjeneste som logger API-kall mot AWS. Denne tjenesten leverer logger som gir informasjon hvem som utførte kall, tidspunkt, IP-adresse, hvilke parameter som var i forespørselen og hva AWS returnerte. CloudTrail lagrer loggene i en S3-database som er mulig å kryptere. Loggene kan også videresendes til CloudWatch, der man kan sette opp forskjellige alarmer. Vi var både innom CloudTrail og CloudWatch, men valgte å ikke gå videre med dette siden oppsettet var komplisert. 


#Konstnader 
Som de fleste tjenester fra AWS så er KMS gratis i bruk opp til 20 000 forespørsler i måneden, deretter koster hver 10 000 forespørsler 0.03$. 

Hvis vi bruker 100 shards vil regnestykker bli følgende: 

![kostnader-kms][kostnader-kms]
Prosessen fra å laste opp kryptert data til DynamoDB, kjøre beregninger på Lambda og laste ned data og dekryptere den vil innebære 204 kall mot KMS. Det vil si at vi kan kjøre den 98 ganger i månenden gratis. Påfølgende kjøring vil enkeltvis koste 0,000612$. 



[kinesis-lambda]:https://bekkopen.blob.core.windows.net/attachments/1b4f1116-3702-4002-8f93-88d61d906993
[AWS-arkitektur]:https://bekkopen.blob.core.windows.net/attachments/b9a2bade-cdda-458e-92ca-8e1a9d857b19
[server-side-kms]:https://bekkopen.blob.core.windows.net/attachments/6a147708-e045-4c49-a4a3-9b3eb06d5503
[hent-datakey-kms]:https://bekkopen.blob.core.windows.net/attachments/0e3f67b7-3559-45a3-be1c-a60b26b82e4c
[kryptering-og-lagring]:https://bekkopen.blob.core.windows.net/attachments/b59c84b6-75e5-4e3d-98c6-6dbf01a689c8
[kryptere-datakey-kms]:https://bekkopen.blob.core.windows.net/attachments/0cb84f4f-cc07-4652-ad45-0e6348d2e6cc
[key-users]: https://bekkopen.blob.core.windows.net/attachments/56de4772-4a07-497c-a60c-4f333ea7a02d
[lambda-og-kms]:https://bekkopen.blob.core.windows.net/attachments/c9e9908f-910a-4635-8ff6-b9e6c6c41a6a
[kostnader-kms]:https://bekkopen.blob.core.windows.net/attachments/9fcb0cfc-10ab-4af7-a4d7-0f0003f0cf33
