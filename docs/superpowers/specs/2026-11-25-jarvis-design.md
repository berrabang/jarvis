# Jarvis v1 – designspec

**Datum:** 2026-11-25
**Ägare:** Anton (berrabang)
**Bas:** fork av [adewaskar/jarvis](https://github.com/adewaskar/jarvis) (MIT), commit `9ae5ec8`
**Status:** godkänd design, väntar på granskning av denna spec

---

## 1. Mål och omfång

En röststyrd assistent på Antons Windows-PC som han pratar med ("Jarvis, …"), som svarar med röst och en holografisk HUD, och som styr datorn, mejl, kalender, filer och enkla videojobb.

**I version 1 (detta dokument):** röst in/ut, HUD, hjärna via Claude-abonnemanget, vakt i tre nivåer, minne, mappstruktur, förmågorna ladda ner / hämta / öppna / göra enkla shorts.

**Inte i version 1** (egna delprojekt med egen spec senare):

| Delprojekt | Innehåll |
|---|---|
| **B – Innehållsteam** | Specialistagenter för research, manus, visuellt, klipp och publicering. Kontoguide för intäkter (YouTube, AdSense, TikTok m.fl.). Uppladdning via officiella API:er/OAuth. Byggs efter v1. |
| **v2 – Svenska** | Svensk talsyntes (t.ex. ElevenLabs/Azure). Taligenkänningen i v1 är redan språkmedveten. |
| **Senare** | Ikon i systemfältet som startar allt vid inloggning, semantisk minnessökning, mobilåtkomst. |

## 2. Fattade beslut

| Fråga | Beslut |
|---|---|
| Huvudsyfte v1 | Röst-Jarvis på PC:n: styr Windows, mejl, kalender |
| Språk | Engelska i v1; arkitekturen förberedd för svenska + engelska |
| Hjärna | Claude-abonnemanget via Claude Agent SDK (Claude Code-inloggning). Byte till API-nyckel ska vara en konfigurationsändring om Jarvis blir en produkt. |
| Lyssna | Väckningsord **och** snabbtangent (mellanslag = push-to-talk, finns redan) |
| Självständighet | Tre nivåer (se §4.4) |
| Gränssnitt | Holografisk HUD (finns i basen: React, Three.js, GLSL) |
| Bygga eller hämta | Forka adewaskar/jarvis (mest stjärnmärkta projektet som matchar ovan) och fylla luckorna |
| Konton | Jarvis skapar aldrig konton. Den öppnar registreringssidan; Anton slutför lösenord, verifiering och CAPTCHA. |

## 3. Arkitektur

Allt körs lokalt på datorn `antons` (Windows 11, x64).

```
 Mikrofon ──► HUD i Chrome/Edge (localhost:5173)
              │  Porcupine "Jarvis" (WASM, offline)
              │  VAD + inspelning av en replik
              │  Kokoro-röst (i webbläsaren)
              │  Bekräftelsekort (nivå 3)
              ▼ WebSocket / HTTP
            Brygga – Node (localhost:8787, bridge/server.mjs)
              │  /stt ──────────────► Taltjänst – Python (localhost:8790)
              │                        faster-whisper, språkdetektering
              │  PreToolUse-hook ───► Vakt (policy.mjs + policy.json)
              │                        allow / ask / deny, audit.log
              ▼
            Claude Agent SDK → Claude Code (abonnemang)
              ├─ claude.ai-kopplingar: Gmail, Google Calendar, Google Drive
              ├─ Windows-MCP (PC-styrning)
              ├─ Playwright-MCP (webb, nedladdningar)
              ├─ jarvis-memory (ny, lokal MCP)
              └─ jarvis-media (ny, lokal MCP: ffmpeg/shorts, filsortering)
```

**Behålls från basen utan ändring:** HUD och scen, bryggans WebSocket-protokoll, SSRF-skydd för WebFetch, filrots-inlåsning, nekade sökvägar för hemligheter (`~/.ssh`, `.env`, `~/.claude` …), befintliga tester.

**Ändras/läggs till:** taltjänst, `/stt`-routning, vakt i tre nivåer med bekräftelseflöde, två nya MCP-servrar, mappstruktur, konfiguration.

## 4. Komponenter

### 4.1 Väckningsord

- Motor: **Porcupine** (`@picovoice/porcupine-web`, redan beroende i basen). Inbyggt nyckelord "Jarvis", körs offline i WASM.
- Aktiveras genom att sätta `VITE_PICOVOICE_ACCESS_KEY` i `.env.local` (gratis nyckel från console.picovoice.ai, Anton skapar kontot).
- Basens reservläge `speech` (webbläsarens SpeechRecognition) skickar ljud till Google och **ska inte användas**. Utan nyckel startar Jarvis i push-to-talk-läge i stället (ändring i `src/config.ts`).
- Om Picovoice-villkoren inte passar: ersätt med openWakeWord-WASM ("hey jarvis"-modellen) bakom samma gränssnitt.

### 4.2 Taltjänst (ny)

- Katalog: `services/stt/` – Python 3.11+, körs med `uv`.
- Motor: `faster-whisper`, modell `small` som standard (konfigurerbar: `base` för svagare dator, `medium` för bättre svenska), CPU int8 eller CUDA om tillgängligt.
- Lyssnar endast på `127.0.0.1:8790`.
- Kontrakt:
  - `POST /transcribe` – body: rå ljudbytes (`audio/webm`, `audio/ogg`, `audio/wav`, `audio/mp4`), header `Content-Type`. Svar `200 {"text": "...", "language": "en", "duration_ms": 1234}`.
  - `GET /health` – `200 {"ok": true, "model": "small"}`.
  - Fel: `400` okänt format, `413` > 25 MB, `500` med felmeddelande.
- Bryggans `/stt` väljer motor via `JARVIS_STT=local|elevenlabs` (standard `local`). I `local` vidarebefordras bytes oförändrade till taltjänsten; svarsformatet mot HUD:n är oförändrat (`{text}`), med `language` som extra fält.
- Ingen molnreserv: om taltjänsten är nere svarar `/stt` med `503` och HUD:n visar textfält.

### 4.3 Röst ut

- `VITE_TTS_ENGINE=kokoro`, röst `bm_george` (brittisk). Lokal och gratis.
- Basens reserv till webbläsarens `speechSynthesis` behålls.

### 4.4 Vakt i tre nivåer

Ersätter basens binära `JARVIS_ALLOW_WRITES` med nivåer. Reglerna ligger i `policy.json` (i repot, med standardvärden) och kan överskridas av `~/.jarvis/policy.json`.

| Nivå | Beslut | Exempel |
|---|---|---|
| 1 Läsa | `allow` | Read/Glob/Grep, WebSearch, WebFetch (med SSRF-skydd), Gmail sök/läs, Calendar lista/läs, Drive sök/läs, Windows-MCP Snapshot/Screenshot, `recall` |
| 2 Små, ångerbara | `allow` + loggas i HUD | Öppna app, volym/media, Gmail `create_draft`, Calendar `create_event` **utan** inbjudna, `remember`, skriva/skapa inne i `~/Jarvis/`, `make_shorts` |
| 3 Fråga | `ask` | Skicka/svara/vidarebefordra mejl, event med inbjudna, ändra/ta bort event, radera, Bash/PowerShell, Windows-MCP Click/Type/Shortcut, flytta/döpa om utanför `~/Jarvis/`, ladda ner, Drive-delning, `forget`, publicering |
| Neka | `deny` | Hemlighetssökvägar, betalningar/köp, stänga av Defender/brandvägg, registerskrivning, skapa konton, läsa/skriva minnesfiler med andra verktyg än `jarvis-memory` |

**Regler:**
- Okänt verktyg ⇒ `ask` tills det klassats i `policy.json`.
- Fel vid utvärdering ⇒ `deny` (stäng vid fel).
- Nivå 3 gäller oavsett ursprung; text i mejl/webbsidor kan inte hoppa över frågan.
- Basens argumentkontroller (`checkToolUse`: filrötter, nekade sökvägar, SSRF) körs före nivåbeslutet och kan bara skärpa det.

**Bekräftelseflöde (nivå 3):**
1. `enforcePolicy` (PreToolUse-hooken, redan async) får `ask`.
2. Bryggan skickar `{"type":"confirm_request","id":"<uuid>","summary":"Skicka mejl till X om Y","tool":"<namn>"}` över WebSocket.
3. HUD:n visar kort och Jarvis säger sammanfattningen + "Ska jag?". Nästa replik tolkas som ja (`yes|yeah|do it|go|ja|kör`) eller nej (`no|stop|cancel|nej|avbryt`); klick på kortet fungerar också.
4. HUD:n svarar `{"type":"confirm_response","id":"<uuid>","approved":true|false}`.
5. Otydligt svar ⇒ en ny fråga; därefter nej. Inget svar inom 20 s ⇒ nej.
6. Hooken returnerar `permissionDecision: "allow"` eller `"deny"` med skäl.
7. `canUseTool` behålls som andra spärr och använder samma nivåbeslut.

**Revisionslogg:** varje beslut skrivs som en JSON-rad i `~/.jarvis/audit.log`: tid, verktyg, nivå, sammanfattning, beslut, svar. Argument med hemligheter maskeras.

### 4.5 Minne (ny MCP: `jarvis-memory`)

- Katalog: `services/memory/` – Python 3.11+ med `uv` och det officiella `mcp`-paketet, stdio-transport. Samma verktygskedja som taltjänsten; `sqlite3` med FTS5 ingår i Pythons standardbibliotek.
- Lagring: `~/.jarvis/memory/`
  - `profil.md` – skickas med i systemprompten varje session, max 2 000 tecken.
  - `fakta/<ämne>.md` – en fil per ämne.
  - `logg/ÅÅÅÅ-MM-DD.md` – kort sammanfattning per dag, skrivs när en session avslutas.
  - `index.sqlite` – FTS5-index över alla filer, byggs om vid start och vid varje skrivning.
- Verktyg:
  - `recall(query: string, limit?: number)` → träffar med fil, rad och utdrag. Nivå 1.
  - `remember(text: string, topic: string)` → lägger till rad i `fakta/<topic>.md`. Nivå 2.
  - `forget(match: string)` → visar träffar och tar bort efter bekräftelse. Nivå 3.
- Hemlighetsfilter före varje skrivning: avvisar text som matchar lösenords-/nyckel-/kortnummermönster (t.ex. `sk-…`, `ghp_…`, `AKIA…`, 13–19 siffror som klarar Luhn, "password: …").
- Allt lokalt, ingen synkning.

### 4.6 Förmågor (ny MCP: `jarvis-media` + konfiguration)

`jarvis-media` ligger i `services/media/` – Python 3.11+ med `uv` och `mcp`-paketet, stdio-transport. Anropar ffmpeg som underprocess och taltjänsten över HTTP.

| Kommando | Implementation | Nivå |
|---|---|---|
| "Ladda ner …" | Playwright-MCP hittar filen; sparas i `~/Jarvis/Nedladdningar/<typ>/`. Bekräftelsen anger filnamn, källa och storlek. | 3 |
| Körbar fil (.exe, .msi, .bat, .cmd, .ps1, .scr, zip som innehåller sådana) | Läggs i `Nedladdningar/Program/`, skannas med `Start-MpScan -ScanType CustomScan`, **körs aldrig automatiskt** | 3 |
| "Gör shorts av …" | `jarvis-media.make_shorts(input, count=3, min_s=15, max_s=60)`: transkribera via taltjänsten → Claude väljer segment → ffmpeg klipper, beskär 9:16, bränner in undertexter → `~/Jarvis/Video/Shorts/<datum>_<namn>/short-NN.mp4` + `.srt`. Originalet rörs inte. | 2 |
| "Hämta …" | Gmail/Drive-kopplingar, filverktyg, webben. Bilagor sparas i `~/Jarvis/Mejlbilagor/ÅÅÅÅ-MM/`. | 1 (läsa) / 2 (spara) |
| "Öppna/starta …" | Windows-MCP `App` | 2 |
| "Sortera … (mapp utanför Jarvis)" | `jarvis-media.plan_sort(path)` visar plan → `apply_sort(plan_id)` | plan 1 / utför 3 |
| "Ladda upp …" | Delprojekt B | 3 |

`yt-dlp` installeras valfritt och används endast för Antons egna videor eller material med fri licens.

### 4.7 Mappstruktur och namnregel

```
%USERPROFILE%\Jarvis\
├─ Nedladdningar\{Dokument,Bilder,Video,Ljud,Program}\
├─ Video\{Original,Shorts\<datum>_<namn>\}
├─ Mejlbilagor\ÅÅÅÅ-MM\
├─ Dokument\
└─ Arbetsyta\          (töms på filer äldre än 7 dagar vid start)

%USERPROFILE%\.jarvis\   (dold, nekad för alla verktyg utom bryggan och jarvis-memory)
├─ memory\
├─ policy.json         (valfri överskrivning)
├─ audit.log
└─ logs\
```

- Namnregel: `ÅÅÅÅ-MM-DD_kort-beskrivning.ext` – gemener, bindestreck, inga mellanslag, åäö translittereras (å→a, ä→a, ö→o).
- Typmappning via filändelse; okänd ändelse ⇒ `Dokument`.
- Krock ⇒ suffix `-2`, `-3` …
- Bryggans standardvärden ändras så att `JARVIS_WORKSPACE` blir `<hemmapp>/Jarvis/Arbetsyta` och `JARVIS_FILE_ROOTS` blir `<hemmapp>/Jarvis`, beräknat med `os.homedir()`. De behöver alltså inte sättas i `.env`.

### 4.8 Konfiguration

`.env` för bryggan:

```
JARVIS_ACCOUNT_CONNECTORS=1
JARVIS_MCP_SERVERS=windows-mcp,playwright,jarvis-memory,jarvis-media
JARVIS_STT=local
JARVIS_STT_URL=http://127.0.0.1:8790
```

`.env.local` för HUD:n:

```
VITE_TTS_ENGINE=kokoro
VITE_KOKORO_VOICE=bm_george
VITE_PICOVOICE_ACCESS_KEY=...   # Antons egen nyckel, committas aldrig
```

MCP-servrarna `windows-mcp`, `playwright`, `jarvis-memory`, `jarvis-media` läggs till i `~/.claude.json`. Nycklar committas aldrig.

## 5. Dataflöde – en replik

1. Porcupine hör "Jarvis" → HUD går från `dormant` till `listening` (eller mellanslag hålls nere).
2. VAD avgör när repliken är slut → ljudet skickas till bryggans `/stt` → taltjänsten → `{text, language}`.
3. Bryggan skickar texten till Agent SDK-sessionen (samma session behålls mellan repliker).
4. Claude planerar och anropar verktyg; varje anrop passerar `enforcePolicy` (nivåbeslut, eventuell bekräftelse, logg).
5. Svarstext strömmas till HUD:n → Kokoro läser upp mening för mening; paneler visar resultat.
6. "Stopp" när som helst ⇒ bryggan avbryter pågående SDK-anrop och tal.

## 6. Felhantering

| Fel | Beteende |
|---|---|
| Taltjänsten nere | Startskriptet startar om den (max 3 försök/minut). HUD visar "Taligenkänning nere" + textfält. Ingen molnreserv. |
| Kokoro misslyckas | Webbläsarens röst tar över (finns i basen). |
| Abonnemangsgräns nådd | Jarvis säger det; HUD visar återställningstid om SDK:n anger den. |
| Koppling/verktyg felar | Ett nytt försök; därefter säger Jarvis vad som inte gick och fortsätter med resten. |
| Vakten felar | `deny`. |
| Lång uppgift | "Working on it" + förlopp i HUD; "stopp" avbryter. |
| Bryggan kraschar | `scripts/start.mjs` startar om bryggan och taltjänsten; fel loggas i `~/.jarvis/logs/`. |

## 7. Test och godkännande

**Automatiska tester (`npm test` + `uv run pytest`):**
- Befintliga tester i `test/` ska fortsatt passera.
- `policy.json`: ett testfall per exempel i tabellen §4.4 ⇒ förväntad nivå; okänt verktyg ⇒ `ask`; kastat fel ⇒ `deny`.
- Bekräftelseflöde: ja, nej, otydligt två gånger, timeout ⇒ rätt beslut.
- Minne: hemlighetsfiltret avvisar alla mönster i §4.5; `remember` → `recall` hittar raden; `forget` tar bort.
- Namnregel och typmappning: åäö, krockar, okänd ändelse.
- Taltjänsten: wav/webm in ⇒ text ut; tom/kort fil ⇒ tom text; 413 över gränsen.
- Shorts: 2 min testvideo ⇒ tre filer i 9:16 med `.srt`, originalets hash oförändrad.

**Kedjetest:** inspelade fraser (`test/fixtures/*.wav`) körs genom hela kedjan i torrkörningsläge (`JARVIS_DRY_RUN=1`), där nivå 2- och 3-verktyg ersätts med attrapper som loggar anropet.

**Godkännande v1 – dessa tio ska fungera med röst:**
1. "Jarvis, what's on my calendar tomorrow?"
2. "Draft a reply to the last email from …" (utkast skapas, inget skickas)
3. "Send it." ⇒ bekräftelsefråga; "yes" skickar, "no" avbryter
4. "Open Spotify."
5. "Download …" ⇒ bekräftelse med filnamn/källa/storlek, hamnar i rätt mapp
6. "Make three shorts from …" ⇒ tre filer i `Video\Shorts\…`
7. "Sort my Downloads folder." ⇒ plan visas, väntar på ja
8. "Remember that …" och sedan "What do you remember about …?"
9. "Stop." mitt i ett långt svar ⇒ avbryter direkt
10. Väckningsordet fungerar med musik i bakgrunden; mellanslag fungerar när det inte gör det

## 8. Steg 0 – säkerhetsgranskning före första körning

Innan något körs på `antons`: läs igenom `bridge/`, `scripts/`, `package.json` (inklusive `postinstall`-skript) och beroendelistan. Kör `npm audit`. Notera avvikelser i en kort rapport. Först därefter `npm install` och första start.

## 9. Det Anton behöver skaffa eller göra själv

| Sak | Varför |
|---|---|
| Claude Code installerat och inloggat med abonnemanget | Hjärnan |
| Node.js 20+, `uv` (Python), ffmpeg | Körmiljö |
| Gratis Picovoice-konto + AccessKey | Väckningsordet offline |
| Headset eller mikrofon | Röst in |
| Chrome eller Edge | HUD |
| (Valfritt) återansluta RUBE på claude.ai | Fler appar senare |

## 10. Risker och sådant som verifieras först

| Risk | Hantering |
|---|---|
| Laddar Agent SDK claude.ai-kopplingarna (Gmail/Calendar/Drive) på Windows? Dokumentationen säger att de laddas när abonnemanget är aktiv inloggning. | Första steget i planen: kort spike som listar verktygen i en SDK-session med `JARVIS_ACCOUNT_CONNECTORS=1`. |
| Basen är främst testad på macOS; Windows nämns delvis. | Spike: `npm start` på Windows, åtgärda sökvägar och skript. |
| Abonnemangets gränser kan nås vid tung användning. | Visa tydligt i HUD; byte till API-nyckel är en konfigurationsändring. |
| Picovoice-villkor (gratisnivå för personligt bruk). | Reserv: openWakeWord-WASM bakom samma gränssnitt. |
| Uppströms ändringar i adewaskar/jarvis. | Håll våra ändringar i egna filer där det går; synka uppströms manuellt vid behov. |
