# STUDENT-QUESTIONS.md — Discussion questions (submit with your repo)

Answer directly under each question. 150–300 words each — **reasoning over length**.

---

## A. Permission design

**A1.** The vault holds `MINTER_ROLE`, so it can `burn` any user's balance. Explain why that is a risk, then write out how you would change `Vault` and `SimpleStablecoin` to remove it.

> Your answer:
The risk lies in the fact that MINTER_ROLE grants Vault the authority to both mint coins and arbitrarily destroy user tokens. If there are vulnerabilities in Vault or it is controlled by attackers, users' stablecoins could be destroyed without their consent. I will retain the minting function restricted by MINTER_ROLE in SimpleStablecoin, remove the permission for this role to arbitrarily destroy other users' tokens, and provide a burn function that only destroys the caller's own balance, as well as a burnFrom function that checks and deducts the user's authorized amount. In Vault, when redeeming, it will be changed to calling burnFrom(msg.sender, amount), requiring users to authorize Vault through approve in advance, then destroying the stablecoins and returning the collateral; these two operations should be completed in the same transaction and retain re-entry protection. Thus, Vault can still mint coins, but can only destroy its tokens within the authorized range of the user.
<br><br><br>

**A2.** In this contract `DEFAULT_ADMIN_ROLE`, `MINTER_ROLE` and `PAUSER_ROLE` all go to the same address. How would you split them in production, and who holds each?

> Your answer:
I will assign the three roles to different addresses: DEFAULT_ADMIN_ROLE is held by the project administrator and is responsible for granting and revoking permissions; MINTER_ROLE is only granted to the Vault contract, requiring users to deposit collateral before minting stablecoins; PAUSER_ROLE is held by another administrator responsible for security, who can suspend the system when abnormalities are detected. This way, even if a certain account is stolen, the attacker will not simultaneously obtain all permissions, and the responsibilities of each role are more clearly defined.
<br><br><br>

---

## B. Pausing and redemption

**B1.** `_update` is the single entry point for every balance change, so `pause()` freezes transfers, minting and redemption together. If you wanted "pause transfers but **allow redemption**", how would you change it? Give the approach — full code not required.

> Your answer:
I will change the pause check in _update to be based on the operation type instead of prohibiting all balance changes when pausing. For normal transfers, both the from and to addresses are not zero addresses, so such operations are prohibited when pausing; redemptions involve destroying stablecoins, and the to address is a zero address, so they can be allowed to execute. At the same time, check the redemption function of the Vault to ensure it is not blocked by whenNotPaused. Whether minting is paused is determined separately. For example, even when paused, minting is still prohibited, while only redemptions are retained, allowing users to exit.
<br><br><br>

**B2.** In 2008, when a money-market fund "broke the buck", redemptions were frozen for days. In 2023 USDC depegged to $0.87 after a reserve bank failed, but redemptions were **not** shut. Compare the two responses — what does closing the redemption channel, or leaving it open, do to a stablecoin?

> Your answer:
Closing the redemption function is like a bank suddenly saying "we can't withdraw money for now". Although it can temporarily stop the outflow of funds, users will be more worried about not being able to get their money back, and might prefer to sell the stablecoin at a price lower than $1, making the depegging more severe. Keeping the redemption function is like the bank still allowing withdrawals: as long as users believe that 1 token can indeed be exchanged for 1 dollar, even if the market price drops to $0.87, there will still be people buying and redeeming to earn the difference, thereby pushing the price back to $1. However, this is only possible if the reserves are sufficient and redemption can be fulfilled; otherwise, continuous redemption may also deplete the available funds.
<br><br><br>

---

## C. Depeg analysis

**C1.** Under what conditions does this coin depeg? Distinguish at least two classes of cause, and say how each one shows up in the invariant `totalCollateral() >= totalSupply()`.

> Your answer:
The first category is insufficient collateralization, such as the theft of collateral, a decline in the price, or excessive minting of stablecoins. In this case, the dollar value of the collateral is lower than the supply of stablecoins, and the invariant does not hold. This indicates that even if all the collateral is used up, it is still impossible to allow all users to redeem each token for 1 dollar, and the stablecoin may fall below 1 dollar. The second category is restricted redemption, such as a system-wide suspension of redemption. In this case, the collateral may still be sufficient, and the invariant still holds, but users cannot exchange the coins for collateral and can only sell them on the market; if many people are eager to exit, the price may fall below 1 dollar. Therefore, both sufficient collateralization and smooth redemption are important. Meeting only this invariant cannot guarantee the absence of deflation.
<br><br><br>

**C2.** Suppose an attacker bribes their way to `MINTER_ROLE`, mints 1,000,000 sUSD out of nothing and redeems it all. Describe the flow of funds, and name the step that could have stopped them.

> Your answer:
The attacker directly calls the mint function to create 1,000,000 sUSD without depositing collateral, then redeems it through the Vault. The Vault burns the attacker’s sUSD and transfers collateral deposited by legitimate users to the attacker. Assuming sufficient collateral is available, the attacker walks away with $1,000,000 worth of collateral, leaving other holders undercollateralized. The attack could have been stopped at the role-granting step: MINTER_ROLE should only be granted to the Vault, so minting must go through the collateral deposit process.
<br><br><br>

---

## D. Toward RWA

**D1.** Right now the collateral is `MockUSDC` and `totalCollateral()` just reads an on-chain balance — simple and reliable. If the collateral were **US Treasuries**, could this invariant still be written that way? What new problems appear?

> Your answer:
The invariant could still be written as totalCollateral() >= totalSupply(), but totalCollateral() could no longer simply read an on-chain balance. US Treasuries are held off-chain, so the contract would need custody reports or audits to confirm ownership and an oracle to report their dollar value. This introduces trust in custodians and data providers, and their reports could be outdated or inaccurate. Treasury prices can also fall as interest rates change. Finally, even if collateral value covers the supply, selling Treasuries and settling the trades takes time, so the system might not have enough cash available for immediate redemptions.
<br><br><br>

**D2.** If the collateral were **a building**, how would you put it inside this vault? Which off-chain roles or legal structures would you have to introduce?

> Your answer:
A building cannot be deposited directly into a smart contract. Instead, a special-purpose company (SPV) could legally own the building and issue tokens representing enforceable rights to it. Those tokens could then be deposited into the Vault. This would require lawyers to connect the tokens to legal rights, a property registry to confirm ownership, an independent appraiser to estimate the building’s value, and a property manager to maintain it and collect rent. The legal agreement must also explain who can sell the building after a default and how the proceeds would repay stablecoin holders. Simply creating a token is not enough, and selling a building takes time, so immediate redemption would be difficult.
<br><br><br>

---

## E. Tests (Tier 1 required — this is Ex4)

Turn the red tests green in `test/exercises/01_LoopTasks.t.sol` to cover the scenarios below, and write your test function names here:

| Scenario | Your test function name |
|---|---|
| Minting by a non-minter reverts |test_Ex4_Mint_RevertsForNonMinter |
| Transfers revert while paused |test_Ex4_Pause_BlocksTransfers |
| **Redemption** reverts while paused |test_Ex4_Pause_BlocksRedeem |
| An attacker cannot burn someone else's balance |test_Ex4_AttackerCannotBurnOthersBalance |
| ...but the vault holding `MINTER_ROLE` can |test_Ex4_VaultHoldsTheKey_CanBurnAnyonesBalance |

That last pair is meant to be read together: the guard is written correctly, but the key was handed to the vault. Keep it in mind when you answer A1.

Now write one more scenario you consider **most likely to be attacked**, and say why you picked it:

> Your answer:
I would test that depositing collateral without approving the Vault reverts and does not mint any sUSD. I chose this scenario because an attacker might try to mint stablecoins without transferring collateral. The test should confirm that the deposit fails and both the Vault’s collateral balance and the total sUSD supply remain unchanged.