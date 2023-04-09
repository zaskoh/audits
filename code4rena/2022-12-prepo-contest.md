## Summary

### Findings

|              | Issue                                                                                 |
| ------------ | ------------------------------------------------------------------------------------- |
| [H-01](#H01) | griefing / blocking / delaying users to withdraw                                      |
| [M-01](#M01) | Hacked owner or malicious owner can immediately steal or lock user and platform funds |
| [M-02](#M02) | Unallowed minting of Short and Long tokens                                            |

---

## High

### <a id="H01"></a> H-01 griefing / blocking / delaying users to withdraw
To withdraw, a user needs to convert his collateral for the base token. This is done in the **withdraw** function in Collateral.  
The WithdrawHook has some security mechanics that can be activated like a global max withdraw in a specific timeframe, also for users to have a withdraw limit for them in a specific timeframe. It also collects the fees.

The check for the user withdraw is wrongly implemented and can lead to an unepexted delay for a user with a position **> userWithdrawLimitPerPeriod**. To withdraw all his funds he needs to be the first in every first new epoch (**lastUserPeriodReset** + **userPeriodLength**) to get his amount out. If he is not the first transaction in the new epoch, he needs to wait for a complete new epoch and depending on the timeframe from **lastUserPeriodReset** + **userPeriodLength** this can get a long delay to get his funds out.

The documentation says, that after every epoch all the user withdraws will be reset and they can withdraw the next set.

```solidity
File: apps/smart-contracts/core/contracts/interfaces/IWithdrawHook.sol
63:   /**
64:    * @notice Sets the length in seconds for which user withdraw limits will
65:    * be evaluated against. Every time `userPeriodLength` seconds passes, the
66:    * amount withdrawn for all users will be reset to 0. This amount is only
```

But the implementation only resets the amount for the first user that interacts with the contract in the new epoch and leaves all other users with their old limit. This can lead to a delay for every user that is on his limit from a previous epoch until they manage to be the first to interact with the contract in the new epoch.

#### Proof of Concept
https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/WithdrawHook.sol#L66-L72

The following test shows how a user is locked out to withdraw if he's at his limit from a previous epoch and another withdraw is done before him.

apps/smart-contracts/core/test/WithdrawHook.test.ts  
```node
  describe('user withdraw is delayd', () => {
    beforeEach(async () => {
      await withdrawHook.setCollateral(collateral.address)
      await withdrawHook.connect(deployer).setWithdrawalsAllowed(true)
      await withdrawHook.connect(deployer).setGlobalPeriodLength(0)
      await withdrawHook.connect(deployer).setUserPeriodLength(TEST_USER_PERIOD_LENGTH)
      await withdrawHook.connect(deployer).setGlobalWithdrawLimitPerPeriod(0)
      await withdrawHook.connect(deployer).setUserWithdrawLimitPerPeriod(TEST_USER_WITHDRAW_LIMIT)
      await withdrawHook.connect(deployer).setDepositRecord(depositRecord.address)
      await withdrawHook.connect(deployer).setTreasury(treasury.address)
      await withdrawHook.connect(deployer).setTokenSender(tokenSender.address)
      await testToken.connect(deployer).mint(collateral.address, TEST_GLOBAL_DEPOSIT_CAP)
      await testToken.connect(deployer).mint(user.address, TEST_GLOBAL_DEPOSIT_CAP)
      await testToken.connect(deployer).mint(user2.address, TEST_GLOBAL_DEPOSIT_CAP)
      await testToken
        .connect(collateralSigner)
        .approve(withdrawHook.address, ethers.constants.MaxUint256)
      tokenSender.send.returns()
    })

    it('reverts if user withdraw limit exceeded for period', async () => {
      
      // first withdraw with the limit amount for a user
      await withdrawHook.connect(collateralSigner).hook(user.address, TEST_USER_WITHDRAW_LIMIT, TEST_USER_WITHDRAW_LIMIT)      
      expect(await withdrawHook.getAmountWithdrawnThisPeriod(user.address)).to.eq(TEST_USER_WITHDRAW_LIMIT)
      
      // we move to a new epoch in the future
      const previousResetTimestamp = await getLastTimestamp(ethers.provider)
      await setNextTimestamp(
        ethers.provider,
        previousResetTimestamp + TEST_USER_PERIOD_LENGTH + 1
      )
      
      // now another user is the first one to withdraw in this new epoch      
      await withdrawHook.connect(collateralSigner).hook(user2.address, TEST_USER_WITHDRAW_LIMIT, TEST_USER_WITHDRAW_LIMIT)      
      expect(await withdrawHook.getAmountWithdrawnThisPeriod(user2.address)).to.eq(TEST_USER_WITHDRAW_LIMIT)
      
      // this will revert, because userToAmountWithdrawnThisPeriod[_sender] is not reset
      // but it should not revert as it's a new epoch and the user didn't withdraw yet
      await expect(
        withdrawHook.connect(collateralSigner).hook(user.address, 1, 1)
      ).to.revertedWith('user withdraw limit exceeded')
      
    })
  })
```
To get the test running you need to add **let user2: SignerWithAddress** and the user2 in **await ethers.getSigners()**

#### Recommended Mitigation Steps
The check how the user periods are handled need to be changed. One possible way is to change the lastUserPeriodReset to a mapping like
**mapping(address => uint256) private lastUserPeriodReset** to track the time for every user seperatly.

With a mapping you can change the condition to
```solidity
File: apps/smart-contracts/core/contracts/WithdrawHook.sol
18:   mapping(address => uint256) lastUserPeriodReset;

File: apps/smart-contracts/core/contracts/WithdrawHook.sol
66:     if (lastUserPeriodReset[_sender] + userPeriodLength < block.timestamp) {
67:       lastUserPeriodReset[_sender] = block.timestamp;
68:       userToAmountWithdrawnThisPeriod[_sender] = _amountBeforeFee;
69:     } else {
70:       require(userToAmountWithdrawnThisPeriod[_sender] + _amountBeforeFee <= userWithdrawLimitPerPeriod, "user withdraw limit exceeded");
71:       userToAmountWithdrawnThisPeriod[_sender] += _amountBeforeFee;
72:     }
```

With this change, we can change the test to how we would normaly expect the contract to work and see that it is correct.

```node
    it('withdraw limit is checked for every use seperatly', async () => {
      
      // first withdraw with the limit amount for a user
      await withdrawHook.connect(collateralSigner).hook(user.address, TEST_USER_WITHDRAW_LIMIT, TEST_USER_WITHDRAW_LIMIT)      
      
      // we move to a new epoch in the future
      const previousResetTimestamp = await getLastTimestamp(ethers.provider)
      await setNextTimestamp(
        ethers.provider,
        previousResetTimestamp + TEST_USER_PERIOD_LENGTH + 1
      )
      
      // now another user is the first one to withdraw in this new epoch      
      await withdrawHook.connect(collateralSigner).hook(user2.address, TEST_USER_WITHDRAW_LIMIT, TEST_USER_WITHDRAW_LIMIT)      
      
      // the first user also can withdraw his limit in this epoch
      await withdrawHook.connect(collateralSigner).hook(user.address, TEST_USER_WITHDRAW_LIMIT, TEST_USER_WITHDRAW_LIMIT)      
      
      // we move the time, but stay in the same epoch
      const previousResetTimestamp2 = await getLastTimestamp(ethers.provider)
      await setNextTimestamp(
        ethers.provider,
        previousResetTimestamp2 + TEST_USER_PERIOD_LENGTH - 1
      )

      // this now will fail as we're in the same epoch
      await expect(
        withdrawHook.connect(collateralSigner).hook(user.address, 1, 1)
      ).to.revertedWith('user withdraw limit exceeded')
      
    })
```

---

## Medium

### <a id="M01"></a> M-01 Hacked owner or malicious owner can immediately steal or lock user and platform funds
The Collateral and PrePOMarket contracts hold protocol collateral funds and user funds.

The protocol will provide the collateral via:
```solidity
File: apps/smart-contracts/core/contracts/PrePOMarket.sol
65:   function mint(uint256 _amount) external override nonReentrant returns (uint256) {
```

Users will use their tokens as collateral via:
```solidity
File: apps/smart-contracts/core/contracts/Collateral.sol
45:   function deposit(address _recipient, uint256 _amount) external override nonReentrant returns (uint256) {
```

If the owner is hacked or malicious there are two attack vectors that lead to locked funds or stolen funds. For easier review I put both together. If needed they could viewed as two different attack vectors.

1. **stealing all user funds**
Here the owner will deactivate the **managerWithdrawHook**, set a new **manger** and just use the **managerWithdraw** to just drain all tokens.

```solidity
File: apps/smart-contracts/core/contracts/Collateral.sol
85:   function setManager(address _newManager) external override onlyRole(SET_MANAGER_ROLE) {
=> set to an attacker address

File: apps/smart-contracts/core/contracts/Collateral.sol
112:   function setManagerWithdrawHook(ICollateralHook _newManagerWithdrawHook) external override onlyRole(SET_MANAGER_WITHDRAW_HOOK_ROLE) {
=> set to address(0)

File: apps/smart-contracts/core/contracts/Collateral.sol
80:   function managerWithdraw(uint256 _amount) external override onlyRole(MANAGER_WITHDRAW_ROLE) nonReentrant {
=> just withdraw balance of contract in _amount
```

2. **locking all user / protocol funds**
Here the owner will set the **_redeemHook** (PrePOMarket) and the **withdrawHook** (Collateral) to an contract that will always revert if called.

```solidity
File: apps/smart-contracts/core/contracts/PrePOMarket.sol
96:       _redeemHook.hook(msg.sender, _collateralAmount, _collateralAmount - _expectedFee); 
=> if this fail, it's not possible to redeem anything for users

File: apps/smart-contracts/core/contracts/Collateral.sol
73:       withdrawHook.hook(msg.sender, _baseTokenAmount, _baseTokenAmountAfterFee);
=> if this fail, it's not possible to withdraw anything for users
```

#### Impact
Hacked owner or malicious owner can immediately steal all assets or lock them forever.

#### Recommended Mitigation Steps
All critical functions (at least in apps/smart-contracts/core/contracts/Collateral.sol) should implement a timelock, to give users at least enough time to withdraw their tokens before some malicious action becomes possible and their tokens are drained from the attacker.


### <a id="M02"></a> M-02 Unallowed minting of Short and Long tokens
The documentation states that minting of the Short and Long tokens should only be done by the governance.

```solidity
File: apps/smart-contracts/core/contracts/interfaces/IPrePOMarket.sol
73:    * Minting will only be done by the team, and thus relies on the `_mintHook`
74:    * to enforce access controls. This is also why there is no fee for `mint()`
75:    * as opposed to `redeem()`.
```

The problem is, that as long as the **_mintHook** is not set via **setMintHook**, everyone can use the mint function and mint short and long tokens.
At the moment the **_mintHook** is not set in the contructor of PrePOMarket and so the transaction that will set the **_mintHook** can be front run to mint short and long tokens for the attacker.

#### Proof of Concept
https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/PrePOMarket.sol#L68
https://github.com/prepo-io/prepo-monorepo/blob/feat/2022-12-prepo/apps/smart-contracts/core/contracts/PrePOMarket.sol#L109

This test shows how a attacker could frontrun the **setMintHook** function

```node
  describe('# mint front run attack', () => {
    let mintHook: FakeContract<Contract>
    beforeEach(async () => {
      prePOMarket = await prePOMarketAttachFixture(await createMarket(defaultParams))
    })
    it('user can frontrun the set setMintHook to mint long and short tokens', async () => {
      
      mintHook = await fakeMintHookFixture()
      await collateralToken.connect(deployer).transfer(user.address, TEST_MINT_AMOUNT.mul(2))
      await collateralToken.connect(user).approve(prePOMarket.address, TEST_MINT_AMOUNT.mul(2))

      // we expect the mintHook to always revert
      mintHook.hook.reverts()

      // attacker frontruns the setMintHook, even we expect the mintHook to revert, we will mint
      await prePOMarket.connect(user).mint(TEST_MINT_AMOUNT)

      // governance sets the mintHook
      await prePOMarket.connect(treasury).setMintHook(mintHook.address)

      // as we expect minthook to revert if not called from treasury, it will revert now
      mintHook.hook.reverts()
      await expect(prePOMarket.connect(user).mint(TEST_MINT_AMOUNT)).to.be.reverted

      // we should now have long and short tokens in the attacker account
      const longToken = await LongShortTokenAttachFixture(await prePOMarket.getLongToken())
      const shortToken = await LongShortTokenAttachFixture(await prePOMarket.getShortToken())
      expect(await longToken.balanceOf(user.address)).to.eq(TEST_MINT_AMOUNT)
      expect(await shortToken.balanceOf(user.address)).to.eq(TEST_MINT_AMOUNT)
    })
  })
```

#### Recommended Mitigation Steps
To prevent the front-running, the **_mintHook** should be set in the deployment in the PrePOMarketFactory.

You could add one more address to the createMarket that accepts the mintHook address for that deployment and just add the address after
```solidity
File: apps/smart-contracts/core/contracts/PrePOMarketFactory.sol
46:     PrePOMarket _newMarket = new PrePOMarket{salt: _salt}(_governance, _collateral, ILongShortToken(address(_longToken)), ILongShortToken(address(_shortToken)), _floorLongPrice, _ceilingLongPrice, _floorValuation, _ceilingValuation, _expiryTime);
47:     deployedMarkets[_salt] = address(_newMarket);
```
the PrePOMarket ist deployed in the Factory.

Alternative you could add a default MintHook-Contract address that will always revert until it's changed to a valid one,