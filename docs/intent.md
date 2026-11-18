# Intent: Collateral Mobility Desk

- **Owner:** Christopher Mensah
- **Status:** Draft for sign-off (Gate 1)
- **Date:** 2026-11-15
- **Sources:** [Product brief](https://claude.ai/artifact/NcLkoAxYT3BPH1zDfqiCJZ) and [concept note](https://claude.ai/artifact/8dpGPkLhDe65Js3JZJL1HB) (October 2026)
- **Next stage reads this:** `spec.md`

## The problem in one sentence

When a sharp market move triggers a large variation margin call, the firm that must pay it often holds the right value in the wrong form, on settlement rails that may be closed, while every party keeps its own copy of the call. So meeting the call is slow, manual and risky exactly when speed matters most.

## Who has it

- **Collateral giver:** a pension scheme or its LDI manager. It holds its liquidity as gilts and money market fund (MMF) units, not cash.
- **Collateral taker:** usually a dealer, which needs eligible collateral on time.
- **In between:** custodians and operations teams, who move the assets and reconcile each firm's records.
- **Oversight:** the ledger operator and a supervisor.

These are the Desk's five seats: buy-side, dealer, custodian, ledger operator and supervisor. All five read one record.

## Root causes

| Root cause | What happens today | Consequence |
| --- | --- | --- |
| Value in the wrong form | VM is mostly paid in cash, so gilts or fund units are sold, redeemed or repo'd first | Forced sales into a falling market |
| Settlement rails close | CHAPS settles 06:00–18:00 on weekdays, so a Friday-evening call waits until Monday | Exposure left uncovered, or assets sold in a hurry |
| Every firm keeps its own copy | Each side calculates, keys and books the call separately | Breaks and disputes found after the fact |
| Terms live in documents | Eligibility rules and haircuts sit in PDFs and spreadsheets | Rekeying errors; new assets such as tokens checked by hand |

Tokenisation fixes the first two causes, because a tokenised MMF unit can move at any hour. It does not fix the last two. Checking the asset against the agreement, agreeing the call and recording the result still happen separately and by hand at each firm. The Desk exists to close that half.

## What to build

A desk that lets a buy-side firm meet a variation margin call with tokenised collateral, any day and any hour, without selling assets first. It is built on two layers:

- **Semantic layer (CDM):** the collateral agreement, the eligibility schedule and every step of the margin call as CDM objects, so every party reads the same meaning.
- **State layer (ledger):** one tamper-evident record of who holds what, and when it moved.

The value sits where the two layers meet: one golden record, collateral that moves between networks, atomic settlement and reporting at source.

The flow:

1. **Bring the terms in.** The Clause-to-CDM agent extracts eligibility terms from the CSA and a person approves them. The cdm-collateral-mapper turns schedules and custodian holdings into validated CDM.
2. **Run the call.** The Desk calculates the exposure, issues the call, checks every holding against the schedule with its haircut, and proposes the cheapest assets that cover it.
3. **Settle and record.** The parties agree, or settle the undisputed amount and dispute the rest. Tokenised assets move on the ledger, and each step is chained to the one before.
4. **Oversee.** Each seat sees the same record from its own view, and anyone can verify that nothing has changed.

## What solving it looks like

- A call is met with the assets already held, without a forced sale.
- Settlement takes minutes, on any day, rather than waiting for the next business day.
- There is one agreed record of the call, so nothing needs reconciling afterwards.
- New kinds of collateral, including tokens, are checked against the agreement automatically.

## Scope

**In scope:** variation margin on bilateral OTC derivatives under a collateral agreement, for a giver holding gilts and MMFs, starting in the UK.

**Out of scope:** initial margin, margin at clearing houses, valuation models and crypto-assets.

## This increment: make the prototype ready for design partners (about 4 weeks)

1. **Let them see their own world.** Load a partner's schedule as an approved, hash-sealed file, from its CSA through the agent or its spreadsheet through the mapper. Load its holdings through the mapper.
2. **Fix the four divergences.** Make the Desk's engine apply the schedule as written, with the mapper's tests as proof.
3. **Control where data goes.** Gemini Explain is off by default, the Desk is self-hosted behind a login, and there is a one-page data-handling note.
4. **Show the business case.** Add a today-versus-Desk screen with editable assumptions: time to an agreed call, manual steps and weekend cover.

**Gate:** the Desk is safe to show with anonymised real data.

## Success measures

**For this increment**

- The eligibility engine passes tests for all four divergences.
- The Desk imports an approved schedule file and rejects a tampered one.
- Five discovery calls are held with LDI and pension managers.
- One design partner signs a letter of intent.

**For the proof of concept that follows** (one partner, one CSA, VM only, 6–10 weeks)

- Extracted eligibility matches the partner's legal review on every in-scope clause.
- There are fewer breaks than the current process across the shadow run.
- The time from call to agreed amount is shorter.
- A weekend call is completed end to end.

## Non-negotiables

- CDM is used as generated by FINOS. The Desk's own additions, including the tokenisation extension, live in a separate extension keyed by the asset identifier and never in modified CDM.
- Schedules are applied as written, not as CDM's reference functions evaluate them. Every change is tested against CDM's functions.
- A person approves every extracted value beside its source clause before a schedule leaves the agent.
- The Desk verifies approval, both hashes and the CDM rules before it uses a schedule. Any failure changes nothing.
- AI features are off by default. No client data reaches an outside AI model unless the partner approves it.
- Networks sit behind one ledger adapter interface, so a partner's ledger is a new adapter, not a rewrite.
- Present mode always runs on the built-in schedule, so the demo figures cannot change.
- The Desk is described as CDM-aligned, not compliant, and as not affiliated with or endorsed by ISDA or FINOS.

## Known findings to carry into `spec.md`

**Four divergences in CDM's `CheckEligibilityByDetails`** (FINOS commit 830b53b, 16 Oct 2026)

1. Exclusion rules are dropped before checking, so excluded assets are never excluded.
2. Fund type is not compared, so an ETF matches an MMF rule.
3. Period units are not converted, so a 6-month bound reads as 6 years.
4. An exclusive lower maturity bound appears to be evaluated in reverse. This needs a test before it is raised with FINOS.

**Open items from connecting the agent and the mapper**

- **A. Inclusive lower bounds:** the mapper's spreadsheet format needs an inclusive marker per bound.
- **B. Index-linked exclusion:** use an in-schedule exclusion rule if CDM's criteria can identify index-linked gilts. Otherwise keep it as a flag labelled as part of the extension.
- **C. MMF limit:** the Desk's schedule must use 50% of aggregate collateral, as the clause states, not a flat amount per delivery.

## Decisions handed to `spec.md`

- The approved-schedule file format that the agent and the mapper both export
- Which ledger and cash leg the first adapter targets. This is chosen with the design partner from a permissioned EVM network, Canton or Hedera.
- The token data record: which facts it carries, who attests them and how it stays current
- How the four divergences are encoded and tested

## What exists today

| Piece | Status |
| --- | --- |
| Desk prototype | Built, on illustrative data, with a simulated ledger. It shows a £180m Friday-evening call met with tokenised MMF units and gilts, across five role views. |
| cdm-collateral-mapper | Built and tested: 15 tests pass, all five FINOS sample schedules validate, and its output passes the `finos-cdm` package. Not yet wired into the Desk. |
| Clause-to-CDM agent | Built: 81 tests pass, and it exports an approved, hash-sealed schedule. A live call to the model is still to be tested. |

## Risks

| Risk | Mitigation |
| --- | --- |
| Legal treatment of tokenised collateral is unsettled | Treat the legal view as a finding of the proof of concept, not a precondition |
| Slow access to partner data | Agree the anonymised extract in week 0 |
| No partner engages | Run paid workshops to build the pipeline |
| IP ownership is unclear | Settle IP terms before outreach |
| Scope creep | VM only, one agreement, at most two asset classes |

## Sign-off

- [ ] Christopher: this intent is right and in my words
