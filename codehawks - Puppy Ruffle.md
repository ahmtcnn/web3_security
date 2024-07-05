



# Solidity CTF Notes: Puppy Ruffle Contract

## 1. Find the security misconfigurations 

- CTF URL: [CodeHawks Contest](https://www.codehawks.com/contests/clnuo221v0001l50aomgo4nyn)
- GitHub Repository: [Cyfrin/2023-10-PuppyRuffle](https://github.com/Cyfrin/2023-10-Puppy-Raffle)

## Puppy Ruffle

This project is to enter a raffle to win a cute dog NFT. The protocol should do the following:

- Call the enterRaffle function with the following parameters:
- address[] participants: A list of addresses that enter. You can use this to enter yourself multiple times, or yourself and a group of your friends.
- Duplicate addresses are not allowed
- Users are allowed to get a refund of their ticket & value if they call the refund function
- Every X seconds, the raffle will be able to draw a winner and be minted a random puppy
- The owner of the protocol will set a feeAddress to take a cut of the value, and the rest of the funds will be sent to the winner of the puppy.

## Issues

## 1. Sending funds to address(0)

- Since the refund function set the index of player to address(0), after a user refund, the winner could be that user and this way the winner gets the price which is address(0) which is a financial loss.
```solidity

    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        payable(msg.sender).sendValue(entranceFee);

        players[playerIndex] = address(0); // here sets the address(0) of playerIndex
        emit RaffleRefunded(playerAddress);
    }

```

```solidity

        delete players;
        raffleStartTime = block.timestamp;
        previousWinner = winner;
        (bool success,) = winner.call{value: prizePool}(""); //here sends money directly to the address without checking if the user is refunded and gone.
        require(success, "PuppyRaffle: Failed to send prize pool to winner");
        _safeMint(winner, tokenId);
```
## 2. Race condition - Reentrancy Attack

- When a user call the refund function, the contract sends the enterance fee to user. After that line, it updates the player index to zero so that the user cannot refund again. Before it updates the addres of player to zero, the player can request the same function and refund again. Like a race condition issue on web2.

```solidity
function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        payable(msg.sender).sendValue(entranceFee);

        players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }
```

### Recommendations

Consider moving the line "players[playerIndex] = address(0);" BEFORE executing "payable(msg.sender).sendValue(entranceFee);", or implement a lock such as "nonReentrant" from OpenZeppelin's ReentrancyGuard."



## 2. Insecure Randomness

The winnerIndex variable is not secure random number since the usec parameters can be known or open to manipulation.


- msg.sender: This is predictable as anyone can see the address initiating the transaction.
- block.timestamp: Miners have some flexibility in setting the timestamp of a block they mine, though it should be greater than the timestamp of the previous  block and less than 15 minutes into the future. This makes it somewhat manipulable.
-block.difficulty: Since the transition to Proof of Stake (PoS) with the Ethereum merge, block.difficulty (renamed to block.difficultyBase) has become more predictable and less random, as it does not vary as widely as under Proof of Work (PoW).



```solidity
function selectWinner() external {
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
        require(players.length >= 4, "PuppyRaffle: Need at least 4 players");
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
        (bool success,) = winner.call{value: prizePool}("");
        require(success, "PuppyRaffle: Failed to send prize pool to winner");
        _safeMint(winner, tokenId);
    }
```


## 3. Loop usage - Gas exceed


- Nested Loop Problem: The function uses a nested loop to check for duplicates within the players array. This process has a time complexity of O(n^2), where n is the number of players. As more players join, the number of comparisons increases quadratically, consuming more gas.

- As the number of participants increases, the gas required for the enterRaffle function also increases significantly. If the number of participants becomes large enough, the gas required to execute the function could exceed the block gas limit, causing the transaction to fail consistently.


```solidity
function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
        }

        // Check for duplicates
        for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
        emit RaffleEnter(newPlayers);
    }
```

### Recommendation

Use of mapping or EnumerableMap: Instead of an array, a mapping could be used to store player entries. Mappings in Solidity provide a more gas-efficient way of ensuring no duplicates, as each address can be mapped to a boolean (or another value) indicating whether it is already in the raffle.