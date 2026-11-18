# AudioLink

Eigen alternatief voor AudioRelay, zonder abonnement.

- **`windows/`** — AudioLinkSender: pakt het geluid van de Windows-laptop (WASAPI loopback),
  zet het om naar 16-bit PCM en stuurt het via UDP over je wifi-netwerk.
- **`android/`** — AudioLink: ontvangt de stream en speelt hem af op de telefoon, of stuurt
  het geluid van de telefoon zelf door naar een ander toestel. De telefoon geeft het geluid
  door aan de Bluetooth-oortjes die er al aan gekoppeld zijn.

Belangrijk verschil met de bestaande apps: **AudioLink vraagt geen audiofocus aan.**
Daardoor stopt de muziek die al op je telefoon speelt niet, maar worden laptopgeluid en
telefoonmuziek door Android gemixt. Je hoort dus beide door dezelfde oortjes.

---

## Hoe het werkt

```
laptop audio ──► WASAPI loopback ──► PCM 16-bit ──► UDP (wifi) ──► AudioTrack ──► Bluetooth-oortjes
                (windows)                                          (android)         (al gekoppeld)
```

- De telefoon kan elke seconde een beacon (broadcast op poort 47810) met zijn naam sturen.
  De Windows-app ziet die beacon, kiest de telefoon en streamt naar poort 47811.
- Die beacon is **uit te zetten** in de app; de laptop verbindt dan op IP (`--ip`).
- Beide kanten gebruiken het protocol uit [PROTOCOL.md](PROTOCOL.md).
- Ongecomprimeerde PCM: 48 kHz stereo 16-bit ≈ 1,5 Mbit/s. Op een normaal thuisnetwerk
  is dat niets; een codec (Opus) is een latere optimalisatie, geen must.

## Wat je nodig hebt

- Windows 10/11 met [.NET 8 SDK](https://dotnet.microsoft.com/download) (of de losse runtime
  als je de kant-en-klare build gebruikt).
- Android 8.0 (API 26) of nieuwer.
- Telefoon en laptop op **hetzelfde wifi-netwerk** (niet "gast"-netwerk: dat isoleert clients).

## Windows installeren

**Aanbevolen: `audiolink/AudioLink-Setup.exe`.** Dubbelklikken en de drie vakjes kiezen:

- **Automatisch starten na het inloggen** — na het inloggen staat AudioLink meteen in het
  systeemvak. Er opent geen venster; het pictogram staat rechts onderin klaar.
- **Snelkoppeling op het bureaublad**
- **AudioLink nu starten**

De installatie komt in `%LOCALAPPDATA%\Programs\AudioLink`. Dat is je eigen profiel, dus er
komt geen beheerdersvraag (UAC) en er is geen wachtwoord nodig. De installer is 132 MB omdat
de complete .NET-runtime erin zit: je hebt op de laptop dus géén .NET-installatie nodig.

### Bijwerken

Draai gewoon een nieuwere `AudioLink-Setup.exe`. De installer ziet dat AudioLink al
geïnstalleerd is en toont **Bijwerken van 1.0.0 naar 1.1.0**;

- een **draaiende** AudioLink (ook als het venster in het systeemvak staat) krijgt een
  verzoek om af te sluiten, bewaart zijn instellingen en gaat netjes weg;
- reageert hij niet, dan wordt hij na een aantal seconden hard gestopt, zodat de bestanden
  vrij zijn;
- daarna worden de bestanden vervangen en start AudioLink weer, in het systeemvak;
- je IP-adres, poort, kwaliteit en buffer blijven bewaard (`%APPDATA%\AudioLink\settings.json`).

Er is dus geen de-installatie nodig: één bestand, dubbelklikken, klaar.

### Verwijderen

Via *Instellingen → Apps → Geïnstalleerde apps → AudioLink (zender) → Verwijderen*, of met:

```powershell
& "$env:LOCALAPPDATA\Programs\AudioLink\AudioLinkSender.exe" --uninstall
```

Dat ruimt de snelkoppelingen, de automatische start en de map op. Je opgeslagen
`settings.json` blijft staan; gooi die map weg als je ook dat kwijt wilt.

### Losse exe (zonder installatie)

**Kant-en-klaar:** `audiolink/AudioLinkSender.exe` (69 MB) is een self-contained build voor
Windows x64 — je hebt dus géén .NET-installatie nodig. Dubbelklikken of in PowerShell:

```powershell
.\AudioLinkSender.exe
```

Deze variant installeert niets en schrijft alleen zijn instellingen weg. Wil je hem
toch automatisch laten starten, gebruik dan de installer (die zet dezelfde registersleutel
als het tray-menu).

Het pictogram zit in de exe, maar staat ook los in `audiolink/AudioLinkSender.ico`
(7 formaten, 16 t/m 256 px) voor als je een bestaande snelkoppeling een eigen icoon wilt
geven: *Eigenschappen → Snelkoppeling → Ander pictogram*.

De exe is niet ondertekend, dus SmartScreen kan "Windows heeft uw pc beschermd" tonen:
klik op *Meer informatie* → *Toch uitvoeren*. De eerste keer vraagt Windows Firewall om
toegang; sta UDP toe voor privénetwerken.

Zelf bouwen kan ook:

```powershell
cd audiolink\windows\AudioLinkSender
dotnet run
```

Of een losse exe:

```powershell
dotnet publish -c Release -r win-x64 --self-contained false -p:PublishSingleFile=true
# resultaat: bin\Release\net8.0-windows\win-x64\publish\AudioLinkSender.exe
```

### Het venster

- **IP-adres telefoon** en **Poort** (standaard 47811) — worden onthouden in
  `%APPDATA%\AudioLink\settings.json`, dus je hoeft ze maar één keer in te vullen.
- **Zoek op netwerk** — luistert 30 seconden naar de beacon en vult adres én toestelnaam in.
- **Kwaliteit** — de bitrate: 1536 kbit/s (48 kHz stereo) tot 256 kbit/s (16 kHz mono).
  Wijzigen tijdens het streamen zet de stream in een paar honderd milliseconden opnieuw op.
- **Buffer** — hoeveel audio de telefoon vooruit buffert: 30 ms voor filmpjes tot 500 ms voor
  muziek op een slechte verbinding. Gaat meteen in, zonder de stream te herstarten.
- **Start / Stop** — start en stopt het opnemen van het bureaubladgeluid.
- **Kruisje** — het venster verdwijnt naar het systeemvak en de stream blijft doorlopen.
  Dubbelklik op het pictogram om het terug te halen; rechtsklik voor *Openen*,
  *Start/Stop streamen*, *Starten met Windows* (vinkje, meteen ingeschakeld of uitgezet) en
  *Afsluiten*. Pas bij *Afsluiten* stopt de zender echt.
- **Statusregel** — laat zien hoeveel pakketten en Mbit/s er weggaan; zo zie je meteen of er
  echt iets verstuurd wordt in plaats van te gokken.
- **Logvak** — capture-formaat, gevonden toestel, fouten.

Commandoregel kan ook:

```
AudioLinkSender                              # venster met de vorige instellingen
AudioLinkSender --ip 192.168.1.50            # vult dit in en start meteen
AudioLinkSender --ip 192.168.1.50 --port 47811
AudioLinkSender --tray                       # meteen in het systeemvak (automatisch starten)
AudioLinkSender --quit                       # een draaiend exemplaar netjes afsluiten
AudioLinkSender --uninstall                  # de geïnstalleerde AudioLink verwijderen
```

`--tray` en `--quit` zijn ook de manier waarop de installer een update doet: hij roept
`--quit` aan en wacht tot het proces weg is. Start je AudioLink twee keer, dan komt het
bestaande venster naar voren in plaats van dat er een tweede exemplaar bijkomt.

**Windows Firewall**: de eerste keer vraagt Windows om toestemming. Sta UDP toe voor
privénetwerken. Gebeurt dat niet, dan komt er geen geluid aan.

## Android-ontvanger bouwen

**Kant-en-klaar:** `audiolink/AudioLink-debug.apk` is al gebouwd met de debug-sleutel van
de build-machine. Zet dat bestand op je telefoon (USB, Google Drive, e-mail) en tik het aan;
je moet dan "Onbekende bronnen" / "Installeren van deze bron toestaan" goedkeuren.

Zelf bouwen: open de map `android/` in Android Studio (Gradle-sync doet de rest) en druk op Run.
Of via de opdrachtregel:

```bash
cd audiolink/android
gradle assembleDebug          # of ./gradlew assembleDebug
# resultaat: app/build/outputs/apk/debug/app-debug.apk
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

## Gebruiken

1. Koppel je Bluetooth-oortjes aan de telefoon en start je muziek (Spotify, YouTube, …).
2. Open **AudioLink** op de telefoon en druk op **Start**. De app toont zijn IP-adres.
3. Start **AudioLinkSender** op de laptop, klik op **Zoek op netwerk** (vult het adres in)
   of typ het adres zelf.
4. Klik op **Start** en speel iets af op de laptop. Je hoort het door de oortjes, samen met
   de telefoonmuziek. De statusregel laat zien dat er pakketten weggaan.
5. **Stop** in het venster, of Stop in de app op de telefoon.

## Verbinden: automatisch of op IP

Er zijn twee manieren, en de keuze staat los van de audio zelf (die gaat altijd unicast):

**1. Automatisch (thuis)** — de telefoon broadcast elke seconde zijn naam op poort 47810,
de laptop pikt het bronadres van die beacon op en streamt daarheen. Geen instellingen nodig.
Gebruikt broadcast, dus alleen geschikt op een netwerk waar dat mag en werkt.

**2. Op IP (bedrijfsnetwerk, gastnetwerk, AP-isolatie)** — zet in de app het vinkje
**Zichtbaar via broadcast** uit. De telefoon zendt dan niets meer uit behalve de audio die
hij ontvangt; de app toont zijn adres. Op de laptop:

```powershell
.\AudioLinkSender.exe --ip 192.168.1.50
```

De laptop luistert in deze modus helemaal niet op 47810 en zendt geen enkel broadcast-
pakket uit: er gaat alleen UDP-verkeer van laptop naar telefoon op poort 47811.

Let op: de beacon-stand wordt in de app onthouden, dus zet hem één keer goed. De muis
op het vinkje doet de wijziging meteen ingaan, ook als de stream al loopt.

**Automatisch herverbinden** — kwam het adres via een beacon binnen, dan blijft de zender
tijdens het streamen meeluisteren. Verhuist de telefoon naar een ander adres (nieuwe
DHCP-huur, wifi opnieuw verbonden, app herstart), dan gaat de stream daar vanzelf verder en
staat er in het logvak `Telefoon op een nieuw adres (...) — stream verplaatst`. Bij een zelf
ingevuld adres (zoals je Tailscale-adres) doet hij dat bewust niet: een LAN-beacon zou de
stream dan naar het verkeerde netwerk sturen.

### Via Tailscale (verschillende netwerken)

Zitten laptop en telefoon niet in hetzelfde netwerk, gebruik dan Tailscale en verbind op het
`100.x.y.z`-adres van de telefoon:

1. Zet in de app **Zichtbaar via broadcast** uit (broadcast komt niet door een VPN heen).
2. De app toont al je adressen; het Tailscale-adres staat er met **(Tailscale)** achter en
   wordt automatisch als voorstel gebruikt.
3. Op de laptop: `AudioLinkSender.exe --ip 100.x.y.z`.

Wat al is aangepast voor deze route: de audiobundels blijven onder de 1252 bytes, zodat ze
binnen de MTU van WireGuard (1280) passen en niet gefragmenteerd worden; en de ontvanger
buffert 5 pakketten in plaats van 3 om jitter van een relay op te vangen.

Schat eerst je doorvoer in: PCM is ~1,5 Mbit/s. Een directe Tailscale-verbinding (peer-to-peer)
haalt dat makkelijk; valt hij terug op een DERP-relay, dan is de doorvoer beperkt en de
vertraging hoger — dan is Opus-compressie (~128 kbit/s) geen luxe meer maar nodig.

## Eigen volume op de telefoon

De app heeft een schuif **Geluid van de laptop**. Die regelt alleen de ontvangen stream
(AudioTrack-volume), niet de media-volume van je toestel. Je kunt dus je eigen muziek hard
zetten en het laptopgeluid zachter erbij mixen — precies omgekeerd kan ook. De stand wordt
onthouden en geldt meteen, ook midden in een stream.

## Tussen twee telefoons (verzenden vanaf Android)

Handig als je oortjes aan je werktelefoon gekoppeld zijn, maar je op je privételefoon iets
wil kijken of luisteren: je hoeft dan niet steeds opnieuw te koppelen.

1. **Telefoon B** (met de oortjes): modus **Ontvangen**, druk op **Start**.
2. **Telefoon A** (waar de muziek of video speelt): modus **Verzenden**, tik op
   **Zoek ontvanger** (vult het adres van B) of typ het adres zelf, en druk op
   **Verzenden starten**.
3. De eerste keer vraagt Android om **audio op te nemen** (eenmalig, daarna nooit meer) en
   daarna om toestemming voor het opnemen van beeld en geluid: kies **Nu beginnen**. Het
   scherm wordt niet opgenomen, alleen het geluid.

Wat je moet weten voordat je hier tijd in steekt:

- **Android 10 of nieuwer** is nodig; playback capture (geluid van een andere app aftappen)
  bestaat pas sinds die versie.
- De toestemming is **per sessie**. Elke keer dat je opnieuw start komt die dialoog terug;
  dat is een beveiligingseis van Android en niet te omzeilen.
- Alleen apps die zich laten opnemen komen door. DRM-diensten (Netflix) en apps die capture
  expliciet uitzetten blijven stil — dat blokkeert Android zelf.
- De kwaliteit ligt vast op 48 kHz stereo (~1,5 Mbit/s, de native rate van vrijwel elk
  toestel) met 30 ms buffer. Voor muziek is dat prima; het is geen instelbare knop omdat
  er bij playback capture geen resampling nodig is.
- Beide telefoons moeten elkaar kunnen bereiken: hetzelfde wifi-netwerk, of beide op
  Tailscale met het `100.x`-adres.

## Bitrate en buffer

Twee knoppen in het venster bepalen hoe de stream zich gedraagt. Ze zijn onafhankelijk:
bitrate is hoeveel data er over de lijn gaat, buffer is hoeveel vertraging je accepteert om
haperingen op te vangen.

**Bitrate** komt bij PCM uit samplefrequentie × kanalen × 16 bit:

| Kwaliteit | Bitrate |
|---|---|
| Hoog (48 kHz stereo) | 1536 kbit/s |
| Normaal (48 kHz mono) | 768 kbit/s |
| Compact (32 kHz mono) | 512 kbit/s |
| Spaarzaam (24 kHz mono) | 384 kbit/s |
| Minimaal (16 kHz mono) | 256 kbit/s |

De zender resamplet en mixt daarvoor zelf: het apparaat levert vaak 48 kHz stereo float,
ook als je mono op 16 kHz wilt versturen. Lage kwaliteit klinkt dunner (alles boven ~8 kHz
valt weg) maar is op een krappe relay een verschil van 6× in doorvoer.

**Buffer** is de jitterbuffer op de telefoon; de laptop stuurt de instelling mee in een
config-pakket, dus je hoeft de telefoon niet aan te raken:

| Buffer | Gebruik |
|---|---|
| 30 ms | filmpje of serie kijken: zo weinig mogelijk vertraging |
| 50 ms | praten/meekijken op een goed netwerk |
| 100 ms | standaard |
| 250 ms | muziek luisteren; vangt horten van wifi of een relay op |
| 500 ms | slechte verbinding: grootste kans op doorlopend geluid |

Kleiner zetten gooit de overtollige pakketten weg, dus de vertraging zakt meteen. Groter
zetten laat de muziek heel even wachten tot de buffer weer gevuld is.

## Latentie

| Stap | Duur |
|---|---|
| WASAPI loopback + PCM-omzetting | 10–30 ms |
| netwerk + buffer (standaard 100 ms) | ~100–150 ms |
| AudioTrack (low-latency-pad, minimum buffer) | 10–20 ms |
| Bluetooth naar de oortjes | 120–250 ms |
| **Totaal** | **≈ 240–420 ms** |

Met de buffer op 30 ms zakt dat naar ongeveer 170–300 ms, met *Ultra laag* (20 ms) naar
160–290 ms. De buffer en de telefoon zijn het enige stuk dat je zelf in de hand hebt; de
Bluetooth-vertraging niet.

Voor filmpjes: zet de kwaliteit op **Hoog (48 kHz stereo)** en de buffer op **Ultra laag**.
48 kHz is namelijk de native rate van vrijwel elk toestel, en alleen dan pakt Android het
low-latency-pad; bij 16 kHz mono wordt er alsnog geresampled en vervalt die winst.

prima voor muziek, podcasts en meekijken met een video-opname; voor lip-sync of gamen blijft
het te veel. De Bluetooth-vertraging is de grootste hap en valt niet weg te programmeren.

## Filmpjes kijken: vertraging compenseren

De Bluetooth-hop naar je oortjes (120–250 ms) is de grootste hap en valt niet weg te
programmeren. Voor lip-sync is dat te veel, maar je kunt het aan de **speler**-kant
compenseren: laat het beeld gewoon dezelfde vertraging oplopen als het geluid.

- **VLC**: *Extra → Synchronisatie van sporen* (of tijdens afspelen `j`/`k`) en zet
  *Audiotrack* op `-0.25` seconde.
- **mpv**: `mpv --audio-delay=-0.25 film.mkv`.
- **Lokale bestanden in een browser of YouTube**: daar is geen schuif voor, dus daar blijft
  het merkbaar. Speel zulke video's in VLC of mpv.

Stel de waarde in op de vertraging die je meet: klap in een video en schuif tot beeld en
klap samenvallen. Eén keer instellen per speler is genoeg.

## Beperkingen van deze MVP

- Alleen stereo/mono PCM; geen Opus-compressie (dus ~1,5 Mbit/s op je wifi).
- Eén telefoon per keer; geen beveiliging/encryptie (vertrouwd thuisnetwerk).
- Beveiligde audio (sommige Netflix/Spotify-DRM-paden) levert stilte op — dat blokkeert
  Windows zelf, niet deze app.
- Geen volume/mix-regeling per bron; gebruik de volumeknoppen van laptop en telefoon.
- Alleen IPv4.
- Geen Bluetooth-route tussen telefoon en laptop: Android kan geen A2DP-sink zijn, en
  Bluetooth zou er 120–250 ms bij optellen in plaats van wegnemen.

## Bestanden

```
audiolink/
├── AudioLink-Setup.exe                # installer (installeren, bijwerken, verwijderen)
├── AudioLinkSender.exe                # losse exe zonder installatie
├── PROTOCOL.md                        # pakketformaat
├── windows/build.sh                   # bouwt zender + installer (werkt ook op Linux)
├── windows/AudioLinkSender/
│   ├── AudioLinkSender.csproj
│   ├── Program.cs                     # entrypoint + commandoregelopties
│   ├── MainForm.cs                    # klein venster: IP, poort, zoeken, start/stop
│   ├── SingleInstance.cs              # één exemplaar, --quit, venster naar voren
│   ├── AutoStart.cs                   # automatisch starten na het inloggen
│   ├── Uninstall.cs                   # --uninstall: zichzelf opruimen
│   └── AudioStreamer.cs               # discovery + WASAPI-capture + UDP-verzending
├── windows/AudioLinkSetup/
│   ├── Program.cs                     # installer-startpunt + /uninstall
│   ├── SetupForm.cs                   # installatievenster met de drie opties
│   └── Installer.cs                   # uitpakken, afsluiten, snelkoppelingen, update
└── android/
    ├── settings.gradle.kts
    ├── build.gradle.kts
    ├── gradle.properties
    └── app/
        ├── build.gradle.kts
        └── src/main/
            ├── AndroidManifest.xml
            └── java/net/audiolink/receiver/
                ├── MainActivity.kt    # UI: modus ontvangen/verzenden, adressen, volume
                ├── ReceiverService.kt # foreground service, beacon, UDP-ontvangst
                ├── SenderService.kt   # verzendmodus: playback capture -> UDP
                └── AudioPlayer.kt     # instelbare buffer + AudioTrack (zonder audiofocus)
```
