# Jarvis – färdplan efter v1

**Princip:** Jarvis styr allt. Varje nytt område kopplas in som verktyg eller specialistagenter under samma hjärna och samma vakt (`policy.json`). Det betyder:

- Att **läsa, analysera och ta fram utkast** sker direkt.
- Att **publicera, skicka, ladda upp, betala eller annonsera** ligger på nivå 3 – Jarvis frågar högt och väntar på ditt ja. **Undantag:** i VD-läget (B+) får Jarvis publicera själv inom ramar som du har godkänt i förväg.
- Att **skapa konton, lägga riktiga trading-order, betala och kringgå sajters botskydd** nekas alltid.

Varje område nedan får egen spec → plan → bygge (samma process som v1) när det är dags. Ordning: v1 → B → C → D.

---

## B – Innehållsteam (nästa efter v1)

**Mål:** Jarvis tar fram, producerar och lägger ut video på YouTube, TikTok och Instagram – med dig som regissör. Du säger t.ex. "Jarvis, gör tre shorts av gårdagens video och lägg ut dem i morgon kl 18", Jarvis gör jobbet, visar resultatet och frågar innan något publiceras.

**Agenter:**

| Agent | Gör | Verktyg |
|---|---|---|
| Research | Trender, nischer, sökord, konkurrenter, vad som fungerar | vidIQ (kopplad) |
| Strategi | Kanalplan, innehållspelare, publiceringsschema, intäktsplan | Promptspelbok (se nedan) + vidIQ-data |
| Manus | Idéer, hooks, manus, titlar, beskrivningar | Claude + blader/humanizer |
| Visuellt | Thumbnails, grafik, bild/video | Canva (kopplad), vidIQ-thumbnails, Open-Higgsfield-AI |
| Produktion | Färdiga videor och shorts | jarvis-media (v1), MoneyPrinterTurbo, HyperFrames (kopplad) |
| Röst | Voiceover, svensk röst | ElevenLabs (kopplad) |
| Publicering | Ladda upp, schemalägga, lägga ut på flera plattformar | vidIQ (`video_upload`, `social_post_publish`) – **alltid nivå 3** |
| Analys | Statistik, vad som funkar, förslag på förbättringar | vidIQ-analys |

**Verktyg att utvärdera:**

| Verktyg | Vad | Status |
|---|---|---|
| [MoneyPrinterTurbo](https://github.com/harry0703/MoneyPrinterTurbo) (MIT) | Ämne in → färdig short med manus, röst, undertexter, musik. Stöder Claude, Kokoro/ElevenLabs/Edge TTS, Windows. **Har egen automatisk uppladdning till TikTok/IG/YouTube – ska vara avstängd; Jarvis publicerar via vakten.** | Utvärdera som produktionsmotor |
| [HyperFrames](https://github.com/heygen-com/hyperframes) (Apache-2.0) | HTML → MP4, samma kod ger samma video. Finns som claude.ai-koppling och som Claude Code-plugin; lokal rendering kräver Node 22+ och FFmpeg (finns redan i v1). Windows-stöd för CLI:t obekräftat. | Kopplad |
| [blader/humanizer](https://github.com/blader/humanizer) (MIT) | Skill som får AI-text att låta mänsklig. Claude Code: `/plugin marketplace add blader/humanizer` → `/plugin install humanizer@humanizer` | Installera i B |
| [Open-Generative-AI](https://github.com/Anil-matcha/Open-Generative-AI) (tidigare Open-Higgsfield-AI, MIT) | Bild/video-studio med hundratals modeller. Mest molnbaserad via Muapi.ai (betalt); lokalt bara några bildmodeller. Marknadsför sig som "uncensored, no guardrails". | Utvärdera med försiktighet – kostnad och innehållsregler |
| vidIQ, Canva, ElevenLabs, HyperFrames | claude.ai-kopplingar | Kopplade – slås på i B (står i `JARVIS_BLOCK_CONNECTORS` i v1) |

**Promptspelbok för strategi-agenten** (från @alfateemacademyofficial): 1) full kanalplan, 2) välj nisch + 100 videoidéer, 3) virala manus/hooks/thumbnails, 4) förstå YouTube-tillväxt och algoritmen, 5) producera 3× snabbare (inkl. återanvändning till Shorts/TikTok/Reels), 6) intäktsplan (YPP, affiliate, sponsorer, produkter), 7) läs statistiken som ett proffs. Skrivs om till agentens systemprompt och matas med riktig data från vidIQ.

**Kontoguide för intäkter** (från v1-specen): Jarvis kollar aktuella krav, visar checklista, öppnar registreringssidor – du skapar kontona själv – och kopplar in dem via OAuth.

**Regler som gäller B:**
- YouTube stänger av intäkter för massproducerat/repetitivt AI-innehåll. Du styr den kreativa riktningen (veckomöte med VD:n, se B+). Varje video ska tillföra något eget. Autopilot bara inom VD-lägets ramar.
- En kanal per plattform.
- Publicering och schemaläggning är nivå 3, utom inom VD-lägets ramar (B+).

---

## B+ – VD-läge: Jarvis driver innehållsteamet som ett företag (Antons beslut 2026-11-26)

**Mål:** Jarvis är VD. Han överser alla agenter, delar ut och omfördelar arbete, "anställer" fler agenter när en roll inte hinner med, och godkänner publicering själv inom ramar som du har satt. Du sätter riktningen och kan stoppa allt, men behöver inte godkänna varje video.

**Organisation (startbemanning, VD:n kan skala varje roll):**

| Roll | Antal | Gör | Verktyg |
|---|---|---|---|
| VD (Jarvis) | 1 | Dagsplan, fördelar uppdrag, granskar, godkänner publicering inom ramarna, rapporterar till dig | Agent SDK + jobbkö |
| Trendspanare | 1+ | Varje morgon: vad får flest visningar just nu (YouTube, TikTok, Instagram). Lämnar topp 5 idéer med data | vidIQ trending/outliers, TikTok/IG-outliers |
| Manusförfattare | 1+ | Krok (0–2 s), struktur (krok → upptrappning → twist → loop), titel, omslagskoncept | Claude + vidIQ-skrivregler |
| Producent | 1+ | Röst, klipp, text ord för ord, vattenstämpel, omslag | `build_short.py`/`roblox_build.py` (finns), jarvis-media, ElevenLabs |
| Kvalitetsgranskare | 1 | Faktakoll mot källor, upphovsrätt (bara eget/fritt material med kreditering), plattformsregler, AI-märkning, "gjord för barn" | Checklista + webbsökning |
| Publicerare | 1 | Lägger ut enligt schema och tak | YouTube Data API, TikTok Content Posting API (se nedan) |
| Analytiker | 1 | Följer upp efter 24 h och 72 h, säger vad som funkar, matar trendspanaren och VD:n | vidIQ-analys |

**Agentvägg i HUD:en (Antons önskemål, som i TikTok-videon):** alla agenter syns på skärmen samtidigt medan de jobbar. Varje agent har ett holografiskt kort med namn, roll, status (ledig, jobbar, väntar på granskning, klar), nuvarande uppdrag, senaste verktyg den använde och en förloppsindikator. VD:n står i mitten med linjer till de agenter han har gett uppdrag, så att man ser hur arbetet flyttas mellan dem. Nya agenter som "anställs" tonar in som nya kort. Tekniskt: bryggan skickar Agent SDK:s subagenthändelser (start, verktygsanrop, klart, fel, kopplade via `parent_tool_use_id`) över WebSocketen som `{type:'agent', id, role, status, task, tool, progress}`. HUD:en ritar korten med Three.js, i samma stil som reaktorn. "Jarvis, visa teamet" växlar till helskärm.

**Regel (Anton 2026-11-26):** alla videor går igenom alla agenter, från trendspanare till analytiker. Inget steg hoppas över.

**Rekrytering:** VD:n startar fler instanser av en roll när kön växer (t.ex. två producenter). Tak: max antal samtidiga agenter och max kostnad per dag (krediter: vidIQ, ElevenLabs, Claude-gränser) i `autonomy.json`.

**Stående fullmakt (`autonomy.json`, du godkänner filen en gång):**
- Jarvis får publicera själv om: formatet finns på godkänd lista, kvalitetsgranskaren har godkänt, och taket inte är nått (start: 2 per dag och plattform).
- Varje publicering schemaläggs med 60 minuters fördröjning. Du får en notis (Telegram eller e-post) med en stoppknapp. Sammanfattning varje kväll.
- Allt utanför ramarna (nytt format, ny plattform, betalning, annonsering) blir nivå 3: Jarvis frågar dig.
- Nödstopp: "Jarvis, stoppa all publicering" pausar kön direkt.
- Alltid nekat, även i VD-läget: upphovsrättsskyddat material (scener ur serier/filmer, andras klipp), att skapa konton, att kringgå botskydd, köp och betalningar, och barnvideor utan "gjord för barn"-märkning.

**Tekniskt:**
- Agent SDK-subagenter per roll + jobbkö i SQLite (research → manus → produktion → granskning → publicering). `start.mjs` och Windows Schemaläggaren startar morgonkörningen.
- vidIQ:s publiceringsverktyg kräver att du har sett varje utkast, och gratisplanen ger 3 YouTube-uppladdningar om dagen via vidIQ. För verklig autonomi behövs egen publicering:
  - YouTube Data API med en OAuth-app i ditt eget Google Cloud-projekt.
  - TikTok Content Posting API: oauditerade appar kan bara posta privat, så appen måste granskas av TikTok först (kan ta upp till två veckor).
- Risk: YouTubes regler mot massproducerat AI-innehåll. Motåtgärd: kvalitetsgranskaren, varierade format och att du sätter riktningen varje vecka.

---

## C – Företagsteam (efter B)

**Mål:** Jarvis driver delar av ett eget företag med specialistagenter.

| Område | Inspiration | Verktyg |
|---|---|---|
| Webbutik: research, annonser, optimering, support | @nolanpearson.ai (10 agenter driver en butik) | Shopify (kopplad), Lena – Meta-annonser (kopplad) |
| Leadgenerering och sälj: hitta företag med dåliga hemsidor, bygga nya sidor, skicka säljbrev | @robertbilling (botar med eget minne som pratar med varandra) | Firecrawl, Gmail, egna agenter |
| Annonsgranskning | [claude-ads](https://github.com/AgriciDaniel/claude-ads) (MIT) – granskar annonskonton på 12 plattformar utifrån exporterad data (inga API-nycklar) | Claude Code-plugin |
| Mejl som AI läser och svarar på | [cloudflare/agentic-inbox](https://github.com/cloudflare/agentic-inbox) (Apache-2.0) | Alternativ – kräver egen domän och Cloudflare-tjänster, bara 11 commits; skriver utkast och skickar bara efter bekräftelse. Jarvis har redan Gmail/Outlook. |
| Helhet | @hannesrkuhn – 48 agenter, "allt riskabelt väntar på ditt ja" | Samma vakt-princip som Jarvis |

**Regler som gäller C:**
- Annonser som kostar pengar, ordrar, prisändringar och utskick är nivå 3.
- Kalla säljmejl inom EU kräver berättigat intresse, tydlig avsändare och avregistreringslänk. Volymgräns per dag.
- Börja med ett område, inte 48 agenter.

---

## D – Ekonomi och trading (utvärdera)

| Verktyg | Vad |
|---|---|
| [FinceptTerminal](https://github.com/Fincept-Corporation/FinceptTerminal) (AGPL-3.0) | Gratis "Bloomberg-terminal" med AI-agenter och många datakällor. Personligt bruk fritt; delar du en ändrad version måste koden publiceras. Windows-installerare finns. |
| [Vibe-Trading](https://github.com/HKUDS/Vibe-Trading) (MIT) | AI-agenter som debatterar affärer; krypto, aktier, optioner. Har MCP-stöd. Om den lägger riktiga order är oklart. |
| [AutoHedge](https://github.com/The-Swarm-Corporation/AutoHedge) (MIT) | Fyra agenter: välj, kontrollera, storlek, lägg order. **Handlar på riktigt på Solana med din plånboks privata nyckel**, bara 37 commits och ingen riskvarning – inte aktuell. |
| Grok-tradingbot (@0xnevsky) | Inspiration |

**Regler som gäller D:**
- Analys, backtesting och pappershandel (låtsaspengar) går bra.
- **Riktiga order är spärrade i vakten** (`policy.json`: verktyg som heter något med *trade* eller *place order* nekas). Vaktens deny-regler kan inte låsas upp via `~/.jarvis/policy.json` – det kräver ett medvetet beslut och en ändring i repots `policy.json`.

---

## Senare idéer

| Idé | Källa |
|---|---|
| En figur (Live2D) i stället för orben – hör, ser skärmen, minns, kan avbrytas; stöder Windows, Claude och faster-whisper | [Open-LLM-VTuber](https://github.com/Open-LLM-VTuber/Open-LLM-VTuber) (MIT) |
| Alternativ datorstyrning om Windows-MCP inte räcker – stöder Windows 10 1809+ via UI Automation | [vercel-labs/agent-desktop](https://github.com/vercel-labs/agent-desktop) (Apache-2.0) |
| Flera AI-botar, var och en med egen virtuell dator; kör Claude/Codex/Grok via dina egna inloggningar – samma princip som Jarvis | [OpenMausBot](https://github.com/milind-soni/OpenMausBot) (Apache-2.0, tidigare OpenGrokBot) |
| Nå Jarvis från mobilen | Telegram-plugin (installerat i Claude) |
| Svensk röst | ElevenLabs (v2) |
| Ikon i systemfältet som startar allt vid inloggning | v1-specen |
| Alla AI-modeller i en app | [LibreChat](https://librechat.ai) – behövs inte så länge hjärnan är Claude |

## Avfärdat tills vidare (och varför)

| Verktyg | Varför |
|---|---|
| [camofox-browser](https://github.com/jo-inc/camofox-browser), [Scrapling](https://github.com/D4Vinci/Scrapling) `--solve-cloudflare` | Byggda för att lura sajters botskydd (Scrapling säger själv att den tar sig förbi Cloudflare Turnstile) – bryter mot villkor, risk för avstängda konton. Vanlig Scrapling utan det läget kan utvärderas i C. |
| [freellmapi](https://github.com/tashfeenahmed/freellmapi) | Snabbinstallation med `curl \| bash`, staplar gratisnivåer från många leverantörer (kräver nycklar hos var och en); hjärnan är redan Claude. |
| VidRush, Roblox + OpenArt | Utanför Jarvis. |
| Claude Code-plugins för modellbyte/kodflöden (@sebintel) | För utvecklingsarbetet, inte Jarvis. |

## Kontrollerade repon (2026-11-26)

Alla repon från videorna finns, är publika och aktiva. Stjärnor vid kontrollen:

| Repo | ★ | Licens | Commits | Anmärkning |
|---|---|---|---|---|
| harry0703/MoneyPrinterTurbo | 143k | MIT | 1 263 | Egen auto-uppladdning – stäng av |
| D4Vinci/Scrapling | 101k | BSD-3 | 1 825 | Kringgår botskydd – avfärdat |
| heygen-com/hyperframes | 90k | Apache-2.0 | 8 142 | Kopplad |
| blader/humanizer | 73k | MIT | 130 | |
| tashfeenahmed/freellmapi | 58k | MIT | 1 175 | Avfärdat |
| LibreChat | 47k | MIT | 6 543 | |
| HKUDS/Vibe-Trading | 38k | MIT | 3 563 | |
| Anil-matcha/Open-Generative-AI | 36k | MIT | 543 | Bytt namn från Open-Higgsfield-AI |
| Fincept-Corporation/FinceptTerminal | 33k | AGPL-3.0 | 1 122 | |
| vercel-labs/agent-desktop | 25k | Apache-2.0 | 351 | Windows-stöd |
| milind-soni/OpenMausBot | 24k | Apache-2.0 | 11 306 | |
| Open-LLM-VTuber/Open-LLM-VTuber | 15k | MIT | 913 | |
| jo-inc/camofox-browser | 12k | MIT | 662 | Avfärdat |
| AgriciDaniel/claude-ads | 11k | MIT | 149 | |
| cloudflare/agentic-inbox | 8.5k | Apache-2.0 | 11 | Få commits |
| The-Swarm-Corporation/AutoHedge | 6.5k | MIT | 37 | Få commits, riktig handel |

Stjärnor kan köpas och säger inget om säkerhet. Innan något av dessa installeras: läs koden (som i v1:s steg 0), kör aldrig `curl | bash` eller installationsskript blint, och installera i en separat mapp.

## Källor (TikTok, 2026-11-26)

- https://www.tiktok.com/@robertbilling/video/7712142390328413462
- https://www.tiktok.com/@martiendejong_dev/photo/7696645043523718432
- https://www.tiktok.com/@byshawnchee/photo/7706071313789586709
- https://www.tiktok.com/@benkimball.ai/video/7701573554617290015
- https://www.tiktok.com/@nolanpearson.ai/video/7701357714449943830
- https://www.tiktok.com/@sebintel/video/7695752696350903574
- https://www.tiktok.com/@philsidehustles/video/7712058531834268961
- https://www.tiktok.com/@alfateemacademyofficial/photo/7701367343267564807
- https://www.tiktok.com/@hannesrkuhn/video/7709566441288158497
- https://www.tiktok.com/@sean.makes/video/7686862913931349279
- https://www.tiktok.com/@0xnevsky/video/7711851304624180519
- Tidigare: https://www.tiktok.com/@jaredrhod/video/7660330574900514061, https://www.tiktok.com/@restructureai/video/7707135325793144085
