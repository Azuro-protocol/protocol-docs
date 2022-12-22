# “V1 > V2 Migration”




> After the migration to the V2,
there were **changes in contracts** + now the client must use **TheGraph**.
The documentation will help you understand the changes.
Below you will find:
 - Description of important changes in contracts
 - **GraphQL** query examples
 - Table with Changes in contracts

_________________
## V2 :


<details><summary>Description of important changes in contracts</summary>
<p>
 
----------------
 
If in v1 we operated with events/conditions, then in v2, the "Game" entity appeared.
In v2, first of all we create a game, now the game contains information about time, league, country, team names, sport ID
  and now,
when the oracle receives information about the postponement of the game.
We no longer need to move many conditions inside separately, only needs to move one game.
conditions are now on the contract along with the games
 
-----------------
 
In v2, we can create multiple sets of contracts, without deploying a new set to different locations each time.
Now we can create new contracts from the Factory contract and connect them.
An LP is created at the factory, core and azuro bet connect to it

What for?

- We can set multiple LPs and one liquidity pool can have multiple core contracts.

Previously, bets could only be placed before the match. In v2, we plan to launch express bets immediately, life after the start of the game

Betting method changes
You still need to bet on the LP contract, if in v1 we pulled only the bet method "how much money do we need", "what outcome do we bet on",
minCoefficient of the bet

All these parameters remain, but a new one appears. Now you need to pass the address of the core contract to the bet v2. To understand on what contract it is delivered.
Now on the air conditioner the fields for obtaining a unique key are the address of the contract + the condition field.
 
-------------
Third difference

Before
Conditions were created and they could have several oracles.
When we created them, we said "This oracle can create a Condition with some of its internal ID in order to know which one to resolve later"
But if on the contract it was the very first Condition created, then it received the number 1. We always had two fields: ConditionId - 1,
  OracleConditionId - 10000, We stored a map on the contract and in order to get one from the other we made a map by the key oracle address + OracleConditionId,
  this is such and such an ordinary ConditionId
 
  This was so that several oracles could work with the same core and create different conditions.
  For example, so that they are responsible for different data providers and do not conflict on the same core.
 
  Now it is possible to deploy several cores within one LP. Or several LPs. we decided to abandon the multiple oracles. There may still be a few oracles, but
  It will be assumed that they always work with the same data set.
  That's why we left the map.
 
  Now:
  1. we create a game (with some kind of own id, which is in the data provider's database)
  2. In it, already on the core contract, we create a Condition with some kind of our own internal ID.
 
  Those. incremental IDs disappear, OracleGameID and OracleConditionID fields are sold, they are all called
GameID and ConditionID and they are all arbitrary, come from the provider's date


Method signature changes (see below)



</p>
</details>


________________________

<details><summary>Difference between V1 and V2 with code examples</summary>
<p>

#### V1:

--------------

- condition is created on Core **(oracleConditionId)

- bet is placed on LP **(by conditionId)

- conditioner shifts, cancelizes, resolves to Core **(oracleConditionId)

- the bet is redimed on LP **(by betId)


#### V2

--------------
##### **The game is created on LP **(oracleGameId)**

<details><summary>function: </summary>
<p>

 ```js
function **createGame**(
bytes32 ipfsHash, //detailed info about game stored in IPFS
uint64 startsAt //timestamp when the game starts
) external onlyOracle
```
</p>
</details>

<details><summary>event:</summary>
<p>

```js
event NewGame(uint256 indexed gameId, bytes32 ipfsHash, uint64 startsAt)
```
</p>
</details>


<details><summary>function: </summary>
<p>

 ```js
function **createCondition**(
    uint256 gameId,
    uint256 oracleConditionId,
    uint64[2] calldata odds,
    uint64[2] calldata outcomes,
    uint128 reinforcement,
    uint64 margin
) external override

```
>This function **creates new conditions and provide contract with initial odds, allowed outcomes and  condition date. 
</p>
</details>

<details><summary>event: </summary>
<p>

```js
event **ConditionCreated**(
    uint256 indexed gameId,
    uint256 indexed oracleConditionId,
    uint256 indexed conditionId
);
```
</p>
</details>

--------------
##### The game is shifted on LP **(by oracleGameId)**

<details><summary>function: </summary>
<p>

 ```js
function **shiftGame**(uint256 gameId, uint64 startsAt) external onlyOracle
```
>admin-oracle function **for changing game’s startsAt
</p>
</details>

<details><summary>event: </summary>
<p>

```js
event **GameShifted**(uint256 indexed gameId, uint64 newStart);
```
</p>
</details>

--------------
##### Bet is placed on LP(core address + conditionId), proxy call - event **is emitted on Core**


<details><summary>function **bet**:</summary>
<p>

 ```js
function **bet**(
    address core,
    uint128 amount,
    uint64 expiresAt,
    ICoreBase.BetData calldata data
) external
```
> function **to put bet, providing BetData

</p>
</details>


<details><summary>function **betFor**:</summary>
<p>

 ```js
function **betFor**(
    address bettor,
    address core,
    uint128 amount,
    uint64 expiresAt,
    ICoreBase.BetData calldata data
) external
```
> function to put bet for bettor

</p>
</details>

<details><summary>function **betNative**:</summary>
<p>

 ```js
function **betNative**(
    address core,
    uint64 expiresAt,
    ICoreBase.BetData calldata data
) external payable
```
>function **to put bet in native tokens

</p>
</details>

<details><summary>**(proxy call - event **is emitted on Core):</summary>
<p>

```js
function **putBet**(function **putBet**(
    address bettor,
    uint128 amount,
    BetData calldata data
) external override onlyLp
```
</p>
</details>

--------------
##### Condition iscanceled and resolved on Core **(oracleConditionId)


<details><summary>function **resolveCondition**:</summary>
<p>

 ```js
function **resolveCondition**(uint256 oracleConditionId, uint64 outcomeWin)
```
</p>
</details>

<details><summary>event:</summary>
<p>

```js
event **ConditionResolved**(
    uint256 indexed conditionId,
    uint8 state,
    uint64 outcomeWin,
    int128 lpProfit
);
```
</p>
</details>



<details><summary>function **cancelByMaintainer**:</summary>
<p>

 ```js
function **cancelByMaintainer**(uint256 conditionId) external onlyMaintainer
```
> admin-maintainer function **for canceling exact conditionID
> 
</p>
</details>

<details><summary>**stopCondition**:</summary>
<p>

 ```js
function **stopCondition**(uint256 conditionId, bool flag) external onlyMaintainer
```
> admin-maintainer function **for stop protocol receiving bets for exact conditionId, flag = true - stop bets for conditionId

</p>
</details>

<details><summary>function **cancelByOracle**:</summary>
<p>

 ```js
function **cancelByOracle**(uint256 oracleConditionId) external onlyOracle
```
> oracle function **for canceling exact oracleConditionId

</p>
</details>

--------------

##### The bet is redeamed on LP indicating Core **(core address + bet id)


<details><summary>function: </summary>
<p>

 ```js
LP.withdrawPayout(
        address core,
        uint256 tokenId,
        bool isNative
    )
```

```js
function withdrawPayout(address core, uint256 tokenId) external isCore(core)
```
>Function to withdraw bet's prize

```js
function withdrawPayoutNative(address core, uint256 tokenId) external isCore(core)
```
>Function to withdraw bet's prize in native tokens


</p>
</details>

</p>
</details>

------------------

<details><summary>V2 Events: </summary>
<p>

> Description of events issued by protocol contracts

### Fabric
#### Common events

```js
event NewPool(address lp);
```
> new pool added

#### Protocol settings changes events

```js
event CoreTypeUpdated(string coreType, address beaconCore);
```

### LP
#### Common events

```js
event BettorWin(address indexed bettor, uint256 tokenId, uint256 amount);
```

> BettorWin issued by withdrawPayout(), withdrawPayoutNative()

```js
event LiquidityAdded(
    address indexed account,
    uint48 indexed leaf,
    uint256 amount
);
```
> LiquidityAdded issued by addLiquidity(), addLiquidityNative()

```js
event LiquidityRemoved(
    address indexed account,
    uint48 indexed leaf,
    uint256 amount
);
```
> LiquidityRemoved issued by withdrawLiquidity(), withdrawLiquidityNative()

#### Protocol settings changes events
```js
event CoreUpdated(address indexed core, bool active);
event MaintainerUpdated(address indexed maintainer, bool active);
event OracleUpdated(address indexed oracle, bool active);
event AffiliateRewardChanged(uint64 newAffiliateFee);
event AffiliateRewarded(address indexed affiliate, uint256 amount);
event DaoRewardChanged(uint64 newDaoFee);
event MinDepoChanged(uint128 newMinDepo);
event OracleRewardChanged(uint64 newOracleFee);
event ReinforcementAbilityChanged(uint128 newReinforcementAbility);
event WithdrawTimeoutChanged(uint64 newWithdrawTimeout);
```


### Core
#### Bettor actions events

```js
event NewBet(
    address indexed bettor,
    address indexed affiliate,
    uint256 indexed conditionId,
    uint256 tokenId,
    uint64 outcomeId,
    uint128 amount,
    uint64 odds,
    uint128[2] funds
);
```
> NewBet issued by LP.bet(), LP.betNative(), LP.betFor()

#### Oracle actions events

```js
event ConditionCreated(
    uint256 indexed gameId,
    uint256 indexed oracleConditionId,
    uint256 indexed conditionId
);
```

> ConditionCreated issued by createCondition()

```js
event ConditionResolved(
    uint256 indexed conditionId,
    uint8 state,
    uint64 outcomeWin,
    int128 lpProfit
);
```
> ConditionResolved issued by resolveCondition(), cancelByOracle(), cancelByMaintainer()

```js
event ConditionStopped(uint256 indexed conditionId, bool flag);
```
> ConditionStopped issued by stopCondition()
</p>
</details>



------------------

<details><summary>Changes in Dictionaries and Entity Ids:</summary>
<p>
> The dictionaries have been updated. More details can be found at the link. Brief information in the table below
https://github.com/Azuro-protocol/dictionaries/tree/main/v2

| v1 | v2 |usage|
| ------ | ------ | -------|
|betTypeOdd|outcome||
|outcome|selection||
|param|points||
|sportType|sport||


> The entities Ids have been also updated. More details can be found at the link. Brief information in the table below https://github.com/Azuro-protocol/azuro-api-subgraph/blob/928c4867d775c56f012293dabc64ae8dc57f27fa/src/utils/schema.ts

| v1        | v2                                               |
| --------- | ------------------------------------------------ |
| game      | LP Address + \_ + gameId                         |
|           |                                                  |
| condition | Core Address + \_ + conditionId                  |
|           |                                                  |
| outcome   | Core Address + \_ + conditionId + \_ + outcomeId |
|           |                                                  |
| bet       | Core Address + \_ + betId                        |
|           |                                                  |
| freebet   | Freebet Address + \_ + freebetId                 |
|           |                                                  |
| LP NFT    | LP Address + \_ + NFT Id                         |
</p>
</details>

------------------

##  TheGraph+v2:
- #### how to use TheGraph - examples

------------------


<details><summary>List of events</summary>
<p>
request:

```
query Sports(
  $sportFilter: Sport_filter, $countryFilter: Country_filter, $leagueFilter: League_filter, $gameFilter: Game_filter, $conditionFilter: Condition_filter!,
  $gameOrderBy: Game_orderBy, $gameOrderDirection: OrderDirection
) {
  sports(where: $sportFilter) {
    id
    sportId
    slug
    name
    sporthub {
      id
    }
    countries(where: $countryFilter, orderBy: turnover, orderDirection: desc) {
      id
      slug
      name
      turnover
      leagues(where: $leagueFilter, orderBy: turnover, orderDirection: desc) {
        id
        name
        slug
        turnover
        games(where: $gameFilter, orderBy: $gameOrderBy, orderDirection: $gameOrderDirection) {
          ...Game
          conditions(where: $conditionFilter) {
            ...GameCondition
          }
        }
      }
    }
  }
}
```

fragment Game :

```
fragment Game on Game {
  id
  gameId
  oracleGameId
  slug
  title
  status
  sport {
    sportId
    slug
    sporthub {
      slug
    }
  }
  league {
    name
    slug
    country {
      name
      slug
    }
  }
  participants {
    image
    name
  }
  startsAt
  hasActiveConditions
  liquidityPool {
    address
  }
}
```

fragment GameCondition:

```
fragment GameCondition on Condition {
  id
  conditionId
  status
  outcomes {
    id
    outcomeId
  }
  core {
    address
    type
  }
}
```

sample parameters for requesting sports for top events:

```json
{
    "sportFilter": {
        "sporthub": "sports",
        "slug_in": [
            "football",
            "basketball",
            "tennis",
            "mma",
            "boxing"
        ]
    },
    "countryFilter": {
        "hasActiveLeagues": true
    },
    "leagueFilter": {
        "games_": {
            "startsAt_gt": "1671183624",
            "liquidityPool": "0xbd3e8643efcdddd033478f485eefcc68ad779af2"
        }
    },
    "gameFilter": {
        "startsAt_gt": "1671183624",
        "hasActiveConditions": true
    },
    "conditionFilter": {
        "core_": {
            "liquidityPool": "0xbd3e8643efcdddd033478f485eefcc68ad779af2"
        }
    },
    "gameOrderBy": "turnover",
    "gameOrderDirection": "desc"
}
```

if you need to query for a specific sport, then the __sportFilter__ has a specific __slug__ and the order is no longer by __turnover__ liquidity, but by __startsAt__ start time

```json
{
    "sportFilter": {
        "sporthub": "sports",
        "slug_in": [
            "football",
            "basketball",
            "tennis",
            "mma",
            "boxing"
        ],
        "slug": "football"
    },
    "countryFilter": {
        "hasActiveLeagues": true
    },
    "leagueFilter": {
        "games_": {
            "startsAt_gt": "1671183878",
            "liquidityPool": "0xbd3e8643efcdddd033478f485eefcc68ad779af2"
        }
    },
    "gameFilter": {
        "startsAt_gt": "1671183878",
        "hasActiveConditions": true
    },
    "conditionFilter": {
        "core_": {
            "liquidityPool": "0xbd3e8643efcdddd033478f485eefcc68ad779af2"
        }
    },
    "gameOrderBy": "startsAt",
    "gameOrderDirection": "asc"
}
```

if you need a specific league, then __country Filter__ add the country __slug__ and the league name in __leagueFilter.slug__

```json
{
    "sportFilter": {
        "sporthub": "sports",
        "slug_in": [
            "football",
            "basketball",
            "tennis",
            "mma",
            "boxing"
        ],
        "slug": "football"
    },
    "countryFilter": {
        "hasActiveLeagues": true,
        "slug": "england"
    },
    "leagueFilter": {
        "games_": {
            "startsAt_gt": "1671184050",
            "liquidityPool": "0xbd3e8643efcdddd033478f485eefcc68ad779af2"
        },
        "slug": "championship"
    },
    "gameFilter": {
        "startsAt_gt": "1671184050",
        "hasActiveConditions": true
    },
    "conditionFilter": {
        "core_": {
            "liquidityPool": "0xbd3e8643efcdddd033478f485eefcc68ad779af2"
        }
    },
    "gameOrderBy": "startsAt",
    "gameOrderDirection": "asc"
}
```

</p>
</details>

------------------

<details><summary>Single event</summary>
<p>

Request:
```
query Game($oracleGameId: BigInt) {
  games(where: {oracleGameId: $oracleGameId}) {
    ...Game
    conditions {
      ...GameCondition
    }
  }
}
```
fragment Game :

```
fragment Game on Game {
  id
  gameId
  oracleGameId
  slug
  title
  status
  sport {
    sportId
    slug
    sporthub {
      slug
    }
  }
  league {
    name
    slug
    country {
      name
      slug
    }
  }
  participants {
    image
    name
  }
  startsAt
  hasActiveConditions
  liquidityPool {
    address
  }
}
```

fragment GameCondition:

```
fragment GameCondition on Condition {
  id
  conditionId
  status
  outcomes {
    id
    outcomeId
  }
  core {
    address
    type
  }
}
```

parameters:

```json
{
    "oracleGameId": "1563997432"
}
```


</p>
</details>

------------------

<details><summary>History of bets</summary>

is formed from two requests to the graph, getting sports bets and toto

## Sports bets
<p>

Request:
```
query Bets($first: Int, $where: Bet_filter) {
  bets(first: $first, orderBy: createdBlockTimestamp, orderDirection: desc, where: $where) {
    ...CommonBet
  }
}
```

fragment CommonBet:
```
fragment CommonBet on Bet {
  id
  betId
  status
  amount
  odds
  outcome {
    id
    outcomeId
    condition {
      ...CommonBetCondition
    }
  }
  createdAt: createdBlockTimestamp
  potentialPayout
  isRedeemed
  freebet {
    contractAddress: freebetContractAddress
  }
  txHash: createdTxHash
  core {
    address
    liquidityPool {
      address
    }
  }
}
```

fragment CommonBetCondition 	:
```
fragment CommonBetCondition on Condition {
  id
  conditionId
  wonOutcome {
    outcomeId
  }
  game {
    ...Game
  }
}
```

fragment Game:
```
fragment Game on Game {
  id
  gameId
  oracleGameId
  slug
  title
  status
  sport {
    sportId
    slug
    sporthub {
      slug
    }
  }
  league {
    name
    slug
    country {
      name
      slug
    }
  }
  participants {
    image
    name
  }
  startsAt
  hasActiveConditions
  liquidityPool {
    address
  }
}
```

parameters:

```json
{
    "first": 500,
    "where": {
        "actor": "0x78a9d33b78d22cc64f9bc1cf3352ac094e50c0a9"
    }
}
```



## Toto

Request:

```
query Bets($first: Int, $where: Bet_filter) {
  bets(first: $first, orderBy: createdAt, where: $where) {
    ...TotoBet
  }
}
```

fragment TotoBet:

```
fragment TotoBet on Bet {
  betId: tokenId
  outcome {
    id
    name
    outcomeId
  }
  amount
  createdAt
  owner
  txHash
  isRedeemed
  game: condition {
    ...TotoBetCondition
  }
}
```
fragment TotoBetCondition:
```
fragment TotoBetCondition on Condition {
  gameId: conditionId
  categoryName
  categorySlug
  icon
  field1
  field2
  condition
  startDate: gameStartsAt
  betsEndsAt: bettingEndsAt
  expiresAt
  opponent1 {
    image
    name
  }
  opponent2 {
    image
    name
  }
  totalPoolOutcome1
  totalPoolOutcome2
  totalPool
  winOutcomeId
  isCanceled
}
```

Parameters:
```json
{
    "first": 500,
    "where": {
        "actor": "0x78a9d33b78d22cc64f9bc1cf3352ac094e50c0a9"
    }
}
```

</p>

</details>

### Table with all changes in contacts v2
<details><summary>LP</summary>
<p>

### functions
| V1                                                                                                                                                            | V2                                                                                                                                                     |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| function **changeCore**(address newCore) external override onlyOwner                                                                                              |                                                                                                                                                        |
| function **changeOracleReward**(uint128 newOracleFee) external onlyOwner                                                                                          |                                                                                                                                                        |
| function **changeDaoReward**(uint128 newDaoFee) external onlyOwner                                                                                                |                                                                                                                                                        |
| function **changeAzuroBet**(address newAzuroBet) external onlyOwner                                                                                               |                                                                                                                                                        |
| function **changeMinDepo**(uint128 newMinDepo) external onlyOwner                                                                                                 | function **changeMinDepo**(uint128 newMinDepo) external onlyOwner                                                                                          |
| function **changeReinforcementAbility**(uint128 newReinforcementAbility) external onlyOwner                                                                       | function **changeReinforcementAbility**(uint64 newReinforcementAbility) external onlyOwner                                                                 |
| function **changeWithdrawTimeout**(uint64 newWithdrawTimeout) external onlyOwner                                                                                  | function **changeWithdrawTimeout**(uint64 newWithdrawTimeout) external onlyOwner                                                                           |
| function **changeClaimTimeout**(uint64 newClaimTimeout) external onlyOwner                                                                                        | function **changeClaimTimeout**(uint64 newClaimTimeout) external onlyOwner                                                                                 |
| function **addLiquidity**(uint128 amount) external                                                                                                                | function **addLiquidity**(uint128 amount) external                                                                                                         |
| function **addLiquidityNative**() external                                                                                                                        | function **addLiquidityNative**() external                                                                                                                 |
| function **withdrawLiquidity**(uint48 depNum, uint40 percent) external                                                                                            | function **withdrawLiquidity**( uint48 depNum, uint40 percent, bool isNative ) external                                                                    |
| function **withdrawLiquidityNative**(uint48 depNum, uint40 percent)                                                                                               |                                                                                                                                                        |
| function **viewPayout**(uint256 tokenId) external view override returns (bool, uint128)                                                                           | function **viewPayout**(address core, uint256 tokenId) external view isCore**(core) returns (uint128 payout)                                                 |
| function **withdrawPayout**(uint256 tokenId) external                                                                                                             | function **withdrawPayout**( address core, uint256 tokenId, bool isNative ) external override isCore**(core)                                                 |
| function **withdrawPayoutNative**(uint256 tokenId) external                                                                                                       |                                                                                                                                                        |
| function **claimDaoReward**() external                                                                                                                            |                                                                                                                                                        |
| function **betFor**( address bettor, uint256 conditionId, uint128 amount, uint64 outcomeId, uint64 deadline, uint64 minOdds ) external override returns (uint256) | function **betFor**( address bettor, address core, uint128 amount, uint64 expiresAt, ICoreBase.BetData calldata data ) external override returns (uint256) |
| function **bet**( uint256 conditionId, uint128 amount, uint64 outcomeId, uint64 deadline, uint64 minOdds ) external override returns (uint256)                    | function **bet**( address core, uint128 amount, uint64 expiresAt, ICoreBase.BetData calldata data ) external override returns (uint256)                    |
| function **bet**( address core, uint128 amount, uint64 expiresAt, ICoreBase.BetData calldata data ) external override returns (uint256)                           | function **bet**( address core, uint128 amount, uint64 expiresAt, ICoreBase.BetData calldata data ) external override returns (uint256)                    |
| function **getReserve**() external view override returns (uint128 reserve)                                                                                        | function **getReserve**() public view override returns (uint128 reserve)                                                                                   |
| function **getPossibilityOfReinforcement**(uint128 reinforcementAmount) external view override returns (bool status)                                              |
| function **getLeaf**() external view override returns (uint48 leaf)                                                                                               | function **getLeaf**() external view override returns (uint48 leaf)                                                                                        |
|                                                                                                                                                               | function **changeFee**(FeeType feeType, uint64 newFee) external onlyOwner                                                                                  |
|                                                                                                                                                               | function **changeFee**(FeeType feeType, uint64 newFee) external onlyOwner                                                                                  |
|                                                                                                                                                               | function **updateRole**( address actor, uint8 role, bool active ) external onlyOwner                                                                       |
|                                                                                                                                                               | function **cancelGame**(uint256 oracleGameId) external onlyRole**(0)                                                                                         |
|                                                                                                                                                               | function **createGame**( uint256 oracleGameId, bytes32 ipfsHash, uint64 startsAt ) external onlyRole**(0)                                                    |
|                                                                                                                                                               | function **shiftGame**(uint256 oracleGameId, uint64 startsAt) external onlyRole**(0)                                                                         |
|                                                                                                                                                               | function **claimReward**() external                                                                                                                        |
|                                                                                                                                                               | function **getGameInfo**(uint256 gameId) external view override returns (uint64, bool)                                                                     |
|                                                                                                                                                               | function **isGameCanceled**(uint256 gameId) external view override returns (bool)                                                                          |
|                                                                                                                                                               | function **updateCore**(address core, bool active) external onlyOwner isCore**(core)                                                                         |
|                                                                                                                                                               | function **updateCore**(address core, bool active) external onlyOwner isCore**(core)                                                                         |
### events

| V1                                                                                                                                                                      | V2                                                                                                       |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| event **NewBet**( address indexed owner, uint256 indexed betId, uint256 indexed conditionId, uint64 outcomeId, uint128 amount, uint256 odds, uint128 fund1, uint128 fund2); |
| event **BetterWin**(address indexed better, uint256 tokenId, uint256 amount);                                                                                               | event **BettorWin**( address indexed core, address indexed bettor, uint256 tokenId, uint256 amount);         |
| event **LiquidityAdded**(address indexed account, uint256 amount, uint48 leaf);                                                                                             | event **LiquidityAdded**( address indexed account, uint48 indexed leaf, uint256 amount);                     |
| event **LiquidityAdded**( address indexed account, uint48 indexed leaf, uint256 amount);                                                                                    | event **LiquidityRemoved**( address indexed account, uint48 indexed leaf, uint256 amount);                   |
| event **LiquidityRequested**( address indexed requestWallet, uint256 requestedValueLp);                                                                                     |
| event **OracleRewardChanged**(uint128 newOracleFee);                                                                                                                        |                                                                                                          |
| event **DaoRewardChanged**(uint128 newDaoFee);                                                                                                                              |                                                                                                          |
| event **AzuroBetChanged**(address newAzuroBet);                                                                                                                             |                                                                                                          |
| event **PeriodChanged**(uint64 newPeriod);                                                                                                                                  | event **MinDepoChanged**(uint128 newMinDepo);                                                                |
| event **MinDepoChanged**(uint128 newMinDepo);                                                                                                                               |                                                                                                          |
| event **WithdrawTimeoutChanged**(uint64 newWithdrawTimeout);                                                                                                                | event **WithdrawTimeoutChanged**(uint64 newWithdrawTimeout);                                                 |
| event **ClaimTimeoutChanged**(uint64 newClaimTimeout);                                                                                                                      | event **ClaimTimeoutChanged**(uint64 newClaimTimeout);                                                       |
| event **ReinforcementAbilityChanged**(uint128 newReinforcementAbility);                                                                                                     | event **ReinforcementAbilityChanged**(uint128 newReinforcementAbility);                                      |
| event **coreChanged**(address newCore);                                                                                                                                     |                                                                                                          |
|                                                                                                                                                                         | event **CoreUpdated**(address indexed core, bool active);                                                    |
|                                                                                                                                                                         | event **RoleUpdated**(address indexed actor, uint8 role, bool active);                                       |
|                                                                                                                                                                         | event **AffiliateRewarded**(address indexed affiliate, uint256 amount);                                      |
|                                                                                                                                                                         | event **FeeChanged**(FeeType feeType, uint64 fee);                                                           |
|                                                                                                                                                                         | event **GameCanceled**(uint256 indexed gameId);                                                              |
|                                                                                                                                                                         | event **GameShifted**(uint256 indexed gameId, uint64 newStart);                                              |
|                                                                                                                                                                         | event **NewGame**( uint256 indexed oracleGameId, uint256 indexed gameId, bytes32 ipfsHash, uint64 startsAt); |


</p>
</details>

_________________

<details><summary>Core</summary>
<p>

### functions
| V1                                                                                                                                                                                       | V2                                                                                                                                                                             |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| function **createCondition**( uint256 oracleCondId, uint128 scopeId, uint64\[2\] memory odds, uint64\[2\] memory outcomes, uint64 timestamp, bytes32 ipfsHash ) external override onlyOracle | function **createCondition**( uint256 gameId, uint256 oracleConditionId, uint64\[2\] calldata odds, uint64\[2\] calldata outcomes, uint128 reinforcement, uint64 margin ) external |
| function **createCondition**( uint256 oracleCondId, uint128 scopeId, uint64\[2\] memory odds, uint64\[2\] memory outcomes, uint64 timestamp, bytes32 ipfsHash ) external override onlyOracle | function **resolveCondition**(uint256 oracleConditionId, uint64 outcomeWin) external                                                                                               |
| function **setLp**(address lp) external override onlyOwner                                                                                                                                   |                                                                                                                                                                                |
| function **setOracle**(address oracle) external onlyOwner                                                                                                                                    |                                                                                                                                                                                |
| function **renounceOracle**(address oracle) external onlyOwner                                                                                                                               |                                                                                                                                                                                |
| function **addMaintainer**(address maintainer, bool active) external onlyOwner                                                                                                               |                                                                                                                                                                                |
| function **cancelByOracle**(uint256 oracleCondId) external onlyOracle                                                                                                                        | function **cancelByOracle**(uint256 oracleConditionId) external onlyOracle                                                                                                         |
| function **cancelByMaintainer**(uint256 conditionId) external onlyMaintainer                                                                                                                 | function **cancelByMaintainer**(uint256 conditionId) external onlyMaintainer                                                                                                       |
| function **shift**(uint256 oracleCondId, uint64 newTimestamp) external onlyOracle                                                                                                            |
| function **claimOracleReward**() external onlyOracle                                                                                                                                         |                                                                                                                                                                                |
| function **changeMaxBanksRatio**(uint64 newRatio) external onlyMaintainer                                                                                                                    |                                                                                                                                                                                |
| function **updateReinforcements**(uint128\[\] memory data) external onlyMaintainer                                                                                                           |
| function **changeDefaultReinforcement**(uint128 reinforcement) external onlyMaintainer                                                                                                       |
| function **updateMargins**(uint128\[\] memory data) external onlyMaintainer                                                                                                                  |                                                                                                                                                                                |
| function **changeDefaultMargin**(uint128 margin) external onlyMaintainer                                                                                                                     |                                                                                                                                                                                |
| function **stopAllConditions**(bool flag) external onlyMaintainer                                                                                                                            |                                                                                                                                                                                |
| function **stopCondition**(uint256 conditionId, bool flag) external onlyMaintainer                                                                                                           | function **stopCondition**(uint256 conditionId, bool flag) external onlyMaintainer                                                                                                 |
| function **getCondition**(uint256 conditionId) external view returns (Condition memory)                                                                                                      | function **getCondition**(uint256 conditionId) external view returns (Condition memory)                                                                                            |
| function **getConditionFunds**(uint256 conditionId) external view returns (uint128\[2\] memory fundBank)                                                                                     |
| function **getConditionReinforcement**(uint256 conditionId) external view returns (uint128 reinforcement)                                                                                    |
| function **getBetInfo**(uint256 betId) external view override returns ( uint128 amount, uint64 odds, uint64 createdAt)                                                                       |
| function **isOracle**(address oracle) external view override returns (bool)                                                                                                                  |
| function **getReinforcement**(uint64 outcomeId) public view returns (uint128)                                                                                                                |
| function **getReinforcement**(uint64 outcomeId) public view returns (uint128)                                                                                                                |
|                                                                                                                                                                                          |                                                                                                                                                                                |
|                                                                                                                                                                                          |                                                                                                                                                                                |
| function **viewPayout**(uint256 tokenId) public view override returns (bool success, uint128 amount)                                                                                         | function **viewPayout**(address account, uint256 tokenId) public view virtual returns (bool, uint128)                                                                              |
| function **calculateOdds**( uint256 conditionId, uint128 amount, uint64 outcome ) public view returns (uint64 odds)                                                                          | function **calcOdds**( uint256 conditionId, uint128 amount, uint64 outcome ) external view override returns (uint64 odds)                                                          |
| function **calcOdds**( uint256 conditionId, uint128 amount, uint64 outcome ) external view override returns (uint64 odds)                                                                    |
|                                                                                                                                                                                          | function **calcOdds**( uint256 conditionId, uint128 amount, uint64 outcome ) external view override returns (uint64 odds)                                                          |
|                                                                                                                                                                                          | function **getTokenInfo**(uint256 tokenId) external view returns (Condition memory, uint256)                                                                                       |
|                                                                                                                                                                                          | function **getTokenInfo**(uint256 tokenId) external view returns (Condition memory, uint256)                                                                                       |
### events

| V1                                                                                                         | V2                                                                                                               |
| ---------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| event ConditionCreated( uint256 indexed oracleConditionId, uint256 indexed conditionId, uint64 timestamp); | event ConditionCreated( uint256 indexed gameId, uint256 indexed oracleConditionId, uint256 indexed conditionId); |
| event ConditionCreated( uint256 indexed oracleConditionId, uint256 indexed conditionId, uint64 timestamp); | event ConditionResolved( uint256 indexed conditionId, uint8 state, uint64 outcomeWin, int128 lpProfit);          |
| event LpChanged(address indexed newLp);                                                                    |                                                                                                                  |
| event MaxBanksRatioChanged(uint64 newRatio);                                                               |                                                                                                                  |
| event MaintainerUpdated(address indexed maintainer, bool active);                                          |                                                                                                                  |
| event OracleAdded(address indexed newOracle);                                                              |                                                                                                                  |
| event OracleRenounced(address indexed oracle);                                                             |                                                                                                                  |
| event AllConditionsStopped(bool flag);                                                                     |                                                                                                                  |
| event ConditionStopped(uint256 indexed conditionId, bool flag);                                            | event ConditionStopped(uint256 indexed conditionId, bool flag);                                                  |
| event ConditionCreated( uint256 indexed oracleConditionId, uint256 indexed conditionId, uint64 timestamp); |
|                                                                                                            | event OddsChanged(uint256 indexed conditionId, uint64\[2\] newOdds);                                             |
|                                                                                                            | event OddsChanged(uint256 indexed conditionId, uint64\[2\] newOdds);                                             |




</p>
</details>

______________________________________

<details><summary>AzuroBet</summary>
<p>
### functions 
| V1 (ERC721)                                                                                            | V2 (ERC1155)                                                                                                                     |
| ------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| function burn(uint256 tokenId) external                                                                | function tokensOfOwner(address owner\_) external view returns (uint256\[\] memory ids)                                           |
| function burn(uint256 tokenId) external                                                                |                                                                                                                                  |
| function mint(address account, address core) external                                                  |                                                                                                                                  |
| function setBaseURI(string calldata uri) external                                                      | function setURI(string memory newUri) external                                                                                   |
| function setLp(address lp) external                                                                    |                                                                                                                                  |
| function getCoreByToken(uint256 tokenId) external view override returns (address core)                 |
|                                                                                                        | function setURI(string memory newUri) external                                                                                   |
|                                                                                                        | function setURI(string memory newUri) external                                                                                   |
|                                                                                                        | function setURI(string memory newUri) external                                                                                   |
|                                                                                                        | function balanceOfBatch(address\[\] memory accounts, uint256\[\] memory ids) external view override returns (uint256\[\] memory) |
|                                                                                                        | function balancePayoutOf(address account, uint256 id) external view override returns (uint256)                                   |
| function tokenOfOwnerByIndex(address owner, uint256 index) public view override returns (uint256)      | function balancePayoutOf(address account, uint256 id) external view override returns (uint256)                                   |
|                                                                                                        | function tokenOfOwnerByIndex(address owner, uint256 index) public view override returns (uint256)                                |
|                                                                                                        | function balanceOf(address account, uint256 id) public view override returns (uint256)                                           |
|                                                                                                        | function isApprovedForAll(address account, address operator) public view override returns (bool)                                 |
| function ownerOf(uint256 tokenId) public view override(ERC721Upgradeable, IAzuroBet) returns (address) |

### events

| V1                           | V2 |
| ---------------------------- | -- |
| event LpChanged(address lp); |    |

</p>
</details>

__________________________________________________________

<details><summary>Factory</summary>
<p>
### function
| V1 | V2                                                                                                                                          |
| -- | ------------------------------------------------------------------------------------------------------------------------------------------- |
|    | function updateCoreType(string calldata coreType, address beaconCore) external onlyOwner                                                    |
|    | function createPool(address token, uint64 daoFee, uint64 oracleFee, uint64 affiliateFee, string calldata coreType, address oracle) external |
|    | function plugCore(address lpAddress, string calldata coreType) external                                                                     |

### events

| V1 | V2                                                         |
| -- | ---------------------------------------------------------- |
|    | event CoreTypeUpdated(string coreType, address beaconCore) |
|    | event NewCore(address lp, address core, string coreType);  |
|    | event NewPool(address lp, address core, string coreType);  |

</p>
</details>

_______________________________

<details><summary>CLICK ME</summary>
<p>

</p>
</details>


