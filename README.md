# Trading-bot

Bot sperimentale per perpetual su azioni USA del dex HIP-3 `xyz` (trade.xyz) di Hyperliquid, con:

- **Claude Opus 5.5** come "oracolo": ogni ora (sessione USA) produce un'analisi strutturata (scenari con probabilità, livelli, caso long e caso short, invalidazioni) e **non decide**;
- **Jev** (decision model di TypeSafe AI) come "decisore": ogni 15 minuti decide azione, ingresso e uscita con probabilità tipizzate, usando lo stato quantitativo e l'analisi dell'oracolo;
- **codice deterministico** per dati, feature, sizing, limiti di rischio, esecuzione e veto sul solo rischio.

Budget di riferimento: 100 € (~110 USDC). Stato attuale: **fase di pianificazione**.

Documenti:

- [PIANO.md](PIANO.md): piano completo (fatti verificati su Hyperliquid e Jev, architettura, ciclo orario, regole di rischio, costi API, roadmap a fasi, struttura del repository, evoluzione verso Rust e interfaccia grafica, note legali/fiscali).
- [ERRORI-E-SOLUZIONI.md](ERRORI-E-SOLUZIONI.md): pre-mortem, errori previsti con rilevazione, prevenzione e reazione automatica, piano di test e runbook.

> Non è consulenza finanziaria. I perpetual con leva possono far perdere più del margine allocato.
