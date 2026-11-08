# Sentinelle créances: French insolvency claim deadlines MCP server

**MCP server and pay-per-call API for AI agents.** For any French company (SIREN), it returns the insolvency proceedings published in the BODACC, the official gazette (safeguard, receivership, liquidation), and for each opening judgment the exact last day for a creditor to file a claim, the number of days left and the court-appointed representative to write to.

- **MCP endpoint (streamable HTTP):** `https://creances.amacunia.fr/mcp`
- **REST endpoint:** `GET https://creances.amacunia.fr/v1/creances?sirens=910759158,552081317`
- **Free sample:** https://creances.amacunia.fr/v1/exemple
- **Docs for agents:** https://creances.amacunia.fr/llms.txt
- **Price:** 0.01 USDC per call (1 to 50 companies per call), paid with the [x402](https://x402.org) protocol on Base. No account, no API key, no subscription.
- **Official MCP Registry:** `io.github.Marcoco27/sentinelle-creances`

## Why it exists

When a French company enters *sauvegarde*, *redressement judiciaire* or *liquidation judiciaire*, the opening judgment is published in the BODACC. From that day, every creditor has **two months** to file its claim with the court-appointed representative, **four months** if it does not live in the territory of the court (art. R622-24 Code de commerce). Miss it and the claim is time-barred: the money is lost.

Nobody tells the creditors, except those the representative already knows. Credit managers, accountants, debt collectors and lawyers have to watch the gazette themselves, for every customer. This server does the watching and the date arithmetic: one call, up to 50 companies, every opening judgment, every open deadline.

## Tools

| Tool | Price | What it does |
|---|---|---|
| `creances_lookup` | 0.01 USDC (x402) | Insolvency proceedings and open claim deadlines for 1 to 50 SIREN |
| `creances_example` | free | A real answer (a liquidation published on 2026-11-08, computed as of 2026-11-09) to see the output shape |

**Input of `creances_lookup`:**

- `sirens`: 1 to 50 SIREN (9 digits, checksum validated) or SIRET (14 digits, reduced to the SIREN), comma-separated.
- `residence` (optional): where the creditor lives. `metropole` (default, mainland France), `etranger` (abroad), or an overseas code such as `974` (La Réunion). The two-month extension applies when it differs from the territory of the court.

## Example output

A liquidation ordered on 29 October 2026 by the Évreux commercial court, published in the BODACC on 8 November 2026. The claim deadline is 8 January 2027.

```json
{
  "date_calcul": "2026-11-09",
  "residence_creancier": "metropole",
  "entreprises_avec_delai_ouvert": 1,
  "resultats": [{
    "siren": "910759158",
    "denomination": "ALTERNATIVE",
    "en_procedure_collective_publiee": true,
    "delais_ouverts": [{
      "nature": "Jugement d'ouverture de liquidation judiciaire",
      "date_jugement": "2026-10-29",
      "publication_bodacc": "2026-11-08",
      "tribunal": "Greffe du Tribunal de Commerce d'Evreux",
      "annonce": "https://www.bodacc.fr/pages/annonces-commerciales-detail/?q.id=id:A202602152607",
      "date_limite_declaration": "2027-01-08",
      "jours_restants": 60,
      "delai_ouvert": true,
      "report_possible_jour_chome": null,
      "releve_de_forclusion_jusqu_au": "2027-05-08",
      "texte_du_jugement": "Jugement prononçant la liquidation judiciaire, … désignant liquidateur SELARL …"
    }],
    "nombre_annonces": 1
  }],
  "marche_a_suivre": "Adresser la déclaration au mandataire judiciaire ou au liquidateur nommé dans le jugement…",
  "source": "BODACC, DILA (Licence Ouverte) ; délais : art. R. 622-24 et L. 622-26 du code de commerce, art. 641 du code de procédure civile",
  "avertissement": "Information, pas un conseil juridique. Seule l'annonce publiée au BODACC fait foi ; la date limite rendue n'applique pas le report des jours chômés."
}
```

Field names are in French. `delai_ouvert` is true while a claim can still be filed; `jours_restants` is 0 once the window is closed; `texte_du_jugement` names the liquidator or representative; `marche_a_suivre` (how to file, in French) appears only when a window is open. `dernier_evenement` gives the latest insolvency notice of any kind, such as a closure.

## Connect

Any MCP client can connect and call `creances_example`. Calling `creances_lookup` returns an x402 *payment required* answer: the client must support x402 payments to pay it (see below).

**Claude Code**

```bash
claude mcp add --transport http sentinelle-creances https://creances.amacunia.fr/mcp
```

**Claude Desktop / claude.ai:** Settings, Connectors, *Add custom connector*, URL `https://creances.amacunia.fr/mcp`.

**Cursor** (`~/.cursor/mcp.json`)

```json
{ "mcpServers": { "sentinelle-creances": { "url": "https://creances.amacunia.fr/mcp" } } }
```

**VS Code** (`.vscode/mcp.json`)

```json
{ "servers": { "sentinelle-creances": { "type": "http", "url": "https://creances.amacunia.fr/mcp" } } }
```

## Paying with x402

The paid tool follows the x402 MCP convention: the first call returns the payment requirements (scheme `exact`, network Base `eip155:8453`, asset USDC, amount `10000` = 0.01 USDC); the client signs a payment, puts it in the request `_meta` under `x402/payment`, and calls again. The payment is settled only if the tool succeeds: invalid input is never charged.

A minimal Python client with the official `x402` SDK is in [`examples/pay_mcp.py`](examples/pay_mcp.py). The same service is also sold over plain HTTP (`GET /v1/creances`, HTTP 402 then retry), which every x402 HTTP client supports.

## Data and limits

- Source: BODACC (Bulletin officiel des annonces civiles et commerciales), published by the Direction de l'information légale et administrative (DILA), read live through its open API, under the Licence Ouverte (Etalab).
- Deadline rule: two months from the BODACC publication, four outside the territory of the court (art. R622-24 Code de commerce); same day of the month, or the last day of the month when that day does not exist (art. 641 Code de procédure civile).
- The deadline returned is the strict one. The extension to the next working day when it falls on a weekend or a public holiday (art. 642) is returned separately, in `report_possible_jour_chome`, and is not guaranteed.
- Opening judgments, and the termination of a plan that opens new proceedings, start a window. Conversions and closures do not.
- A creditor holding a published security interest or contract must be notified personally, and its deadline runs from that notice. Lifting the time bar can be asked within six months of publication (art. L622-26).
- Information only, not legal advice. The notice published in the BODACC prevails.

## Also listed on

- [data.gouv.fr reuse](https://www.data.gouv.fr/reuses/sentinelle-creances-date-limite-de-declaration-de-creance-apres-une-procedure-collective-pour-agents-ia)
- Coinbase x402 Bazaar
- Official MCP Registry: `io.github.Marcoco27/sentinelle-creances`

## Same author

- [Sentinelle catnat](https://github.com/Marcoco27/sentinelle-catnat-mcp): French natural-disaster decrees per commune and the open insurance-claim deadline.

## About this repository

This repository holds the public documentation, the MCP Registry manifest ([`server.json`](server.json)) and client examples. The service itself runs at `creances.amacunia.fr`. Issues and questions are welcome.

---

### En français

Serveur MCP et API payable à l'appel pour les agents IA : pour une entreprise française (SIREN), les procédures collectives publiées au BODACC et la date limite de déclaration de créance, avec les jours restants et le mandataire à qui écrire. Un centime de dollar par appel (protocole x402, USDC sur Base), jusqu'à 50 entreprises. Présentation : https://creances.amacunia.fr
