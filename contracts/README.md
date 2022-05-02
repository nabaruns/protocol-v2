# Understanding of code base

# Isolation mode introduced in v3
New assets can be listed as isolated in Aave V3, which *previously were not allowed* to be listed due to certain issues like a lack of oracle price feed.
- Borrowers supplying an isolated asset as collateral cannot supply other assets as collateral (though they can still supply to capture yield). 
- Borrowers using an isolated collateral **can only borrow stablecoins** that have been permitted by the Aave governance to be borrowable in isolation mode, up to a specified debt ceiling.

## LendingPool
The [LendingPool contract](./protocol/lendingpool/LendingPool.sol) is the main contract of the protocol. It exposes all the user-oriented actions that can be invoked using either Solidity or web3 libraries.

 > LendingPool methods deposit, borrow, withdraw and repayare **only for ERC20**, if you want to deposit, borrow, withdraw or repay using native ETH (or native MATIC incase of Polygon), use WETHGateway instead.

Main point of interaction with an Aave protocol's market, users can:
- Deposit
- Withdraw
- Borrow
- Repay
- Swap their loans between variable and stable rate
- Enable/disable their deposits as collateral rebalance stable rate borrow positions
- Liquidate positions
- Execute Flash Loans

Functions and associated steps:

- [initialize](./protocol/lendingpool/LendingPool.sol#L86): invoked by the proxy contract when the LendingPool contract is added to the LendingPoolAddressesProvider of the market, 
    1. Set _maxStableRateBorrowSizePercent = 2500, _flashLoanPremiumTotal = 9, _maxNumberOfReserves = 128

- [deposit](./protocol/lendingpool/LendingPool.sol#L104): Deposits an `amount` of underlying `asset` into the reserve, receiving in return overlying aTokens. E.g. User deposits 100 USDC and gets in return 100 aUSDC. This can be called with separate onBehalfOf instead of the sender. 
    1. Get the reserve data using the `asset`'s address
    2. Get aToken address from reserve
    3. updateState of the reserve
    4. updateInterestRates of the reserve, using (reserveAddress, aTokenAddress, liquidityAdded=`amount`, liquidityTaken=0)
    5. IERC20(`asset`).safeTransferFrom(from=msg.sender, to=aToken, `amount`)
    6. If this isFirstDeposit for the user, setUsingAsCollateral(reserve.id, true) in config and emit ReserveUsedAsCollateralEnabled
    8. emit Deposit

- [withdraw](./protocol/lendingpool/LendingPool.sol#L142): Withdraws an `amount` of underlying asset from the reserve, burning the equivalent aTokens owned. E.g. User has 100 aUSDC, calls withdraw() and receives 100 USDC, burning the 100 aUSDC
    1. Get the reserve data using the `asset`'s address
    2. Get aToken address from reserve
    3. Get userBalance of the aToken
    4. validate withdraw with pre defined validation logic and getPriceOracle
    5. updateState of the reserve
    6. updateInterestRates of the reserve, using (asset, aTokenAddress, liquidityAdded=0, liquidityTaken=`amount`)
    7. Burn aToken
    8. emit Withdraw

- [repay](./protocol/lendingpool/LendingPool.sol#L236): Repays a borrowed `amount` on a specific reserve, burning the equivalent debt tokens owned. E.g. User repays 100 USDC, burning 100 variable/stable debt tokens of the `onBehalfOf` address
    1. Get the reserve data using the `asset`'s address
    2. Get stableDebt and variableDebt part of UserCurrentDebt
    3. Get interestRateMode using `rateMode` input
    4. Validate repay
    5. If interestRateMode is STABLE, then paybackAmount = stableDebt, else variableDebt
    6. if `amount` < paybackAmount, then paybackAmount = amount
    7. updateState of the reserve
    8. Burn stable or debt tokens as per the interestRateMode
       1. Variable debt has a variableBorrowIndex value in reserve
    9. Get aToken address from reserve
    10. updateInterestRates of the reserve, using (asset, aTokenAddress, liquidityAdded=paybackAmount, liquidityTaken=0)
    11. if (stableDebt + variableDebt - paybackAmount) == 0, then set user config borrowing status to false
    12. IERC20(`asset`).safeTransferFrom(msg.sender, aToken, paybackAmount)
    13. IAToken(aToken).handleRepayment(msg.sender, paybackAmount);
    14. emit Repay

- [swapBorrowRateMode](./protocol/lendingpool/LendingPool.sol#L297): Allows a borrower to swap his debt between stable and variable mode, or viceversa. 
    1. Get the reserve data using the `asset`'s address
    2. Get stableDebt and variableDebt part of UserCurrentDebt
    3. Get interestRateMode using `rateMode` input
    4. Validate swap rate
    5. updateState of the reserve
    6. If existing interestRateMode == STABLE, then burn IStableDebtToken from reserveand mint IVariableDebtToken with variableBorrowIndex. Else do vice-versa with currentStableBorrowRate
    7. updateInterestRates(asset, reserve.aTokenAddress, liquidityAdded=0, liquidityTaken=0);
    8. emit Swap

- [rebalanceStableBorrowRate](./protocol/lendingpool/LendingPool.sol#L350): Rebalances the stable interest rate of a user to the current stable rate defined on the reserve. Users can be rebalanced if the following conditions are satisfied: 1. Usage ratio is above 95%, 2. the (current deposit APY < REBALANCE_UP_THRESHOLD * maxVariableBorrowRate), which means that too much has been borrowed at a stable rate and depositors are not earning enough
    1. Get the reserve data using the `asset`'s address
    2. Get IERC20 stableDebtToken and variableDebtToken addresses and aTokenAddress from reserve reserve
    3. Get `user`'s stableDebt balance of stableDebtToken
    4. Validate rebalance
    5. updateState of the reserve
    6. Burn IStableDebtToken and mint new stableDebt using currentStableBorrowRate
    7. updateInterestRates(asset, aTokenAddress, liquidityAdded=0, liquidityTaken=0);
    8. emit RebalanceStableBorrowRate

- [setUserUseReserveAsCollateral](./protocol/lendingpool/LendingPool.sol#L387): unction to liquidate a non-healthy position collateral-wise, with Health Factor below 1. The caller (liquidator) covers `debtToCover` amount of debt of the user getting liquidated, and receives
   *   a proportionally amount of the `collateralAsset` plus a bonus to cover market risk
    1. Get the reserve data using the `asset`'s address
    2. validate SetUseReserveAsCollateral
    3. Set user config to use reserve as collateral
    4. emit ReserveUsedAsCollateralEnabled if `useAsCollateral`=1, else emit ReserveUsedAsCollateralDisabled

- [liquidationCall](./protocol/lendingpool/LendingPool.sol#L425): Function to liquidate a non-healthy position collateral-wise, with Health Factor below 1. The caller (liquidator) covers `debtToCover` amount of debt of the user getting liquidated, and receives a proportionally amount of the `collateralAsset` plus a bonus to cover market risk
  - receiveAToken is set true if the liquidators wants to receive the aTokens, false if he wants to receive the underlying asset directly
    1. getLendingPoolCollateralManager address

- [flashLoan](./protocol/lendingpool/LendingPool.sol#L483): Allows smartcontracts to access the liquidity of the pool within one transaction, as long as the amount taken plus a fee is returned.
  - Takes input `assets`[], `amounts`[], `modes`[]. Modes are the types of the debt to open if the flash loan is not returned:
    - 0 -> Don't open any debt, just revert if funds can't be transferred from the receiver
    - 1 -> Open debt at stable rate for the value of the amount flash-borrowed to the `onBehalfOf` address
    - 2 -> Open debt at variable rate for the value of the amount flash-borrowed to the `onBehalfOf` address
  - If the user chose to not return the funds, the system checks if there is enough collateral and eventually opens a debt position

- [_executeBorrow](./protocol/lendingpool/LendingPool.sol#L855): Allows users to borrow a specific `amount` of the reserve underlying asset, provided that the borrower already deposited enough collateral, or he was given enough allowance by a credit delegator on the corresponding debt token (StableDebtToken or VariableDebtToken). E.g. User borrows 100 USDC passing as `onBehalfOf` his own address, receiving the 100 USDC in his wallet and 100 stable/variable debt tokens, depending on the `interestRateMode`
    1. getAssetPrice amountInETH using IPriceOracleGetter
    2. mint debt tokens
    3. updateInterestRates
    4. if releaseUnderlying is true, then transferUnderlyingTo
    5. emit Borrow

## Lending Pool Collateral Manager
[LendingPoolCollateralManager](./protocol/lendingpool/LendingPoolCollateralManager.sol): Implements actions involving management of collateral in the protocol, the main one being the liquidations.

- [liquidationCall](./protocol/lendingpool/LendingPoolCollateralManager.sol#L81): Function to liquidate a position if its Health Factor drops below 1. The caller (liquidator) covers `debtToCover` amount of debt of the user getting liquidated, and receives a proportionally amount of the `collateralAsset` plus a bonus to cover market risk

- [_calculateAvailableCollateralToLiquidate](./protocol/lendingpool/LendingPoolCollateralManager.sol#L272): Calculates how much of a specific collateral can be liquidated, given a certain amount of debt asset.

## LendingPoolConfigurator
[LendingPoolConfigurator](./protocol/lendingpool/LendingPoolConfigurator.sol): implements the configuration methods for the Aave protocol

- [configureReserveAsCollateral](./protocol/lendingpool/LendingPoolConfigurator.sol#L287): Configures the reserve collateralization parameters all the values are expressed in percentages with two decimals of precision. A valid value is 10000, which means 100.00%
  - `asset` The address of the underlying asset of the reserve
  - `ltv` The loan to value of the asset when used as collateral
  - `liquidationThreshold` The threshold at which loans using this asset as collateral will be considered undercollateralized
  - `liquidationBonus` The bonus liquidators receive to liquidate this asset. The values is always above 100%. A value of 105% means the liquidator will receive a 5% bonus

## GenericLogic
[GenericLogic](./protocol/libraries/logic/GenericLogic.sol): Implements protocol-level logic to calculate and validate the state of a user

- [balanceDecreaseAllowed](./protocol/libraries/logic/GenericLogic.soll#L55): Checks if a specific balance decrease is allowed (i.e. doesn't bring the user borrow position health factor under HEALTH_FACTOR_LIQUIDATION_THRESHOLD)

- [calculateUserAccountData](./protocol/libraries/logic/GenericLogic.soll#L150): Calculates the user data across the reserves. This includes the total liquidity/collateral/borrow balances in ETH, the average Loan To Value, the average Liquidation Ratio, and the Health factor.

## AToken
[ATokens](./protocol/tokenization/AToken.sol) are yield-generating tokens that are minted and burned upon deposit and withdraw. The aTokens' value is pegged to the value of the corresponding deposited asset at a 1:1 ratio, and can be safely stored, transferred or traded. All interest collected by the aTokens reserves are distributed to aTokens holders directly by continuously increasing their wallet balance. 

All standard **EIP20** methods are implemented, such as `balanceOf()`, `transfer()`, `transferFrom()`, `approve()`, `totalSupply()`, etc.
> `balanceOf()` will always return the most up to date balance of the user, which includes their **principal balance + the interest** generated by the principal balance.

Also EIP2612 Methods like the function permit() allows a user to **permit another account (or contract) to use their funds** using a signed message. This enables gas-less transactions and single approval/transfer transactions.

Dependencies:

- [IERC20](./dependencies/openzeppelin/contracts/IERC20.sol)
- [SafeERC20](./dependencies/openzeppelin/contracts/SafeERC20.sol)
- [ILendingPool](./interfaces/ILendingPool.sol)
- [IAToken](./interfaces/IAToken.sol)
- [WadRayMath](./protocol/libraries/math/WadRayMath.sol)
- [Errors](./protocol/libraries/helpers/Errors.sol)
- [VersionedInitializable](./protocol/libraries/aave-upgradeability/VersionedInitializable.sol)
- [IncentivizedERC20](./IncentivizedERC20.sol)
- [IAaveIncentivesController](./interfaces/IAaveIncentivesController.sol)

Functions and associated steps:

- [initialize](./protocol/tokenization/AToken.sol#L55): Initializes the aToken

- [burn](./protocol/tokenization/AToken.sol#L120): Burns aTokens from `user` and sends the equivalent amount of underlying to `receiverOfUnderlying`. Only callable by the LendingPool, as extra state updates there need to be managed, 
    1. amountScaled = amount / liquidity index of the reserve
    2. _burn(user, amountScaled)
    3. IERC20(_underlyingAsset).safeTransfer

- [mint](./protocol/tokenization/AToken.sol#L144): Mints `amount` aTokens to `user`. Only callable by the LendingPool, as extra state updates there need to be managed
    1. previousBalance = balanceOf(user)
    2. amountScaled = mount / liquidity index of the reserve
    3. _mint(user, amountScaled)

- [permit](./protocol/tokenization/AToken.sol#L336): implements the [permit function](https://github.com/ethereum/EIPs/blob/8a34d644aacf0f9f8f00815307fd7dd5da07655f/EIPS/eip-2612.md) as for extending ERC-20 which allows for approvals to be made via secp256k1 signatures. This kind of "account abstraction for ERC-20" brings about two main benefits:
  - transactions involving ERC-20 operations can be paid using the token itself rather than ETH,
  - approve and pull operations can happen in a single transaction instead of two consecutive transactions
    1. previousBalance = balanceOf(user)
    2. amountScaled = mount / liquidity index of the reserve
    3. _mint(user, amountScaled)

## Debt Tokens
Debt tokens are interest-accruing tokens that are minted and burned on  and , representing the debt owed by the token holder. There are 2 types of debt tokens:
1. [Stable debt tokens](./protocol/tokenization/StableDebtToken.sol), representing a debt to the protocol with a stable interest rate
2. [Variable debt tokens](./protocol/tokenization/VariableDebtToken.sol), representing a debt to the protocol with a variable interest rate

Although debt tokens are modelled on the **ERC20/EIP20** standard, they are **non-transferrable**. Therefore they do **not** implement any of the standard ERC20/EIP20 functions *relating to transfer()* and allowance().
> `balanceOf()` will always return the most up to date accumulated debt of the user.
> `totalSupply()` will always return the most up to date total debt accrued by all protocol users for that specific type (stable vs variable) of debt token.

## Liquidation Atoken test
This [test script](../test-suites/test-aave/liquidation-atoken.spec.ts) mentions all possible tests for the Liquidation module.
- Deposits WETH, borrows DAI/Check liquidation fails because health factor is above 1
  - user 1 deposits 1000 DAI
  - User 2 deposits 1 WETH
  - User 2 borrows DAI
    - daiPrice = await oracle.getAssetPrice(dai.address)
    - amountDAIToBorrow = (availableBorrowsETH / daiPrice) * 0.95
    - pool.connect.borrow with params (dai.address, amountDAIToBorrow, RateMode.Variable)
- Drop the health factor below 1
- Tries to liquidate a different currency than the loan principal
- Tries to liquidate a different collateral than the borrower collateral
- Liquidates the borrow
- User 3 deposits 1000 USDC, user 4 1 WETH, user 4 borrows - drops HF, liquidates the borrow