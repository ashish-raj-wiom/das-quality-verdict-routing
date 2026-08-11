# CLAUDE.md — Quality Verdict in Routing

Guidance for Claude Code working in this folder. Read this first; it is written so a fresh session can continue without re-deriving anything.

## What this is

A PRD: **when Quality OS changes a CSP's composite state, that verdict must reach DAS and update the CSP's state there**, across every zone he works. Built with the `wiom-prd-gen` skill.

| | |
|---|---|
| **Status** | **v1.1 — SIGNED OFF, 11 Aug 2026** |
| Owner | Ashish Raj (PM) · Reviewer **Saurabh Goyal (EM)** · Consulted Quality OS **Akhil** |
| Canon | `DAS_Quality_Verdict_Routing_PRD.md` |
| Page | `index.html` — **generated. Never hand-edit.** |
| Tradeoffs | `DAS_Quality_Verdict_Routing_Tradeoffs.md` — 19 decisions + rejected options. Not part of the PRD |
| Live | https://ashish-raj-wiom.github.io/das-quality-verdict-routing/ |
| Repo | `ashish-raj-wiom/das-quality-verdict-routing` (public) |
| Chat | `csx resume b66d3a27` then `claude --resume` |

Shape: 4 guardrails · 1 metric · 4 rules · 3 transitions · 4 MQs · 24 ACs · 8 Overrides · **zero C-ids**.

## Build workflow — non-negotiable

```bash
node build-html.js && node check-parity.js
```

The markdown is canon. `check-parity.js` reduces both files to word bags and compares **both directions**; it exits 1 on any drift, so it gates the commit. Never edit `index.html`. Always rebuild + parity-check + commit both files together, then confirm the Pages build:

```bash
gh api repos/ashish-raj-wiom/das-quality-verdict-routing/pages/builds/latest --jq '.status'
```

Scripts are copied verbatim from `../returning-booking-context/`; only the two filename constants differ. Do not "improve" them.

## The finding the whole spec rests on

**The Quality→DAS pipe never existed.** DAS subscribes to event type `QUALITY_SLA_STATE_UPDATED` (`csp-demand-allocation-service/src/main/resources/application.yml:193`) and **no service in the platform produces it**. Quality emits `QUAL_STATE_COMPLIANT / AT_RISK / NON_COMPLIANT / RECOVERY / INSUFFICIENT_DATA`; Notification, Visibility, Capability Intervention and Exit subscribe — DAS subscribes to none. So `handleQualitySlaStateUpdated` → `updateExposureBandForQualityChange` → `INELIGIBLE_QUALITY` is complete, tested, and has **never executed**. Every `csp_zone_exposures.sla_state` in prod came from the `QUALITY_SEED_20260718` bulk script.

Also: Quality is **CSP-scoped** (INV-QUAL-26, one `sla_state_records` row per `csp_id`) and the prod daily job passes the literal string `"DEFAULT"` as the zone (`DailyWindowController.java:24,63`).

## Locked decisions — do not re-litigate

Full list with rejected options in the tradeoffs register. The ones most likely to be re-opened by accident:

- **Scope is the signal, not the algorithm.** *"The spec listens to the quality verdict for the csp and updates the csp state which might impact routing. Nothing else."* The PRD does **not** redefine the verdict→band mapping.
- **G4 "Downstream untouched"** — *"however the system works today, it should work."* The known faults (D1–D4) must reproduce unchanged. AC-GRD-4 tests exactly that.
- No zone named → all his zones. A real zone named → that pair only. `DEFAULT` means no zone.
- Never create an exposure record from a verdict; never widen an unknown named zone — drop and log.
- Newest **assessment time** wins, not newest arrival. The "or at the same instant" clause in R3/G3/T2 is load-bearing: it is what makes a re-delivery a no-op. Do not remove it.
- DAS never subscribes to the no-verdict signal, so a blocked CSP cannot free himself by going quiet.
- A new exposure record starts clean; nothing is backdated onto it.
- Block new work only; nothing already assigned, accepted or active moves.
- One metric: verdict accuracy, Quality vs DAS, 100%.

## Code ground truth — verified 10–11 Aug 2026, `csp-os-yaml` qa branch

Do not re-trace this; verify only if the branch has moved.

- `ExposureServiceImpl.updateExposureBandForQualityChange` (`:39-59`) — sets `sla_state`, derives the quality band, merges with `deriveNonQualityBand`, then increments/resets `consecutiveCompliantCycles` **after** the band is derived.
- `deriveExposureBandFromQuality` (`:145-160`) — cases for COMPLIANT / AT_RISK / NON_COMPLIANT / NOT_EVALUABLE_ZERO_BASE / INSUFFICIENT_DATA. **No RECOVERY case** → `default -> ELIGIBLE_BASELINE`.
- `resolveQualityTier` (`RoutingEngineServiceImpl:251-263`) — switches on the **band**, then asks only "is he AT_RISK?". D&A OS §4.3 defines **Tier 1 as the verdict `QUAL_STATE = COMPLIANT`**, not the band.
- Routing selects `eligibleSet.getFirst()` after sorting by (tier rank, load ratio, onboarding date).
- Quality's `CompositeSlaStateMachine` (enforced at `EvaluationCycleServiceImpl:242,563`): COMPLIANT→{AT_RISK,NON_COMPLIANT,INSUFFICIENT_DATA}; AT_RISK→{NON_COMPLIANT,RECOVERY,INSUFFICIENT_DATA}; NON_COMPLIANT→{RECOVERY,INSUFFICIENT_DATA}; RECOVERY→{COMPLIANT,AT_RISK,NON_COMPLIANT}; INSUFFICIENT_DATA→{COMPLIANT,AT_RISK,NON_COMPLIANT}. **No direct NON_COMPLIANT→COMPLIANT edge.**
- `QUAL_STATE_*` is emitted **every cycle, not only on change** — the emit call sits outside the `if (finalComposite != previousComposite)` block. That is what makes the P42 counter tick.
- Parameters **declared and never read**: P43 (recovery window), P44 (0.5 reduced multiplier), P45 (ramp), P53 (0.3 baseline), P194, P196. Read and live: P42 (=3), P46, P51, P59, P195.

## NEXT PIECE OF WORK — the anomaly follow-up

Deliberately excluded from v1.0/v1.1. *"Lets finalise the PRD first, then we look into this anomaly."* Belongs to the **allocation domain**, not this spec. Also recorded at the end of the tradeoffs register.

1. **Three rank inversions** — DAS moves the CSP the opposite way to Quality's judgement. at_risk→recovering demotes T2→T3; recovering→compliant demotes T3→**T4, the bottom**; recovering→at_risk **promotes** T3→T2. A CSP who dips one cycle is ranked worst exactly when he is fully compliant again, for ~4 cycles. (D1, D2, D3 in the PRD; the third inversion is **not** in the PRD.)
2. **Merge discipline — only 1 of 4 writers obeys OS rule B-DA-2.** `ExposureBand.mostRestrictive` itself is **correct** (order 8→1 matches the OS exactly). But quality (`:44`) merges all four inputs; enforcement (`:69`) merges only enforcement+quality, dropping exit and capacity; exit (`:87`) and capacity (`:100`) **hard-set a band with no merge** — which can *lower* severity and never recomputes on the way back up. This is the mechanism behind the band-stuck rows in the July audit. **Go-live consequence: the quality handler is the only correct writer, so the first real verdict will unstick stale `INELIGIBLE_EXIT` / `INELIGIBLE_CAPACITY` rows. Rows will move for reasons unrelated to their verdict.**
3. **No intensity anywhere.** Bands are a gate plus a sort key; §4.2's "reduced share" was never built. In a single-CSP zone, at_risk and compliant are functionally identical.
4. `ELIGIBLE_CAPPED` is never written by anything — the INV-DAO-16 Tier-3 throttle does not exist. If it were written, a quality recompute would silently clear it.
5. `ExposureStateMachine` is injected into `ExposureServiceImpl` and **never called**. Band writes are unvalidated.
6. DAS's `SlaState` carries `NOT_EVALUABLE_ZERO_BASE`, which Quality's `CompositeSlaState` does not have.

## Working style on this project

- **Verify against code before asserting.** Every claim in the PRD was traced to a file and line; the PM checks.
- **Small.** He cuts hard — a whole section, three metrics to one, six flow-chart branches to two. Write the small version first.
- **Never name a person from memory.** Unfilled consulted seats were *removed*, not invented.
- Windows + bash: forward slashes, `/dev/null` not `NUL`.
