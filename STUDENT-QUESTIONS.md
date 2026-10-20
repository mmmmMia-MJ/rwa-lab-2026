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
The closest on-chain equivalent is the assertion tBill.totalSupply() == custodian.realHoldings(), or, if we want a value-conservation view, custodian.realHoldings() + vault.reserveBalance() >= tBill.totalSupply() * nav / 1e20. It asserts that the system's own books are internally consistent: the value of every outstanding share is covered by the vault's cash plus the holdings the custodian reports.

What it stops proving is the original guarantee that the bills actually exist. realHoldings() is not measured on-chain; it is a number that a permissioned address types in. If that address is wrong or dishonest, the invariant still passes while no Treasury exists. Therefore, the check turns a claim about the physical world into a claim about a database. It catches bugs on-chain, but the real guarantee has moved off-chain—to the custodian's statements, auditors, and the law.
<br><br><br>

**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> Your answer:
The token is only a pointer; what makes it worth anything is who legally owns the Treasuries. If the issuer held them on its own balance sheet, its bankruptcy would pull the assets into the estate, and token holders would queue behind every other creditor. A bankruptcy-remote SPV is a separate legal entity with one job, ensuring its assets are ring-fenced from the sponsor's creditors. If the custodian fails, the bills sit in segregated accounts in the SPV's name and are not part of the custodian's estate.

What the token gives you is a pro-rata claim on the SPV's assets, exercised through a trustee, not title to a specific bill. That claim is enforced by courts, not by code. On-chain, realHoldings() cannot show any of this; it just trusts the SPV's word.
<br><br><br>

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> Your answer:
There are three primary failure modes.

Wrong data: A typo or bad data feed posts an incorrect NAV, repricing every share at once. Mitigation: cap how far one attestation can move the NAV and require a second signer for large moves.

Compromised or colluding: A stolen REPORTER_ROLE key makes one call to mint or redeem at a false price. Mitigation: put the role behind a multisig with a timelock and enforce legal liability with audit rights.

Absent (Stale NAV): No update arrives, leaving the NAV stale. Mitigation: store the update timestamp and make subscribe and enqueue revert when the NAV is older than a heartbeat limit.

Underneath all three is the single point of failure. Using a median of several independent reporters, plus an independent auditor, ensures no single number is final.
<br><br><br>

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> Your answer:

<brT-Bills accrue interest every day, so an honest NAV posted at close is already slightly out of date the next morning. Someone who knows the true value has moved can trade against it. If the true NAV is 1.0010 but the posted one is 1.0000, they subscribe at 1.0000 and get more shares per dollar than they should. They wait for the update, then enqueue to redeem at the corrected, higher price. This is a near risk-free profit paid for by existing holders, whose stake is diluted. The reverse works when true NAV falls: redeem at the stale, higher price before the correction. Nobody lied; the loss comes purely from timing. Defenses include forward pricing, cut-off times, strict staleness limits, and small entry/exit fees that exceed the daily drift.><br><br>

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> Your answer:
The price is locked at enqueue, but cash is paid at settle a day later, meaning the risk of NAV moving in that gap falls on someone other than the person who decided to leave. If NAV rises after enqueue, the redeemer is paid the old, lower amount, while remaining holders keep the gain. If NAV falls, the redeemer is paid the higher locked amount, so remaining holders absorb the loss, and the fund may be short of cash at settlement, threatening the settlement-solvency invariant. To reallocate risk, the queue could pay the lower of the locked amount and the NAV at settlement. The redeemer then carries the downside of the T+1 gap, protecting the remaining holders. A small redemption fee can cover the residual gap.
<br><br><br>

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> Your answer:
In 2008, the Reserve Primary Fund fell below $1 after losses on Lehman paper; a run followed, and redemptions were frozen. In March 2023, USDC dropped to roughly $0.87 when part of its reserves was stuck at Silicon Valley Bank. The lesson is that when holders cannot redeem at NAV, the secondary price falls below it, and panic feeds itself. For a tokenized T-Bill fund, closing the channel means tickets pile up and nobody can exit.

The queue should keep its FIFO order, keep accepting tickets, delay settlement only as far as cash is missing, and keep claim open, because settled cash already belongs to the claimants. A global pause() is a poor tool: it also blocks claim and transfers, trapping money already owed. A targeted gate on settle and enqueue fits a liquidity squeeze much better.
<br><br><br>

---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> Your answer:
The check lives in TBillToken._update, the one function every mint, transfer, and burn passes through, and it tests both endpoints. The legal duty attaches to who holds the security. Checking only the receiver would let a delisted holder move or exit shares freely. Checking only the sender would let a verified holder send shares to an unverified address. Putting the check in the token itself means every path (vault, queue, direct transfer) inherits it, with no side doors.

Whoever can call setWhitelisted moves the line. It should belong to a compliance officer's multisig, separate from the NAV reporter, the minter, and the operator, with events for every change and a timelock on additions. Separation of duties means one stolen key cannot both approve a buyer and mint to them.
<br><br><br>

**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Your answer:
For: The issuer is regulated and must comply with sanctions, court orders, and theft recovery; without freeze and burn, it cannot meet those duties. Against: The same powers let a compromised or abusive issuer erase anyone's holdings, so the holder's claim is only as strong as the issuer's discretion.

In this system, removing an address from the whitelist freezes it and also blocks the burn (burning from a non-whitelisted address reverts). Freezing and seizing pull in opposite directions, creating a deadlock. Safeguards include: splitting the roles (compliance officer can freeze; a separate legal-authority multisig can burn), requiring 3-of-5 signatures and a timelock on burns, emitting events with a case or court-order reference, and sending seized value to an escrow address rather than the issuer.
<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario | Your test function name |
|---|---|---|
| a | Shares exist while the custodian holds nothing |test_Ex2_SharesExistWhileTheCustodianHoldsNothing|
| a | `realHoldings()` moves with no cash moving |test_Ex2_RealHoldingsIsJustANumber|
| b | The claim value floats while the share count stays |test_Ex3_TheClaimFloats_TheShareCountDoesNot|
| b | One `attest` re-prices every holder at once |test_Ex3_OneCallMovesTheWholeBook|
| d | A non-whitelisted address cannot receive shares, even by paying |test_Ex4_NoKyc_NoShares_EvenIfYouPay|
| d | A whitelisted holder cannot send shares to a non-whitelisted address |test_Ex4_TransferChecksBothEndpoints|
| d | Removing an address freezes that holder — and blocks a burn |test_Ex4_FreezeBeatsConfiscation|

Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:

> Your answer:
The scenario most likely to be attacked: Plank B (Attestation) — NAV Manipulation.

A single compromised or bribed REPORTER_ROLE key can move the value of every share in one attest() call. Combined with the price lock at enqueue, this creates a highly profitable arbitrage for an insider or a stolen key. The attack path is straightforward: the attacker subscribes or enqueues at the current NAV, calls attest() to artificially inflate the NAV, and then redeems or settles at the new, higher price. The profit is paid by every other holder in the fund. The core vulnerability is that the contract only checks who is calling (the role), not whether the number is true. The chain trusts the number blindly.
