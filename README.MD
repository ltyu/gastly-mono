This is the mono repo for gastly

# Description
As web3 users navigate the diverse ecosystems across many chains, we will inevitably accumulate an abundance of disparate wallets, with many different tokens.

The goal of building Gastly is to unify all these wallets & tokens in a task that is ubiquitous across every chain: paying for gas.

Gastly is the glue that enables any of your wallets pay for a transaction, using any token, regardless of the chain.

# Definitions
- Root Pool
	- The pool on a home chain (currently Polygon) in which all other branch chains send their gas payments to
- Branch Pool
	- The pool on a chain other than the main chain that the user sends their gas token to. 
- Safe
    - Smart contract wallet
- 1Balance
	- Paymaster for the Safe wallet
- [xERC20](https://github.com/connext/EIPs/blob/master/EIPS/eip-7281.md)
    - A bridgable LP token minted to liquidity providers
    
# Features
## Native Asset Pools
Root and Branch Pools do not rely on a lock and mint mechanism. They instead are liquidity pools that providers directly depost the native token (USDC on Polygon mainnet, goerli Eth on goerli) into. The accounting is done using virtual crediting that is sent through Wormhole

## Dynamic Fee
Fees are charged based on a [utilization curve] of the Root Pool. Utilization is calculatd as
```
Utilization Rate = balanceUsed / gasToken.balanceOf(rootPool)
```
where `balanceUsed` is adjusted everytime stable coins are deposited in 1Balance. 

As utilization rate approaches 1 (or, 100%), it means that liquidity on the root pool is being depleted, and branch pool liquidity is increasing. High utilization means users pay higher fees, which also incentivizes liquidity to be 1) rebalanced from the branch pools, and 2) deposited directly into the root pool.

## Dynamic Balancing Pools
[Diagram](https://raw.githubusercontent.com/ltyu/gastly-mono/master/diagram.png)
At the heart of Gastly is a hub-and-spoke pool model. When a user pays with Branch Pool tokens, intuitively, the tokens should be sent to the Root Pool. However, this would be costly for users if their tokens were always bridged over.

Instead, it relies on internal accounting to determine how much can be bridged.  Rebalancers are incentivizes through rebalancing fees.
```
BranchPool
LP: 100
assets: 100
last known balance of Root: 100
Bridgable bandwidth of BranchPool (b): 100

RootPool
LP: 100
assets: 100
last known balance of Branch: 100
Bridgable bandwidth of RootPool (b): 100

==Transfer 50==
BranchPool
LP: 100
assets: 100 + 50 = 150
last known balance of Root: 100 + 50 = 150
Bridgable bandwidth of BranchPool (b): 100 - 50 = 50

RootPool
LP: 100
assets: 100 - 50 = 50
last known balance of Branch: 100 - 50 = 50
Bridgable bandwidth of RootPool (b): 100+ 500 = 150
```

In the above scenario, 50 tokens are bridged from Branch to Root. This reduces the bandwidth to 50, which mean only 50 more tokens can be bridged. In the Root, the bandwidth has increased to 150, this means up to 150 can be bridged over to Branch. Also to note, that the bandwidth has diverged from the currently liquidity (LP). We can incentivize users to 1) deposit more LP to bring it closer to the bandwidth 1:1, or 2) bridge to Branch

## Account Abstraction
Gastly uses Safe Accounts with 1Balance to pay for transactions. 1Balance receives tokens from the Root Pool.

## Known issues
- Incase crosschain fails, there is no withdraw deposit function
- Gas overages does not get refunded
- No LP fee curve


## Addresses
- RootPool 
    - https://goerli.etherscan.io/address/0x4325df4f116a96846885d3a2c7ebab23b652659d
- BranchPool 
    - https://mumbai.polygonscan.com/address/0x837cfe91eb2d0d309b086faaf94258f379176e52
- Connext xERC20 LP token (goerli) 
    - TX: https://goerli.etherscan.io/tx/0x9ea5db10dc8f45bc5d97d057983bc9b85e4157a26dddf7a85f492f2535336bf6
    - Contract:https://goerli.etherscan.io/address/0xA34488C6C27542cdCDD381D41eb359755B232dc5#code
- Connext xERC20 LP token (mumbai) 
    - TX: https://mumbai.polygonscan.com/tx/0x868edb89cdbd057d994b48569c208828143506387a8e6083cd1a01b145ebc42f
    - Contract: https://mumbai.polygonscan.com/address/0xa34488c6c27542cdcdd381d41eb359755b232dc5

## To verify XERC20
```
forge verify-contract  --verifier-url  https://api-testnet.polygonscan.com/api 0xA34488C6C27542cdCDD381D41eb359755B232dc5 lib/xERC20/solidity/contracts/XERC20.sol:XERC20 --chain 80001 --constructor-args 000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000000a0000000000000000000000000c10fa06d401c05bce65b55efb1ef2158e55ff1d7000000000000000000000000000000000000000000000000000000000000000e676173746c792d6c702d676574680000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000007676c706745746800000000000000000000000000000000000000000000000000 --watch
```