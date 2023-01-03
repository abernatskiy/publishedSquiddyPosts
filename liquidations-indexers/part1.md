Developing Blockchain Indexers Part 1: A Newbie Investigation Of Smart Contracts
================================================================================

In my [previous post](https://medium.com/subsquid/indexing-uniswap-v3-with-subsquid-vs-the-graph-first-impressions-68a5216d107b), I wrote about what blockchain indexers are and compared the experience of local indexer deployment for two popular frameworks - [Subsquid](https://subsquid.io) and [The Graph](https://thegraph.com). In this post, the first one in "Developing Blockchain Indexers" series, I will discuss the process of developing simple indexers using the same pair of frameworks, including the investigations that have to be done before or during the development process.

My first task was to index the blockchain data on liquidations happening on the [AAVE lending platform](https://aave.com). I went in as a complete newbie, without proficiency in smart contracts or the AAVE protocol. I began by skimming over [the Subsquid tutorials](https://docs.subsquid.io/tutorials/). All of them follow the same top-down structure - define a database schema for the data you want, then get your data. However, being a complete newbie, I did not know *what data is safe to want*. Or, in this particular case, what can possibly be known about an AAVE liquidation. So I turned the approach upside down and began by taking a look at the involved smart contracts. But first I had to get up to speed with how smart contracts operate and what kind of data they store on chain. This post summarizes my findings and tells the story of me applying them to AAVE liquidations.

**Disclaimer: this post is sponsored by Subsquid.**

**Prerequisites:** browser, Internet access.  
**Dependencies:** none.  
**Difficulty:** easy, hopefully.  

# Intro

Smart contracts are pieces of software that run on a blockchain. In practice that means that they are run on nodes of a blockchain network, but unlike regular code the nodes reach [consensus](https://en.wikipedia.org/wiki/Consensus_(computer_science)#Permissionless_consensus_protocols) on the code being executed, its input and any results of the computation. That makes it irrelevant which node actually ended up executing the code. It becomes a property of a blockchain as a whole rather than a property of any particular node.

The most popular blockchain implementing smart contracts is Ethereum. Many other chains strive to be compatible with it. It also hosts AAVE, the lending platform that I aim to get data from. For these reasons from here on I will only talk about smart contracts on Ethereum. Smart contracts that you might find on other chains will likely look and behave in a similar way.

Smart contracts are deployed by users by making a special transaction. A user attaches the contract binary code to the transaction, pays a gas fee and poof - a contract is created. It gets its own address similar to that of a user. From here on, users and other contracts can call its *methods* - chunks of contract code identified by their call signatures. To call a contract method, user - again - sends it a transaction, attaching the encoded call signature and argument values as input data. The contract may, in turn, make its own transactions, including calls of methods of other contracts.

Method calls can emit *events* - log entries in a special format that are saved onto the blockchain. Each such entry consists of zero to four topics and a data section. *Topics* are 32-byte binary blobs that are used to lookup and filter events efficiently and *event data* is just an arbitrary length binary blob attached to the log entry. Most events behave as special functions that can be called from within the methods and that do nothing but encode their argument values into a blob and write the log entry with the blob as its data. The signature of such a special function is called *event name*. For any event emitted by any such function the first topic is set to the hash of the function's signature. This allows those who know an event name to filter out all events with this name and decode their data sections to reconstruct the values of arguments of the event-emitting special function.

Contracts may have state, meaning that method calls can cause some data to be stored onto the blockchain and that this data can change the execution of subsequent calls. Typically, a single service involves multiple contracts performing different duties. One [common pattern](https://eips.ethereum.org/EIPS/eip-1967) (which I learned the hard way, as you'll see) is to have two separate contracts, with one ("proxy") holding state but little actual code and the other ("implementation") holding all the contract logic but no state. Contracts can also be split along logical lines, allowing for modularity. For example, for services that handle multiple token types it is common to have a separate smart contract for state and/or logic specific to every supported token. A document describing all contracts involved in a service and protocols for their interaction is called a *service protocol*. [AAVE](https://aave.com) is a service protocol used by the service whose front end is at [app.aave.com](https://app.aave.com).

Here are some of the resources I found useful while figuring all of this out:
 - [an overview of the virtual machine that executes the smart contracts](https://medium.com/mycrypto/the-ethereum-virtual-machine-how-does-it-work-9abac2b7c9e);
 - [technical details on how events are emitted](https://medium.com/mycrypto/understanding-event-logs-on-the-ethereum-blockchain-f4ae7ba50378);
 - [technical details on how contract methods are called](https://medium.com/@hayeah/how-to-decipher-a-smart-contract-method-call-8ee980311603);
 - [the three common patterns of event usage in smart contracts](https://consensys.net/blog/developers/guide-to-events-and-logs-in-ethereum-smart-contracts/).

And of course many thanks to [Etherscan](https://etherscan.io) for their amazing resource that allows observing how smart contracts operate in the wild.

# Finding liquidations data in contracts

For the task at hand, the first step was to figure out which data is stored onto the chain when a liquidation happens. It should be clear by now that there are only three types of traces that might be left - transactions triggering the execution of contract methods, events that the contracts might emit and public contract state changes. Out of these three, events are by far the easiest to work with, but they aren't always available.

To find out what is available for AAVE I looked at its documentation. There I found a [detailed description of the service protocol](https://docs.aave.com/developers/getting-started/contracts-overview) and even a reference implementation for its latest version, V3, in [Solidity](https://soliditylang.org). Debt liquidation is a procedure that is central to functioning of any lending platform, so I figured that I might find the traces in some of the [core smart contracts](https://github.com/aave/aave-v3-core/tree/master/contracts) of the service. I began by searching for methods that might be related to liquidation. Methods are `function`s in Solidity, so I included that keyword as well:

![Github methods search screenshot](/liquidations-indexers/githubLiquidationsMethodsSearch.png)

This returns three pages of results, most of which are clearly intended for internal use or for preparing for the deed, with declarations like `setLiquidationBonus()` and `isLiquidationAllowed()`. But there were three results that looked like methods that can actually be called to perform the liquidation:

![Github IPool contract liquidationCall](/liquidations-indexers/githubLiquidationsMethodsIPool.png)

![Github Pool contract liquidationCall](/liquidations-indexers/githubLiquidationsMethodsPool.png)

![Github L2Pool contract liquidationCall](/liquidations-indexers/githubLiquidationsMethodsL2Pool.png)

The search for events included the `event` keyword and returned just two relevant results leading to identical definitions:

![Github events search screenshot](/liquidations-indexers/githubLiquidationsEventsSearch.png)

![Github LiquidationLogic contract liquidationCall](/liquidations-indexers/githubLiquidationsEventsLiquidationLogic.png)

![Github IPool contract liquidationCall](/liquidations-indexers/githubLiquidationsEventsIPool.png)

I could track liquidations by finding all transactions calling the `liquidationCall` methods. However, since there are special events that are supposed to announce liquidations to the network, I chose to use these.

At this point I set out to find the smart contract that emits `LiquidationCall` events within the infrastructure of the [AAVE lending service](https://app.aave.com), with the only lead being that its name will likely include "aave" and "pool".

# Finding the right contract

I tried three different approaches while searching for the right contract. Listing them all because the readers may need any of them.

1. Addresses of the contracts can sometimes be found in documentation of their respective projects. In case of AAVE only the protocol is documented, not any of its deployments.

2. [Etherscan](https://etherscan.io) maintains incomplete lists of contracts associated with larger projects. They can be looked up directly on Etherscan by project name (requires free registration) or found with Google. Google search for `aave pool contract` actually returns the [etherscan page of the contract I wanted](https://etherscan.io/address/0x7d2768de32b0b80b7a3454c06bdac94a69ddc7a9) as the second search result. However, I missed it because I did not yet know about [proxy contracts](ttps://eips.ethereum.org/EIPS/eip-1967) at this point. More on that later.

3. Finally, I took the approach which I consider a "silver bullet" for this issue: I used the service and traced the resulting transactions. After connecting my Metamask wallet on the AAVE app page I clicked "Supply" and deposited some ether. While confirming the transaction I saw the address of the contract that my wallet was going to interact with:

![AAVE supply contract in Metamask](/liquidations-indexers/aaveSupplyScreenshot.png)

The address pointed to [this contract](https://etherscan.io/address/0xeffc18fc3b7eb8e676dac549e0c693ad50d1ce31/advanced). I looked at its internal transactions and found that it talks a lot to [AAVE: Lending Pool V2](https://etherscan.io/address/0x7d2768de32b0b80b7a3454c06bdac94a69ddc7a9), the contract from the Google search.

There was, however, an issue with this contract. When I tried downloading its *ABI* - a bundle of signatures of events and methods that is necessary to interface with the contract - I found no mentions of `LiquidationCall` events inside, or any of the event names I previously saw in `IPool.sol`. Instead, there just a few methods with `upgrade` and `proxy` in their names. And that's when I noticed The Button:

![The Button](/liquidations-indexers/theButton.png)

After clicking it I saw the following message:

![The Button clicked](/liquidations-indexers/theButtonClicked.png)

And that was how I found the [LendingPool implementation contract](https://etherscan.io/address/0xc6845a5c768bf8d7681249f8927877efda425baf). Turned out that the AAVE Lending Pool V2 contract was a proxy which held state and handled upgrades, while the actual contract logic lived at this address. ABI I downloaded for this contract did contain a call signature for `LiquidationCall`s, but the events were actually emitted not by this contract, but by the proxy. I checked that by visiting the "Events" tab on [the proxy contract page](https://etherscan.io/address/0x7d2768de32b0b80b7a3454c06bdac94a69ddc7a9) and observing that other events defined alongside `LiquidationCall` were regularly emitted there. The only reason I did not see any `LiquidationCall`s was because those are fairly rare.

At this point I knew that I need to scrape the data about `LiquidationCall` events (or more precisely `LiquidationCall(address,address,address,uint256,uint256,address,bool)` - remember that events are identified by their full signatures) emitted by contract `0x7d2768de32b0b80b7a3454c06bdac94a69ddc7a9` aka AAVE Lending Pool V2. In the next post I will use this knowledge to develop indexers powered by Subsquid and The Graph.
