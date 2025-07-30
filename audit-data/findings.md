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

This issue deems Critical severity because a reentrancy attack can end up causing serious damage by draining funds from the contract. Particularly for a contract handling withdrawal functions, this vulnerability exposes the contract to a substantial risk of loss.

**Recommended Mitigation**:

Add [ReentrancyGuardTransient](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/ReentrancyGuardTransient.sol) contract to the `PuppyRaffle` contract and add the `nonReentrant` modifier to the  `PuppyRaffle::refund` function as a precautionary measure. This will prevent recursive calls and therefore protect against potential reentrancy attacks.

The OpenZeppelin version would need to be bumped to 5.1.

### [H-2] Reentrancy Vulnerability in `PuppyRaffle::selectWinner` Function.

**Description**: A significant security issue has been located in the `PuppyRaffle::selectWinner` function.

The vulnerability is embedded within the potential for a reentrancy attack. In this scenario, a hacker could repetitively call the `PuppyRaffle::selectWinner` function until the contract is fully drained of its funds.

This issue deems Critical severity because a reentrancy attack can end up causing serious damage by draining funds from the contract. Particularly for a contract handling withdrawal functions, this vulnerability exposes the contract to a substantial risk of loss.

**Recommended Mitigation**:

Add [ReentrancyGuardTransient](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/ReentrancyGuardTransient.sol) contract to the `PuppyRaffle` contract and add the `nonReentrant` modifier to the  `PuppyRaffle::selectWinner` function as a precautionary measure. This will prevent recursive calls and therefore protect against potential reentrancy attacks.

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

**Impact**: If the contract balance is higher than the total fees, the `PuppyRaffle::withdrawFees` function will always revert, preventing the withdrawal of fees.

**Recommended Mitigation**: Replace the strict equality with a more robust comparison, such as:

```javascript
    function withdrawFees() external {
@>      require(address(this).balance >= uint256(totalFees),"PuppyRaffle: There are currently players active!");
    // ... code logic after
    }
```

Add [ReentrancyGuardTransient](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/ReentrancyGuardTransient.sol) contract to the `PuppyRaffle` contract and add the `nonReentrant` modifier to the  `PuppyRaffle::selectWinner` function as a precautionary measure. This will prevent recursive calls and therefore protect against potential reentrancy attacks.

The OpenZeppelin version would need to be bumped to 5.1.

### [M-1] Missing zero address validation leads to lost of funds.

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

**Recommended Mitigation**: Check that the inputed `_feeAddress` value is different than zero address by using a conditional statement.

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



### [I-1] Replace `require` Statements with Custom Errors, only if solc version is 0.8.4 or higher.

**Description**: As stated in the official release of (Solidity 0.8.4)[https://soliditylang.org/blog/2021/04/21/custom-errors/], utilizing custom errors can reduce runtime and deployment costs, as indicated by the following benchmark, while also improving clarity in error handling.

**Recommended Mitigation**: Consider update to solidity version 0.8.4 or higher and replacing all require statements with custom errors.

### [I-2] `PuppyRaffle::_isActivePlayer` function is not being used in the contract.

**Description**: The `PuppyRaffle::_isActivePlayer` function is not being used in the contract.

**Recommended Mitigation**: Remove this function as it is not being used anywhere in the contract.

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