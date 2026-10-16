# STUDENT-QUESTIONS.md — Discussion questions (goes inside your submission zip)

Answer directly under each question. 150–300 words each — **reasoning over length**.

The sections follow the four planks of the bridge. There are no standard answers; these are the
real point of the lab.

---

## A. Custody — where the asset actually is (plank a)

**A1.** In Lab 1, `totalCollateral()` read the vault's own ERC-20 balance, and the chain could
prove the invariant. Here, `MockTBillCustodian.realHoldings()` is just a number that a permissioned
address can move. Write the equivalent of `totalCollateral() == totalSupply()` for *this* system.
What does it actually assert, and what does it no longer prove?

> Your answer:
>
> The equivalent is `vault.reserveBalance() + custodian.realHoldings() == vault.totalClaimValue()`, before considering pending redemptions, fees and rounding. Both sides use 6-decimal USDC value, because comparing raw USDC with 18-decimal shares would be wrong. This only says the reported assets match the reported share value. It no longer proves that actual T-Bills exist or belong to this fund.
>
> From Ex2, we can see the problem already: shares can exist before investment, and the custodian can increase its number without receiving cash. For queued redemptions, a fuller calculation must value circulating shares at current NAV, escrowed shares at their locked payouts, and include settled claims and queue cash. Otherwise we double-count or misprice liabilities.

<br><br><br>

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> Your answer:
>
> The legal structure is part of the design because Solidity cannot recover assets from a bank. The SPV is intended to separate fund assets from the issuer's own debts. The token needs a documented connection to that arrangement, otherwise it is only a balance in a contract.
>
> If the custodian fails, holders' rights depend on the fund documents, asset ownership, segregation and applicable insolvency rules. They may have a beneficial interest or a contractual claim pursued through the SPV or trustee; holding a token does not automatically give direct ownership of particular T-Bills. Recovery can take time and may be incomplete. This mock provides no legal entitlement by itself, so just having an on-chain balance is not enough lah.

<br><br><br>

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> Your answer:
>
> The main risk is trusting one price writer, because every subscription and new redemption uses that price. A stolen reporter key needs separate signers, a multisig and key rotation. A dishonest reporter needs independent valuation checks and accountable audits. A decimal or typing mistake needs price bounds and alerts. An unavailable reporter needs a backup and a freshness deadline. These reduce different risks, but more signers does not make bad source data correct.
>
> There is also a direct bypass in this lab: `MockPriceFeed.setPrice()` has no access control. Anyone can change the feed without calling `attest()`. Production must restrict that setter as well. The Ex6 invariants can still pass with a dishonest NAV, because they check escrow and settled cash accounting, not whether the price is true.

<br><br><br>

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> Your answer:
>
> A stale price can give an informed user a better deal, because the contract still trades at the old value. If actual NAV rises from 1.00 to 1.10 while the feed stays at 1.00, paying 1,000 USDC buys 1,000 shares instead of about 909.09. Existing holders lose out a bit here because their shares get diluted. If actual NAV falls to 0.90, an existing holder can enqueue at the old 1.00 and lock an excessive payout, subject to cash becoming available.
>
> I would reject subscriptions and new redemption pricing after a freshness deadline, then use a published valuation cutoff. The feed must store the real update time: this mock returns the current block timestamp on every read, so checking that timestamp alone would not detect an old price.

<br><br><br>

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> Your answer:
>
> The redeemer misses the later gain, because enqueue fixes the payout. For 100 shares queued at NAV 1.00, the claim stays 100 USDC even if NAV reaches 1.10 before settlement. Economically, remaining holders retain the subsequent upside, assuming the assets really gained value; the contract does not automatically calculate that gain for them. If NAV falls, the remaining holders instead bear the loss against the fixed claim.
>
> I think this allocation is okay, as long as everyone know the cutoff clearly. I would use a published dealing cutoff with a validated NAV and explain when exposure ends. Pricing at settlement is another option, but makes the payout uncertain and gives settlement timing more influence. Neither approach guarantees immediate liquidity.

<br><br><br>

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> Your answer:
>
> The main difference is liquidation versus restoring normal redemption. The Reserve Primary Fund fell to $0.97 in 2008 and suspended redemptions before distributing assets through liquidation ([SEC](https://www.sec.gov/divisions/investment/guidance/reservefundmmffaq.htm)). During USDC's March 2023 disruption, banking access constrained redemption; Circle subsequently processed redemptions and reported substantially clearing the backlog by March 15 ([Circle](https://www.circle.com/blog/march-15-update-on-usdc-operations?hss_channel=tw-2151686839)).
>
> Closing this fund's redemption channel should suspend the affected requests or settlements under a clear policy. Existing tickets, locked amounts and ordering should remain recorded; cannot just delete them and start again. Any change to payout treatment needs disclosed rules.
>
> The token pause is too broad for a cash shortage because it blocks transfers, subscriptions, enqueue and settlement burns together. Already-settled USDC claims still work, because `claim()` does not move tBILL. I would separate subscription, request and settlement controls, keep funded claims available, and publish reopening conditions. A temporary cash delay should not freeze every useful operation.

<br><br><br>

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> Your answer:
>
> The right line is allowing eligible, verified investors to hold and transfer shares, because this fund has admission rules beyond owning a wallet. Both sender and receiver need checking, including mint and burn endpoints. An ordinary user waiting for KYC should receive a clear explanation and a review route; they should not be treated as malicious simply because paperwork is unfinished.
>
> A designated compliance team should manage the list under documented policy, using a multisig for sensitive changes. Developers should not change the guest list just because it is more convenient. I would record reasons and approvals privately, emit public change events, and provide an appeal process. Personal KYC documents should stay off-chain. Emergency restrictions need fast handling, but mistakes also need a clear way to be corrected.

<br><br><br>

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Your answer:
>
> Issuer powers can support compliance and recovery, because a fund may need to restrict an account or act on a valid legal instruction. However, a stolen or dishonest key can censor users, destroy their balances, or mint unsupported shares. A burn alone also does not transfer assets to a lawful recipient or settle the holder's legal rights.
>
> Ex4 shows that once the address is frozen, the burn also cannot go through. Temporarily whitelisting it is a poor workaround because it opens a transfer window. If forced recovery is required, I would design a separate, narrowly scoped process with independent approval and a recorded reason.
>
> The compliance multisig should hold whitelist authority; a separate recovery multisig should approve exceptional actions. Routine mint and burn permissions should stay with the vault and queue. Administrative changes need a timelock, while an emergency pause key should have limited authority and prompt review.

<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario | Your test function name |
|---|---|---|
| a | Shares exist while the custodian holds nothing | `test_Ex2_SharesExistWhileTheCustodianHoldsNothing` |
| a | `realHoldings()` moves with no cash moving | `test_Ex2_RealHoldingsIsJustANumber` |
| b | The claim value floats while the share count stays | `test_Ex3_TheClaimFloats_TheShareCountDoesNot` |
| b | One `attest` re-prices every holder at once | `test_Ex3_OneCallMovesTheWholeBook` |
| d | A non-whitelisted address cannot receive shares, even by paying | `test_Ex4_NoKyc_NoShares_EvenIfYouPay` |
| d | A whitelisted holder cannot send shares to a non-whitelisted address | `test_Ex4_TransferChecksBothEndpoints` |
| d | Removing an address freezes that holder — and blocks a burn | `test_Ex4_FreezeBeatsConfiscation` |

Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:

> Your answer:
>
> The scenario I would prioritise in this implementation is direct NAV manipulation through the public `MockPriceFeed.setPrice()`. It breaks plank **b, attestation**, because an attacker does not even need the reporter key.
>
> A whitelisted attacker can subscribe at NAV 1.00, call `setPrice()` with a much higher value, and enqueue shares to lock an inflated payout. If the authorised settler later processes that ticket with sufficient reserves, the attacker can claim excess USDC. The attacker cannot call settlement without its role, but the fake price has already entered the queue. An unwhitelisted caller can also change the feed and disrupt pricing, although they cannot normally acquire shares.
>
> I choose this as the most exposed scenario in the lab because the setter is public and there is no price-change limit. This is a judgement about this code, not a claim about attack frequency across all real funds. Restricting feed writes, validating changes and checking genuine update timestamps should come before production use. A regression test should first demonstrate the bypass, then confirm unauthorised writes revert after the feed is secured. That extra test is proposed here; it is not one of the seven completed Ex2–Ex4 tests.
