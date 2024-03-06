# Case study with Kolibri
- Documentation: https://kolibri.finance/docs/general/intro
- Source code: https://github.com/Hover-Labs/kolibri-contracts/tree/master/smart_contracts

- `kolibri_minter_only.tzw` : Simplified version of Kolibri. 2 contracts are implemented.
    - `TokenContract` : Corresponds to the [Token](https://github.com/Hover-Labs/kolibri-contracts/blob/master/smart_contracts/token.py) contract. It records the amount of `kUSD` tokens for each addresses have.  For simplicity, only `mint` and `burn` entrypoints are ported.
    - `Minter` : Corresponds to the [Minter](https://github.com/Hover-Labs/kolibri-contracts/blob/master/smart_contracts/minter.py) contract.
    It has the following 5 core entripoints. Currently, only `borrow`, `repay`, and `liquidate` entypoints are ported.
        - `deposit` : deposit `XTZ` to oven.
        - `withdraw` : withdraw `XTZ` from oven. 
        - `borrow` : borrow `kUSD` tokens using `XTZ` in the oven as collateral.
        - `repay` : repay `kUSD` tokens.
        - `liquidate` : If the amount of `XTZ` in the oven is not enough to the amount of borrowed `kUSD`s, the oven can be liquidated.

- Verification goals for `kolibri_minter_only.tzw` :
    - The parameters (borrowedToken, stabilityFeeTokens, interestIndex) are correctly calculated
    - When a sufficient amount of XTZ is deposited, no one can liquidate the oven.
    - When the OraclePrice falls, the oven can be liquidated, and the liquidator will benefit financially.
    <!--
        In the original Kolibri code, the functions of the Minter contract is called from [Oven](https://github.com/Hover-Labs/kolibri-contracts/blob/master/smart_contracts/oven.py) contracts through the [Oven-Proxy](https://github.com/Hover-Labs/kolibri-contracts/blob/master/smart_contracts/oven-proxy.py) contract, and the Minter contract linked with the proxy contract can be updated with governance.
    -->

