# Trading-bot

Bot sperimentale per perpetual su azioni USA del dex HIP-3 `xyz` (trade.xyz) di Hyperliquid, con:

- **Claude Opus 5.5** come "analista": ogni ora (sessione USA) produce un segnale strutturato (direzione, convinzione, tesi, invalidazione);
- **Jev** (decision model di TypeSafe AI) come "giudice": risponde con probabilità tipizzate (regime, qualità del setup, azione, uscita), anche come "guardia" ogni 15 minuti sulle uscite;
- **codice deterministico** per dati, feature, sizing, limiti di rischio, esecuzione e veto.

Budget di riferimento: 100 € (~110 USDC). Stato attuale: **fase di pianificazione**.

Leggi [PIANO.md](PIANO.md) per il piano completo: fatti verificati su Hyperliquid e Jev, architettura, ciclo operativo, regole di rischio, costi API, roadmap a fasi, struttura del repository e note legali/fiscali.

> Non è consulenza finanziaria. I perpetual con leva possono far perdere più del margine allocato.
