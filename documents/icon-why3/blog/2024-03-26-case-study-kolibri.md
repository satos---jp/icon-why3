---
slug: case-study-kolibri
title: 'Case Study ; Kolibri'
authors:
  name: Sota Sato
  title: Engineer
tags: []
---

## What is Kolibri

[Kolibri](https://kolibri.finance/docs/general/intro) is a stablecoin built on Tezos. Kolibri has the `makeOven` entrypoint for users to create their ovens. The owner of the oven can deposit XTZ in their oven to mint `kUSD` tokens. The amount of mintable `kUSD` tokens is limited with the value of the deposited XTZ converted at the current XTZ-USD rate, so that the value of 1 `kUSD` token has at least 1 USD.

Each oven has following 4 entripoints for increasing/decreasing XTZ/kUSD.

```txt title="Cited from https://kolibri.finance/docs/general/intro"
Deposit: Place XTZ into the Oven
Withdraw: Remove XTZ from the Oven
Borrow: Borrow kUSD against the Oven using XTZ as collateral
Repay: Repay kUSD that was borrowed against the Oven.
```

The ratio of the deposited XTZ to the borrwed kUSD tokens is called as `collateralization ratio`. In order to ensure the convertability of kUSD to XTZ, the contract code forces the oven owners to maintain the ratio to be greater than 200%. Assertions prevent to drop the ratio to be lower than 200% in the `borrow` and the `withdraw` entripoints. However, the XTZ price might fluctuate, so that the ratio could drop below 200%. If it happens, anyone can call the `liquidate` entrypoint of the oven as the liquidator. The liquidator pays all of the kUSDs minted from the Oven, and receives all of the deposited XTZs with paying some fees. Assuming that the XTZ price doesn't change so rapidly, there would be a sufficient time when the collateralization ratio is still high enough to financially benefit the liquidator.

## Porting Kolibri code

The Kolibri contracts are implemented with [SmartPy](https://smartpy.io/), and the source code are available at https://github.com/Hover-Labs/kolibri-contracts/tree/master/smart_contracts.

Kolibri consists of about 10 contracts. We ported 2 of them, with some simplifications.
- [minter.py](https://github.com/Hover-Labs/kolibri-contracts/blob/master/smart_contracts/minter.py)\
  This contract implements the logic for four main functions; `Deposit`, `Withdraw`, `Borrow`, and `Repay`. This time we ported two of them; `Borrow` and `Repay`. We also ported the `liquidate` entripoint.

- [token.py](https://github.com/Hover-Labs/kolibri-contracts/blob/master/smart_contracts/token.py)\
  This contract implements the ledger for `kUSD` tokens with [FA1.2](https://tezos.gitlab.io/user/fa12.html) interfaces. We ported `mint` and `burn` entrypoints, those are called from the minter contract.

We ported the Python code to the why3 code. Each Python code could be straightforwardly converted to why3 code.

- Variable declartions are converted to `let` definitions.
```python title="minter.py"
timeDeltaSeconds = sp.as_nat(sp.now - self.data.lastInterestIndexUpdateTime)
```

```ml title="kolibri.tzw"
let timeDeltaSeconds = st.now - s.lastInterestIndexUpdateTime in
```

- Assertions are converted to terms followed by the conjunction `/\`.

```python title="minter.py"
sp.verify(collateralizationPercentage < self.data.collateralizationPercentage, message = Errors.NOT_UNDER_COLLATERALIZED)
```

```ml title="kolibri.tzw"
currentcollateralizationPercentage < s.collateralizationPercentage /\
```

- In iCon, user can define functions used for the specifications in the `Preamble` scope, so the function declarations and applications can be converted without code duplication.

```python title="minter.py"
def compoundWithLinearApproximation(params):
    sp.set_type(params, sp.TPair(sp.TNat, sp.TPair(sp.TNat, sp.TNat)))
    initialValue = sp.fst(params)
    stabilityFee = sp.fst(sp.snd(params))
    numPeriods = sp.snd(sp.snd(params))
    sp.result((initialValue * (Constants.PRECISION + (numPeriods * stabilityFee))) // Constants.PRECISION)
```

```ml title="kolibri.tzw"
function compoundWithLinearApproximation
  (initialValue : nat)
  (stabilityFee : nat)
  (numPeriods : nat) : nat =
  (initialValue * (v_PRECISION + (numPeriods * stabilityFee))) / v_PRECISION
```

- In SmartPy, `sp.transfer` is a function call with side-effects that updates the current operation array. It is ported to iCon as updating the `ops` variable.

```python title="minter.py"
tokenContractParam = sp.record(address= address, value= tokensToMint)
contractHandle = sp.contract(
    sp.TRecord(address = sp.TAddress, value = sp.TNat),
    self.data.tokenContractAddress,
    "mint"
).open_some()
sp.transfer(tokenContractParam, sp.mutez(0), contractHandle)
```

```ml title="kolibri.tzw"
let ops = Cons (Xfer (Gp'0mint'0address4nat ownerAddress tokensToMint) 0 s.tokenContractaddr) ops in
```

It would be hard to port everything faithfully, so we simplified the code as follows.
- In the original code, many oven parameters can be updated with governance. This function is omitted.
- In the original code, users originate their own `Oven`contract through the `makeOven` entrypoint. The actual oven logic is implemented in the `Minter` contract and it is called from the `Oven` contracts through the `OvenProxy` contract. This design enables updating the behaviour of the every ovens by updating the pointer to the `Minter` contract saved in the `OvenProxy` contract. We simplified them so that there is only one `Minter` contract which works as an oven.
- In the original code, the parameter `oraclePrice`, the exchange rate between XTZ and USD, is retrieved from the `Oracle` contract. For simplicity, we defined the value as `function oraclePrice : nat` without restriction.

## Verified properties

We verified following properties.

- Some basic properties; e.g. the constants are immutable / the storage of the minter contract is not modified when the token contract is called / the storage of the Minter becomes immutable after the `isLiquidated` flag is set.

- The `borrowedTokens` value recorded in the storage of the Minter is equal to the sum of the tokens recorded in the Token contract.
```ml
  predicate borrowedTokens_inv (c : ctx) =
    let ms = c.minter_storage in
    let balances = c.tokenContract_storage.TokenContract.balances in
    ms.borrowedTokens =
    balances[ms.ownerAddress] +
    balances[ms.developerFundContractAddress] +
    balances[ms.stabilityFundContractAddress]
```

- When liquidating, if the `collateralizationPercentage` is high enough, the liquidator benefits financially; i.e. the amount of XTZ earned is greater than the amount of kUSD tokens paid, by exchanging with the `oraclePrice` rate.
```ml
predicate liquidator_will_benefit (st : step) (gp : gparam) (c : ctx) (c' : ctx) =
  ((not c.minter_storage.isLiquidated) /\ c'.minter_storage.isLiquidated) ->
  match gp, st.entrypoint with
  | ICon.Gp (():unit), ICon.Contract.Liquidate ->
    let s = c.minter_storage in

    (* vvvvvvvvvvvv Following code helps z3 to verify the main property vvvvvvvvvvvv *)

    (* ommit *)

    (* ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ *)

    ((v_PRECISION + s.liquidationFeePercent) / v_PRECISION < currentcollateralizationPercentage ->
    st.sender = Dummy_Liquidator.addr ->
    (totalOutstandingTokens > 0 -> (* XXX: Introduced to avoid division by 0 in computeCollateralizationPercentage *)
    
    let earned_mutez = c'.dummy_Liquidator_balance - c.dummy_Liquidator_balance in
    let paid_token =
      c.tokenContract_storage.TokenContract.balances[st.sender] -
      c'.tokenContract_storage.TokenContract.balances[st.sender]
    in

    (* vvvvvvvvvvvv Following code helps z3 to verify the main property vvvvvvvvvvvv *)

    earned_mutez = st.balance /\
    ovenBalance >= 0 /\

    (* ommit *)

    (* ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ *)

    (* The following term is the main property we want to verify *)

    paid_token < earned_mutez * oraclePrice * v_MUTEZ_TO_KOLIBRI_CONVERSION * 100
    ))
  | _ -> false
  end
```

Especially the third property was the most difficult to verify.
The core term for the third property is `paid_token < earned_mutez * oraclePrice * v_MUTEZ_TO_KOLIBRI_CONVERSION * 100`, where the right side represents the paid kUSD token, and the left side of the inequality sign represents XTZs transfered to the liquidator account converted to the USD.
In the predicate, there are many other terms like `earned_mutez = st.balance /\`, `ovenBalance >= 0 /\`, etc. which seem to be redundant for verification, but they help z3 to verify the main property. If we remove them, the verification fails with timeouts.

The verification terms are too complex, so we need to used `z3-ce` instead of `z3` to verify.

```txt
$ why3 prove -P z3-ce examples/kolibri/kolibri.tzw 
File examples/kolibri/kolibri.tzw:
Goal interestindex_is_weakly_increasing'vc.
Prover result is: Valid (0.01s, 6437 steps).

File examples/kolibri/kolibri.tzw:
Goal totalOutstandingTokens_is_greaer_than_zero'vc.
Prover result is: Valid (0.08s, 140875 steps).

File examples/kolibri/kolibri.tzw:
Goal unknown'vc.
Prover result is: Valid (0.02s, 56162 steps).

File examples/kolibri/kolibri.tzw:
Goal dummy_Liquidator_func'vc.
Prover result is: Valid (0.02s, 40934 steps).

File examples/kolibri/kolibri.tzw:
Goal minter_func'vc.
Prover result is: Valid (3.33s, 17911566 steps).

File examples/kolibri/kolibri.tzw:
Goal tokenContract_func'vc.
Prover result is: Valid (0.02s, 45473 steps).
```

## References

- Kolibri documentation\
  https://kolibri.finance/docs/general/intro

- The source code of Kolibri\
  https://github.com/Hover-Labs/kolibri-contracts/tree/master/smart_contracts
