# Spec: Collateral Mobility Desk — ready for design partners

- **Owner:** Christopher Mensah
- **Status:** Draft for review (Gate 2)
- **Date:** 2026-11-18
- **Reads:** [`intent.md`](intent.md) (signed off 2026-11-18)
- **Next stage reads this:** `plan.md`

This spec covers the current increment only: about 4 weeks to make the prototype safe to show a design partner with anonymised real data. Anything not here is out of scope for `plan.md`.

> **Assumptions to confirm before Gate 2.** The Desk, mapper and agent code is not yet in this repo. This spec assumes:
>
> - the Desk is a TypeScript web app;
> - the mapper is the TypeScript `cdm-collateral-mapper`;
> - CDM objects are handled as JSON matching CDM 7.x.
>
> Field names in the CDM sections must be checked against the FINOS source once the code is imported.

## 1. What this increment delivers

| # | Capability | Section |
| --- | --- | --- |
| 1 | One approved-schedule file format, exported by both the agent and the mapper | §2 |
| 2 | The Desk imports an approved schedule only after verifying it | §3 |
| 3 | The eligibility engine applies a schedule as written, with the four CDM divergences and open items A–C fixed and tested | §4 |
| 4 | Holdings import through the mapper | §5 |
| 5 | A token data record for tokenised assets, kept in the extension | §6 |
| 6 | One ledger adapter interface; the simulated ledger is its first implementation | §7 |
| 7 | Data handling ready for a partner: AI off by default, self-hosting behind a login, a data-handling note | §8 |
| 8 | A business-case screen comparing today with the Desk | §9 |

## 2. Approved-schedule file format

One JSON file, whatever the source. Both the Clause-to-CDM agent and the mapper write it; only the Desk reads it.

**File name:** `<schedule-id>.approved-schedule.json`

**Top-level shape**

```json
{
  "formatVersion": "1.0",
  "scheduleId": "string, stable across versions of the same schedule",
  "scheduleVersion": 3,
  "source": {
    "kind": "agent | mapper",
    "tool": "clause-to-cdm-agent | cdm-collateral-mapper",
    "toolVersion": "semver",
    "inputRef": "file name of the CSA text or spreadsheet; no client names",
    "cdmVersion": "7.x.y"
  },
  "schedule": { "...": "CDM EligibleCollateralSpecification, unmodified" },
  "tokenConditions": { "...": "extension object, see §2.3" },
  "citations": [ { "...": "one per value, see §2.2" } ],
  "approval": { "...": "see §2.4" },
  "hashes": {
    "algorithm": "SHA-256",
    "canonicalisation": "RFC 8785 (JCS)",
    "scheduleHash": "hex",
    "fileHash": "hex"
  }
}
```

### 2.1 Field rules

- `schedule` is a CDM `EligibleCollateralSpecification` exactly as generated. No extra fields; anything the Desk adds goes in `tokenConditions`.
- `scheduleVersion` goes up by 1 each time the same `scheduleId` is approved again.
- `inputRef` is a file name or label only. It must never hold a counterparty name (rule 9 in `CLAUDE.md`).

### 2.2 Citations

Every value in `schedule` and `tokenConditions` that came from a source has one citation.

```json
{
  "path": "JSON Pointer into the file, e.g. /schedule/criteria/3/treatment/valuationTreatment/haircutPercentage",
  "value": "the value as written in the file",
  "sourceKind": "clause | spreadsheetCell",
  "quote": "verbatim clause text (agent) or the cell's raw value (mapper)",
  "location": "paragraph number, or sheet!cell",
  "decision": "approved | edited",
  "editedFrom": "original proposed value, only when decision = edited"
}
```

- Agent: the `quote` must be found verbatim in the input text, or the export fails. This check exists today and stays.
- Mapper: the `quote` is the raw cell content, with sheet, row and column.

### 2.3 Token conditions (extension)

The conditions the collateral taker sets on tokenised assets, which CDM can't hold yet:

```json
{
  "approvedNetworks": ["hedera-testnet", "..."],
  "acceptedTokenStandards": ["HTS", "ERC-3643", "..."],
  "acceptedCustodyModels": ["direct", "custodian-held"],
  "finalityRequired": "legal | technical",
  "acceptedCashLegTypes": ["tokenised-deposit", "stablecoin", "central-bank-money"],
  "flags": { "excludeIndexLinked": true }
}
```

`flags.excludeIndexLinked` is used only if open item B ends with the flag route (see §4.3).

### 2.4 Approval

```json
{
  "status": "approved",
  "approvedBy": "role label, e.g. 'legal reviewer'; no personal names in fixtures",
  "approvedAt": "ISO 8601 UTC",
  "valuesTotal": 24,
  "valuesApproved": 22,
  "valuesEdited": 2,
  "valuesRejected": 0
}
```

The file is valid only if `status = "approved"`, `valuesRejected = 0`, and `valuesApproved + valuesEdited = valuesTotal = citations.length`.

### 2.5 Hashes

1. `scheduleHash` = SHA-256 of the JCS-canonical JSON of `{ "schedule": …, "tokenConditions": … }`.
2. `fileHash` = SHA-256 of the JCS-canonical JSON of the whole file with `hashes.fileHash` set to `""`.

`scheduleHash` identifies the rules themselves. Two files with the same rules but different citations or approvers share a `scheduleHash`. `fileHash` seals everything.

### 2.6 Changes to the producers

| Producer | Change |
| --- | --- |
| Clause-to-CDM agent | Already exports a hash-sealed file. Align its fields and canonicalisation with §2–2.5 and add `formatVersion`. |
| cdm-collateral-mapper | New: export spreadsheet schedules in this format. Requires a person to approve in a review step (rule 4). Add an inclusive/exclusive marker per maturity bound to the spreadsheet format (open item A). |

## 3. Desk import: "Load approved schedule"

### 3.1 Verification, in order

Any failure stops the import and changes nothing (rule 5). The Desk shows which check failed and why.

| # | Check | Fails when |
| --- | --- | --- |
| V1 | Parse | Not valid JSON, or `formatVersion` unsupported |
| V2 | Approval | Any §2.4 condition is false |
| V3 | File hash | Recomputed `fileHash` ≠ stated |
| V4 | Schedule hash | Recomputed `scheduleHash` ≠ stated |
| V5 | CDM rules | `schedule` fails CDM validation (types, cardinality, conditions) |
| V6 | Bands | Maturity bands for the same asset class leave a gap or overlap, given each bound's inclusive flag |
| V7 | Citations | A cited `path` does not resolve, or a value has no citation |

### 3.2 Compare, then apply

1. **Compare.** Show the current and incoming schedules side by side: haircuts, maturity bands, limits, exclusions and token conditions. Highlight every difference.
2. **Apply.** On confirmation, rerun eligibility and allocation for the open call. Write a `ScheduleImported` event to the ledger with `scheduleId`, `scheduleVersion`, `scheduleHash`, `fileHash` and the importing role.
3. **Header chip.** Show which schedule is in force: *Built-in*, *Approved: `<scheduleId>` v`<n>`*, or *Modified* (an approved schedule edited in the Desk, which is not allowed in partner mode).

### 3.3 Present mode

Present mode always runs on the built-in schedule (rule 7). "Load approved schedule" is disabled in Present mode, and entering Present mode switches back to the built-in schedule without losing the imported one.

## 4. Eligibility engine: apply schedules as written

The engine is deterministic code (rule 3). CDM's `CheckEligibilityByDetails` is never used for a decision. It is run alongside in tests so every divergence is recorded.

### 4.1 Decision order for one holding against one schedule

1. **Exclusions first.** If any criterion with `isIncluded = false` matches, the holding is ineligible. Stop.
2. **Match inclusion rules.** Find every criterion with `isIncluded = true` whose conditions all match. Every condition must match: asset type, fund type, issuer type, country, currency, maturity band, rating and token conditions.
3. **No match → ineligible.**
4. **Several matches → apply the highest haircut** (most conservative). Log that more than one rule matched.
5. **Apply limits** across the whole posted portfolio, not per delivery (open item C).
6. **Apply token conditions** from `tokenConditions` against the asset's token data record (§6). Any failure → ineligible, with the reason.

Every decision returns `{eligible, haircut, ruleMatched, reasons[]}`. The reasons are shown in the buy-side and dealer views.

### 4.2 The four divergences

| ID | CDM behaviour today | Desk behaviour | Test case (minimum) |
| --- | --- | --- | --- |
| D1 | Exclusion rules dropped before checking | Exclusions are checked first and win (§4.1 step 1) | A holding matching both an inclusion rule and an exclusion rule is ineligible |
| D2 | Fund type not compared | Fund type must match when the rule states one | An equity ETF against a rule for sterling MMFs is ineligible |
| D3 | Period units not converted | All periods are converted to days before comparison: D=1, W=7, M=30.4375, Y=365.25. Bounds are compared on calendar dates where a reference date exists. | A 6-month upper bound rejects a 2-year bill; a 6M bound and a 0.5Y bound give the same result |
| D4 | Exclusive lower maturity bound appears reversed | Lower bound: `inclusive = true` means ≥, `false` means >. Upper bound likewise ≤ or <. | A 2054 gilt against an exclusive 10Y lower bound is eligible; a bond exactly at an exclusive bound is not |

For each, the test also records CDM's own answer, so the difference is proven and can be raised with FINOS. D4 is to be confirmed by this test before it is raised.

**Units for D3.** Converting months and years to days is an approximation. Where the call date is known, compare actual dates (call date + period) instead. The test suite covers both paths.

### 4.3 Open items from connecting the agent and the mapper

| ID | Decision for this spec | Test case |
| --- | --- | --- |
| A | Every maturity bound carries an explicit `inclusive` flag; the mapper's spreadsheet gets a column per bound. "Not less than X" → lower bound X, inclusive. | A gilt with exactly 1 year left matches the "≥ 1Y" band, and no gap appears between bands |
| B | Try the in-schedule route first: an exclusion criterion matching index-linked debt. If CDM's criteria can't identify index-linked gilts, use `tokenConditions.flags.excludeIndexLinked` and label it as extension. Plan week 1 decides which. | An index-linked gilt is ineligible under either route; a conventional gilt with the same maturity is eligible |
| C | The MMF concentration limit is 50% of aggregate posted collateral value after haircuts. It is applied across the portfolio for the call, not per delivery. | A response putting 60% in MMFs is reduced to 50%, and the optimiser fills the rest from the next cheapest eligible asset |

## 5. Holdings import through the mapper

- **Input:** a custodian CSV or the Desk's JSON registry, described by a field map.
- **Output:** a CDM `CollateralPortfolio`, plus one token data record (§6) per tokenised holding.
- **Rejects:** any row with an error, reported by file, row and column, and never half-mapped. The checks are ISIN check digit, currency, maturity date and settlement status.
- **Desk:** a "Load holdings" action with the same verify → compare → apply pattern as §3. There is no hash requirement for holdings in this increment.

## 6. Token data record (extension)

One record per tokenised asset, keyed by the underlying asset identifier. It lives in `extension/` and is never added to CDM objects.

Each fact has the same shape:

```json
{ "value": "...", "source": "issuer | transfer-agent | custodian | rating-agency | desk-fixture", "asOf": "ISO 8601", "attestedBy": "role label" }
```

| Group | Facts |
| --- | --- |
| Underlying asset (CDM can test) | ISIN, asset type, fund type, issuer, issuer type, issuer country, currency, maturity date, ratings by agency, debt features (e.g. index-linked), issue outstanding |
| Token (no CDM field yet) | network, token standard, token ID (ISO 24165 DTI where available), token issuer, relation to the underlying (native or record of a security held elsewhere), underlying issuer, custody model, finality point, cash-leg type, operating hours, transfer restrictions, redemption terms (fund units), legal opinion status |

**Rules**

- In this increment every fact has `source = "desk-fixture"`, so the data is clearly illustrative.
- A fact older than its schedule's tolerance is flagged *stale* in the view. The tolerance is 1 business day for ratings and 1 day for transfer restrictions.
- This record is also the draft **token eligibility profile** for the FINOS working group. Keep its field names stable and documented in `extension/README.md`.

## 7. Ledger adapter interface

All ledger access goes through one interface in `desk/adapters/` (rule 8). The simulated hash-chained ledger becomes its first implementation; no behaviour changes.

```ts
interface LedgerAdapter {
  readonly network: string;               // e.g. "simulated", "hedera-testnet"
  readonly capabilities: {
    atomicMultiLeg: boolean;              // collateral and cash in one transaction
    privacy: "public" | "permissioned";
    operatingHours: "24x7" | string;
  };

  getBalance(account: AccountRef, asset: AssetRef): Promise<Amount>;

  // Lock collateral pending acceptance; returns a lock that can be released or settled
  lock(req: TransferRequest): Promise<LockRef>;
  release(lock: LockRef, reason: string): Promise<TxReceipt>;

  // Settle one or more legs atomically; all or none
  settle(legs: TransferRequest[], lock?: LockRef): Promise<TxReceipt>;

  // Append a Desk event (CDM hash + event hash) to the record
  recordEvent(evt: DeskEvent): Promise<TxReceipt>;

  // Recompute the chain from the start; tamper evidence
  verifyChain(): Promise<{ ok: boolean; brokenAt?: number }>;
}
```

- `settle` on an adapter without `atomicMultiLeg` must refuse a multi-leg request rather than settle legs one by one.
- `TxReceipt` carries the network's transaction ID and the finality status (`pending | technical | legal`).
- Real-network adapters are out of scope for this increment. The interface must be implementable on Hedera (HTS atomic transfers), Canton and a permissioned EVM network; check that with a short paper exercise, not code.

## 8. Data handling

| Item | Requirement |
| --- | --- |
| Gemini Explain | Off by default. Turning it on needs an explicit setting and shows a banner stating that data leaves the environment. It is never available in partner mode unless a partner-approval flag is set. |
| Hosting | Runs self-hosted in a container behind a login (single shared password or OIDC, to decide in `plan.md`). No public demo URL with partner data. |
| Data at rest | Imported schedules and holdings are stored only in the deployment, never in browser storage shared across users. A "Clear partner data" action removes them. |
| Data-handling note | One page: what data the Desk needs, where it is stored, who can see it, what leaves the environment (nothing, by default), and how it is deleted. Lives at `docs/data-handling.md`. |
| Determinism | The same inputs give the same outputs and the same hashes. A "Reset" button returns to the built-in scenario. |

## 9. Business-case screen

A screen that compares today's process with the Desk for one VM call, using assumptions the partner can edit.

| Input (editable) | Default (illustrative) |
| --- | --- |
| Call size | £180m |
| Calls per year | 50 |
| Manual steps per call today | 12 |
| Time from call to agreed amount today | 4 hours |
| Share of calls landing outside CHAPS hours | 10% |
| Cost of carry of cash held for VM | 0.5% a year |

**Outputs:** time to an agreed call, manual steps removed, calls met outside payment-rail hours, and the cash buffer that could be held as gilts or MMFs instead.

Defaults are labelled illustrative, and a partner's own figures replace them in week 0 of the proof of concept (the baselines in `intent.md`).

## 10. Tests and evals

| Suite | Contents | Pass bar |
| --- | --- | --- |
| Divergences | D1–D4 and A–C (§4.2–4.3); each records the Desk answer and CDM's answer | 100% |
| FINOS samples | All five FINOS sample eligibility schedules validate | 0 errors |
| Format | Valid file accepted; one failing file per check V1–V7 rejected | 100% |
| Tamper | Change one byte in `schedule`, `tokenConditions`, `citations` or `approval`; each is rejected by V3 or V4 | 100% |
| Cross-check | Mapper and agent output passes the `finos-cdm` Python package | 0 errors |
| Present mode | Demo figures identical before and after importing a schedule | Identical |
| Determinism | Two runs of the scenario give identical events and hashes | Identical |

## 11. Out of scope for this increment

- Real ledger adapters, a second ledger and a real cash leg
- Multiple CSAs, or concentration limits beyond open item C
- Reporting at source (DRR)
- Single sign-on and role-based access beyond the login in §8

## 12. Open questions for Gate 2

1. Open item B: which route? This needs a look at CDM's debt-type criteria in week 1.
2. Login for self-hosting: a shared password for the demo, or OIDC from the start?
3. D3: is the calendar-date comparison acceptable as the default when a call date exists?
4. Business-case defaults: are these illustrative figures reasonable, or should the screen start empty?

## Sign-off

- [ ] Christopher: this spec is right and fits the 4-week increment
