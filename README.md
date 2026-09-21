# Spout Finance beta review

[Download the complete source PDF](spout-beta-review.pdf)

> **The interface said my order was signed. The chain never saw it.**

I tested Spout Finance on Solana devnet on 18 and 19 September 2026. I funded a new embedded wallet with 5 SOL and 20 USDC, completed the identity flow, and tried to buy $20 of NVDA twice. Neither attempt became a buy transaction by my final check.

The first attempt returned `Blockhash not found` from `/api/orders/submit`. The second built an order but never issued the submit request. Spout showed the same green signed/submitting state in both cases and did not record either failure in Activity or Transaction History. That false-success state is the most urgent issue in this report.

## Test detail

| Test detail | Value |
|---|---|
| Build tested | `beta.spout.finance`, Solana devnet, v1.0.0 (`5d92f4c`) |
| Wallet | `DiiNsocoaZ9SeSk3DgxJH8gdGZynFcc9ydvRYMzuN69J` |
| Access card | No. 201 |
| Funding | 5 SOL and 20 USDC devnet |
| Test dates | 18-19 September 2026 |
| Assets used | Valueless devnet assets only |

## Findings

| # | Priority | Finding | Evidence status |
|---:|---|---|---|
| 1 | Blocker | The order UI reported a signed/submitting state after two different failures; neither buy appeared on-chain by my final check | Direct browser observation, reproduced twice |
| 2 | High, pre-mainnet | The NVDA mint has a permanent delegate, defaults new token accounts to frozen, and has a freeze authority, but the product does not explain these controls | Confirmed on-chain |
| 3 | High, pre-mainnet | Orders and Vault were upgradeable through the same on-curve authority during my check | Confirmed on-chain for the listed programs |
| 4 | High disclosure risk | The interface displayed `Reserves 100.2%` as a fixed client string and provided no path to an attestation | Confirmed in the client bundle; devnet reserve account checked separately |
| 5 | High design question | At the displayed 50% maximum LTV, a 2.0x position implies Health Factor 1.00 if Spout uses the same 50% as the liquidation threshold | Derived, not tested with a live position |
| 6 | Medium | `0% interest` is technically different from displayed Borrow Cost, but the prominent claim hides the economic cost in capped upside | Confirmed UI wording; accounting interpretation needs clearer disclosure |
| 7 | Resolved during testing | The production CSP blocked Spout's configured devnet RPC on 18 September | Confirmed during testing; no longer present in the 19 September check |
| 8 | Medium | NVDA market data disagreed between the table and asset page | Direct browser observation |
| 9 | Medium | The leverage slider was not keyboard operable; light-mode labels measured 3.32:1 contrast | Measured in the tested build |
| 10 | Medium | Terms and Privacy controls did not open a document or route | Direct browser observation |
| 11 | Low to medium | Amount-entry, rounding, minimum-order, responsive-layout, and stale Sell-panel defects remained | Direct browser observation |

## Scope and limitations

- Everything in this report concerns devnet. It does not establish how Spout will operate on mainnet.
- I did not complete a purchase, so borrowing, repayment, selling, collateral release, liquidation, and Earn were not tested end to end.
- I did not verify brokerage ownership, legal title, custodian records, real-world reserve attestations, or investment performance.
- I used the product UI for user flows and public, read-only RPC calls for chain inspection. I did not attempt an exploit, submit a hand-built transaction, or bypass a product control.
- `Blocker` means the issue stopped my flow in this beta account. It is not a security rating or evidence of production-wide impact.

## 1. Signed did not mean submitted

I attempted to buy $20 of NVDA. Privy opened, I signed, and Spout displayed:

> Order signed  
> Submitting it to Solana now. This can take a moment.

No position appeared, the USDC balance did not change, and the interface showed no error.

The first Network-panel path was:

```text
POST /api/orders/buy       200
POST /api/orders/submit    400
{"error":"Simulation failed. \nMessage: Transaction simulation failed: Blockhash not found."}
```

The second attempt followed a different path:

```text
POST /api/orders/buy       200
No following /api/orders/submit request
```

The common problem is the state model: wallet signature was treated as success before the backend had accepted, broadcast, confirmed, or settled the transaction. Neither attempt appeared in Portfolio Activity or Transaction History. The only new signature was a KYC identity transaction, not a buy.

### Recommended change

1. Treat wallet signature as an intermediate state, not success.
2. Show separate `signed`, `submitted`, `confirmed`, `settled`, and `failed` states.
3. Track `lastValidBlockHeight`; rebuild with a fresh blockhash when needed.
4. Make recovery idempotent so retries cannot create duplicate orders.
5. Log pending and failed attempts with reason codes and a transaction signature when one exists.
6. Tell the user plainly when nothing was charged.

## 2. Token controls are appropriate, but invisible

The NVDA asset uses a Token-2022 mint:

| Control | Address / value |
|---|---|
| Mint | `FkzjAJX584L1LncQb8AKSnGhc5sLvZjPGCgZLGpeHyTH` |
| Permanent delegate | `28bLmwghfAyZLCXpRZHpxa7uefN1TupnCftXfTqHzJm` |
| Default account state | `frozen` |
| Freeze authority | `GDBgF4AdFnK3ku8e7euZZLUag3JU9zDzbGktXTrzfMtE` |

These controls can be appropriate for restricted or regulated assets, but the interface uses simple ownership language without explaining the issuer and program authorities attached to the token. Each asset page should disclose the mint, token program, permanent delegate, default state, freeze authority, and the entity or program responsible for each authority.

## 3. The devnet upgrade model concentrates control

| Program | Address | Upgrade authority at check time |
|---|---|---|
| Orders | `SPoRXgsB4gWZmWPwyndoRWQrmZXKUc7o7oPMdkkGRcG` | `7N31cE8BRTpyAVDeczkutP4EnGdQJLSQm4q3SJ6aYWEp` |
| Vault | `spvaDgABYdpFKyatqo4Jr3nfwvVgWF5BbFxKFkzN3Am` | `7N31cE8BRTpyAVDeczkutP4EnGdQJLSQm4q3SJ6aYWEp` |
| KYC identity | `SKYCVrkX3mQaHwZcLrUtuvzii43kma7kBUim5MaQm6k` | `BD29wQ5Tj7b1MEFqquRxASsTqoN38oCrjxpU4riTZx7C` |

Orders and Vault shared the same direct, on-curve authority during the check. This is normal for a devnet beta, not an exploitable production finding by itself. Before mainnet, publish the multisig threshold, timelock or notice period, immutability plan, shared-authority policy, and emergency key-compromise procedure.

## 4. `Reserves 100.2%` looked measured, but was static

Every tested order panel displayed `Held 1:1 at Alpaca Securities - Reserves 100.2%`, while the footer said `US-regulated custody | Onchain proof of reserves`. I found the percentage as a literal string in the client bundle rather than a value fetched or computed from a visible reserve feed. The proof-of-reserves label did not link to an attestation, account, methodology, or timestamp.

The client declared devnet USDC account `ADZRYU7F5t4vzhBCihr9faNEYPbGAHYCiC8c3mkZKR2d`; its token balance was 0 USDC when checked. That does not establish anything about real-world reserves. It does show that the displayed percentage had no user-verifiable path to the devnet account shown in the same configuration.

A reserves page should show token supply, custodian-held shares or attested quantity, the ratio and formula, attestation scope, last-updated time, and links to the on-chain accounts involved.

## 5. Maximum leverage may coincide with the displayed liquidation boundary

The tested interface said a 2.0x position puts up half and borrows half, while eligible stock supports borrowing up to 50% LTV and Health Factor 1.00 is the liquidation point. If the liquidation threshold is also 50%, then:

```text
LTV = C / 2C = 50%
Health Factor = (2C * 0.50) / C = 1.00
```

I did not open a leveraged position, so this is a design question, not a live liquidation finding. The Buy panel should display projected Health Factor and liquidation price as leverage changes, and publish the exact LTV, threshold, oracle, market-closure, and gap-risk rules.

## 6. 0% interest needs the cost beside the claim

The interface showed `0% Interest on borrowing. Always` while the trade table showed nonzero annual Borrow Cost values. This may be economically consistent if the cost is capped upside rather than recurring cash interest, but the explanation was hidden in a tooltip and labels such as `Borrow Cost` and `Interest Rate` were inconsistent.

A clearer disclosure is: **0% cash interest. Your borrowing cost is capped upside. A covered call funds the loan. Spout shows the estimated annualized cost for each asset.** The estimate should state how strike, expiry, volatility, assignment, roll timing, and fees affect realized outcomes.

## 7. The CSP regression was no longer present on 19 September

On 18 September, the funded wallet displayed 0 USDC because the browser CSP blocked `https://api.devnet.solana.com`. By 19 September, the HTTP and WebSocket devnet RPC origins were present in `connect-src`, and balance, price, and KYC reads recovered. A deployment smoke test should fail whenever a configured RPC origin is absent from CSP.

## 8. Market data disagreed between screens

NVDA showed a `$5.36T` market cap in the trade table and `$2.59T` on its detail page. Its fundamentals grid also displayed incompatible units: `EPS $6.31`, `ROE $6.31`, `Dividend Yield 0.41%`, and `EPS (TTM) 0.41%`. Field-level schema validation should reject incompatible units before publication, and every value should carry a source and freshness timestamp.

## 9. Financial controls need keyboard and contrast fixes

The custom leverage slider had `tabindex="-1"`; ArrowRight did not change `aria-valuenow`. Light-mode 12px ticker labels measured 3.32:1 contrast against white, below the WCAG AA 4.5:1 requirement. At 1024px the asset-detail View control sat beneath the order panel; at 375px Borrow Cost text clipped `/yr`.

Use a native range input or a correctly focusable slider, increase small-text contrast, and add responsive layout tests at desktop and mobile widths.

## 10. Legal and notification controls were incomplete

Terms of Service and Privacy Policy were rendered as buttons that opened no route, modal, document, or new tab. Settings exposed enabled notification toggles, including a Health Factor alert, without showing an email address, phone number, push permission, or delivery channel. Legal links should resolve to versioned documents, and critical-risk alerts should show destination, opt-in status, and last successful delivery test.

## 11. Smaller interaction defects

- Switching Buy to Sell carried the Buy amount into a zero-holdings Sell panel.
- Switching dollar and share units changed the economic amount because shares rounded too early.
- Command+A followed by typing appended instead of replacing existing amount or search text on macOS.
- A `$1` order stayed enabled even though the client configuration declared a `$2` minimum.
- The Privy modal mixed decimal separators.
- Ask Spout did not submit with Enter.
- The product tour described Earn flows while the Earn page said coming soon.
- Export CSV remained enabled with no transactions.

Keep monetary precision in the model, round only for display, reset state when trade direction changes, respect native text selection, and validate minimums before opening the wallet flow.

## What I actually executed

The wallet dialog labelled the cluster **DEVNET TEST FUNDS** and stated the tokens have no value. External funding delivered 20 test USDC and 1 devnet SOL. One Spout buy order committed 10 test USDC at 1x. Placement finalized in slot `495881855` with no error; the order fulfilled at 13:31:09 UTC, 69 seconds after the displayed 13:30 opening. These are devnet protocol operations, not evidence of regulated real-world share settlement.

The fulfillment log identified `FulfillBuyOrderFreezeGated` for the same wallet and delivered `0.045302013` Token-2022 units. Looking only at classic SPL Token accounts misses that holding.

## What I would fix first

1. Replace the false-success order state with explicit signed, submitted, confirmed, settled, or failed states.
2. Log pending and failed orders in Activity with a reason code.
3. Disclose Token-2022 controls on every asset page.
4. Link the reserve claim to an attestation, or remove the percentage until one exists.
5. Publish the mainnet authority and upgrade plan.
6. Show Health Factor and liquidation price before a leveraged order is signed.
7. Put capped-upside cost beside the 0% cash-interest claim.
8. Add deployment tests for CSP, market-data schemas, keyboard controls, and responsive clipping.
9. Wire Terms, Privacy, and alert-delivery settings before public launch.

## Overall assessment

Spout's product thesis is worth pursuing. The interface explains a difficult options-backed lending model better than most early DeFi products I have tested, and the Token-2022 controls are consistent with a restricted-asset design. The beta still asks the user to trust important states that are not verifiable from the interface.

The strongest improvement would be to make every important claim traceable: an order should have a state and reference, a reserve percentage should have an attestation and timestamp, a token should disclose its authorities, and a leveraged position should show its liquidation assumptions before the user signs.

## Reproduction notes

The complete source report is preserved in [spout-beta-review.pdf](spout-beta-review.pdf). The report's raw RPC checks use `https://api.devnet.solana.com`; historical UI and Network-panel observations are labeled separately from later public snapshots.

This repository contains the source PDF and this readable Markdown version. No additional screenshots or raw RPC snapshot files were included with the supplied PDF.
