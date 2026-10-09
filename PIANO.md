# Piano dettagliato: trading bot su azioni "tokenizzate" di Hyperliquid con Claude Opus 5.5 (segnali) + Jev (decisioni)

> Stato: piano operativo, aggiornato al 9 ottobre 2026. Tutti i numeri su Hyperliquid sono stati verificati direttamente sull'API pubblica il 9/10/2026; i prezzi e i limiti di Jev vengono dalla documentazione pubblica di TypeSafe (early access, possono cambiare).
> Non è consulenza finanziaria. I perpetual con leva possono far perdere più del margine allocato.

---

## 0. Sintesi in 10 righe

1. **Cosa si costruisce.** Un bot Python con tre ruoli separati. **Claude Opus 5.5 è l'oracolo**: ogni ora, alla chiusura della candela 1h, produce un'analisi strutturata (scenari con probabilità, livelli chiave, caso long e caso short, invalidazioni, rischi) e **non decide**. **Jev (decision model di TypeSafe AI) è il decisore**: ogni 15 minuti riceve lo stato quantitativo etichettato più l'analisi dell'oracolo e decide azione, ingresso e uscita con probabilità calibrate. **Il codice** fa tutta la matematica (sizing, stop, limiti di rischio), esegue gli ordini e ha diritto di veto solo sulle regole di rischio.
2. **Dove si opera.** Sui perpetual su azioni USA del dex HIP-3 `xyz` (trade.xyz) di Hyperliquid: `xyz:TSLA`, `xyz:NVDA`, `xyz:AAPL`, `xyz:MSFT`, `xyz:GOOGL`, `xyz:AMZN`, `xyz:META` (leva max 20x, cross margin, "growth mode" con fee ridotte) più `xyz:SP500`. Sono **perpetual sul prezzo dell'azione**, non azioni tokenizzate in senso stretto: nessun dividendo, nessuna proprietà del titolo. Collaterale: USDC.
3. **Budget.** 100 € ≈ 110 USDC netti dopo i costi di on-ramp. Ordine minimo su Hyperliquid: 10 $ di nozionale. Si può operare con 1 posizione alla volta da 50–100 USDC di nozionale a leva effettiva ≤ 1x sul capitale (leva impostata 3x isolata solo per ridurre il margine bloccato).
4. **Cadenza oraria: costi API sotto controllo.** Con una chiamata Opus 5.5 all'ora durante la sessione USA il costo è di circa **3 $/mese per simbolo** (≈ 9 $ per TSLA, NVDA e SP500; ≈ 32 $ se si copre tutto il 24/5), contro i ~166 $/mese che sarebbe costata la cadenza a 5 minuti 24/7. Jev costa **~0,2 $/mese** anche con una "guardia" ogni 15 minuti. Il vero collo di bottiglia diventa la **validazione statistica**: a 1–2 trade al giorno servono mesi di paper per avere centinaia di trade, quindi la validazione principale è il **backtest sulle candele orarie storiche** (≈ 7 mesi disponibili dall'API, ≈ 20 $ di chiamate per un backtest completo).
5. **Aspettative realistiche.** Nell'esperimento pubblico Alpha Arena (Nof1, ottobre–novembre 2025) sei LLM hanno fatto trading reale su Hyperliquid con 10.000 $ ciascuno, **su perpetual crypto** (BTC, ETH, SOL, BNB, DOGE, XRP) a leva 10–20x: Claude Sonnet 4.5 ha chiuso a −30,8 %, GPT‑5 a −62,7 %, solo Qwen3 Max ha guadagnato (+22,3 %; cifre finali riportate da fonti secondarie, non verificate sul sito di Nof1). Non esistono esperimenti pubblici equivalenti sui perps azionari, ma la lezione vale lo stesso: il problema era il metodo (previsioni direzionali frequenti, a leva, senza un edge misurato), non l'asset. Sulle azioni ci sono differenze da tenere presenti: il perp **insegue** l'oracolo del titolo reale (TSLA scambia ~28 M$/giorno su Hyperliquid contro decine di miliardi sul Nasdaq), la volatilità è più bassa, e ci sono gap, earnings e dati macro a orari fissi. Per questo la cadenza oraria con filtri di sessione è più adatta dei 5 minuti. In Alpha Arena decideva l'LLM stesso, in testo libero, ad ogni ciclo e con leva 10–20x; qui l'LLM non decide, Jev decide con probabilità tipizzate e il codice impone i limiti: questo rimuove due delle tre cause di quei risultati (decisioni incoerenti e over‑trading a leva alta). La terza, l'assenza di un vantaggio predittivo dimostrato, resta aperta e si misura con backtest e paper: Jev giudica solo ciò che gli viene passato, non ha dati di mercato propri e non c'è evidenza pubblica che preveda i prezzi da solo. Un LLM da solo non è un vantaggio statistico: il valore del progetto sta nella disciplina di rischio, nei costi bassi e nella misurazione. Con 100 € l'obiettivo sensato è **imparare e validare il sistema senza perdere più di 20–30 €**, non "guadagnare".
6. **Jev è in early access** (waitlist su typesafe.ai). Il piano prevede un "surrogato" a regole con la stessa interfaccia, così lo sviluppo e il paper trading partono subito anche senza chiave Jev.
7. **Orari.** I perps `xyz` scambiano 24/7 (verificato: candele e trade anche sabato e domenica), ma il prezzo esterno dell'azione arriva solo 24/5 (dom 20:05 → ven 20:00 ET). Fuori da quella finestra il prezzo è "interno" (order book + EMA 30 min) con bande di scoperta: il bot **non apre posizioni** nel weekend/festivi e chiude tutto prima del venerdì sera. Tra un'analisi oraria e l'altra, Jev decide ogni 15 minuti (ingressi e uscite) senza chiamare Opus.
8. **Sicurezza.** Wallet dedicato con solo il capitale di trading, API wallet ("agent") che può fare ordini ma non prelievi, chiavi solo in variabili d'ambiente, kill‑switch a soglie di perdita.
9. **Roadmap.** 6 fasi in ~6–8 settimane: setup → dati storici e paper engine → segnali Claude → decisioni Jev → rischio, backtest storico ed esecuzione (paper ≥ 3 settimane) → live con 20–30 USDC, poi 100.
10. **Legale/fiscale (Italia).** Hyperliquid non geo‑blocca l'UE (blocca USA, Ontario e paesi sanzionati), ma i perps sono derivati in una zona grigia regolatoria (dibattito MiFID II vs MiCA, warning FCA nel Regno Unito). In Italia le plusvalenze su cripto‑attività sono tassate al 33 % dal 2026 (verifica con un commercialista).

---

## 1. Obiettivo, perimetro e vincoli

| Voce | Scelta | Perché |
|---|---|---|
| Capitale | 100 € → ~110 USDC su Hyperliquid | Dopo fee di acquisto e prelievo su Arbitrum (1–3 €) e gas (centesimi). Deposito minimo bridge: 5 USDC. |
| Venue | Hyperliquid, dex HIP‑3 `xyz` (trade.xyz) | 98 % del volume HIP‑3, 133 mercati, oracolo Pyth, USDC, API unica HyperCore. |
| Strumenti | Perps su TSLA, NVDA, AAPL, MSFT, GOOGL, AMZN, META, SP500 | Sono i mercati `xyz` a leva 20x–50x con cross margin abilitato e volumi giornalieri 10–140 M$. Le small cap `xyz` sono isolated‑only, 10x, poco liquide: escluse. |
| Cadenza | Oracolo Opus **ogni ora** (chiusura candela 1h); decisioni Jev **ogni 15 minuti** sull'ultima analisi valida | Scelta dell'utente per le azioni: meno rumore, 6–7 analisi per sessione USA, costi API bassi; Jev costa quasi nulla e può decidere spesso. Candele 1h disponibili dall'API (≈ 7 mesi di storico per richiesta). |
| Oracolo | Claude Opus 5.5 (`claude-opus-5-5`): ogni ora un'analisi JSON strutturata (scenari, livelli, caso long/short, invalidazioni), effort `low`; **non decide** | Richiesta dell'utente. Costo e latenza controllati con prompt caching e output corto. |
| Decisioni | Jev (`jev-1.13.0` pinnato) via `typesafe-sdk`: ogni 15 minuti decide azione, ingresso e uscita | Risposte tipizzate con probabilità e confidenza in 70–500 ms; 0,042 $/M token in input, output gratis. |
| Esecuzione | `hyperliquid-python-sdk` 0.24.0 con `perp_dexs=["xyz"]` | Supporto nativo HIP‑3 (`xyz:TSLA` → asset id 110001). |
| Modalità | `paper` (default) → `live` | Il live si attiva solo dopo i criteri di uscita della fase 4. |

**Cosa NON è nel perimetro:** market making, HFT, più venue, azioni tokenizzate spot (xStocks e simili su altre chain: spread e fee non compatibili con 100 € e trading intraday), ottimizzazione di portafoglio.

---

## 2. Fatti verificati (9 ottobre 2026)

### 2.1 Hyperliquid, dex `xyz`

| Fatto | Valore verificato | Fonte |
|---|---|---|
| Dex HIP‑3 attivi | `xyz` (133 asset), `flx`, `vntl`, `hyna`, `km`, `cash`, `para`, `mkts`, `io`, `abcd` | `POST /info {"type":"perpDexs"}` |
| Nome asset | `xyz:TSLA`, `xyz:NVDA`, `xyz:SP500`… (prefisso dex + `:`) | docs API "Info endpoint → perpetuals" |
| Asset id | `100000 + 10000 × indice_dex + indice_in_meta` → `xyz:TSLA` = 110001, `xyz:NVDA` = 110002, `xyz:SP500` = 110052 | docs "Asset IDs", confermato con l'SDK |
| Collaterale | USDC (`collateralToken: 0`) | `meta` con `dex:"xyz"` |
| Leva max / margine | TSLA, NVDA, AAPL, MSFT, GOOGL, AMZN, META: 20x, cross consentito. SP500: 50x. XYZ100: 30x. HOOD, PLTR, COIN, AMD, NFLX, TSM, ORCL…: 10x **isolated only** | `meta` con `dex:"xyz"` |
| Decimali size | TSLA/NVDA/AAPL/META/GOOGL/AMZN/MSFT: 3; SP500: 3; XYZ100: 4 | `meta` |
| Ordine minimo | 10 $ di nozionale (errore "minimum value of $10") | docs API "Error responses" |
| Fee base (tier 0) | Taker 0,045 %, maker 0,015 %; HIP‑3 con `deployerFeeScale 1.0` → ×2 = **0,09 % / 0,03 %**; i mercati principali `xyz` sono in **growth mode** (riduzione ≥ 90 %, taker all‑in ~0,0045–0,009 %). Verifica il valore reale con `userFees` dopo il primo trade. | docs "Fees", `meta` (`growthMode: enabled`) |
| Funding | Moltiplicatore 0,5 su `xyz`; osservato 0,0006 %/h su TSLA/NVDA/AAPL (≈ 5 %/anno) | docs trade.xyz, `metaAndAssetCtxs` |
| Spread/profondità TSLA (ven 20:50 UTC) | Spread 0,8 bps; top‑5 book ≈ 33 k$ bid / 7,7 k$ ask | `l2Book` |
| Volumi 24h (9/10) | XYZ100 182 M$, SP500 141 M$, NVDA 81 M$, TSLA 28 M$, GOOGL 27 M$, AAPL 21 M$, META 31 M$, MSFT 11 M$ | `metaAndAssetCtxs` |
| Volatilità e volume orari (15/3 → 9/10/2026, 5.000 candele 1h) | ATR(14) 1h mediano: TSLA 0,48 % (range orario mediano in sessione USA 0,97 %), NVDA 0,44 % (0,81 %), AAPL 0,31 % (0,57 %), SP500 0,17 % (0,27 %). Volume orario mediano in sessione: TSLA 1,8 M$, NVDA 3,4 M$, AAPL 0,7 M$, SP500 18,5 M$; di notte 7–8× più basso. | `candleSnapshot` 1h |
| Weekend | Candele 5m presenti sab 3/10 e dom 4/10 con trade reali (TSLA: 3,4 k unità sabato vs 98 k lunedì; SP500 molto più attivo) | `candleSnapshot` |
| Prezzo esterno | 24/5, da domenica 20:05 a venerdì 20:00 ET (pre, regular, post, overnight). Fuori: oracolo "interno" = ultimo prezzo esterno + EMA 30 min dell'impact price del book, con bande di scoperta ±(1/leva max) | docs trade.xyz "Specification index", "Architecture" |
| Mark price | Mediana di: oracolo, oracolo+basis EMA 150 s, mediana(best bid, best ask, last) | docs trade.xyz |
| Rate limit IP | 1200 "peso"/min; `l2Book`, `allMids`, `clearinghouseState` pesano 2; `candleSnapshot` e `metaAndAssetCtxs` 20 (+ extra ogni 60 candele) | docs "Rate limits" |
| Rate limit indirizzo | 10.000 azioni iniziali + 1 ogni 1 USDC scambiato; a limite 1 azione/10 s | docs "Rate limits" |
| Deposito | Bridge Arbitrum (USDC nativo, **non** USDC.e), minimo 5 USDC, accredito < 1 min; prelievo ~1 USDC di fee (fonte terza) | docs "USDC", guide |
| Modalità account | Standard (saldi separati per dex: serve `sendAsset` da `""` a `"xyz"`) oppure **Unified account** (azione `userSetAbstraction: "unifiedAccount"`, nell'SDK `user_set_abstraction`; un saldo condiviso tra dex; limite 50 k azioni/giorno). "Dex abstraction" è deprecata. | docs "Account abstraction modes", SDK 0.24.0 |
| Testnet | Ha dex di prova (`test`, `unit`, …) ma **non `xyz`**: la testnet serve solo a provare la meccanica ordini sul dex principale | `api.hyperliquid-testnet.xyz` |
| Geografia | ToS: vietato per USA, Ontario, paesi sanzionati; UE raggiungibile senza KYC (test da Italia riportato da FinTelegram); warning FCA UK; richiesta MiFID II in revisione MiCA (ottobre 2026) | fonti in §16 |

### 2.2 Jev (TypeSafe AI)

| Fatto | Valore |
|---|---|
| Cos'è | Modello "System One": non genera testo, risponde a domande tipizzate su uno *stato* che gli passi. Rilascio early access 15/9/2026, accesso via waitlist su typesafe.ai; chiavi su console.typesafe.ai. |
| Endpoint | `POST https://api.typesafe.ai/v1/systemone`, header `Authorization: Bearer $TYPESAFE_API_KEY` |
| SDK | Python `pip install typesafe-sdk` (`TypeSafeClient`, `AsyncTypeSafeClient`, `RetryPolicy`); JS `@typesafe-ai/sdk` |
| Modelli | `jev-latest`, `jev-preview`, pin `jev-1.13.0` (loggare sempre il campo `model` della risposta) |
| Domande | **Choice** (1 opzione tra ≤ 255, con `probabilities` + `confidence`), **Score** (2–10 livelli ordinati, `score` + `probabilities` + `confidence`), **Noul** (probabilità 0–1 che un'affermazione sia vera) |
| Batching | Più domande nella stessa chiamata, valutate in parallelo e in modo indipendente |
| Prezzo | 0,042 $/M token in input, output gratis (early access) |
| Limiti | 1.200 richieste/min, 250 k token/s; contesto 64 k (32 k per stato + domanda più lunga) |
| Latenza | 70–500 ms dichiarati |
| Limiti noti di `jev-1.13` | Niente aritmetica, conteggi e date; lettura letterale delle istruzioni; bias verso la **prima opzione** di una Choice; accuratezza cala con stato lungo o rumoroso; inglese molto meglio delle altre lingue; non genera testo. Workaround ufficiali: codice fa i conti, stato piccolo ed etichettato, opzioni mescolate, domande dirette. |
| Confidenza | `confidence` = 1 se tutta la probabilità è su un'opzione, 0 se uniforme. Linee guida: sotto 0,5 non agire; sopra 0,9 per azioni "distruttive". |

### 2.3 Claude Opus 5.5

| Fatto | Valore |
|---|---|
| Modello | `claude-opus-5-5`, contesto 1 M, output fino a 128 k |
| Prezzo | 4 $/M input, 20 $/M output, cache read 0,20 $/M (scrittura cache 1,25×) |
| Thinking | Sempre attivo (adattivo); si controlla con `output_config.effort` (`low`…`max`, default `medium`). Per segnali orari: `low` (oppure `medium`, ≈ 1,8× il costo, se si vuole più analisi). |
| Output strutturato | `client.messages.parse(..., output_format=ModelloPydantic)` oppure `output_config.format = json_schema` |
| Caching | `cache_control` sul system prompt (stabile) → paga pieno solo lo stato di mercato variabile |
| Fallback | In produzione aggiungere `fallbacks: "default"` con beta `server-side-fallback-2026-07-01` (gestione dei rari `stop_reason: "refusal"`) |

---

## 3. Architettura

```
          ┌──────────────────────────────────────────────────────────────────────┐
          │  SCHEDULER: oracolo ogni ora (hh:00 +5 s); decisione ogni 15 minuti  │
          └──────────────────────────────┬───────────────────────────────────────┘
                                         ▼
┌───────────────────────── LIVELLO 1: CODICE (deterministico) ─────────────────────────┐
│  Hyperliquid Info API ──► candele 1h/4h/15m, l2Book, metaAndAssetCtxs, posizione,   │
│  Feature engine: EMA/VWAP/RSI/ATR, rendimenti, volume relativo, spread, profondità, │
│                  funding, OI, sessione (pre/regular/post/overnight/weekend), eventi │
│  State builder:  (a) testo compatto con numeri per l'oracolo                        │
│                  (b) JSON piccolo con ETICHETTE per il decisore, incluso il blocco  │
│                      "oracle" (analisi tradotta in etichette + età dell'analisi)    │
└───────────────┬──────────────────────────────────────────────────────────────────────┘
                ▼  (ogni ora)
┌──────── LIVELLO 2: ORACOLO = CLAUDE OPUS 5.5 ───────────────────────────────────────┐
│  Legge lo stato, ragiona, restituisce un'ANALISI JSON: scenari 4h con probabilità,  │
│  bias, livelli chiave, caso long / caso short, invalidazioni, rischi, cosa cambierebbe│
│  la lettura. NON decide, NON dimensiona. (effort low, cache, ~10 s, ~0,02 $)         │
└───────────────┬──────────────────────────────────────────────────────────────────────┘
                ▼  (ogni 15 minuti, finché l'analisi è valida)
┌──────── LIVELLO 3: DECISORE = JEV ───────────────────────────────────────────────────┐
│  Stato etichettato + blocco "oracle" → in < 0,5 s risponde a domande tipizzate:      │
│  action (long/short/flat ×3, opzioni mescolate), setup_quality, risk_ok, exit_now,  │
│  oracle_consistent, regime. La direzione la decide SOLO Jev.                        │
└───────────────┬──────────────────────────────────────────────────────────────────────┘
                ▼
┌──────── LIVELLO 4: RISCHIO + ESECUZIONE (codice) ────────────────────────────────────┐
│  Veto solo su regole di rischio: sessione, eventi, spread/profondità, limiti di     │
│  perdita, drawdown, cooldown, numero di trade, soglie di confidenza di Jev          │
│  Sizing: nozionale, stop (ATR + invalidazione dell'oracolo), target, time‑stop      │
│  Esecuzione: ingresso + stop + target nello stesso batch (normalTpsl), cloid        │
│  Riconciliazione ad ogni ciclo; SQLite + log JSONL + Telegram + kill‑switch          │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

**Principio guida:** *il codice fa i conti, Opus analizza, Jev decide, il codice esegue e può porre il veto solo sul rischio*. Nessun modello calcola stop, size o P&L; nessun modello invia ordini; l'oracolo non ha mai l'ultima parola sulla direzione.

**Perché questa divisione.** Un LLM è bravo a sintetizzare contesto e a scrivere un'analisi, ma come decisore è incoerente: la stessa situazione descritta in modo leggermente diverso produce decisioni diverse, e in testo libero tende a "raccontare" più che a scegliere. Jev è costruito per l'opposto: risponde solo a domande chiuse, con probabilità calibrate e confidenza, sempre nello stesso formato, in meno di mezzo secondo e a costo quasi nullo, quindi può decidere spesso (ogni 15 minuti) senza che il costo esploda. Quello che Jev **non** aggiunge è un vantaggio predittivo proprio: giudica solo ciò che riceve. Per questo l'oracolo deve dargli un'analisi ricca (scenari, livelli, casi contrapposti) e il codice deve dargli uno stato quantitativo pulito; e per questo la validazione misura separatamente il contributo di ciascuno (§12).

---

## 4. I due cicli: oracolo ogni ora, decisione ogni 15 minuti

### 4.1 Ciclo dell'oracolo (hh:00 + 5 s, solo nella finestra operativa; obiettivo < 60 s)

| Passo | Cosa succede | Tempo | Costo |
|---|---|---|---|
| 0 | Scheduler scatta alla chiusura della candela 1h (orologio NTP). Se il ciclo precedente è ancora in corso → skip e alert. | – | – |
| 1 | Riconcilia stato reale: `clearinghouseState(dex="xyz")`, `openOrders`, `userFills`. Se la posizione reale ≠ attesa → safe mode. | 1 s | peso 2–20 |
| 2 | Dati: candele 1h (ultime 72), 4h (ultime 60), 15m (ultime 16), `l2Book`, `metaAndAssetCtxs`. | 2 s | peso ~60 |
| 3 | Gate "hard": prezzo esterno attivo, sessione (preferenza 9:30–16:00 ET), nessun evento ±60 min, spread ≤ 3 bps, profondità ≥ 20× size, limiti di perdita non raggiunti. Se fallisce → nessuna chiamata a Opus, l'analisi precedente viene marcata non valida. | 0 s | 0 |
| 4 | **Oracolo Opus 5.5** (§6): stato compatto → `AnalisiOracolo` JSON. Timeout 60 s, 2 retry. Analisi valida per 60 minuti o finché il prezzo non supera un livello di invalidazione. | 5–20 s | ~0,02 $ |
| 5 | Persistenza dell'analisi (con `usage`) e traduzione in etichette per il blocco `oracle` dello stato di Jev. Poi parte subito il ciclo di decisione. | 0,5 s | – |

### 4.2 Ciclo di decisione (hh:00, :15, :30, :45; obiettivo < 10 s)

| Passo | Cosa succede | Tempo | Costo |
|---|---|---|---|
| 1 | Riconcilia stato reale; nuovi fill → aggiorna posizione e P&L in R. | 1 s | peso 2–20 |
| 2 | Dati leggeri: ultime candele 15m, `l2Book`, `allMids(dex="xyz")`; aggiorna le etichette quantitative. | 1 s | peso ~25 |
| 3 | Gate "hard" (come sopra) + età dell'analisi: se non esiste un'analisi valida → **nessun nuovo ingresso** (le uscite vengono comunque valutate). | 0 s | 0 |
| 4 | **Decisore Jev** (§7): stato etichettato + blocco `oracle` + posizione → `action` ×3, `setup_quality`, `risk_ok`, `exit_now`, `oracle_consistent`, `regime`. | 0,5 s | ~0,0001 $ |
| 5 | Policy (codice, §7.3): applica **solo** le soglie di confidenza di Jev e l'isteresi. Esito: `OPEN_LONG` / `OPEN_SHORT` / `CLOSE` / `HOLD` / `NOOP`. | 0 s | 0 |
| 6 | Risk engine: size dal rischio per trade, stop = max(0,8 %, 1,5 × ATR 1h, invalidazione dell'oracolo), target 1,5 R, leva 3x isolata, nozionale ≥ 10 $ e multipli dei decimali. **Veto** se una regola di §5.2 non è rispettata. | 0 s | 0 |
| 7 | Esecuzione (`live`) o simulazione (`paper`): ingresso IOC (slippage max 0,3 %) + stop + target nello stesso batch con `cloid`. Verifica fill e trigger. | 1–3 s | 1–3 azioni |
| 8 | Persistenza: stato, risposte Jev (con `model`), decisione, ordini, costi. Alert Telegram su apertura/chiusura/errori. | 0,5 s | – |

Quando non c'è posizione e il gate fallisce, il ciclo di decisione non chiama Jev. Quando c'è una posizione, Jev viene sempre interrogato (almeno `exit_now`), anche fuori finestra, finché la posizione non è chiusa. Costo totale di Jev con 4 decisioni l'ora in sessione: ~0,2 $/mese.

### 4.3 Il feed: chi produce cosa

Il "feed" è un file JSON per simbolo e per ciclo (`state.json`), unica fonte di verità per Jev, per il backtest e per la futura interfaccia grafica. Lo costruisce il codice; Opus lo arricchisce; Jev lo legge; nessuno dei due modelli scarica dati o scrive domande.

| Passo | Chi | Quando | Cosa produce |
|---|---|---|---|
| 1 | Codice | ogni 15 min | `quant.json`: candele 1h/4h/15m, indicatori, book, funding, OI, sessione, eventi, posizione e P&L in R, già tradotti in etichette (§7.1) |
| 2 | Opus (oracolo) | ogni ora | `oracle.json`: scenari 4 h con probabilità, bias, livelli, caso long e caso short, invalidazioni, rischi (§6.2), calcolati **a partire da `quant.json`**, mai da dati scaricati da Opus |
| 3 | Codice | ogni 15 min | `state.json` = `quant` + `oracle` (con età e validità) + `position`; salvato nel log degli eventi |
| 4 | Jev (decisore) | ogni 15 min | Risposte alle **domande fisse e versionate** (`questions_vN.yaml`) su `state.json`: azione ×3, qualità del setup, rischio, uscita, coerenza dell'oracolo, regime (§7.2) |
| 5 | Codice | ogni 15 min | Soglie e veti (§7.3), size e stop (§5.2), ordini (§8), persistenza e alert |

Tre regole che rendono il feed affidabile:

- **I dati li scarica e li calcola il codice.** Un feed costruito dal codice è deterministico, costa zero, arriva in un secondo e si rigioca sui 7 mesi di storico. Se li scaricasse Opus con strumenti di rete: 3–10× più token e latenza per ciclo, errori di intervallo o di campo, aritmetica inaffidabile sugli indicatori, nessuna riproducibilità (quindi niente backtest) e una superficie di attacco in più (contenuti esterni che condizionano l'output).
- **Le domande a Jev sono fisse e versionate.** Se cambiassero ad ogni ciclo, le risposte non sarebbero confrontabili, le soglie non si potrebbero tarare e la calibrazione non si potrebbe misurare. Opus può aiutare a scriverle una volta, offline (come nel cookbook TypeSafe in cui un LLM genera la specifica delle domande), ma non le cambia in produzione. Ciò che Opus può variare ogni ora è il **contenuto** dello stato, ad esempio "condizione long: prezzo sopra VWAP, soddisfatta", che il codice trasforma in un'etichetta.
- **La previsione di Opus è un campo tra gli altri.** Se Jev decidesse solo in base alla previsione di Opus, sarebbe un formattatore dell'opinione dell'LLM: lo schema di Alpha Arena con un involucro tipizzato. Jev aggiunge valore perché vede anche lo stato quantitativo, la posizione e i rischi, e può dissentire; per questo la previsione è probabilistica (tre scenari), non una direzione.

**Estensione v2:** una ricerca a bassa frequenza in cui Opus, una volta al giorno e con ricerca web, produce `context.json` (earnings, eventi macro, catalizzatori dei titoli dell'universo) da rivedere a mano e aggiungere al feed come etichette. Costa pochi centesimi al giorno ed è l'unico caso in cui ha senso che sia Opus a "scaricare" qualcosa.

---

## 5. Universo e regole di rischio con ~110 USDC

### 5.1 Universo
- **Core:** `xyz:TSLA`, `xyz:NVDA` (volume e volatilità intraday), `xyz:SP500` (liquido anche nel weekend, utile come regime di mercato).
- **Secondari (attivabili dopo il paper):** `xyz:AAPL`, `xyz:MSFT`, `xyz:GOOGL`, `xyz:AMZN`, `xyz:META`.
- Un solo simbolo per ciclo viene "promosso" al passo 5 (quello con il miglior punteggio pre‑filtro), per non moltiplicare le chiamate Opus.

### 5.2 Regole hard (nel codice, non nei prompt)

| Regola | Valore iniziale | Nota |
|---|---|---|
| Posizioni aperte contemporanee | 1 | Con 110 USDC due posizioni rendono il sizing insignificante. |
| Nozionale per posizione | 50–100 USDC (≈ 0,5–0,9× equity) | Sopra il minimo di 10 $, sotto i limiti di profondità del book. |
| Leva impostata | 3x, **isolated** (`update_leverage(3, coin, is_cross=False)`) | Margine bloccato 17–33 USDC; la perdita massima resta limitata al margine anche in caso di gap. |
| Rischio per trade | ≤ 1,5 % dell'equity (≈ 1,6 USDC) | Stop = max(0,8 %, 1,5 × ATR(14) 1h) dal prezzo di ingresso (per TSLA ≈ 0,8–1,1 %); la size si ricava dal rischio, non viceversa. |
| Target | 1,5–2 R oppure uscita Jev (`exit_now ≥ 0,7`) | Partial take‑profit non ha senso con queste size. |
| Time‑stop | 4 candele 1h senza progresso (< +0,3 R) → chiudi; chiusura forzata alle 15:45 ET | v1 intraday: niente overnight (gap all'apertura). In v2, dopo il backtest, overnight consentito con size dimezzata solo con prezzo esterno attivo. |
| Finestra operativa | Solo con prezzo esterno attivo (dom 20:05 → ven 20:00 ET); ingressi preferibilmente 9:30–16:00 ET; **flat entro ven 19:30 ET** | Weekend: liquidità al 3–5 % e prezzo interno con bande di scoperta. |
| Eventi | Nessun ingresso ±60 min da earnings del titolo, FOMC, CPI, NFP; nessuna posizione aperta attraverso gli earnings | Calendario mantenuto in `config/events.yaml`. |
| Limite perdita giornaliera | 4 % equity → stop fino a mezzanotte UTC | |
| Limite settimanale | 8 % → stop fino a lunedì + revisione | |
| Max drawdown | 15 % dal picco → bot fermo, serve riavvio manuale | |
| Max trade/giorno | 2; cooldown 2 ore dopo una perdita; nessun nuovo ingresso dopo le 14:00 ET | Con 6–7 decisioni per sessione, 1–2 trade al giorno è il massimo sensato. |
| Qualità esecuzione | Spread ≤ 3 bps, top‑5 book ≥ 20× size, slippage IOC ≤ 0,3 % | Altrimenti nessun ordine. |
| Costi API | Budget mensile configurabile (es. 40 $); al superamento il bot passa a `paper` | Contatore su `usage` di Claude e `input_tokens` di Jev. |

**Esempio numerico.** Equity 110 USDC, TSLA a 382,6 $, ATR(14) 1h 0,48 % → stop = max(0,8 %, 0,72 %) = 0,8 %. Rischio 1,5 % = 1,65 USDC → nozionale = 1,65 / 0,008 = 206 USDC, ma il cap è 100 USDC → size = 100 / 382,6 = 0,261 → arrotondata a 3 decimali `0.261` (nozionale 99,9 $). Rischio effettivo 0,80 USDC (0,73 %). Fee andata/ritorno in growth mode ≈ 0,02 USDC, nominali ≈ 0,18 USDC; spread ≈ 0,01 USDC. Target 1,5 R = +1,2 % ≈ 1,20 USDC. Con queste cifre servono **≥ 41 % di trade vincenti a 1,5 R con le fee growth mode, ≥ 49 % con quelle nominali** solo per pareggiare: la precisione del sistema va misurata su centinaia di trade (backtest storico + paper) prima del live.

### 5.3 Come si usa la leva (e cosa non fa)

Su Hyperliquid la leva ha due significati separati, e il bot li tratta in modo diverso.

**Leva impostata** (il selettore 1x–20x su TSLA): decide solo quanto collaterale viene bloccato e dove sta la liquidazione. Il margine richiesto è nozionale ÷ leva; la liquidazione scatta quando margine + P&L non realizzato scende sotto il margine di mantenimento, che è metà del margine iniziale alla leva massima (TSLA: 2,5 % del nozionale; SP500: 1 %; titoli a 10x: 5 %). Posizione da 100 USDC su TSLA a 382,6 $ (formula ufficiale `liq = px − side · margin_available / size / (1 − l · side)`):

| Leva impostata | Margine bloccato | Liquidazione long | Liquidazione short |
|---|---|---|---|
| 1x | 99,9 USDC | −100 % | +95 % |
| **3x (scelta)** | **33,3 USDC** | **−31,6 %** | **+30,1 %** |
| 5x | 20,0 USDC | −17,9 % | +17,1 % |
| 10x | 10,0 USDC | −7,7 % | +7,3 % |
| 20x | 5,0 USDC | −2,6 % | +2,4 % |

Con stop allo 0,8–1,1 % la liquidazione a 3x è trenta volte più lontana dello stop: è solo la rete di sicurezza estrema. A 20x una normale oscillazione oraria liquida prima che lo stop lavori. Perché 3x e non 1x: a 1x si bloccherebbero 100 dei 110 USDC in una sola posizione, con il rischio di "margine insufficiente" per fee, funding e perdite non realizzate; a 3x restano 77 USDC liberi. Perché isolata: in uno scenario estremo (titolo sospeso che riapre a −40 %) la perdita si ferma al margine della posizione.

**Leva effettiva** (esposizione ÷ capitale): è l'unica che conta per il rischio, e non è un obiettivo ma una conseguenza di rischio per trade e distanza dello stop (nozionale = rischio ÷ stop). Con 110 USDC:

| Rischio per trade | Stop | Nozionale | Leva effettiva |
|---|---|---|---|
| 1,0 % (1,10 USDC) | 0,8 % | 138 USDC | 1,2x |
| 1,5 % (1,65 USDC) | 0,8 % | 206 USDC | 1,9x |
| 1,5 % (1,65 USDC) | 1,0 % | 165 USDC | 1,5x |
| 1,5 % (1,65 USDC) | 0,5 % | 330 USDC | 3,0x |

Il rischio per trade resta identico in tutte le righe; la leva effettiva serve solo a usare stop più stretti senza ridurre il rischio sotto una soglia irrilevante. Nella v1 il tetto è 100 USDC di nozionale (0,9x). Dopo il backtest il tetto può salire fino a ~2x, mai oltre; per eventuali posizioni overnight scende a 0,5x. Il funding (0,0006 %/h su TSLA, ≈ 5 %/anno sul nozionale) raddoppia a 2x ma su 4 ore vale millesimi; le fee crescono in proporzione al nozionale.

**In pratica:** all'avvio, da flat, `update_leverage(3, coin, is_cross=False)` per ogni simbolo; dopo ogni apertura il bot verifica in `user_state` tipo `isolated` e valore 3; se trova una leva oltre 5x chiude la posizione. La leva impostata non cambia mai durante il trading: cambia la size, non la leva. `update_isolated_margin` esiste per aggiungere margine a una posizione, ma con questi stop non serve.

---

## 6. Design dell'oracolo Claude Opus 5.5

### 6.1 Cosa riceve (≈ 1.000–1.500 token variabili + system prompt ≈ 2.000 token in cache)
- **System prompt (stabile, in cache):** ruolo di *analista di ricerca* intraday su perps azionari; regola fondamentale: **descrivere, non decidere** (niente "compra/vendi", niente size); orizzonte di analisi 1–4 ore; definizione degli scenari (up = chiusura oltre +0,8 % entro 4 h, down = sotto −0,8 %, range = né l'uno né l'altro); divieti (non inventare notizie o livelli non supportati dai dati, non fare aritmetica sui prezzi: le distanze sono in percentuale e vengono ricalcolate dal codice); descrizione dei campi dello stato; formato di output.
- **Stato (variabile, in ordine stabile, senza timestamp inutili):** simbolo; sessione e minuti alla chiusura USA; ultime 24 candele 1h (o,h,l,c,v compatte) + ultime 4 candele 15m + riassunto 4h/1g (trend, range, VWAP di sessione, EMA 20/50/200, RSI 14, ATR %, volume relativo, distanza da apertura/massimo/minimo di giornata); mark vs oracle (premium), funding, variazione OI 1h; spread e profondità; posizione aperta (lato, P&L in R, ore); esito delle ultime 3 decisioni (feedback); flag calendario.
- **Niente notizie in tempo reale** nella v1: aggiungerle costa (web search) e introduce rumore; valutare in v2 con un filtro Jev di rilevanza.

### 6.2 Cosa restituisce (schema Pydantic → `messages.parse`)
```python
from pydantic import BaseModel, Field
from typing import Literal, List

class Scenari4h(BaseModel):
    up: float = Field(ge=0, le=1)        # P(chiusura > +0,8 % entro 4 h)
    down: float = Field(ge=0, le=1)      # P(chiusura < -0,8 % entro 4 h)
    range: float = Field(ge=0, le=1)     # il codice normalizza la somma a 1

class AnalisiOracolo(BaseModel):
    regime_view: Literal["trend_up", "trend_down", "range", "volatile_chop", "unclear"]
    scenari_4h: Scenari4h
    bias: Literal["bullish", "bearish", "neutral"]
    support_pct: float = Field(ge=0, le=5)             # distanza % del supporto chiave sotto il prezzo
    resistance_pct: float = Field(ge=0, le=5)          # distanza % della resistenza chiave sopra il prezzo
    long_case: str = Field(max_length=200)             # in inglese: perché un long potrebbe funzionare
    short_case: str = Field(max_length=200)            # in inglese: perché uno short potrebbe funzionare
    long_invalidation_pct: float = Field(gt=0, le=3)   # sotto questa distanza % il caso long è falso
    short_invalidation_pct: float = Field(gt=0, le=3)  # sopra questa distanza % il caso short è falso
    risk_flags: List[Literal["event_soon", "thin_book", "extended_move", "funding_extreme", "session_edge", "none"]]
    what_changes_view: str = Field(max_length=160)     # cosa invaliderebbe l'intera lettura
```
Non c'è un campo "direzione" né una "convinzione": l'oracolo espone entrambi i casi con le rispettive invalidazioni e le probabilità degli scenari. La scelta spetta a Jev.

### 6.3 Chiamata (Python, SDK `anthropic` ≥ 1.13)
```python
import anthropic
client = anthropic.Anthropic()  # ANTHROPIC_API_KEY dall'ambiente

SYSTEM = [{"type": "text", "text": SYSTEM_PROMPT,            # testo stabile
           "cache_control": {"type": "ephemeral"}}]           # cache → ~0,20 $/M invece di 4 $/M

resp = client.messages.parse(
    model="claude-opus-5-5",
    max_tokens=4000,                       # include il thinking adattivo
    output_config={"effort": "low"},       # thinking sempre attivo su Opus 5.5: effort è la leva di costo
    system=SYSTEM,
    messages=[{"role": "user", "content": stato_testo}],
    output_format=AnalisiOracolo,
)
analisi = resp.parsed_output
log(usage=resp.usage)   # input_tokens, cache_read_input_tokens, output_tokens → contatore costi
```
Note operative: verifica che `cache_read_input_tokens > 0` dal secondo ciclo (se è 0 c'è qualcosa di variabile nel prefisso); un `stop_reason == "max_tokens"` o `"refusal"` → nessuna analisi valida per quest'ora (Jev non apre, può solo chiudere); in produzione abilita `fallbacks: "default"` (beta `server-side-fallback-2026-07-01`) per i rari refusal.

### 6.4 Traduzione dell'analisi per Jev (codice)
Il codice converte l'analisi in etichette e aggiunge l'età: `"oracle": {"age": "fresh (12 minutes)", "bias": "bullish", "scenarios": "up 55%, range 30%, down 15%", "levels": "support 0.6% below, resistance 1.1% above", "long_case": "...", "short_case": "...", "invalidation": "long case fails 0.7% below; short case fails 1.0% above", "risk_flags": "none"}`. Se il prezzo supera un livello di invalidazione prima della scadenza, il blocco diventa `"status": "invalidated"` e Jev lo sa.

### 6.5 Controllo costi
- Effort `low`, `max_tokens` 4000, stato compatto, system prompt in cache: ≈ 0,02 $/chiamata, 7 chiamate per sessione USA per simbolo.
- Fuori finestra operativa il bot gira ma non chiama Opus.
- Contatore mensile con soglia (default 20 $) → oltre, passaggio automatico a `paper` + alert.

---

## 7. Design delle decisioni Jev

### 7.1 Stato per Jev: piccolo, etichettato, in inglese (Jev non fa aritmetica)
```json
{
  "symbol": "TSLA perpetual",
  "session": "US regular hours, 94 minutes to close",
  "trend_1h": "up, price above VWAP and EMA20",
  "trend_15m": "pullback of 0.4% into EMA20 in the last 3 candles",
  "momentum": "RSI 58, neutral-bullish",
  "volatility": "ATR normal (0.5% per hour); 4-hour range 1.4%",
  "volume": "last hour volume is 1.4x the session average",
  "order_book": "spread tight (0.8 bps), bid depth 4x ask depth",
  "derivatives": "mark at oracle, funding neutral, open interest rising",
  "event_risk": "no scheduled events in the next 2 hours",
  "position": "flat",
  "oracle": {"age": "fresh (12 minutes)", "bias": "bullish",
             "scenarios": "up 55%, range 30%, down 15%",
             "levels": "support 0.6% below, resistance 1.1% above",
             "long_case": "Trend continuation after a shallow pullback into VWAP with rising volume",
             "short_case": "Rejection at yesterday's high with momentum fading",
             "invalidation": "long case fails 0.7% below; short case fails 1.0% above",
             "risk_flags": "none"},
  "recent_performance": "last 3 trades: +1.2R, -1.0R, +0.4R"
}
```
Tutti i numeri vengono convertiti dal codice in bucket descrittivi ("tight", "normal", "1.4x average"). Nessun calcolo è lasciato a Jev. L'analisi dell'oracolo è **un campo dello stato**, non un ordine.

### 7.2 Domande (una sola chiamata, valutate in parallelo)
La versione di riferimento delle domande, con criteri completi, vocabolario dello stato, soglie della policy e casi di test, è in [`config/questions_v1.yaml`](config/questions_v1.yaml): il bot la carica all'avvio e la traduce negli oggetti dell'SDK, come mostrato qui sotto. Jev riceve solo il JSON risultante.
```python
from typesafe_sdk import TypeSafeClient, Choice, Score, Noul
import random

jev = TypeSafeClient()   # TYPESAFE_API_KEY dall'ambiente; pinna "jev-1.13.0" (parametro model, vedi doc SDK "Choosing a model")

def q_action():          # opzioni mescolate ad ogni domanda: Jev tende a preferire la prima
    opts = {"long": "Open or hold a long position now",
            "short": "Open or hold a short position now",
            "flat": "Stay out, or close the open position"}
    items = list(opts.items()); random.shuffle(items)
    return Choice(instructions="Given the quantitative state and the oracle analysis, what is the best action for the next 2 to 4 hours?",
                  criteria=dict(items))

r = jev.system_one(state=stato_jev, questions={
    "action_1": q_action(), "action_2": q_action(), "action_3": q_action(),
    "setup_quality": Score(instructions="Rate the quality of the trading opportunity in the chosen direction.",
                           criteria=["Poor", "Weak", "Acceptable", "Good", "Excellent"]),
    "risk_ok": Noul(instructions="Is opening a new position acceptable from a risk standpoint?",
                    criteria={"true": "Session, liquidity and event risk are acceptable",
                              "false": "Any risk flag makes a new position unacceptable"}),
    "exit_now": Noul(instructions="If a position is open, should it be closed now?",
                     criteria={"true": "The case for the position has failed or risk is rising", "false": "The case is intact"}),
    "oracle_consistent": Noul(instructions="Is the oracle analysis consistent with every quantitative field of the state?",
                              criteria={"true": "No field contradicts the analysis", "false": "At least one field contradicts it"}),
    "regime": Choice(instructions="What market regime best describes the state?",
                     criteria={"trend_up": "Higher highs, price above VWAP/EMAs",
                               "trend_down": "Lower lows, price below VWAP/EMAs",
                               "range": "Sideways inside a defined band",
                               "volatile_chop": "Large, directionless swings"}),
})
log(jev_model=r.model, usage=r.usage)
```

### 7.3 Policy (codice): Jev decide, il codice applica soglie e veti — valori iniziali, da tarare
| Condizione per **aprire** | Soglia |
|---|---|
| `action_1`, `action_2`, `action_3` | tutte uguali (long o short), `confidence` media ≥ 0,60 |
| `setup_quality` | livello ≥ 3/5 ("Acceptable") con `confidence` ≥ 0,5 |
| `risk_ok` | ≥ 0,70 |
| Analisi dell'oracolo | presente e valida (età < 60 min, non invalidata); `oracle_consistent` < 0,4 non blocca, ma viene registrato e l'analisi viene marcata "contestata" per la taratura |
| `regime` | `volatile_chop` → mai; `range` → target ridotto a 1 R |
| Isteresi | invertire una posizione richiede due decisioni concordi a 15 minuti di distanza (la prima chiude, la seconda apre) |
| Veto di rischio (codice) | tutte le regole di §5.2 (sessione, eventi, spread/profondità, limiti di perdita, drawdown, cooldown, max 2 trade/giorno) |

| Condizione per **chiudere** (basta una) | Soglia |
|---|---|
| Stop/target/time‑stop | gestiti dal codice e da trigger on‑chain |
| `exit_now` | ≥ 0,70 per 2 decisioni consecutive (30 min) |
| `action` = `flat` o opposta alla posizione | tutte e tre concordi con `confidence` media ≥ 0,60 (chiusura immediata; eventuale inversione solo alla decisione successiva) |
| Finestra operativa in chiusura (15:45 ET; ven 19:30 ET) o evento imminente | regola di codice |

Quando l'oracolo e Jev **divergono** (ad esempio bias bullish e decisione short, o scenario "up 60 %" e decisione flat), il bot segue Jev e registra la divergenza: questi casi, insieme all'esito, sono i dati più utili per capire se l'oracolo aggiunge valore.

### 7.4 Surrogato (sviluppo senza chiave Jev)
`decision/surrogate.py` implementa le stesse domande con regole (es. regime da EMA/VWAP, `risk_ok` da ATR/spread/eventi, `action` da scenari dell'oracolo e trend) e restituisce lo stesso schema (`choice`, `probabilities`, `confidence`, `noul`). Il resto del sistema non sa quale decisore sta usando; il campo `model` nel log distingue `surrogate-v1` da `jev-1.13.0`. Quando arriva la chiave, si confrontano le due versioni sugli stessi stati registrati (replay).

---

## 8. Esecuzione su Hyperliquid

### 8.1 Setup account (una tantum)
1. **Wallet dedicato** (nuovo seed, mai usato altrove). Compra USDC su un exchange (Kraken/Bitpanda/Coinbase), preleva **USDC nativo su Arbitrum** (non USDC.e) + ~0,5 € di ETH per il gas.
2. Deposita su app.hyperliquid.xyz (minimo 5 USDC, accredito < 1 min).
3. Scegli la modalità account: **Unified account** → un solo saldo USDC usabile su `xyz` senza trasferimenti. Si imposta dall'app oppure con `exchange.user_set_abstraction(indirizzo, "unifiedAccount")` (azione firmata dal **wallet principale**, non dall'agent). In alternativa modalità standard + `exchange.send_asset(indirizzo, "", "xyz", "USDC", importo)`.
4. Crea un **API wallet (agent)** in app.hyperliquid.xyz/API (o `exchange.approve_agent("bot")`): può firmare ordini ma **non** prelevare. Nel bot: `secret_key` = chiave dell'agent, `account_address` = wallet principale.
5. Imposta leva e margine per ciascun simbolo: `exchange.update_leverage(3, "xyz:TSLA", is_cross=False)`.

### 8.2 Codice di riferimento (SDK 0.24.0)
```python
from hyperliquid.info import Info
from hyperliquid.exchange import Exchange
from hyperliquid.utils import constants
import eth_account, os, time

info = Info(constants.MAINNET_API_URL, skip_ws=True, perp_dexs=["xyz"])
wallet = eth_account.Account.from_key(os.environ["HL_AGENT_PRIVATE_KEY"])
ex = Exchange(wallet, constants.MAINNET_API_URL, account_address=os.environ["HL_ACCOUNT_ADDRESS"], perp_dexs=["xyz"])

# Dati
end = int(time.time() * 1000)
candles = info.candles_snapshot("xyz:TSLA", "1h", end - 72 * 3600 * 1000, end)
book = info.l2_snapshot("xyz:TSLA")
meta, ctxs = info.post("/info", {"type": "metaAndAssetCtxs", "dex": "xyz"})
stato = info.user_state(os.environ["HL_ACCOUNT_ADDRESS"], dex="xyz")

# Apertura: ordine IOC "market" con slippage massimo 0,3 %, poi stop e target reduce-only (trigger on-chain)
res = ex.market_open("xyz:TSLA", is_buy=True, sz=0.261, slippage=0.003)
ex.order("xyz:TSLA", is_buy=False, sz=0.261, limit_px=stop_px,
         order_type={"trigger": {"triggerPx": stop_px, "isMarket": True, "tpsl": "sl"}}, reduce_only=True)
ex.order("xyz:TSLA", is_buy=False, sz=0.261, limit_px=target_px,
         order_type={"trigger": {"triggerPx": target_px, "isMarket": True, "tpsl": "tp"}}, reduce_only=True)

# Chiusura totale
ex.market_close("xyz:TSLA", slippage=0.003)
```
Regole: arrotonda la size ai `szDecimals` dell'asset e il prezzo a 5 cifre significative; nozionale ≥ 10 $; dopo ogni ordine leggi la risposta (`statuses`: `filled`, `resting`, `error`) e riconcilia con `user_state`; cancella i trigger orfani quando la posizione è chiusa; non superare ~20 azioni/ora (il buffer di 10.000 azioni basta per mesi).

### 8.3 Paper trading realistico
`execution/paper.py` simula i fill sul book reale: ingresso al miglior prezzo opposto + metà spread, stop/target sulle candele successive (worst‑case: se nella stessa candela vengono toccati entrambi, conta lo stop), fee da `userFees` (nominali se growth mode non è confermato), funding orario pro‑rata. Stesse interfacce dell'esecutore live.

---

## 9. Costi stimati e opzioni (decisione tua)

Ipotesi per chiamata Opus: 2.000 token di system prompt in cache, 1.500 token di stato, 700 token di output (thinking incluso, effort `low`) ≈ 0,020 $. Sessione USA = 7 decisioni orarie × 22 giorni.

| Scenario | $/chiamata | $/giorno | $/mese |
|---|---|---|---|
| **A. Opus 5.5 ogni ora, sessione USA, 1 simbolo** | 0,020 | 0,14 | **~3** |
| **B. Opus 5.5 ogni ora, sessione USA, 3 simboli (TSLA, NVDA, SP500) [consigliato]** | 0,020 | 0,43 | **~9** |
| C. Opus 5.5 ogni ora 24/5 (pre, post e overnight), 3 simboli | 0,020 | 1,47 | ~32 |
| Opus 5.5 ogni ora 24/7, 1 simbolo (weekend incluso: sconsigliato, prezzo interno) | 0,020 | 0,49 | ~15 |
| Opus 5.5 effort `medium` (output ~1.500), sessione USA, 1 simbolo | 0,036 | 0,25 | ~6 |
| Opus 5.5 come "stratega" 1 volta/giorno (20 k in, 4 k out: rivede i log e propone modifiche ai parametri) | 0,16 | 0,16 | ~5 |
| Jev: decisioni ogni 15 min in sessione + controlli con posizione aperta | ~0,0001 | 0,01 | **~0,2** |
| Backtest storico: 1.000 stati orari (≈ 7 mesi di sessione USA su 1 simbolo), 1 chiamata Opus + 1 Jev per stato | 0,020 | – | ~19 una tantum |
| Per confronto: la cadenza iniziale a 5 minuti 24/7 con Opus | 0,019 | 5,5 | ~166 |
| VPS (Hetzner CX22 o simile) | – | – | ~4 |
| Hyperliquid: deposito/prelievo | – | – | ~1–3 una tantum |

**Raccomandazione:** scenario **B** (≈ 9 $/mese) con decisioni Jev ogni 15 minuti, budget API mensile di sicurezza 20 $ (oltre, il bot torna in `paper`), e un backtest storico (~20 $) ad ogni modifica importante di prompt, domande o soglie. Il pre‑filtro Jev del passo 4 resta come filtro di qualità, non più come risparmio. Lo schema "Opus stratega giornaliero + modello leggero per i segnali" (quello dell'articolo "Opus 5.5 + Jev") resta una variante attivabile cambiando il parametro `signals.model`.

---

## 10. Roadmap a fasi (criteri di uscita espliciti)

| Fase | Durata | Attività | Criterio di uscita |
|---|---|---|---|
| **0. Setup** | 1–2 giorni | Wallet dedicato, USDC su Arbitrum, deposito 20 USDC (il resto arriva alla fase 5), API wallet, unified account, chiave Anthropic, waitlist Jev, VPS/Docker, Telegram bot, repo con `.env.example` | Tutte le chiavi in `.env`, `python -m bot.healthcheck` verde (legge saldo, meta `xyz`, candele; Claude e Jev/surrogato rispondono) |
| **1. Dati e paper engine** | Settimana 1 | `data/`, `features/`, `state/`, `storage/`, `execution/paper.py`, scarico dello storico (candele 1h: 5.000 per richiesta ≈ 7 mesi, da marzo 2026; 15m ≈ 52 giorni; 4h tutto lo storico) e raccolta continua su SQLite, script `replay.py` | ≈ 7 mesi di storico 1h salvati per l'universo; replay deterministico; test unitari su feature e size |
| **2. Oracolo Claude** | Settimana 1–2 | Prompt di sistema, schema `AnalisiOracolo`, traduzione in etichette, cache verificata, contatore costi, eval su 200 stati registrati: coerenza (stesso stato → stessa analisi ≥ 90 %), calibrazione degli scenari (Brier score contro l'esito a 4 h), costo medio | Costo medio ≤ 0,025 $/chiamata, cache hit ≥ 90 %, nessun errore di schema, scenari meglio calibrati di un prior fisso |
| **3. Decisioni Jev** | Settimana 2 | `decision/surrogate.py`, `decision/jev_decider.py`, `decision/policy.py`, ciclo a 15 minuti, test di stabilità (ordine opzioni, bucket), log del `model` | Accordo delle 3 domande `action` ≥ 85 %; distribuzione di confidenza sensata; replay con surrogato e (se disponibile) Jev |
| **4. Rischio, backtest ed esecuzione in paper** | Settimane 2–5 | `risk/engine.py`, kill‑switch, calendario eventi; **backtest storico** su ≥ 1.000 stati orari (stati costruiti solo con dati fino alla candela chiusa, prezzi normalizzati e senza date, per limitare look‑ahead e "memoria" del modello sul periodo); esecutore live provato su **testnet** (dex principale, es. BTC) per la meccanica ordini/trigger; **paper trading su dati mainnet ≥ 3 settimane** | Backtest: ≥ 150 trade simulati, profit factor netto ≥ 1,2, max DD ≤ 10 %, meglio di tutte e tre le baseline di §12; paper: 20–40 trade coerenti col backtest; costo API entro budget; zero incidenti di riconciliazione |
| **5. Live progressivo** | Dalla settimana 6 | Live con 20–30 USDC per 1–2 settimane (size minima 10–15 $); poi 100 USDC se le metriche live sono in linea con il paper (slippage reale ≤ 2× stimato) | Metriche live ≈ paper; nessun errore di esecuzione |
| **6. Operatività e iterazione** | Continuo | Report giornaliero su Telegram; revisione settimanale con Opus (legge i log, propone modifiche a soglie/prompt in un PR, **mai** applicate in automatico); versioning di prompt e domande; A/B surrogato vs Jev | Decisione mensile: continuare, cambiare, fermare |

---

## 11. Struttura del repository proposta

```
Trading-bot/
├── PIANO.md                      ← questo documento
├── README.md
├── pyproject.toml                ← python ≥ 3.11; deps: hyperliquid-python-sdk==0.24.0, anthropic>=1.13,
│                                    typesafe-sdk, pydantic, httpx, apscheduler, pyyaml, python-telegram-bot, pandas
├── .env.example                  ← HL_ACCOUNT_ADDRESS, HL_AGENT_PRIVATE_KEY, ANTHROPIC_API_KEY, TYPESAFE_API_KEY,
│                                    TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID, BOT_MODE=paper|live
├── config/
│   ├── settings.yaml             ← universo, soglie §7.3, limiti §5.2, budget API, finestra oraria, modello segnali
│   ├── questions_v1.yaml         ← domande a Jev (criteri completi), soglie della policy, casi di test; versionato
│   └── events.yaml               ← earnings, FOMC, CPI, NFP, festività USA
├── src/bot/
│   ├── main.py                   ← loop scheduler + orchestrazione del ciclo §4
│   ├── healthcheck.py
│   ├── data/hyperliquid_client.py
│   ├── features/indicators.py
│   ├── state/builder.py          ← stato testuale per l'oracolo + stato etichettato (con blocco oracle) per Jev
│   ├── oracle/claude_oracle.py
│   ├── decision/{jev_decider.py, surrogate.py, policy.py}
│   ├── risk/engine.py
│   ├── execution/{hyperliquid_exec.py, paper.py, reconcile.py}
│   ├── storage/db.py             ← SQLite: candele, stati, segnali, decisioni, ordini, fill, costi
│   ├── notify/telegram.py
│   └── calendar/events.py
├── scripts/{collect_history.py, replay.py, backtest.py, report.py}
├── tests/
├── Dockerfile, docker-compose.yml, systemd/trading-bot.service
└── .gitignore                    ← .env, config.json, *.db, logs/
```

Stack: Python 3.11+, SQLite (basta e si copia con un `scp`), APScheduler o loop `asyncio` con sleep allineato al minuto, logging JSONL, Docker su un VPS Linux con NTP attivo e timezone UTC.

---

## 12. Backtest, paper e metriche: cosa vuol dire "funziona"

- **Replay offline:** ogni ciclo salva lo stato esatto inviato ai modelli, le risposte e la decisione; `replay.py` ripete il ciclo su stati storici (le risposte di Claude e Jev vengono rilette dal log, non richieste di nuovo: costo zero). Permette di cambiare soglie e sizing e misurare l'effetto.
- **Baseline obbligatorie:** (1) random con la stessa frequenza e durata dei trade, (2) Jev senza oracolo (decide solo sullo stato quantitativo), (3) oracolo tradotto in regole fisse senza Jev (es. scenario up ≥ 55 % → long), (4) surrogato a regole al posto di Jev. Così si misura il contributo di ciascun componente. Se il sistema completo non batte in modo netto tutte le baseline su ≥ 100 trade, non è il momento del live.
- **Metriche per trade:** hit rate, R medio, profit factor, expectancy netta (fee nominali + spread + funding), max drawdown, tempo medio in posizione, slippage.
- **Metriche di sistema:** P&L netto **meno costi API**, costo API per trade, tasso di divergenza oracolo↔Jev e suo esito, calibrazione degli scenari dell'oracolo, calibrazione di Jev (quando dice 0,8 ha ragione ~80 % delle volte?), latenza del ciclo, cache hit ratio.
- **Limite di un bot orario con 100 €:** ogni trade rischia ~0,8–1,6 USDC e si fanno 1–2 trade al giorno: in paper servono mesi per accumulare centinaia di trade. Per questo la validazione principale è il **backtest sugli stati orari storici** (≈ 7 mesi di candele 1h disponibili), mentre il paper conferma esecuzione, slippage e costi reali. Il giudizio va fatto sulla **distribuzione** (expectancy e drawdown), non sul saldo dopo una settimana.
- **Backtest con modelli:** ogni stato storico richiede una chiamata Opus e una Jev (≈ 0,02 $). Regole anti‑inganno: lo stato contiene solo dati fino alla candela chiusa; niente date assolute né notizie; prezzi come rendimenti percentuali e livelli relativi (distanza da VWAP/EMA), così il modello non può "ricordare" il periodo; stessa pipeline del live, nessun parametro ottimizzato sul periodo di test (split train/test per mese).

---

## 13. Monitoraggio, sicurezza, kill‑switch

- **Alert Telegram:** apertura/chiusura con motivo (segnale, confidenze, R), errori API, kill‑switch scattato, costi API giornalieri, heartbeat ogni ora. Comandi: `/status`, `/flat` (chiude tutto), `/pause`, `/resume`.
- **Kill‑switch automatico:** perdita giornaliera/settimanale/drawdown (§5.2), 3 errori consecutivi di esecuzione, posizione inattesa, latenza ciclo > 120 s, budget API superato, prezzo esterno assente (weekend/festivi) con posizione aperta → chiudi.
- **Sicurezza chiavi:** solo variabili d'ambiente; `.env` e `config.json` nel `.gitignore`; l'agent wallet non può prelevare; il wallet principale contiene solo il capitale di trading; 2FA sull'exchange di on‑ramp; nessuna chiave nei log e nei prompt.
- **Operatività:** NTP, timezone UTC, riavvio automatico (systemd/Docker `restart: unless-stopped`) con riconciliazione all'avvio; backup giornaliero del DB.
- **Rate limit:** ≤ 100 "peso"/ciclo sul limite di 1.200/min; backoff esponenziale su 429; un solo processo per indirizzo.
- **Pre‑mortem completo:** errori previsti, rilevazione, prevenzione, reazione automatica, test e runbook sono in [ERRORI-E-SOLUZIONI.md](ERRORI-E-SOLUZIONI.md). Tre scelte tecniche che ne derivano e che vanno nel codice dalla fase 1: `cloid` deterministico su ogni ordine, ingresso + stop + target nello stesso batch (`grouping="normalTpsl"`), `set_expires_after` su ogni azione firmata.

---

## 14. Note legali e fiscali (Italia) — da verificare con un professionista

- **Accesso:** i ToS di Hyperliquid vietano USA, Ontario e giurisdizioni sanzionate; l'UE non è bloccata (ottobre 2026). Nel Regno Unito la FCA ha emesso un warning; a Bruxelles è in corso la revisione MiCA con la richiesta (dal Hyperliquid Policy Center) di trattare i perps come derivati MiFID II. Lo scenario può cambiare: tienilo tra i rischi di progetto.
- **Natura dello strumento:** perpetual con leva su prezzi di azioni, regolati in USDC: è un derivato, non un'azione. Nessun dividendo, nessun diritto.
- **Fisco:** dal 1/1/2026 le plusvalenze su cripto‑attività sono tassate con imposta sostitutiva del 33 % senza franchigia (L. 199/2025; 26 % solo per e‑money token in euro); obblighi di monitoraggio (quadro RW) e reporting DAC8. Tieni il log di tutti i fill (`userFills`) e un export mensile in CSV per il commercialista.
- **Rischio controparte:** i mercati HIP‑3 dipendono dal deployer (trade.xyz) per oracolo, parametri, `haltTrading`; non sono garantiti dalla liquidità del protocollo. Con 100 € il rischio è contenuto per costruzione, ma esiste.

---

## 15. Decisioni che spettano a te prima della fase 2

1. **Copertura oraria di Opus:** solo sessione USA regolare (scenario B, ~9 $/mese, consigliato) oppure tutto il 24/5 con pre/post/overnight (~32 $/mese, liquidità 7–8× più bassa)?
2. **Budget API mensile massimo** oltre il quale il bot torna in paper (proposta: 20 $).
3. **Universo iniziale:** TSLA + NVDA + SP500 (proposta) oppure tutte le 8 blue chip (~24 $/mese)?
4. **Modalità account Hyperliquid:** unified account (proposta) o standard con trasferimenti espliciti?
5. **Ospitare il bot:** VPS (~4 €/mese) o macchina di casa (Raspberry/PC sempre acceso)?
6. **Jev:** hai già accesso early access o partiamo con il surrogato e aggiungiamo Jev quando arriva la chiave?
7. **Overnight:** confermi la v1 solo intraday (chiusura forzata alle 15:45 ET)?

---

## 16. Evoluzione: piattaforma in Rust e interfaccia grafica

**È fattibile**, e conviene prepararsi ora senza cambiare linguaggio subito.

| Domanda | Risposta |
|---|---|
| Rust ha senso per questo bot? | Per un ciclo orario la velocità di Rust non conta. I vantaggi reali sono tipizzazione forte, un binario unico senza dipendenze a runtime, memoria minima (un VPS da 2 €). Il costo è la velocità di iterazione: nelle prime settimane si cambiano prompt, domande e soglie ogni giorno, e in Python si fa in minuti. |
| Cosa esiste in Rust | Hyperliquid ha un SDK Rust nella sua organizzazione GitHub (`hyperliquid_rust_sdk`, 116 commit, 31 issue aperte; supporto ai dex HIP‑3 da verificare prima di contarci). Anthropic **non** ha un SDK Rust ufficiale: si usa l'API HTTP con `reqwest` (`POST /v1/messages`, output strutturato con `output_config.format`, caching con `cache_control`). Jev ha SDK solo Python e JavaScript: in Rust si chiama `POST /v1/systemone` direttamente, è un solo endpoint JSON. |
| Ordine consigliato | 1) Python finché la strategia non è validata (fasi 1–5). 2) Interfaccia grafica come **dashboard web** che legge il log degli eventi: prima in sola lettura, poi con i comandi `pause`/`flat`/`resume` attraverso lo stesso "command bus" usato da Telegram. 3) Rust come riscrittura del **percorso critico** (dati → rischio → esecuzione) in un servizio separato, lasciando a Python l'orchestrazione dei modelli; oppure riscrittura completa, se nel frattempo lo stack Rust di Hyperliquid è maturo. |
| Interfaccia grafica: con cosa | Fase rapida: una pagina web servita da FastAPI (o Streamlit) con equity, posizioni, ultimi segnali e decisioni, costi API, stato dei kill‑switch. Fase "prodotto": frontend web (React/Svelte) oppure app desktop con **Tauri** (backend Rust + frontend web), che è il punto d'incontro naturale fra le due idee. |
| Cosa fare **adesso** perché tutto questo sia facile dopo | Architettura "porte e adattatori": interfacce esplicite per `DataSource`, `Oracle` (Claude), `Decider` (Jev/surrogato), `RiskEngine`, `Executor` (live/paper), `Store`. Messaggi fra i moduli in JSON con schema versionato (Pydantic ora, `serde` dopo): lo stato per i modelli, il segnale, la decisione, l'ordine. Log degli eventi come unica fonte di verità, leggibile da qualunque linguaggio. Configurazione in YAML. Nessuna logica di business dentro Telegram o dentro la futura GUI: sono solo client del command bus. |

In pratica: il repository Python che nasce dalla fase 1 è già il "contratto" che una piattaforma Rust o una GUI dovranno rispettare; quando arriverà il momento, si riscrive un modulo alla volta tenendo i test di contratto come rete di sicurezza.

---

## 17. Fonti

**Hyperliquid (verifiche dirette sull'API pubblica il 9/10/2026)**: `perpDexs`, `meta`/`metaAndAssetCtxs` con `dex:"xyz"`, `candleSnapshot` 5m (2–9 ottobre) e 1h (15 marzo – 9 ottobre 2026), `l2Book`, `userFees`; SDK `hyperliquid-python-sdk` 0.24.0 (`perp_dexs`, `name_to_asset`, `market_open`, `update_leverage`, `send_asset`, `approve_agent`, `user_dex_abstraction`).
- Docs Hyperliquid: [HIP‑3 builder‑deployed perpetuals](https://hyperliquid.gitbook.io/hyperliquid-docs/hyperliquid-improvement-proposals-hips/hip-3-builder-deployed-perpetuals), [Info endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint), [Perpetuals info](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint/perpetuals), [Asset IDs](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/asset-ids), [Rate limits](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits), [Fees](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/fees), [Error responses](https://hyperliquid.gitbook.io/Hyperliquid-docs/for-developers/api/error-responses), [USDC/bridge](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/usdc), [Account abstraction modes](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/account-abstraction-modes.md), [How to start trading](https://hyperliquid.gitbook.io/hyperliquid-docs/onboarding/how-to-start-trading), [Python SDK](https://github.com/hyperliquid-dex/hyperliquid-python-sdk).
- trade.xyz: [Specification index](https://docs.trade.xyz/perpetuals/specifications-and-schedules/specification-index.md), [Architecture](https://docs.trade.xyz/architecture.md), [Risk & margining](https://docs.trade.xyz/perpetuals/risk-and-margining.md).
- Guide terze: [Equity perps guide](https://hyperliquidguide.com/guides/trading/equity-perps-guide), [XYZ explained](https://hyperliquidguide.com/guides/trading/hyperliquid-xyz-explained), [HIP‑3 explained (Nansen)](https://nansen.ai/post/what-is-hip-3-hyperliquid), [Trade[XYZ] 98 % del volume HIP‑3](https://www.kucoin.com/news/flash/trade-xyz-builds-92-markets-and-98-hip-3-volume-on-hyperliquid-in-8-months), [Open interest equities (CryptoBriefing)](https://cryptobriefing.com/hyperliquid-open-interest-tokenized-equities/), [Deposits & withdrawals (OneKey)](https://onekey.so/blog/ecosystem/complete-guide-to-hyperliquid-deposits-withdrawals-2026-fbe041), [Chainstack: build a Hyperliquid bot](https://chainstack.com/how-to-build-a-hyperliquid-trading-bot/).

**Jev / TypeSafe AI**: [docs.typesafe.ai](https://docs.typesafe.ai/) ([quickstart](https://docs.typesafe.ai/introduction/quickstart), [state](https://docs.typesafe.ai/concepts/state), [confidence](https://docs.typesafe.ai/confidence), [Python SDK](https://docs.typesafe.ai/sdk/python), [limiti noti jev‑1.13](https://docs.typesafe.ai/model-jaggedness/jev-1.13), [cookbook function calling per trading](https://docs.typesafe.ai/cookbooks/function_calling)); [cheatsheet non ufficiale](https://jev-cheatsheet.vercel.app/); [analisi Apex Logic](https://www.apex-logic.net/news/jev-ai-model-for-trading-2026); [pricing (eesel)](https://www.eesel.ai/blog/typesafe-jev-pricing); repo di riferimento [L1vsun/JEV-Trading-BOT](https://github.com/L1vsun/JEV-Trading-BOT) e [jarrodwatts/jev-trader](https://github.com/jarrodwatts/jev-trader); articolo ["Opus 5.5 + Jev" (Miles Deutscher)](https://x.com/milesdeutscher/article/2104937986869567565) e [riassunto Phemex](https://phemex.com/news/article/new-ai-trading-bot-architecture-combines-opus-55-and-jev-for-automated-execution-98272).

**Claude**: riferimento API/prezzi dalla skill `claude-api` (modelli e prezzi al 6/10/2026).

**Risultati LLM in trading reale**: [Alpha Arena, lezioni (Buildix)](https://www.buildix.trade/blog/nof1-alpha-arena-lessons-ai-trading-byok-strategy-advisor-2026), [SCMP](https://www.scmp.com/tech/tech-trends/article/3329784/deepseek-outperforms-ai-rivals-real-money-real-market-crypto-showdown), [Blockworks](https://empire-blockworks.beehiiv.com/p/ai-trades-on-hyperliquid).

**Regolamentazione e fisco**: [The Block – MiCA review](https://theblock.co/news/regulation/2026-10-01-hyperliquid-circle-mica-review-417452), [FinTelegram – Perps at the EU perimeter](https://fintelegram.com/perps-at-the-eu-perimeter-when-permissionless-meets-mifid-ii/), [FinTelegram – FCA warning](https://fintelegram.com/hyperliquid-after-the-fca-warning-defi-branding-meets-regulated-derivatives-reality/), [Milano Finanza – tasse cripto 2026](https://www.milanofinanza.it/news/tasse-criptovalute-2026-la-guida-completa-con-aliquote-scadenze-e-adempimenti-202605131159008630), [Studio Cerbone](https://www.studiocerbone.com/la-disciplina-fiscale-delle-cripto-attivita-in-italia-un-quadro-normativo-in-evoluzione/).
