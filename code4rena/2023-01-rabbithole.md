## Summary

### Findings

|              | Issue                                                                                |
| ------------ | ------------------------------------------------------------------------------------ |
| [H-01](#H01) | Unclaimed user rewards are on risk after endTime is reached in Erc20Quest            |
| [M-01](#M01) | Unclaimed rewards in Erc1155Quest will be lost for users                             |
| [M-02](#M02) | Possible cross-chain replay-attacks                                                  |
| [M-03](#M03) | Reentrance in Quest.claim() function                                                 |
| [L-01](#L01) | Follow the suggested Proxy pattern for TransparentUpgradeableProxy in QuestFactory   |
| [L-02](#L02) | Emit event after updating a important state variable by the admin                    |
| [L-03](#L03) | Check fee adjustments before updating state                                          |
| [L-04](#L04) | Check if token exists before calculating the royalty                                 |
| [L-05](#L05) | Rename missleading modifier name                                                     |
| [L-06](#L06) | Move questFactoryContract from Erc20Quest to base Quest                              |
| [L-07](#L07) | onlyMinter in RabbitHoleReceipt and RabbitHoleTickets should check for factory       |
| [L-08](#L08) | Creation of Erc1155Quest's is missing a check and needs additional privileged access |

---

## High

### <a id="H01"></a> H-01 Unclaimed user rewards are on risk after endTime is reached in Erc20Quest
Currently an Erc20Quest has the ability to withdraw fees via `Erc20Quest.withdrawFee`. This function is protected by the modifier `onlyAdminWithdrawAfterEnd` but the modifier doesn't check if an Admin is calling this function and is only checking if endTime is reached. So the function can be called by everyone as soon as the endTime is reached.

```solidity
File: contracts/Erc20Quest.sol
102:     function withdrawFee() public onlyAdminWithdrawAfterEnd {
103:         IERC20(rewardToken).safeTransfer(protocolFeeRecipient, protocolFee());
104:     }

File: contracts/Quest.sol
76:     modifier onlyAdminWithdrawAfterEnd() {
77:         if (block.timestamp < endTime) revert NoWithdrawDuringClaim();
78:         _;
79:     }
```

Also the function does not track, if it the fees were already collected and so it can be called multiple times and drain the funds in the Quest contract leading resulting in a contract state where users can't claim their rewards as the contract doesn't hold the needed amount anymore.

As the function can be called by everyone any attacker can grief the protocol and bring every contract in a bad state for all the users that didn't claimed their rewards as soon as the endTime is reached.

#### Proof of Concept
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc20Quest.sol#L102
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L76

The test shows, that an users is not able to claim their rewards, after the withdrawFee() function was called multiple times.

File: test/Erc20Quest.spec.ts add second time call `await deployedQuestContract.withdrawFee()` and the test will fail as it collected to much fees.
```solidity
describe('withdrawFee()', async () => {
    it('should transfer protocol fees back to owner', async () => {
      const beginningContractBalance = await deployedSampleErc20Contract.balanceOf(deployedQuestContract.address)

      await deployedFactoryContract.connect(firstAddress).mintReceipt(questId, messageHash, signature)
      await deployedQuestContract.start()
      await ethers.provider.send('evm_increaseTime', [86400])
      expect(await deployedSampleErc20Contract.balanceOf(protocolFeeAddress)).to.equal(0)

      await deployedQuestContract.connect(firstAddress).claim()
      expect(await deployedSampleErc20Contract.balanceOf(firstAddress.address)).to.equal(1 * 1000)
      expect(beginningContractBalance).to.equal(totalRewardsPlusFee * 100)

      await ethers.provider.send('evm_increaseTime', [100001])
      await deployedQuestContract.withdrawFee()

      await deployedQuestContract.withdrawFee() // @audit this should not transfer a second time and change the expected behaviour

      expect(await deployedSampleErc20Contract.balanceOf(deployedQuestContract.address)).to.equal(
        totalRewardsPlusFee * 100 - 1 * 1000 - 200
      )
      expect(await deployedQuestContract.receiptRedeemers()).to.equal(1)
      expect(await deployedQuestContract.protocolFee()).to.equal(200) // 1 * 1000 * (2000 / 10000) = 200
      expect(await deployedSampleErc20Contract.balanceOf(protocolFeeAddress)).to.equal(200)

      await ethers.provider.send('evm_increaseTime', [-100001])
      await ethers.provider.send('evm_increaseTime', [-86400])
    })
})
```

#### Tools Used
manual review

#### Recommended Mitigation Steps
Add a variable to track if the fee's were collected and revert if it is called a second time.

```soldity
bool private feesClaimed;
function withdrawFee() public onlyAdminWithdrawAfterEnd {
    if(feesClaimed) return;

    feesClaimed = true;
    IERC20(rewardToken).safeTransfer(protocolFeeRecipient, protocolFee());
}
```

Also you should consider adding a admin check in the modifier onlyAdminWithdrawAfterEnd or rename it to onlyWithdrawAfterEnd so it is clear what it is doing.

---

## Medium

### <a id="M01"></a> M-01 Unclaimed rewards in Erc1155Quest will be lost for users
The `Erc1155Quest.withdrawRemainingTokens(address to_)` is for an admin to withdraw any additional tokens that are too much in the contract.

The current implementation just transfers all the holdings of the contract. It is not reducing the amount of all the users that are allowed to claim rewards and so the users can't claim their rewards after the Admin called the withdrawRemainingTokens function.

```solidity
File: contracts/Erc1155Quest.sol
54:     function withdrawRemainingTokens(address to_) public override onlyOwner {
55:         super.withdrawRemainingTokens(to_);
56:         IERC1155(rewardToken).safeTransferFrom(
57:             address(this),
58:             to_,
59:             rewardAmountInWeiOrTokenId,
60:             IERC1155(rewardToken).balanceOf(address(this), rewardAmountInWeiOrTokenId),
61:             '0x00'
62:         );
63:     }
```

Even if we trust and assume the current Rabbit team will return or move funds to the users that didn't claimed, with a growing allowed list of addresses that have the `CREATE_QUEST_ROLE` role to start new quests it get's more and more possible that there will be a malicious actor that will not transfer the funds to the users.

#### Proof of Concept
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc1155Quest.sol#L60

The test shows, that an users is not able to claim their rewards, after the admin called withdrawRemainingTokens.

Add: File: test/Erc1155Quest.spec.ts
```node
describe('claim() for user should still be possible after withdrawRemainingTokens() was falled from admin', async () => {    
    it('it should be possible to claim the correct amount after the admin has called withdrawRemainingTokens()', async () => {
        await deployedRabbitholeReceiptContract.mint(owner.address, questId)
        await deployedQuestContract.start()

        await ethers.provider.send('evm_increaseTime', [86400])

        expect(await deployedSampleErc1155Contract.balanceOf(owner.address, rewardAmount)).to.equal(0)

        const totalTokens = await deployedRabbitholeReceiptContract.getOwnedTokenIdsOfQuest(questId, owner.address)
        expect(totalTokens.length).to.equal(1)

        expect(await deployedQuestContract.isClaimed(1)).to.equal(false)
        
        // owner calles withdrawRemainingTokens before user calls claim
        await deployedQuestContract.connect(owner).withdrawRemainingTokens(firstAddress.address)

        await deployedQuestContract.claim() // @audit this now failes with ERC1155: insufficient balance for transfer but should transfer it to the user
        expect(await deployedSampleErc1155Contract.balanceOf(owner.address, rewardAmount)).to.equal(1)
        await ethers.provider.send('evm_increaseTime', [-86400])
    })
})
```

#### Tools Used
manual review

#### Recommended Mitigation Steps
Reduce from the total allowed claimers the amount that are already claimed, and reduce this amount from the current balance.  
Like in the Erc20Quest.sol implementation.

### <a id="M02"></a> M-02 Possible cross-chain replay-attacks
The current implementation for validating receipts before they can mint their token is vulnarable to a crosschain replay-attack.  

If RabbitHole wants to give users the ability to do quests on different Chains and the signer for the tickets that is signing the message off-chain will stay the same, all questIDs can be replayed by the users.

#### Proof of Concept
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L222

Let's assume RabbitHole is active on Ethereum with a quest `questId = Game #xyz` where user 1 played and got a ticket and minted it on ethereum via `QuestFactory.mintReceipt(string memory questId_, bytes32 hash_, bytes memory signature_)`.

A bit later, RabbitHole decides to also start on Polygon and create the quest `questId = Game #xyz` on polygon as it was one of the best quests and they want to give players on polygon also the chance to play it.

As the logic for signing and checking is implemented already for ethereum they just use the same address for `claimSignerAddress` to not have to handle two different signers.

User 1 from ethereum now can also mint the Receipt on polygon without getting a new valid hash.

```solidity
// this check will pass, as the msg.sender and questId_ are the same as they were on ethereum
if (keccak256(abi.encodePacked(msg.sender, questId_)) != hash_) revert InvalidHash();

// this will also not revert, as the message that is checked is only checking for
// bytes32 messageDigest = keccak256(abi.encodePacked('\x19Ethereum Signed Message:\n32', hash_));
if (recoverSigner(hash_, signature_) != claimSignerAddress) revert AddressNotSigned();
```

#### Tools Used
manual review

#### Recommended Mitigation Steps
Consider adding a DOMAIN_SEPARATOR for the check of the message that is signed. For this you could extend the _hash that is checked to also contain the chainId.

```solidity
// add block.chainid to check the hash, also consider to not use encodePacked and stay with encode
if (keccak256(abi.encodePacked(msg.sender, questId_, block.chainid)) != hash_) revert InvalidHash();
```

### <a id="M03"></a> M-03 Reentrance in Quest.claim() function
Currently in `Quest.claim()` there is an external call before the state variable `redeemedTokens` is updated and can lead to an re-entrance attack.

With the current implementation of Erc20Quest and Erc1155Quest it's not to critical, as both implementations don't depend on the current value of `redeemedTokens` and just transfer the tokens.
Still, if any external contract or instance is depending of the value of `uint256 public redeemedTokens;` it can lead to a read re-entrance attack.

#### Proof of Concept
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L96
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L114-L115

State variable is updated after the external call in `_transferRewards(totalRedeemableRewards)`.

```solidity
File: contracts/Quest.sol
113:         _setClaimed(tokens);
114:         _transferRewards(totalRedeemableRewards);
115:         redeemedTokens += redeemableTokenCount;
```

#### Tools Used
manual review

#### Recommended Mitigation Steps
Change the order and first update `redeemedTokens += redeemableTokenCount;` and then make the transfer.

```solidity
        _setClaimed(tokens);        
        redeemedTokens += redeemableTokenCount;
        _transferRewards(totalRedeemableRewards);
```

---

## Low

### <a id="L01"></a> L-01 Follow the suggested Proxy pattern for TransparentUpgradeableProxy in QuestFactory
[Openzepplin](https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable-_disableInitializers-- ) recommends using `_disableInitializers();` since 4.6.0 instead of the modifier `initilizer`. 

Currently you have a constructor with the `initilizer` modifier in QuestFactory. You should change it to use `_disableInitializers();` inside the constructor to follow the suggested implementation from Openzepplin.

```solidity
File: contracts/QuestFactory.sol
35:     constructor() initializer {}
```

### <a id="L02"></a> L-02 Emit event after updating a important state variable by the admin
After updating an important state variable from the admin you should emit an event with the update, so it's easier to get notified off-chain if for example an important destination contract, the fees are changed after an admin update.

```solidity
File: contracts/QuestFactory.sol
159:     function setClaimSignerAddress(address claimSignerAddress_) public onlyOwner {
160:         claimSignerAddress = claimSignerAddress_;
161:     }

File: contracts/QuestFactory.sol
165:     function setProtocolFeeRecipient(address protocolFeeRecipient_) public onlyOwner {
166:         if (protocolFeeRecipient_ == address(0)) revert AddressZeroNotAllowed();
167:         protocolFeeRecipient = protocolFeeRecipient_;
168:     }

File: contracts/QuestFactory.sol
172:     function setRabbitHoleReceiptContract(address rabbitholeReceiptContract_) public onlyOwner {
173:         rabbitholeReceiptContract = RabbitHoleReceipt(rabbitholeReceiptContract_);
174:     }

File: contracts/QuestFactory.sol
179:     function setRewardAllowlistAddress(address rewardAddress_, bool allowed_) public onlyOwner {
180:         rewardAllowlist[rewardAddress_] = allowed_;
181:     }

File: contracts/QuestFactory.sol
186:     function setQuestFee(uint256 questFee_) public onlyOwner {
187:         if (questFee_ > 10_000) revert QuestFeeTooHigh();
188:         questFee = questFee_;
189:     }

File: contracts/RabbitHoleReceipt.sol
65:     function setReceiptRenderer(address receiptRenderer_) public onlyOwner {
66:         ReceiptRendererContract = ReceiptRenderer(receiptRenderer_);
67:     }

File: contracts/RabbitHoleReceipt.sol
71:     function setRoyaltyRecipient(address royaltyRecipient_) public onlyOwner {
72:         royaltyRecipient = royaltyRecipient_; 
73:     }

File: contracts/RabbitHoleReceipt.sol
77:     function setQuestFactory(address questFactory_) public onlyOwner {
78:         QuestFactoryContract = IQuestFactory(questFactory_);
79:     }

File: contracts/RabbitHoleTickets.sol
54:     function setTicketRenderer(address ticketRenderer_) public onlyOwner {
55:         TicketRendererContract = TicketRenderer(ticketRenderer_);
56:     }

File: contracts/RabbitHoleTickets.sol
60:     function setRoyaltyRecipient(address royaltyRecipient_) public onlyOwner {
61:         royaltyRecipient = royaltyRecipient_;
62:     }
```

### <a id="L03"></a> L-03 Check fee adjustments before updating state
Before updating fees you should check at least that the new value is not greater than X. Contract can get in a bad state if the fee is too high.

```solidity
File: contracts/RabbitHoleTickets.sol
66:     function setRoyaltyFee(uint256 royaltyFee_) public onlyOwner {
67:         royaltyFee = royaltyFee_;
68:         emit RoyaltyFeeSet(royaltyFee_);
69:     }

File: contracts/RabbitHoleReceipt.sol
90:     function setRoyaltyFee(uint256 royaltyFee_) public onlyOwner {
91:         royaltyFee = royaltyFee_;
92:         emit RoyaltyFeeSet(royaltyFee_);
93:     }
```

### <a id="L04"></a> L-04 Check if token exists before calculating the royalty
At the moment the `RabbitHoleTickets.royaltyInfo` function is not checking if the token exists. You should consider adding a check if the token exists before you calculate the amount, or it can lead to unexpected behaviour for calling contracts.

```solidity
File: contracts/RabbitHoleTickets.sol
109:     function royaltyInfo(
110:         uint256 tokenId_,
111:         uint256 salePrice_
112:     ) external view override returns (address receiver, uint256 royaltyAmount) {
113:         uint256 royaltyPayment = (salePrice_ * royaltyFee) / 10_000;
114:         return (royaltyRecipient, royaltyPayment);
115:     }
```

### <a id="L05"></a> L-05 Rename missleading modifier name
The modifier `onlyAdminWithdrawAfterEnd` in Quest is missleading. It suggests that there is a Admin check, but it only checks for the endTime.  
Rename the modifier to `onlyWithdrawAfterEnd` or add the admin check.

```solidity
File: contracts/Quest.sol
76:     modifier onlyAdminWithdrawAfterEnd() {
77:         if (block.timestamp < endTime) revert NoWithdrawDuringClaim();
78:         _;
79:     }
```

### <a id="L06"></a> L-06 Move questFactoryContract from Erc20Quest to base Quest
At the moment the factory address is only stored in an Erc20Quest but it should be also accessible in the Erc1155Quest as a genral speaking, a Quest is most likely always connected to the deploying factory contract. It's very likely that the factory will also be needed in the Erc1155Quest later on, so it should be moved in the base contract Quest.

```solidity
File: contracts/Erc20Quest.sol
40:         questFactoryContract = QuestFactory(msg.sender);
```

### <a id="L07"></a> L-07 onlyMinter in RabbitHoleReceipt and RabbitHoleTickets should check for factory
Currently RabbitHoleReceipt and RabbitHoleTickets check in the modifier that the caller is the minterAddress. This means, that the minter can be different to the QuestFactory and lead to unexpected behaviour.

You should consider checking for `msg.sender == address(QuestFactoryContract)` as it's always the factory that should be allowed to mint. With this you take out complexity and remove a potential breach or miss-configuration if address(QuestFactoryContract) != minterAddress.

```solidity
File: contracts/RabbitHoleReceipt.sol
58:     modifier onlyMinter() {
59:         msg.sender == minterAddress;
60:         _;
61:     }

File: contracts/RabbitHoleTickets.sol
47:     modifier onlyMinter() {
48:         msg.sender == minterAddress;
49:         _;
50:     }
```

### <a id="L08"></a> L-08 Creation of Erc1155Quest's is missing a check and needs additional privileged access
Currently in a creation for a new Erc1155Quest it's not checked if the reward-token is approved and also it needs the caller to be the owner.

As it requires the caller to be also the owner, it assume that the owner will always have the CREATE_QUEST_ROLE. You should consider if it's not better to seperate the owner of the contract from the priviliged user that can create a Erc1155Quest. You could split up the create quest role in two roles, one for ERC20 and one for ERC1155.

```solidity
File: contracts/QuestFactory.sol
// restricts for only CREATE_QUEST_ROLE
69:     ) public onlyRole(CREATE_QUEST_ROLE) returns (address) { 
105:         if (keccak256(abi.encodePacked(contractType_)) == keccak256(abi.encodePacked('erc1155'))) {
// missing the check if (rewardAllowlist[rewardTokenAddress_] == false) revert RewardNotAllowed();
// requires owner to have CREATE_QUEST_ROLE
106:             if (msg.sender != owner()) revert OnlyOwnerCanCreate1155Quest(); 
```