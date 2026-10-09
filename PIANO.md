# Piano dettagliato: trading bot su azioni "tokenizzate" di Hyperliquid con Claude Opus 5.5 (segnali) + Jev (decisioni)

> Stato: piano operativo, aggiornato al 9 ottobre 2026. Tutti i numeri su Hyperliquid sono stati verificati direttamente sull'API pubblica il 9/10/2026; i prezzi e i limiti di Jev vengono dalla documentazione pubblica di TypeSafe (early access, possono cambiare).
> Non è consulenza finanziaria. I perpetual con leva possono far perdere più del margine allocato.

---

## 0. Sintesi in 10 righe

1. **Cosa si costruisce.** Un bot Python che ogni 5 minuti legge i dati di mercato da Hyperliquid, chiede a **Claude Opus 5.5** un segnale strutturato (long/short/flat + convinzione + tesi + invalidazione), passa uno "stato" etichettato a **Jev** (decision model di TypeSafe AI) che risponde con scelte/probabilità tipizzate (regime, qualità del setup, azione, uscita), e lascia al **codice** tutta la matematica: sizing, stop, limiti di rischio, esecuzione e veto.
2. **Dove si opera.** Sui perpetual su azioni USA del dex HIP-3 `xyz` (trade.xyz) di Hyperliquid: `xyz:TSLA`, `xyz:NVDA`, `xyz:AAPL`, `xyz:MSFT`, `xyz:GOOGL`, `xyz:AMZN`, `xyz:META` (leva max 20x, cross margin, "growth mode" con fee ridotte) più `xyz:SP500`. Sono **perpetual sul prezzo dell'azione**, non azioni tokenizzate in senso stretto: nessun dividendo, nessuna proprietà del titolo. Collaterale: USDC.
3. **Budget.** 100 € ≈ 110 USDC netti dopo i costi di on-ramp. Ordine minimo su Hyperliquid: 10 $ di nozionale. Si può operare con 1 posizione alla volta da 50–100 USDC di nozionale a leva effettiva ≤ 1x sul capitale (leva impostata 3x isolata solo per ridurre il margine bloccato).
4. **Il problema numero uno è il costo delle API, non il mercato.** Opus 5.5 ogni 5 minuti 24/7 costa circa **166 $/mese** (effort `low`, cache attiva): più dell'intero capitale. Limitando le chiamate alla sessione USA (6,5 h × 22 giorni) si scende a **~33 $/mese**; con un pre-filtro Jev si risparmia un altro 30–50 %. Jev costa **~0,5 $/mese**. La scelta finale è tua (sezione 9).
5. **Aspettative realistiche.** Nell'esperimento pubblico Alpha Arena (Nof1, ottobre–novembre 2025) sei LLM hanno fatto trading reale su Hyperliquid con 10.000 $ ciascuno: Claude Sonnet 4.5 ha chiuso a −30,8 %, GPT‑5 a −62,7 %, solo Qwen3 Max ha guadagnato (+22,3 %; cifre finali riportate da fonti secondarie, non verificate sul sito di Nof1). Un LLM da solo non è un vantaggio statistico: il valore del progetto sta nella disciplina di rischio, nei costi bassi e nella misurazione. Con 100 € l'obiettivo sensato è **imparare e validare il sistema senza perdere più di 20–30 €**, non "guadagnare".
6. **Jev è in early access** (waitlist su typesafe.ai). Il piano prevede un "surrogato" a regole con la stessa interfaccia, così lo sviluppo e il paper trading partono subito anche senza chiave Jev.
7. **Orari.** I perps `xyz` scambiano 24/7 (verificato: candele e trade anche sabato e domenica), ma il prezzo esterno dell'azione arriva solo 24/5 (dom 20:05 → ven 20:00 ET). Fuori da quella finestra il prezzo è "interno" (order book + EMA 30 min) con bande di scoperta: il bot **non apre posizioni** nel weekend/festivi e chiude tutto prima del venerdì sera.
8. **Sicurezza.** Wallet dedicato con solo il capitale di trading, API wallet ("agent") che può fare ordini ma non prelievi, chiavi solo in variabili d'ambiente, kill‑switch a soglie di perdita.
9. **Roadmap.** 6 fasi in ~5–6 settimane: setup → dati e paper engine → segnali Claude → decisioni Jev → rischio ed esecuzione (paper ≥ 2 settimane) → live con 20–30 USDC, poi 100.
10. **Legale/fiscale (Italia).** Hyperliquid non geo‑blocca l'UE (blocca USA, Ontario e paesi sanzionati), ma i perps sono derivati in una zona grigia regolatoria (dibattito MiFID II vs MiCA, warning FCA nel Regno Unito). In Italia le plusvalenze su cripto‑attività sono tassate al 33 % dal 2026 (verifica con un commercialista).

---

## 1. Obiettivo, perimetro e vincoli

| Voce | Scelta | Perché |
|---|---|---|
| Capitale | 100 € → ~110 USDC su Hyperliquid | Dopo fee di acquisto e prelievo su Arbitrum (1–3 €) e gas (centesimi). Deposito minimo bridge: 5 USDC. |
| Venue | Hyperliquid, dex HIP‑3 `xyz` (trade.xyz) | 98 % del volume HIP‑3, 133 mercati, oracolo Pyth, USDC, API unica HyperCore. |
| Strumenti | Perps su TSLA, NVDA, AAPL, MSFT, GOOGL, AMZN, META, SP500 | Sono i mercati `xyz` a leva 20x–50x con cross margin abilitato e volumi giornalieri 10–140 M$. Le small cap `xyz` sono isolated‑only, 10x, poco liquide: escluse. |
| Cadenza | Ciclo ogni 5 minuti allineato alla chiusura della candela 5m | Richiesta dell'utente; coerente con la granularità delle candele Hyperliquid (`candleSnapshot` 5m). |
| Segnali | Claude Opus 5.5 (`claude-opus-5-5`), output JSON strutturato, effort `low` | Richiesta dell'utente. Costo e latenza controllati con prompt caching e output corto. |
| Decisioni | Jev (`jev-1.13.0` pinnato) via `typesafe-sdk` | Risposte tipizzate con probabilità e confidenza in 70–500 ms; 0,042 $/M token in input, output gratis. |
| Esecuzione | `hyperliquid-python-sdk` 0.24.0 con `perp_dexs=["xyz"]` | Supporto nativo HIP‑3 (`xyz:TSLA` → asset id 110001). |
| Modalità | `paper` (default) → `live` | Il live si attiva solo dopo i criteri di uscita della fase 4. |

**Cosa NON è nel perimetro:** market making, HFT, più venue, azioni tokenizzate spot (xStocks e simili su altre chain: spread e fee non compatibili con 100 € e 5 minuti), ottimizzazione di portafoglio.

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
| Thinking | Sempre attivo (adattivo); si controlla con `output_config.effort` (`low`…`max`, default `medium`). Per segnali ogni 5 minuti: `low`. |
| Output strutturato | `client.messages.parse(..., output_format=ModelloPydantic)` oppure `output_config.format = json_schema` |
| Caching | `cache_control` sul system prompt (stabile) → paga pieno solo lo stato di mercato variabile |
| Fallback | In produzione aggiungere `fallbacks: "default"` con beta `server-side-fallback-2026-07-01` (gestione dei rari `stop_reason: "refusal"`) |

---

## 3. Architettura

```
                ┌──────────────────────────────────────────────────────────┐
                │  SCHEDULER (ogni 5 min, +3 s dopo la chiusura candela)   │
                └───────────────┬──────────────────────────────────────────┘
                                ▼
┌───────────────────────── LIVELLO 1: CODICE (deterministico) ─────────────────────────┐
│  Hyperliquid Info API ──► candele 5m/1h, l2Book, metaAndAssetCtxs, posizione, fills │
│  Feature engine: EMA/VWAP/RSI/ATR, rendimenti, volume relativo, spread, profondità, │
│                  funding, OI, sessione (pre/regular/post/overnight/weekend), eventi │
│  State builder:  (a) testo compatto con numeri per Claude                           │
│                  (b) JSON piccolo con ETICHETTE ("rsi: overbought") per Jev          │
└───────────────┬──────────────────────────────────────────┬──────────────────────────┘
                ▼                                          ▼
┌──────── LIVELLO 2: CLAUDE OPUS 5.5 ────────┐   ┌──────── LIVELLO 3: JEV ─────────────────┐
│ "Analista": legge lo stato, ragiona,        │   │ "Giudice": risponde in <0,5 s a domande  │
│ restituisce JSON: direction, conviction,    │──►│ tipizzate: regime, setup_quality,        │
│ thesis, invalidation_pct, horizon, flags    │   │ action (x3, opzioni mescolate), exit_now,│
│ (effort low, cache, ~10 s, ~0,02 $)          │   │ thesis_consistent, risk_ok               │
└─────────────────────────────────────────────┘   └───────────────────┬─────────────────────┘
                                                                      ▼
┌───────────────────────── LIVELLO 4: RISK ENGINE + ESECUZIONE (codice) ───────────────┐
│  Gate: soglie di confidenza, coerenza Claude↔Jev, regime, sessione, calendario,      │
│        spread/profondità, limiti giornalieri, drawdown, cooldown  → può porre VETO    │
│  Sizing: nozionale, stop (ATR/invalidazione), target, time‑stop                      │
│  Esecuzione: hyperliquid-python-sdk → ordine IOC + trigger SL/TP reduce‑only         │
│  Riconciliazione: stato reale da clearinghouseState/userFills ad ogni ciclo           │
│  Persistenza SQLite + log JSONL + alert Telegram + kill‑switch                        │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

**Principio guida (dalla documentazione Jev e dai bot di riferimento):** *il codice fa i conti, Claude ragiona, Jev giudica, il codice esegue e può porre il veto*. Nessun modello calcola stop, size o P&L; nessun modello invia ordini.

**Perché due modelli e non uno.** Opus è lento (secondi) e costoso ma capace di sintetizzare contesto; Jev è rapido ed economico e restituisce probabilità calibrate su domande chiuse. Jev serve a tre cose: (1) **pre‑filtro** economico ("il regime è tradabile?") che evita chiamate Opus inutili, (2) **secondo parere indipendente** sull'azione, (3) **audit della tesi** di Claude ("la tesi è coerente con lo stato?"), che è il caso d'uso che TypeSafe stessa documenta.

---

## 4. Il ciclo ogni 5 minuti (tempo totale obiettivo < 45 s)

| Passo | Cosa succede | Budget tempo | Costo |
|---|---|---|---|
| 0 | Scheduler scatta a hh:m5+3 s (orologio NTP). Se il ciclo precedente è ancora in corso → skip e alert. | – | – |
| 1 | Riconcilia stato reale: `clearinghouseState(dex="xyz")`, `openOrders`, `userFills` (nuovi fill dal ciclo precedente). Se la posizione reale ≠ posizione attesa → modalità "safe": chiudi/annulla e notifica. | 1 s | peso 2–20 |
| 2 | Dati: candele 5m (ultime 120) e 1h (ultime 48) per i simboli dell'universo, `l2Book`, `metaAndAssetCtxs` (mark, oracle, funding, OI, volume). Cache locale per non riscaricare. | 2 s | peso ~60 |
| 3 | Gate "hard" pre‑modello: sessione (solo con prezzo esterno attivo; preferenza regular 9:30–16:00 ET), nessun evento programmato ±15 min (earnings del titolo, FOMC, CPI, NFP), spread ≤ 3 bps, profondità top‑of‑book ≥ 20× la size, limiti di perdita non raggiunti, cooldown non attivo. Se fallisce → log e fine ciclo (nessuna chiamata a modelli). | 0 s | 0 |
| 4 | Jev pre‑filtro (opzionale ma consigliato): `regime` (Choice) + `is_tradable` (Noul). Se `is_tradable < 0,55` e non ci sono posizioni → fine ciclo. | 0,5 s | ~0,0001 $ |
| 5 | Claude Opus 5.5: stato compatto → JSON segnale (vedi §6). Timeout 40 s, 1 retry. | 5–20 s | ~0,02 $ |
| 6 | Jev decisione (vedi §7): stato etichettato + tesi di Claude → `action` ×3 (opzioni mescolate), `setup_quality`, `thesis_consistent`, `exit_now`, `risk_ok`. | 0,5 s | ~0,0001 $ |
| 7 | Policy (codice): combina soglie (§7.3). Esito: `OPEN_LONG` / `OPEN_SHORT` / `CLOSE` / `HOLD` / `NOOP`. | 0 s | 0 |
| 8 | Risk engine: size, stop, target, leva, controllo nozionale ≥ 10 $ e multipli dei decimali. Veto se qualcosa non torna. | 0 s | 0 |
| 9 | Esecuzione (`live`) o simulazione (`paper`): ordine IOC con slippage max 0,3 %, poi trigger SL/TP reduce‑only. Verifica fill. | 1–3 s | 1–3 azioni |
| 10 | Persistenza: stato, prompt, risposte (con `model` di Jev e `usage` di Claude), decisione, ordini, costi API stimati. Alert Telegram su apertura/chiusura/errori. | 0,5 s | – |

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
| Rischio per trade | ≤ 1,5 % dell'equity (≈ 1,6 USDC) | Stop = max(0,6 %, 1,2 × ATR(14) 5m) dal prezzo di ingresso; la size si ricava dal rischio, non viceversa. |
| Target | 1,5–2 R oppure uscita Jev (`exit_now ≥ 0,7`) | Partial take‑profit non ha senso con queste size. |
| Time‑stop | 6 candele (30 min) senza progresso → chiudi; durata massima 4 h | Trading intraday: niente overnight nelle prime settimane. |
| Finestra operativa | Solo con prezzo esterno attivo (dom 20:05 → ven 20:00 ET); ingressi preferibilmente 9:30–16:00 ET; **flat entro ven 19:30 ET** | Weekend: liquidità al 3–5 % e prezzo interno con bande di scoperta. |
| Eventi | Nessun ingresso ±15 min da earnings del titolo, FOMC, CPI, NFP; nessuna posizione aperta attraverso gli earnings | Calendario mantenuto in `config/events.yaml`. |
| Limite perdita giornaliera | 4 % equity → stop fino a mezzanotte UTC | |
| Limite settimanale | 8 % → stop fino a lunedì + revisione | |
| Max drawdown | 15 % dal picco → bot fermo, serve riavvio manuale | |
| Max trade/giorno | 6; cooldown 15 min dopo una perdita | Limita anche le chiamate API. |
| Qualità esecuzione | Spread ≤ 3 bps, top‑5 book ≥ 20× size, slippage IOC ≤ 0,3 % | Altrimenti nessun ordine. |
| Costi API | Budget mensile configurabile (es. 40 $); al superamento il bot passa a `paper` | Contatore su `usage` di Claude e `input_tokens` di Jev. |

**Esempio numerico.** Equity 110 USDC, TSLA a 382,6 $, ATR 5m 0,25 % → stop 0,6 % (il minimo). Rischio 1,5 % = 1,65 USDC → nozionale = 1,65 / 0,006 = 275 USDC, ma il cap è 100 USDC → size = 100 / 382,6 = 0,261 → arrotondata a 3 decimali `0.261` (nozionale 99,9 $). Rischio effettivo 0,60 USDC (0,55 %). Fee andata/ritorno in growth mode ≈ 0,02 USDC, nominali ≈ 0,18 USDC; spread ≈ 0,01 USDC. Target 1,5 R = +0,9 % ≈ 0,90 USDC. Con queste cifre servono **≥ 45–50 % di trade vincenti a 1,5 R** solo per pareggiare le fee nominali: la precisione del sistema va misurata su centinaia di trade in paper prima del live.

---

## 6. Design del segnale Claude Opus 5.5

### 6.1 Cosa riceve (≈ 1.000–1.500 token variabili + system prompt ≈ 2.000 token in cache)
- **System prompt (stabile, in cache):** ruolo (analista intraday su perps azionari), regole del gioco (orizzonte 5–60 min, costi, niente overnight), formato di output, criteri di convinzione, divieti (non calcolare size, non inventare notizie, dichiarare "flat" quando non c'è edge), descrizione dei campi dello stato.
- **Stato (variabile, in ordine stabile, senza timestamp inutili):** simbolo; sessione e minuti alla chiusura USA; ultime 24 candele 5m (o,h,l,c,v compatte) + riassunto 1h/4h (trend, range, VWAP, EMA 20/50/200, RSI 14, ATR %, volume relativo); mark vs oracle (premium), funding, variazione OI 1h; spread e profondità; regime Jev (dal pre‑filtro); posizione aperta (lato, P&L in R, minuti); ultimi 3 segnali con esito (feedback); flag calendario.
- **Niente notizie in tempo reale** nella v1: aggiungerle costa (web search) e introduce rumore; valutare in v2 con un filtro Jev di rilevanza.

### 6.2 Cosa restituisce (schema Pydantic → `messages.parse`)
```python
from pydantic import BaseModel, Field
from typing import Literal, List

class Segnale(BaseModel):
    direction: Literal["long", "short", "flat"]
    conviction: float = Field(ge=0, le=1)           # 0–1, Claude dichiara quanto crede nella tesi
    horizon_min: int = Field(ge=5, le=240)          # orizzonte atteso
    thesis: str = Field(max_length=280)             # 1–2 frasi, in inglese (Jev legge meglio l'inglese)
    invalidation_pct: float = Field(gt=0, le=3)     # distanza % oltre cui la tesi è falsa (il codice la trasforma in stop)
    target_pct: float = Field(gt=0, le=5)
    regime_view: Literal["trend_up", "trend_down", "range", "volatile_chop", "unclear"]
    risk_flags: List[Literal["event_soon", "thin_book", "extended_move", "funding_extreme", "session_edge", "none"]]
```

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
    output_format=Segnale,
)
segnale = resp.parsed_output
log(usage=resp.usage)   # input_tokens, cache_read_input_tokens, output_tokens → contatore costi
```
Note operative: verifica che `cache_read_input_tokens > 0` dal secondo ciclo (se è 0 c'è qualcosa di variabile nel prefisso); un `stop_reason == "max_tokens"` o `"refusal"` → segnale `flat`; in produzione abilita `fallbacks: "default"` (beta `server-side-fallback-2026-07-01`) per i rari refusal.

### 6.4 Controllo costi (il punto critico)
- Effort `low`, `max_tokens` 4000, stato compatto, system prompt in cache: ≈ 0,019 $/chiamata.
- Chiamare Opus **solo** quando: finestra operativa attiva **e** (pre‑filtro Jev tradabile **oppure** posizione aperta da gestire). Fuori sessione USA il bot gira ma non chiama Opus.
- Contatore mensile con soglia (default 40 $) → oltre, passaggio automatico a `paper` + alert.

---

## 7. Design delle decisioni Jev

### 7.1 Stato per Jev: piccolo, etichettato, in inglese (Jev non fa aritmetica)
```json
{
  "symbol": "TSLA perpetual",
  "session": "US regular hours, 94 minutes to close",
  "trend_1h": "up, price above VWAP and EMA20",
  "trend_5m": "pullback of 0.4% into EMA20 in the last 3 candles",
  "momentum": "RSI 58, neutral-bullish",
  "volatility": "ATR normal (0.25% per 5m); 1-hour range 0.9%",
  "volume": "last 15m volume is 1.4x the session average",
  "order_book": "spread tight (0.8 bps), bid depth 4x ask depth",
  "derivatives": "mark at oracle, funding neutral, open interest rising",
  "event_risk": "no scheduled events in the next 2 hours",
  "position": "flat",
  "analyst_signal": {"direction": "long", "conviction": "moderate (0.65)",
                     "thesis": "Trend continuation after shallow pullback; invalidation below VWAP (-0.7%)"},
  "recent_performance": "last 3 trades: +1.2R, -1.0R, +0.4R"
}
```
Tutti i numeri vengono convertiti dal codice in bucket descrittivi ("tight", "normal", "1.4x average"). Nessun calcolo è lasciato a Jev.

### 7.2 Domande (una sola chiamata, valutate in parallelo)
```python
from typesafe_sdk import TypeSafeClient, Choice, Score, Noul
import random

jev = TypeSafeClient()   # TYPESAFE_API_KEY dall'ambiente; pinna "jev-1.13.0" (parametro model, vedi doc SDK "Choosing a model")

def q_action():          # opzioni mescolate ad ogni domanda: Jev tende a preferire la prima
    opts = {"long": "Open or hold a long position now",
            "short": "Open or hold a short position now",
            "flat": "Do not trade; close any open position if the thesis is invalid"}
    items = list(opts.items()); random.shuffle(items)
    return Choice(instructions="Given the state and the analyst signal, what is the best action for the next 30 minutes?",
                  criteria=dict(items))

r = jev.system_one(state=stato_jev, questions={
    "regime": Choice(instructions="What market regime best describes the state?",
                     criteria={"trend_up": "Higher highs, price above VWAP/EMAs",
                               "trend_down": "Lower lows, price below VWAP/EMAs",
                               "range": "Sideways inside a defined band",
                               "volatile_chop": "Large, directionless swings"}),
    "is_tradable": Noul(instructions="Is this a state where a disciplined intraday trader would consider a new position?",
                        criteria={"true": "Clear regime, normal volatility, no imminent events, good liquidity",
                                  "false": "Unclear regime, event risk, thin book, or extreme volatility"}),
    "setup_quality": Score(instructions="Rate the quality of the analyst's setup.",
                           criteria=["Poor", "Weak", "Acceptable", "Good", "Excellent"]),
    "thesis_consistent": Noul(instructions="Is the analyst's thesis consistent with every field of the state?",
                              criteria={"true": "No field contradicts the thesis", "false": "At least one field contradicts it"}),
    "action_1": q_action(), "action_2": q_action(), "action_3": q_action(),
    "exit_now": Noul(instructions="If a position is open, should it be closed now?",
                     criteria={"true": "Thesis invalidated or risk rising", "false": "Thesis intact"}),
    "risk_ok": Noul(instructions="Is opening a new position acceptable from a risk standpoint?",
                    criteria={"true": "Session, liquidity and event risk are acceptable",
                              "false": "Any risk flag makes a new position unacceptable"}),
})
log(jev_model=r.model, usage=r.usage)
```

### 7.3 Policy (codice) — valori iniziali, da tarare in paper
| Condizione per **aprire** | Soglia |
|---|---|
| Claude `direction` ≠ flat e `conviction` | ≥ 0,60 |
| Jev `action_1/2/3` | tutte e tre uguali alla direzione di Claude, `confidence` media ≥ 0,60 |
| Jev `setup_quality` | livello ≥ 3/5 ("Acceptable") con `confidence` ≥ 0,5 |
| Jev `thesis_consistent` | ≥ 0,65 |
| Jev `risk_ok` | ≥ 0,70 |
| Jev `regime` | coerente con la direzione (trend_up → long, trend_down → short); `range` ammesso solo con target ≤ 1 R; `volatile_chop` → mai |
| Gate hard §5.2 | tutti verdi |

| Condizione per **chiudere** (basta una) | Soglia |
|---|---|
| Stop/target/time‑stop | gestiti dal codice e da trigger on‑chain |
| Jev `exit_now` | ≥ 0,70 per 2 cicli consecutivi (evita falsi allarmi) |
| Claude `direction` opposta con `conviction` ≥ 0,7 e Jev concorde | |
| Finestra operativa in chiusura (ven 19:30 ET) o evento imminente | |

Quando Claude e Jev **non** concordano, il bot non fa nulla e registra il disaccordo: questi casi sono i dati più preziosi per la taratura.

### 7.4 Surrogato (sviluppo senza chiave Jev)
`decision/surrogate.py` implementa le stesse 9 domande con regole (es. regime da EMA/VWAP, `is_tradable` da ATR/spread/eventi) e restituisce lo stesso schema (`choice`, `probabilities`, `confidence`, `noul`). Il resto del sistema non sa quale "giudice" sta usando; il campo `model` nel log distingue `surrogate-v1` da `jev-1.13.0`. Quando arriva la chiave, si confrontano le due versioni sugli stessi stati registrati (replay).

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
candles = info.candles_snapshot("xyz:TSLA", "5m", end - 120 * 5 * 60 * 1000, end)
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

Ipotesi per chiamata Opus: 2.000 token di system prompt in cache, 1.200 token di stato, 700 token di output (thinking incluso, effort `low`).

| Scenario | $/chiamata | $/giorno | $/mese |
|---|---|---|---|
| **A. Opus 5.5 ogni 5 min, 24/7** (richiesta letterale) | 0,019 | 5,5 | **~166** |
| **B. Opus 5.5 ogni 5 min solo sessione USA regolare** (6,5 h × 22 gg) | 0,019 | 1,5 | **~33** |
| B + pre‑filtro Jev (stima −30/50 % chiamate) | 0,019 | ~0,9 | **~18–23** |
| Opus 5.5 effort `medium` (output ~1.500) 24/7 | 0,035 | 10,1 | ~304 |
| C. Sonnet 5.5 ogni 5 min 24/7 (per confronto) | 0,010 | 2,8 | ~85 |
| D. Haiku 5.5 ogni 5 min 24/7 (per confronto) | 0,0005 | 0,14 | ~4 |
| Opus 5.5 come "stratega" 1 volta/giorno (20 k in, 4 k out: rivede i log e propone modifiche ai parametri) | 0,16 | 0,16 | ~5 |
| Jev, 288 chiamate/giorno × 1.500 token | ~0,00006 | 0,018 | **~0,5** |
| VPS (Hetzner CX22 o simile) | – | – | ~4 |
| Hyperliquid: deposito/prelievo | – | – | ~1–3 una tantum |

**Raccomandazione:** scenario **B + pre‑filtro Jev** (≈ 20–35 $/mese) nelle prime 4–6 settimane; è l'unico modo per usare Opus a 5 minuti senza che il costo API superi il capitale. Lo scenario A resta disponibile con un flag di configurazione (`signals.schedule: always`), ma va acceso consapevolmente. Lo schema "Opus stratega giornaliero + modello leggero per i segnali" (quello dell'articolo "Opus 5.5 + Jev") è una variante che il repo supporterà con un semplice cambio del parametro `signals.model`.

---

## 10. Roadmap a fasi (criteri di uscita espliciti)

| Fase | Durata | Attività | Criterio di uscita |
|---|---|---|---|
| **0. Setup** | 1–2 giorni | Wallet dedicato, USDC su Arbitrum, deposito 20 USDC (il resto arriva alla fase 5), API wallet, unified account, chiave Anthropic, waitlist Jev, VPS/Docker, Telegram bot, repo con `.env.example` | Tutte le chiavi in `.env`, `python -m bot.healthcheck` verde (legge saldo, meta `xyz`, candele; Claude e Jev/surrogato rispondono) |
| **1. Dati e paper engine** | Settimana 1 | `data/`, `features/`, `state/`, `storage/`, `execution/paper.py`, raccolta continua candele 5m/1h (ogni richiesta copre max 5.000 candele ≈ 17 giorni: accumulare storico su SQLite), script `replay.py` | 7 giorni di storico salvati; replay deterministico; test unitari su feature e size |
| **2. Segnali Claude** | Settimana 1–2 | Prompt di sistema, schema `Segnale`, cache verificata, contatore costi, eval su 200 stati registrati: coerenza (stesso stato → stesso segnale ≥ 90 %), distribuzione long/short/flat, costo medio | Costo medio ≤ 0,025 $/chiamata, cache hit ≥ 90 %, nessun errore di schema |
| **3. Decisioni Jev** | Settimana 2 | `decision/surrogate.py`, `decision/jev_judge.py`, `decision/policy.py`, test di stabilità (ordine opzioni, bucket), log del `model` | Accordo delle 3 domande `action` ≥ 85 %; distribuzione di confidenza sensata; replay con surrogato e (se disponibile) Jev |
| **4. Rischio ed esecuzione in paper** | Settimane 2–4 | `risk/engine.py`, kill‑switch, calendario eventi, esecutore live provato su **testnet** (dex principale, es. BTC) per la meccanica ordini/trigger; **paper trading su dati mainnet ≥ 2 settimane** | ≥ 100 trade paper; profit factor netto ≥ 1,2; max DD ≤ 10 %; costo API/mese entro budget; zero incidenti di riconciliazione |
| **5. Live progressivo** | Dal weekend 4–5 | Live con 20–30 USDC per 1–2 settimane (size minima 10–15 $); poi 100 USDC se le metriche live sono in linea con il paper (slippage reale ≤ 2× stimato) | Metriche live ≈ paper; nessun errore di esecuzione |
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
│   └── events.yaml               ← earnings, FOMC, CPI, NFP, festività USA
├── src/bot/
│   ├── main.py                   ← loop scheduler + orchestrazione del ciclo §4
│   ├── healthcheck.py
│   ├── data/hyperliquid_client.py
│   ├── features/indicators.py
│   ├── state/builder.py          ← stato testuale per Claude + stato etichettato per Jev
│   ├── signals/claude_analyst.py
│   ├── decision/{jev_judge.py, surrogate.py, policy.py}
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
- **Baseline obbligatorie:** (1) random con la stessa frequenza e durata dei trade, (2) solo Claude senza Jev, (3) solo surrogato/Jev senza Claude. Se il sistema completo non batte in modo netto tutte e tre su ≥ 100 trade, non è il momento del live.
- **Metriche per trade:** hit rate, R medio, profit factor, expectancy netta (fee nominali + spread + funding), max drawdown, tempo medio in posizione, slippage.
- **Metriche di sistema:** P&L netto **meno costi API**, costo API per trade, tasso di disaccordo Claude↔Jev e suo esito, calibrazione di Jev (quando dice 0,8 ha ragione ~80 % delle volte?), latenza del ciclo, cache hit ratio.
- **Limite di un bot a 5 minuti con 100 €:** ogni trade rischia ~0,6–1,6 USDC; il rumore statistico è enorme. Il giudizio va fatto sulla **distribuzione** (expectancy e drawdown) dopo centinaia di trade in paper, non sul saldo dopo una settimana.

---

## 13. Monitoraggio, sicurezza, kill‑switch

- **Alert Telegram:** apertura/chiusura con motivo (segnale, confidenze, R), errori API, kill‑switch scattato, costi API giornalieri, heartbeat ogni ora. Comandi: `/status`, `/flat` (chiude tutto), `/pause`, `/resume`.
- **Kill‑switch automatico:** perdita giornaliera/settimanale/drawdown (§5.2), 3 errori consecutivi di esecuzione, posizione inattesa, latenza ciclo > 120 s, budget API superato, prezzo esterno assente (weekend/festivi) con posizione aperta → chiudi.
- **Sicurezza chiavi:** solo variabili d'ambiente; `.env` e `config.json` nel `.gitignore`; l'agent wallet non può prelevare; il wallet principale contiene solo il capitale di trading; 2FA sull'exchange di on‑ramp; nessuna chiave nei log e nei prompt.
- **Operatività:** NTP, timezone UTC, riavvio automatico (systemd/Docker `restart: unless-stopped`) con riconciliazione all'avvio; backup giornaliero del DB.
- **Rate limit:** ≤ 100 "peso"/ciclo sul limite di 1.200/min; backoff esponenziale su 429; un solo processo per indirizzo.

---

## 14. Note legali e fiscali (Italia) — da verificare con un professionista

- **Accesso:** i ToS di Hyperliquid vietano USA, Ontario e giurisdizioni sanzionate; l'UE non è bloccata (ottobre 2026). Nel Regno Unito la FCA ha emesso un warning; a Bruxelles è in corso la revisione MiCA con la richiesta (dal Hyperliquid Policy Center) di trattare i perps come derivati MiFID II. Lo scenario può cambiare: tienilo tra i rischi di progetto.
- **Natura dello strumento:** perpetual con leva su prezzi di azioni, regolati in USDC: è un derivato, non un'azione. Nessun dividendo, nessun diritto.
- **Fisco:** dal 1/1/2026 le plusvalenze su cripto‑attività sono tassate con imposta sostitutiva del 33 % senza franchigia (L. 199/2025; 26 % solo per e‑money token in euro); obblighi di monitoraggio (quadro RW) e reporting DAC8. Tieni il log di tutti i fill (`userFills`) e un export mensile in CSV per il commercialista.
- **Rischio controparte:** i mercati HIP‑3 dipendono dal deployer (trade.xyz) per oracolo, parametri, `haltTrading`; non sono garantiti dalla liquidità del protocollo. Con 100 € il rischio è contenuto per costruzione, ma esiste.

---

## 15. Decisioni che spettano a te prima della fase 2

1. **Cadenza Opus:** scenario A (24/7, ~166 $/mese), **B (sessione USA, ~33 $/mese) [consigliato]**, o B + pre‑filtro Jev (~20 $/mese)?
2. **Budget API mensile massimo** oltre il quale il bot torna in paper (proposta: 40 $).
3. **Universo iniziale:** solo TSLA + NVDA + SP500 (proposta) oppure tutte le 8 blue chip?
4. **Modalità account Hyperliquid:** unified account (proposta) o standard con trasferimenti espliciti?
5. **Ospitare il bot:** VPS (~4 €/mese) o macchina di casa (Raspberry/PC sempre acceso)?
6. **Jev:** hai già accesso early access o partiamo con il surrogato e aggiungiamo Jev quando arriva la chiave?

---

## 16. Fonti

**Hyperliquid (verifiche dirette sull'API pubblica il 9/10/2026)**: `perpDexs`, `meta`/`metaAndAssetCtxs` con `dex:"xyz"`, `candleSnapshot` 5m (2–9 ottobre), `l2Book`, `userFees`; SDK `hyperliquid-python-sdk` 0.24.0 (`perp_dexs`, `name_to_asset`, `market_open`, `update_leverage`, `send_asset`, `approve_agent`, `user_dex_abstraction`).
- Docs Hyperliquid: [HIP‑3 builder‑deployed perpetuals](https://hyperliquid.gitbook.io/hyperliquid-docs/hyperliquid-improvement-proposals-hips/hip-3-builder-deployed-perpetuals), [Info endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint), [Perpetuals info](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint/perpetuals), [Asset IDs](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/asset-ids), [Rate limits](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits), [Fees](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/fees), [Error responses](https://hyperliquid.gitbook.io/Hyperliquid-docs/for-developers/api/error-responses), [USDC/bridge](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/usdc), [Account abstraction modes](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/account-abstraction-modes.md), [How to start trading](https://hyperliquid.gitbook.io/hyperliquid-docs/onboarding/how-to-start-trading), [Python SDK](https://github.com/hyperliquid-dex/hyperliquid-python-sdk).
- trade.xyz: [Specification index](https://docs.trade.xyz/perpetuals/specifications-and-schedules/specification-index.md), [Architecture](https://docs.trade.xyz/architecture.md), [Risk & margining](https://docs.trade.xyz/perpetuals/risk-and-margining.md).
- Guide terze: [Equity perps guide](https://hyperliquidguide.com/guides/trading/equity-perps-guide), [XYZ explained](https://hyperliquidguide.com/guides/trading/hyperliquid-xyz-explained), [HIP‑3 explained (Nansen)](https://nansen.ai/post/what-is-hip-3-hyperliquid), [Trade[XYZ] 98 % del volume HIP‑3](https://www.kucoin.com/news/flash/trade-xyz-builds-92-markets-and-98-hip-3-volume-on-hyperliquid-in-8-months), [Open interest equities (CryptoBriefing)](https://cryptobriefing.com/hyperliquid-open-interest-tokenized-equities/), [Deposits & withdrawals (OneKey)](https://onekey.so/blog/ecosystem/complete-guide-to-hyperliquid-deposits-withdrawals-2026-fbe041), [Chainstack: build a Hyperliquid bot](https://chainstack.com/how-to-build-a-hyperliquid-trading-bot/).

**Jev / TypeSafe AI**: [docs.typesafe.ai](https://docs.typesafe.ai/) ([quickstart](https://docs.typesafe.ai/introduction/quickstart), [state](https://docs.typesafe.ai/concepts/state), [confidence](https://docs.typesafe.ai/confidence), [Python SDK](https://docs.typesafe.ai/sdk/python), [limiti noti jev‑1.13](https://docs.typesafe.ai/model-jaggedness/jev-1.13), [cookbook function calling per trading](https://docs.typesafe.ai/cookbooks/function_calling)); [cheatsheet non ufficiale](https://jev-cheatsheet.vercel.app/); [analisi Apex Logic](https://www.apex-logic.net/news/jev-ai-model-for-trading-2026); [pricing (eesel)](https://www.eesel.ai/blog/typesafe-jev-pricing); repo di riferimento [L1vsun/JEV-Trading-BOT](https://github.com/L1vsun/JEV-Trading-BOT) e [jarrodwatts/jev-trader](https://github.com/jarrodwatts/jev-trader); articolo ["Opus 5.5 + Jev" (Miles Deutscher)](https://x.com/milesdeutscher/article/2104937986869567565) e [riassunto Phemex](https://phemex.com/news/article/new-ai-trading-bot-architecture-combines-opus-55-and-jev-for-automated-execution-98272).

**Claude**: riferimento API/prezzi dalla skill `claude-api` (modelli e prezzi al 6/10/2026).

**Risultati LLM in trading reale**: [Alpha Arena, lezioni (Buildix)](https://www.buildix.trade/blog/nof1-alpha-arena-lessons-ai-trading-byok-strategy-advisor-2026), [SCMP](https://www.scmp.com/tech/tech-trends/article/3329784/deepseek-outperforms-ai-rivals-real-money-real-market-crypto-showdown), [Blockworks](https://empire-blockworks.beehiiv.com/p/ai-trades-on-hyperliquid).

**Regolamentazione e fisco**: [The Block – MiCA review](https://theblock.co/news/regulation/2026-10-01-hyperliquid-circle-mica-review-417452), [FinTelegram – Perps at the EU perimeter](https://fintelegram.com/perps-at-the-eu-perimeter-when-permissionless-meets-mifid-ii/), [FinTelegram – FCA warning](https://fintelegram.com/hyperliquid-after-the-fca-warning-defi-branding-meets-regulated-derivatives-reality/), [Milano Finanza – tasse cripto 2026](https://www.milanofinanza.it/news/tasse-criptovalute-2026-la-guida-completa-con-aliquote-scadenze-e-adempimenti-202605131159008630), [Studio Cerbone](https://www.studiocerbone.com/la-disciplina-fiscale-delle-cripto-attivita-in-italia-un-quadro-normativo-in-evoluzione/).
