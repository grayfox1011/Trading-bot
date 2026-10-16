# Errori previsti e soluzioni (pre‑mortem del bot)

> Complemento di [PIANO.md](PIANO.md). Per ogni errore: come si manifesta, come lo rileviamo, come lo preveniamo, cosa fa il bot da solo. Le soluzioni citano metodi reali dell'SDK `hyperliquid-python-sdk` 0.24.0, dell'SDK `anthropic` 1.13 e di `typesafe-sdk`, verificati il 9/10/2026.

---

## 0. Dieci principi che evitano la maggior parte dei guai

1. **Fail‑closed.** In caso di dubbio (dati stantii, modello irraggiungibile, stato incoerente) il bot **non apre** nulla. Le posizioni esistenti sono sempre protette da stop e target **on‑chain**, quindi un bot fermo è un bot sicuro.
2. **Ordini idempotenti.** Ogni ordine ha un `cloid` deterministico (hash di ciclo + simbolo + lato + scopo). Prima di qualsiasi retry si chiede a Hyperliquid se l'ordine esiste già (`info.query_order_by_cloid`). Mai un retry "cieco".
3. **Riconciliare prima di agire.** All'inizio di ogni ciclo e di ogni guardia lo stato reale (`clearinghouseState`, `openOrders`, `userFills`) vince sul database. Se non coincidono, il bot entra in *safe mode*: nessun nuovo ingresso, alert, e chiude solo ciò che non riconosce.
4. **Protezione nello stesso istante dell'ingresso.** Ingresso + stop + target partono nello stesso batch (`bulk_orders(..., grouping="normalTpsl")`), così la posizione non è mai "nuda". Comportamento OCO (stop e target si annullano a vicenda) da verificare sul testnet prima del live.
5. **Un processo, una chiave.** Un solo processo per indirizzo e una chiave agent diversa per ambiente (dev, paper, live): i nonce di Hyperliquid devono essere crescenti per firmatario.
6. **Il log degli eventi è la verità.** Ogni dato, segnale, decisione, ordine e fill è un evento append‑only (JSONL + SQLite in modalità WAL). Il database si ricostruisce dal log; i report si calcolano dal log.
7. **Budget guard su tutto.** Perdita giornaliera, settimanale, drawdown, numero di azioni API, costi Claude e Jev: ogni contatore ha una soglia e una reazione automatica.
8. **Degradazione graduale.** Senza oracolo (Opus) → niente nuovi ingressi, Jev continua a gestire le uscite. Senza decisore (Jev) → surrogato a regole con soglie più severe (o niente ingressi, a scelta). Senza dati freschi → niente. Senza rete → i trigger on‑chain proteggono.
9. **Dead‑man switch.** Heartbeat ogni 15 minuti su Telegram e in un file; due heartbeat mancati → alert sul telefono; da telefono `/flat` chiude tutto.
10. **Il live è difficile da attivare per sbaglio.** `BOT_MODE=live` da solo non basta: serve anche `LIVE_CONFIRM=<indirizzo del wallet>` e un tetto `MAX_EQUITY_USDC` (es. 150). Se l'equity supera il tetto il bot si rifiuta di partire.

---

## 1. Esecuzione e account Hyperliquid

| Errore previsto | Come si manifesta / rilevazione | Prevenzione | Reazione automatica |
|---|---|---|---|
| Ordine sotto il minimo | Risposta con `error: "Order must have minimum value of $10"` dopo l'arrotondamento della size ai `szDecimals` | `round_sz` → ricalcolo nozionale → se < 10,5 $ alza la size al minimo o salta il trade | Trade annullato, log |
| "Insufficient margin" con fondi presenti | USDC nel saldo sbagliato (modalità standard: il dex `xyz` ha un saldo separato) oppure margine bloccato da ordini aperti | Unified account, oppure `send_asset(addr, "", "xyz", "USDC", importo)` manuale; controllo `withdrawable`/`accountValue` da `user_state(addr, dex="xyz")` prima di ogni ordine | Nessun ordine, alert "fondi nel saldo sbagliato" |
| Prezzo non valido | Rifiuto: più di 5 cifre significative o più di `6 − szDecimals` decimali | Funzione `round_px` con test unitari; `float_to_wire` dell'SDK | Ordine non inviato |
| Size non valida | Rifiuto per decimali | `round_sz` ai `szDecimals` letti da `meta` ad ogni avvio | Ordine non inviato |
| Ordine IOC non eseguito | `"Order could not immediately match against any resting orders"`: slippage 0,3 % troppo stretto in un book sottile | Gate spread ≤ 3 bps e profondità ≥ 20× size prima dell'invio | Un solo retry a 0,5 % se il gate è ancora verde; altrimenti il trade salta (non si insegue) |
| Fill parziale | `userFills` mostra una size minore di quella richiesta | – | Stop e target dimensionati sulla posizione **reale**; se il residuo è sotto 10 $ di nozionale (non riducibile) si chiude tutto |
| Stop/target non piazzati dopo l'ingresso | Risposta `error` sui trigger, posizione senza protezione | Batch unico `normalTpsl`; verifica `openOrders` subito dopo | 3 retry in 10 s; se fallisce → `market_close` immediato e alert. Regola: mai più di 10 s senza stop |
| Trigger orfani | Dopo un target eseguito resta lo stop (o viceversa) | OCO via grouping; trigger `reduce_only=True` (non possono aprire posizioni inverse) | Ad ogni ciclo: cancella trigger senza posizione (`cancel_by_cloid`) |
| Stop eseguito molto peggio del previsto (gap) | Fill dello stop `isMarket` lontano dal trigger: perdita > 1 R | Niente posizioni attraverso earnings/macro; chiusura forzata 15:45 ET; margine isolato (perdita massima = margine) | Registra "gap loss", cooldown 1 giorno sul simbolo |
| Ordine duplicato | Timeout di rete dopo l'invio, poi retry: due posizioni | `cloid` deterministico; `query_order_by_cloid` prima di ogni retry | Se esiste già: nessun reinvio, riconciliazione |
| Nonce non valido | `"Invalid nonce"`: orologio sfasato o due processi con la stessa chiave | NTP/chrony; lock di istanza; una chiave agent per processo; `set_expires_after(now + 60 s)` così un'azione in ritardo viene rifiutata invece di eseguita tardi | Safe mode + alert |
| Chiave agent scaduta o revocata | `"User or API Wallet does not exist"` o firma rifiutata (nell'app si può impostare una scadenza) | Healthcheck giornaliero con un'azione innocua (`update_leverage` allo stesso valore); alert 7 giorni prima della scadenza registrata in config | Nessun nuovo ingresso; runbook "rinnovo agent" |
| Rate limit | HTTP 429 o `"Too many requests"` | Peso per ciclo ≤ 100 su 1.200/min; address limit 10.000 azioni iniziali (consumo previsto < 30/giorno); backoff esponenziale | Ciclo saltato, alert se ricorrente |
| Leva o margine impostati male | `assetPositions[].position.leverage.type` ≠ `isolated` o valore ≠ 3 | `update_leverage(3, coin, is_cross=False)` all'avvio e verifica dopo ogni apertura; controllo `maxLeverage` e `onlyIsolated` da `meta` | Chiude la posizione se la leva reale è > 5× quella prevista |
| Stato ignoto dopo caduta di rete a metà ordine | Nessuna risposta all'invio | `cloid` + `set_expires_after` | Riconciliazione via `query_order_by_cloid` e `userFills`; poi safe mode fino a stato coerente |
| Il deployer cambia i parametri | `deployerFeeScale` (0,1–3), `marginTableId`, OI cap, `growthMode` cambiano in `meta`/`perpDexs` | Snapshot giornaliero dei parametri e confronto ad ogni ciclo | Se cambiano fee o margine: nessun nuovo ingresso + alert + ricalcolo del break‑even; se `haltTrading`: il deployer liquida a mark, il bot registra e si ferma |
| OI cap raggiunto | Ordine che aumenta l'esposizione rifiutato | Leggere `assetToStreamingOiCap` | Trattato come "no trade", non come errore |
| ADL o liquidazione inattesa | La riconciliazione trova la posizione chiusa senza un nostro ordine | Leva bassa, margine isolato | Registra il fill con motivazione, pausa 1 giorno, alert |
| Azioni che richiedono la chiave principale | `send_asset`, `approve_agent`, `user_set_abstraction` falliscono con la chiave agent | Queste operazioni restano manuali e documentate nel runbook; il bot non le tenta mai | – |

## 2. Dati di mercato

| Errore previsto | Rilevazione | Prevenzione / reazione |
|---|---|---|
| Dati stantii | L'ultima candela 1h chiusa ha `T` più vecchio di 90 s; `time` di `clearinghouseState` lontano dall'ora locale | Nessuna decisione; alert se persiste 2 cicli |
| Candele mancanti o duplicate (downtime, riavvio) | Buchi nella sequenza `t` | `collect_history` riallinea per `t` (chiave primaria); le feature si calcolano sempre dal DB, mai da strutture in memoria |
| Candele senza trade (`n == 0`) di notte | Volume 0 → divisioni per zero negli indicatori | Guard sugli indicatori; fuori finestra non si decide comunque |
| Prezzo interno (weekend, festivi, overnight) | Oracolo che segue il book anziché il titolo | Sessioni calcolate in `America/New_York` con `zoneinfo` + calendario festività NYSE in `config/events.yaml`; **mai offset UTC fissi** (cambio ora: USA 1/11/2026, UE 25/10/2026: una settimana con offset diversi) |
| Book vuoto o spread anomalo | `l2Book` con livelli mancanti o spread > 3 bps | Gate fallisce → nessun trade |
| Indici degli asset spostati o mappa SDK vecchia | `name_to_asset("xyz:TSLA") ≠ 110001` | Ricreare `Info`/`Exchange` ogni giorno; assert sugli id attesi all'avvio |
| Cambio di schema dell'API | Campi nuovi, rinominati o mancanti | Modelli Pydantic con `extra="ignore"` e campi obbligatori minimi; healthcheck giornaliero di contratto (lettura) che fallisce rumorosamente |
| Split, dividendi straordinari, corporate action | Salto > 20 % tra due oracoli consecutivi con book coerente | Pausa sul simbolo + alert; calendario corporate action; nessuna posizione aperta nei giorni di split |

## 3. Claude Opus 5.5 (oracolo)

| Errore previsto | Rilevazione | Prevenzione / reazione |
|---|---|---|
| Timeout, 529 "overloaded", 429 | Eccezioni `APITimeoutError`, `InternalServerError`, `RateLimitError` | Timeout 300 s (streaming), 1 retry con backoff; poi nessuna analisi valida → nessun nuovo ingresso; le posizioni restano gestite da trigger on‑chain e dalle decisioni di uscita di Jev |
| Output troncato | `stop_reason == "max_tokens"` (il thinking ha consumato il budget) | `max_tokens` 64000 in streaming con effort `max` (`messages.stream` + `output_format`); un retry a 128000; se ricorre, rivedere il prompt |
| Refusal | `stop_reason == "refusal"` | `fallbacks: "default"` (beta `server-side-fallback-2026-07-01`); se anche il fallback rifiuta → nessuna analisi valida per quest'ora |
| JSON non valido o fuori schema | `messages.parse` solleva un errore di validazione | Nessuna analisi valida (Jev può solo chiudere), testo grezzo nel log, contatore; 3 errori consecutivi → alert "prompt rotto" |
| Analisi che cambia lettura ogni ora | Bias e scenari che oscillano | L'oracolo non decide: Jev vede l'analisi come un campo dello stato, con la sua età, accanto ai dati quantitativi; invertire una posizione richiede due decisioni Jev concordi a 15 minuti di distanza (isteresi nel codice); il cooldown dopo una perdita evita il ping‑pong |
| Numeri inventati o calcoli sbagliati | – | Nessun numero dell'oracolo entra direttamente in size o prezzi; le distanze di invalidazione sono clampate e combinate con l'ATR dal codice; le probabilità degli scenari vengono normalizzate |
| Costi che esplodono | `usage.cache_read_input_tokens == 0` per 3 chiamate; costo giornaliero > budget/22 | Alert "cache persa" (qualcosa di variabile è finito nel prefisso); budget mensile → passaggio automatico a `paper` |
| Crediti API esauriti (crediti inclusi nel piano Max o prepagati) | HTTP 400 con "Your credit balance is too low to access the Anthropic API" | Workspace Console dedicato con limite di spesa; alert quando il saldo scende sotto il 20 %; auto‑ricarica opzionale; alla risposta di errore: nessuna analisi valida → nessun nuovo ingresso, Jev gestisce solo le uscite |
| Modello deprecato o id errato | HTTP 404 `NotFoundError` | Il modello è un parametro di config; nessun nuovo ingresso + alert |
| "Memoria" del periodo nel backtest | Risultati storici troppo belli | Stati senza date né prezzi assoluti (rendimenti e distanze relative), split train/test per mese |
| Prompt injection | Non applicabile nella v1 (stato generato dal codice da numeri) | Se in v2 entrano notizie: filtro di rilevanza con Jev, troncamento, nessuna istruzione nel testo delle notizie |
| Latenza > 300 s | Timer del ciclo | A effort `max` le risposte possono durare 1–5 minuti; timeout 300 s, poi skip con log e nessuna analisi per quest'ora; la decisione delle hh:00 usa l'analisi precedente finché è valida |

## 4. Jev (decisore)

| Errore previsto | Rilevazione | Prevenzione / reazione |
|---|---|---|
| Nessuna chiave (waitlist early access) | `TYPESAFE_API_KEY` assente | Surrogato a regole con la stessa interfaccia; campo `model = "surrogate-v1"` nel log |
| 401 / 422 | `AuthenticationError`, `UnprocessableEntityError` (criteri malformati) | Test di contratto all'avvio con una domanda di prova; errore = bug di configurazione → nessun ingresso + alert immediato |
| 429 / 529 | `RateLimitError`, `InternalServerError` | `RetryPolicy` dell'SDK; se fallisce → modalità degradata (nessun ingresso, oppure surrogato con soglie +0,10: scelta in config) |
| Cambio di versione silenzioso | `response.model ≠ "jev-1.13.0"` | Modello pinnato; alert se diverso; le soglie sono tarate per versione |
| Bias verso la prima opzione | Risposte che cambiano con l'ordine delle opzioni | Opzioni mescolate ad ogni domanda; test mensile di stabilità (stesso stato, 3 ordini → stessa risposta ≥ 90 %) |
| Calibrazione che peggiora | Brier score mensile su `action` ed `exit_now` contro gli esiti | Se peggiora oltre soglia: alza le soglie di confidenza o pausa |
| Stato troppo lungo o nella lingua sbagliata | Accuratezza in calo ("context rot") | Stato < 1.500 token, in inglese, solo campi rilevanti, numeri già trasformati in etichette |
| Domande contraddittorie | Istruzioni e criteri non allineati | Domande versionate (`questions_v3.yaml`) con changelog e test sui casi limite |
| Latenza > 2 s | Timer | Guardia saltata; nel ciclo orario si attende fino a 5 s |
| Servizio chiuso o prezzi cambiati (startup in early access) | Errori persistenti; `usage` fuori budget | Il bot non dipende da Jev per proteggere le posizioni; surrogato; budget guard |

## 5. Infrastruttura e processo

| Errore previsto | Prevenzione / reazione |
|---|---|
| Crash o riavvio | `systemd`/Docker `restart: unless-stopped`; all'avvio: riconciliazione, ripristino dal DB, verifica che ogni posizione abbia stop e target |
| Due istanze in parallelo | Lock file + record "leader" nel DB con heartbeat; chiavi agent diverse per dev/paper/live |
| Disco pieno | `logrotate`, `VACUUM` mensile, alert spazio < 20 % |
| SQLite "database is locked" | Modalità WAL, un solo writer (coda), `timeout=30` |
| Orologio sfasato | `chrony`; se `time` del server differisce > 5 s: nessuna azione firmata, alert |
| Rete instabile o proxy | Timeout espliciti (5 s info, 10 s exchange), retry limitati, circuit breaker |
| Variabili d'ambiente mancanti o errate | Validazione all'avvio (Pydantic Settings): il processo non parte |
| Dipendenze che cambiano firma (SDK Hyperliquid) | Versioni pinnate + lockfile; CI con test di contratto in sola lettura contro l'API pubblica |
| Fuso orario del server | Container in UTC; sessioni calcolate in `America/New_York` |
| Perdita del VPS | Backup giornaliero del DB (rclone/S3), runbook di ripristino; nel frattempo proteggono i trigger on‑chain e `/flat` da telefono |

## 6. Sicurezza

| Errore previsto | Prevenzione / reazione |
|---|---|
| Chiave in un log o in un commit | Logger che maschera le stringhe `0x…`; `.gitignore`; `gitleaks` in pre‑commit; chiave agent senza prelievi; wallet dedicato |
| Comandi Telegram da estranei | Whitelist `chat_id`; `/flat` chiede conferma |
| Dipendenza compromessa | Versioni pinnate, `pip-audit` in CI, nessuna dipendenza esotica |
| Furto della chiave agent | Danno massimo = capitale di trading; revoca dall'app (nuovo `approveAgent`); scadenza impostata |
| Endpoint falsi (TypeSafe avverte di usare solo `api.typesafe.ai`) | Endpoint fissi nel codice, TLS verificato, nessun `base_url` da env in live |

## 7. Strategia e statistica

| Errore previsto | Prevenzione / reazione |
|---|---|
| Overfitting delle soglie sul backtest | Split train/test per mese, walk‑forward, massimo 3 iterazioni sul test |
| Look‑ahead | Stati costruiti solo da candele chiuse; test automatico `T < now` su ogni stato |
| Cambio di regime | Re‑validazione mensile; auto‑pausa se l'expectancy delle ultime 30 operazioni è < 0 con drawdown > 8 % |
| Costi > profitti | Report mensile "P&L netto − fee − funding − costi API"; negativo per 2 mesi → stop e revisione |
| Pochi trade per trarre conclusioni | Nessuna conclusione prima di 100 trade; intervalli di confidenza (bootstrap) nel report |
| Slippage reale > stimato | Confronto fill vs mid atteso; se > 2× per 10 trade → size ridotta o ordine limit al mid con attesa 30 s |

## 8. Piattaforma, legale, fiscale

| Errore previsto | Prevenzione / reazione |
|---|---|
| Geo‑block UE/Italia introdotto all'improvviso | Capitale piccolo; procedura per chiudere via API e prelevare; hash settimanale della pagina dei ToS → alert se cambia |
| Delisting del mercato o `haltTrading` | Settlement automatico a mark; la riconciliazione registra e il bot si ferma sul simbolo |
| Prezzi o limiti di Jev/Anthropic che cambiano | Budget guard + alert; modelli e prezzi unitari in config |
| Fisco | Export mensile CSV di fill e funding (`userFills`, `userFunding`), log conservati |

## 9. Errori umani

| Errore previsto | Prevenzione / reazione |
|---|---|
| Trade manuali sullo stesso account | La riconciliazione segnala "posizione non attesa" → safe mode; account usato solo dal bot |
| Modifica di config a caldo | Ricarica solo all'inizio del ciclo, con validazione; config versionata in git |
| Deploy di un prompt o di domande rotte | Eval su 50 stati registrati in CI prima del deploy (schema valido, distribuzione delle risposte, costo) |
| Avvio in live per sbaglio | `LIVE_CONFIRM` + `MAX_EQUITY_USDC` (principio 10) |

---

## 10. Come testiamo tutto questo

- **Unit test:** arrotondamenti, sizing, gate, policy, parser delle risposte, calendario sessioni (inclusa la settimana di cambio ora).
- **Test di contratto (giornalieri, sola lettura):** `perpDexs`, `meta` xyz, `candleSnapshot`, `l2Book`, una domanda di prova a Jev, una chiamata minima a Claude.
- **Integrazione su testnet (dex principale):** apertura, stop/target in batch, OCO, cancel, fill parziale, riconciliazione dopo riavvio.
- **Chaos test (in paper):** timeout di Claude, 429 di Jev, risposta malformata, rete giù subito dopo l'invio di un ordine, crash a metà ciclo, orologio sfasato. Criterio: nessuna posizione mai senza stop, nessun ordine duplicato, bot in safe mode con alert.
- **Replay:** rigiocare giornate storiche (inclusi earnings e festività) con esecuzione simulata.

## 11. Runbook (procedure manuali)

1. **Posizione senza stop:** `/flat` da Telegram, oppure dall'app chiudi la posizione; poi verifica `openOrders` e cancella i trigger residui.
2. **Bot bloccato in safe mode:** leggi l'ultimo evento `reconcile_mismatch`, allinea manualmente (chiudi/cancella dall'app), riavvia con `--reconcile-ack`.
3. **Rinnovo chiave agent:** nell'app crea un nuovo API wallet, aggiorna `HL_AGENT_PRIVATE_KEY`, riavvia, verifica con l'healthcheck, revoca la vecchia.
4. **Ripristino VPS:** nuovo server, `docker compose up`, ripristina l'ultimo backup del DB, avvia in `paper`, controlla la riconciliazione, poi `live`.
5. **Sospetto furto di chiave:** revoca l'agent dall'app, chiudi le posizioni, sposta i fondi su un wallet nuovo.
6. **Fermare tutto:** `/pause` (nessun nuovo ingresso) o `/flat` (chiude tutto e mette in pausa); `docker compose down` ferma il processo, i trigger on‑chain restano attivi.
