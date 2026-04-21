---
name: dflow-sponsored-swaps
description: Build gasless DFlow flows where the app (sponsor) pays the Solana transaction fee, ATA creation rent, or prediction market initialization cost on behalf of the user. Use when the user wants gasless swaps, "we pay the gas", a sponsor wallet that co-signs trades, `sponsor` + `sponsorExec` parameters on `/order` and `/quote`, the `sponsoredSwap` flag, fee-payer flows for new users with no SOL, or `predictionMarketInitPayer` for sponsoring Kalshi market initialization. Do NOT use for setting Solana validator priority fees (use `dflow-priority-fees`), for taking a builder cut on trades (use `dflow-platform-fees`), or for placing the trade itself (use `dflow-spot-trading` or `dflow-kalshi-trading`).
---
