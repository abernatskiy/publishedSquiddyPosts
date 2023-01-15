Developing Blockchain Indexers Part 2: Simplest Nontrivial Examples
===================================================================

This post continues the "Developing Blockchain Indexers" series. In [Part 1](https://medium.com/subsquid/developing-blockchain-indexers-part-1-a-newbie-investigation-of-smart-contracts-38e140db6a2e) I summarized my investigaton on how Ethereum smart contracts operate and what kind of data they produce. I also told the story of how I found the exact location of data produced by liquidations on the [AAVE lending platform](https://app.aave.com) and laid out a fairly general approach to such searches.

The conclusion was that I need to scrape the data contained within events with signature `LiquidationCall(address,address,address,uint256,uint256,address,bool)` (or `LiquidationCall` for short) emitted by the contract `0x7d2768de32b0b80b7a3454c06bdac94a69ddc7a9`. This is perhaps the simplest possible task for a blockchain indexer. In this post I describe the development of two indexers that perform it using the two popular frameworks: [Subsquid](https://subsquid.io) and [The Graph](https://thegraph.com).

**Disclaimer: this post is sponsored by Subsquid.** I will, however, try to keep things as objective as possible and highlight the pros and cons of both frameworks.

**Prerequisites:** a Linux system, familiarity with command line. For The Graph also a local indexer node ([installation instructions](https://medium.com/subsquid/indexing-uniswap-v3-with-subsquid-vs-the-graph-first-impressions-68a5216d107b#51b7)) and a browser Ethereum wallet if you want to check out Subgraph Studio.  
**Dependencies:** NodeJS. For Subsquid also make and Docker. For The Graph also yarn.  
**Difficulty:** easy.  

# Subsquid

As mentioned in [Part 1](https://medium.com/subsquid/developing-blockchain-indexers-part-1-a-newbie-investigation-of-smart-contracts-38e140db6a2e), I used the bottom-up approach to the indexer ("squid" in [Subsquid](https://subsquid.io) terminology) development. This goes into the opposite direction to all [Subsquid tutorials](https://docs.subsquid.io/tutorials/), but I think it is a more newbie-friendly approach. Still, with how simple the development process is, doing such an inversion on any tutorial should be easy.

I began by making a new `liquidations-subsquid` repository out of [squid-ethereum-template](https://github.com/subsquid/squid-ethereum-template)[^1]:

![Using the template](/liquidations-indexers/usingTheTemplate.png)

Then I cloned it, installed the dependencies and started the database:
```bash
$ git clone https://github.com/abernatskiy/liquidations-squid
$ cd liquidations-squid
$ npm ci
$ make up # starts a database in a docker container
```
The next step was to obtain the application binary interface (ABI) for the AAVE Pool smart contract. Although in principle knowing just the event signature is sufficient for finding and parsing all the event data, the practical tools for ABI handling like Etherscan API and `squid-evm-typegen` usually work with more or less complete contract interfaces. It is simply easier to work with ABIs provided by these tools than it is to manually define a partial ABI for just one event using just its signature. Here I follow this easier path, but it is certainly possible to do things the other way around. This might be useful in cases when full ABI is unavailable for any reason.

For [proxy contracts](https://eips.ethereum.org/EIPS/eip-1967) a truly complete ABI capable of processing all the contract data would be a combination of the ABIs of the proxy and the implementation. The [AAVE Pool](https://etherscan.io/address/0x7d2768de32b0b80b7a3454c06bdac94a69ddc7a9#readProxyContract) I am interested in is among such contracts, but since I only want to track `LiquidationCall`s and not any of the proxy-specific activity (such as updates) I can use just the implementation contract ABI. You can read more on the proxy pattern and on how to find an implementation contract given the proxy in [Part 1](https://medium.com/subsquid/developing-blockchain-indexers-part-1-a-newbie-investigation-of-smart-contracts-38e140db6a2e) of this series of posts.

The most common format for describing ABIs of Ethereum contracts is [JSON](https://docs.soliditylang.org/en/v0.8.17/abi-spec.html#abi-json). To actually interface with the contract I needed to transform it into Typescript code using the `squid-evm-typegen` tool.

Originally the whole process involved fetching the JSON ABI from Etherscan using `curl`, then stripping the metadata Etherscan adds to it and then generating Typescript code. Since then this procedure has been automated and now it can be done with a single command:
```bash
$ npx squid-evm-typegen src/abi 0xC6845a5C768BF8D7681249f8927877Efda425baf#aave-lending-pool-v2
```
Here, `0xC6845a5C768BF8D7681249f8927877Efda425baf` is the address of the *implementation contract* and `aave-lending-pool-v2` is the name of the resulting Typescript module that gets generated at `src/abi`.

I encourage the readers who might follow in my footsteps to take a look at `src/abi/aave-lending-pool-v2.ts` and note the `events` object defined just below `export const abi`. Each of its properties holds a `LogEvent` object that is used to access and decode the events corresponding to its name. Important properties of `LogEvent` objects include:
 * `.topic` - holds the first topic of the event log entries. Useful for finding them within the blockchain data.
 * `.decode` - a method for decoding the data sections of the log entries and retrieving event arguments values.

I now had everything in place to configure the squid's [EVM processor](https://docs.subsquid.io/develop-a-squid/evm-processor/). It is constructed and used at `src/processor.ts`:
```typescript
/*** src/processor.ts ***/

import {TypeormDatabase} from '@subsquid/typeorm-store'
import {EvmBatchProcessor} from '@subsquid/evm-processor'
import * as lendingPoolAbi from './abi/aave-lending-pool-v2'

const processor = new EvmBatchProcessor()
  .setBlockRange({from: 11362579})
  .setDataSource({
    chain: process.env.ETHEREUM_MAINNET_WSS,
    archive: 'https://eth.archive.subsquid.io',
  })
  .addLog('0x7d2768dE32b0b80b7a3454c06BdAc94A69DDc7A9', {
    filter: [[lendingPoolAbi.events.LiquidationCall.topic]],
    data: {
      evmLog: {
        data: true
      }
    } as const
  })

processor.run(new TypeormDatabase(), async (ctx) => {
  for (let c of ctx.blocks) {
    for (let i of c.items) {
      console.log(i); // simply log the items
    }
  }
});

/*** src/processor.ts - END ***/
```
This instructs the processor to go through all blocks starting at 11362579 (the block at which the proxy contract was deployed), getting data from the Subsquid-provided Ethereum archive and looking for logs (events) generated by `0x7d2768dE32b0b80b7a3454c06BdAc94A69DDc7A9` (the proxy contract). Further, it should only get the events with topics containing `lendingPoolAbi.events.LiquidationCall.topic` that is, the `LiquidationCall` events. For now I only request the access to `.evmLog.data` field of each event data item - that is, to the event data binary blob.

It is enlightening to pause here, run `make build; make process` to get the syncing started and observe the output:

![Trivial processor logs screenshot](/liquidations-indexers/trivialProcessorLogs.png)

Two kinds of items are passing the processor's filters: the events we need and the transactions that emitted them. Transactions are duplicated within the events, so they can be safely ignored. A lot more data is actually available in the events than what is requested. Data access permissions are implemented through static typing. When I tried to access it without requesting it in the processor constructor, I got compilation errors at build stage:
```typescript
processor.run(new TypeormDatabase(), async (ctx) => {
  for (let c of ctx.blocks) {
    for (let i of c.items) {
      if(i.kind==='evmLog') {
        console.log(i.evmLog.data); // OK
        console.log(i.evmLog.topics); // error TS2339: Property 'topics' does not exist on type ...
      }
    }
  }
});
```
Some fields are straight up unaccessible:
```typescript
const processor = new EvmBatchProcessor()
  .setBlockRange({from: 11362579})
  .setDataSource({
    chain: process.env.ETHEREUM_MAINNET_WSS,
    archive: 'https://eth.archive.subsquid.io',
  })
  .addLog('0x7d2768dE32b0b80b7a3454c06BdAc94A69DDc7A9', {
    filter: [[lendingPoolAbi.events['LiquidationCall(address,address,address,uint256,uint256,address,bool)'].topic]],
    data: {
      evmLog: {
        blockNumber: true
      }
    } as const
  })

processor.run(new TypeormDatabase(), async (ctx) => {
  for (let c of ctx.blocks) {
    for (let i of c.items) {
      if(i.kind==='evmLog') {
        console.log(i.evmLog.blockNumber) // error TS2769: No overload matches this call
      }
    }
  }
});
```
This is the expected behavior documented at the [data selectors subsection](https://docs.subsquid.io/develop-a-squid/evm-processor/configuration/#data-selectors) of processor documentation. Like `blockNumber`, these fields are redundant and are likely sealed off to keep the one obvious way to do things, although I must admit it was not always that obvious. For example, to actually get the block number one has to access `c.header.height`.

By now I began to get a picture for what data I want to keep for each event log entry. Aside from the event argument values I also wanted to keep the block number at which the event occured and the hash of its parent transaction. I also did not want any extra data about any of the event parameter values, for now anyway.

The next thing I did was decode the event arguments. The decoder for `LiquidationCall` events data is located at `lendingPoolAbi.events.LiquidationCall.decode`. Properties available within the objects it returns can be deduced from the arguments of the `LogEvent` generic type of the `lendingPoolAbi.events.LiquidationCall` object (see `src/abi/aave-lending-pool-v2.ts`):
```typescript
// reformatted for readability
export const events = {
  ...
  LiquidationCall: new LogEvent<(
    {
      collateralAsset: string, debtAsset: string, user: string,
      debtToCover: ethers.BigNumber, liquidatedCollateralAmount: ethers.BigNumber,
      liquidator: string, receiveAToken: boolean
    } &
    [
      // ...same fields, but within an array
    ]
  )>(abi, '0xe413a321e8681d831f4dbccbca790d2952b56f977908e45be37335533e005286'),
  ...
}
```
After destructuring this and declaring a couple of convenience variables I was done with the "extract" step of ETL. Find the full code of `src/processors.ts` at this point:
```typescript
/*** src/processor.ts ***/

import {TypeormDatabase} from '@subsquid/typeorm-store'
import {EvmBatchProcessor} from '@subsquid/evm-processor'
import * as lendingPoolAbi from './abi/aave-lending-pool-v2'

const processor = new EvmBatchProcessor()
  .setBlockRange({from: 11362579})
  .setDataSource({
    chain: process.env.ETHEREUM_MAINNET_WSS,
    archive: 'https://eth.archive.subsquid.io',
  })
  .addLog('0x7d2768dE32b0b80b7a3454c06BdAc94A69DDc7A9', {
    filter: [[lendingPoolAbi.events.LiquidationCall.topic]],
    data: {
      evmLog: {
        topics: true,
        data: true
      },
      transaction: {
        hash: true
      }
    } as const
  })

processor.run(new TypeormDatabase(), async (ctx) => {
  for (let c of ctx.blocks) {
    for (let i of c.items) {
      if (i.kind==='evmLog') {
        const {
          collateralAsset,
          debtAsset,
          user,
          debtToCover,
          liquidatedCollateralAmount,
          liquidator,
          receiveAToken
        } = lendingPoolAbi.events.LiquidationCall.decode(i.evmLog)
        const block = c.header.height
        const hash = i.transaction.hash
      }
    }
  }
});

/*** src/processor.ts - END ***/
```

To store ("load") the data I wrote the following database schema:
```graphql
# schema.graphql

type LiquidationEvent @entity {
  id: ID!
  collateralAsset: String! @index
  debtAsset: String! @index
  user: String! @index
  debtToCover: BigInt!
  liquidatedCollateralAmount: BigInt!
  liquidator: String! @index
  receiveAToken: Boolean!
  block: BigInt!
  hash: String! @index
}

# schema.graphql - END
```
This instructs the data model code generator to make a single (one per every `@entity` directive) table called `liquidation_event` with columns corresponding to the fields of the entity. Columns with the `@index` directive will be indexed for faster lookup and comparison. I thought I may want to search for liquidation events by collateral asset, debt asset, user and liquidator, so I stuck this directive in front of these fields. `ID` is a required column with values that should be unique.

A full (and good) description of the GraphQL schema dialect used by Subsquid can be found [here](https://docs.subsquid.io/develop-a-squid/schema-file/).

The code generator will also make a TypeORM type called `LiquidationEvent` (not to be confused with the `LiquidationCall` in the event name). Its instances map onto rows of the `liquidation_event` table. The type is to be available at the `./model` module.

Once done with the schema I ran
```bash
$ npx squid-typeorm-codegen
$ make build
$ make down
$ rm -rf db/migrations/*.js
$ make up
$ make migration
$ make migrate # optional, will be executed at make process anyway
```
to generate the TypeORM code and update the schema in the database. Then I was almost ready to store the extracted data.

Almost - because I also needed to supply unique ID for every `LiquidationEvent` entity. I tried using the parent transaction hash for that, but it turns out that it is possible to a single transaction to emit multiple `LiquidationCall`s. Instead, I used the unique string from the `id` field of the event log data item.

After adding a few type conversions and a buffer for the finished `LiquidationEvent` entities I was done:
```typescript
/*** src/processor.ts ***/

import {TypeormDatabase} from '@subsquid/typeorm-store'
import {EvmBatchProcessor} from '@subsquid/evm-processor'
import * as lendingPoolAbi from './abi/aave-lending-pool-v2'
import {LiquidationEvent} from './model'

// The so-called AAVE V2 (0x7d2768de32b0b80b7a3454c06bdac94a69ddc7a9) - starts at 11362579
// Tis' a proxy, implementation is at 0xc6845a5c768bf8d7681249f8927877efda425baf

const processor = new EvmBatchProcessor()
  .setBlockRange({from: 11362579})
  .setDataSource({
    chain: process.env.ETHEREUM_MAINNET_WSS,
    archive: 'https://eth.archive.subsquid.io',
  })
  .addLog('0x7d2768dE32b0b80b7a3454c06BdAc94A69DDc7A9', {
    filter: [[lendingPoolAbi.events.LiquidationCall.topic]],
    data: {
      evmLog: {
        id: true,
        topics: true,
        data: true
      },
      transaction: {
        hash: true
      }
    } as const
  })

processor.run(new TypeormDatabase(), async (ctx) => {
  const liquidations: LiquidationEvent[] = [];

  for (let c of ctx.blocks) {
    for (let i of c.items) {
      if (i.kind==='evmLog') {
        const {
          collateralAsset,
          debtAsset,
          user,
          debtToCover,
          liquidatedCollateralAmount,
          liquidator,
          receiveAToken
        } = lendingPoolAbi.events.LiquidationCall.decode(i.evmLog)
        const block = c.header.height
        const hash = i.transaction.hash
        const eventId = i.evmLog.id

        liquidations.push(new LiquidationEvent({
          id: eventId,
          collateralAsset: collateralAsset,
          debtAsset: debtAsset,
          user: user,
          debtToCover: debtToCover.toBigInt(),
          liquidatedCollateralAmount: liquidatedCollateralAmount.toBigInt(),
          liquidator: liquidator,
          receiveAToken: receiveAToken,
          block: BigInt(block),
          hash: hash
        }))
      }
    }
  }

  await ctx.store.save(liquidations)
});

/*** src/processor.ts - END ***/
```
To start this extremely simple squid all I had to do now was to run `make build; make process`. The squid requires no Ethereum RPC endpoint and syncs in about 9 minutes. Its full code is available [here](https://github.com/abernatskiy/liquidations-squid/).

# The Graph

To make a subgraph with functionality equivalent to that of the squid I loosely followed the [main subgraph creation tutorial](https://thegraph.com/docs/en/developing/creating-a-subgraph/).

For a start I installed `graph-cli` globally [^2]:
```bash
$ npm install -g @graphprotocol/graph-cli
```
[The next step](https://thegraph.com/docs/en/developing/creating-a-subgraph/#from-an-existing-contract) was to generate the initial code with the following command:
```bash
$ graph init \
  --product subgraph-studio \
  --from-contract <CONTRACT_ADDRESS> \
  [--network <ETHEREUM_NETWORK>] \
  [--abi <FILE>] \
  <SUBGRAPH_SLUG> [<DIRECTORY>]
```
After the investigations of part 1 nothing caused any difficulties here except `<SUBGRAPH_SLUG>`. According to the tutorial,

> The <SUBGRAPH_SLUG> is the ID of your subgraph in Subgraph Studio, it can be found on your subgraph details page.

There were a few unknowns in this statement, so I began digging. [This video tutorial](https://www.youtube.com/watch?v=HfDgC2oNnwo) was particularly helpful in understanding what is really going on here.

[Subgraph Studio](https://thegraph.com/studio/) appears to be TheGraph's cloud service for developing subgraphs and managing them after they are deployed ([see more details here](https://thegraph.com/docs/en/deploying/subgraph-studio/)). It handles billing, among other things, and requires connecting a wallet for registration. It is a centralized frontend to a mostly centralized service.

To create a new subgraph, a user is supposed to go to the Studio and introduce themselves by attaching their wallet. Then they can "create a subgraph" there by registering a string handle. Lowercase of that handle is the aforementioned `<SUBGRAPH_SLUG>`.

![Subgraph creation at Studio](/liquidations-indexers/subgraphStudioCreationScreenshot.png)

Does one really need to go through all of this just to play around making subgraphs to run on their local indexer node? Turns out, the answer is no. The code created by `graph init` works perfectly fine with subgraph slugs not registered with the Studio, up to and including local deployment. For those who plan to eventually deploy the subgraph into one of TheGraph's clouds or its decentralized network, it is probably wise to keep the slug lowercase and different from slugs of any other subgraphs you might have.

In my case I chose `abliquidationstracker` as my slug. For ABI I used the JSON I obtained using the now-obsolete instructions from Subsquid documentation:
```bash
$ curl "https://api.etherscan.io/api?module=contract&action=getabi&address=0xC6845a5C768BF8D7681249f8927877Efda425baf" --output /tmp/aave-lending-pool-v2-raw.json
$ python -c "import json; ifile=open('/tmp/aave-lending-pool-v2-raw.json', 'r'); print(json.load(ifile)['result']); ifile.close()" > /tmp/aave-lending-pool-v2.json
$ rm /tmp/aave-lending-pool-v2-raw.json
```
The final code generation command was:
```bash
$ graph init \
  --product subgraph-studio \
  --from-contract 0x7d2768de32b0b80b7a3454c06bdac94a69ddc7a9 \
  --abi /tmp/aave-lending-pool-v2.json \
  abliquidationstracker abLiquidationsTracker/
```
This code generator asked me a few straightforward questions. The only caveat I found is that the contract name must not include any "`-`" symbols, as in `aave-v2-pool`. This is because it will be directly used as a module name in the generated Typescript code. Since I wanted to index an event emitted by the contract, I chose `Y` in `Index contract events as entities`.

![Subgraph code generation by graph cli](/liquidations-indexers/subgraphGraphInitSuccess.png)

This creates a folder `abLiquidationsTracker` with a newly initialized Git repository that tracks all files specific to this subgraph:
```bash
$ git ls-files 
abis/aavev2pool.json
networks.json
package.json
schema.graphql
src/aavev-2-pool.ts
subgraph.yaml
tests/aavev-2-pool-utils.ts
tests/aavev-2-pool.test.ts
tsconfig.json
```
The main configuration for the subgraph is at `subgraph.yaml`. It defined *entities* (things in the GraphQL schema that map to database tables, similar to Subsquid) and Typescript handlers for every event listed in the contract ABI (saved at `abis/aavev2pool.json`). I removed all the code irrelevant to `LiquidationCall`s. Then I added a `startBlock` entry to the `.source` section of the data source definition to avoid scanning blocks from back when the AAVE Lending Pool v2 contract did not exist.

And for the main configuration file that was it.
```yaml
# subgraph.yaml

specVersion: 0.0.5
schema:
  file: ./schema.graphql
dataSources:
  - kind: ethereum
    name: aavev2pool
    network: mainnet
    source:
      address: "0x7d2768de32b0b80b7a3454c06bdac94a69ddc7a9"
      abi: aavev2pool
      startBlock: 11362579
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - LiquidationCall
      abis:
        - name: aavev2pool
          file: ./abis/aavev2pool.json
      eventHandlers:
        - event: LiquidationCall(indexed address,indexed address,indexed address,uint256,uint256,address,bool)
          handler: handleLiquidationCall
      file: ./src/aavev-2-pool.ts

# subgraph.yaml - END
```
Then I went ahead and removed all irrelevant entities from the database schema definition at `schema.graphql`. What remained was very similar to the schema I defined for the squid:
```graphql
# schema.graphql

type LiquidationCall @entity(immutable: true) {
  id: Bytes!
  collateralAsset: Bytes! # address
  debtAsset: Bytes! # address
  user: Bytes! # address
  debtToCover: BigInt! # uint256
  liquidatedCollateralAmount: BigInt! # uint256
  liquidator: Bytes! # address
  receiveAToken: Boolean! # bool
  blockNumber: BigInt!
  blockTimestamp: BigInt!
  transactionHash: Bytes!
}

# subgraph.yaml - END
```
Finally, I removed all the handlers except for `handleLiquidationCall` from `src/aavev-2-pool.ts`. The result:
```typescript
/*** src/aavev-2-pool.ts ***/

import {
  LiquidationCall as LiquidationCallEvent
} from "../generated/aavev2pool/aavev2pool"
import {
  LiquidationCall
} from "../generated/schema"

export function handleLiquidationCall(event: LiquidationCallEvent): void {
  let entity = new LiquidationCall(
    event.transaction.hash.concatI32(event.logIndex.toI32())
  )
  entity.collateralAsset = event.params.collateralAsset
  entity.debtAsset = event.params.debtAsset
  entity.user = event.params.user
  entity.debtToCover = event.params.debtToCover
  entity.liquidatedCollateralAmount = event.params.liquidatedCollateralAmount
  entity.liquidator = event.params.liquidator
  entity.receiveAToken = event.params.receiveAToken

  entity.blockNumber = event.block.number
  entity.blockTimestamp = event.block.timestamp
  entity.transactionHash = event.transaction.hash

  entity.save()
}

/*** src/aavev-2-pool.ts - END ***/
```
And with this I was done! All that was left to do was to regenerate types, rebuild the subgraph and I could deploy it on my local graph indexer node[^3]:
```bash
$ yarn codegen
$ yarn build
$ yarn create-local
$ yarn deploy-local
```
In the terminal where the node ran I saw the subgraph being created, deployed and beginning to sync.

![Subgraph begins syncing](/liquidations-indexers/subgraphBeginsSyncingMarkings.png)

It was done after about 100 minutes.

# Conclusions

Both Subsquid and TheGraph make it very easy for me to develop a simple indexer. However, only for the subgraph I would describe the procedure as "braindead". If you just need to index events emitted by any given contract, the `graph` tool will do almost all the work for you. In my case all I needed to do was to remove the irrelevant code and add a starting block.

That is, after I spent an hour figuring out what subgraph slug is and why (and if) I should get it off the cloud. The "get on our platform first, ask questions later" marketing policy of TheGraph is confusing and annoying. A more decent solution would be to enable the developers to work locally without talking to the cloud at all.

Another significant drawback of subgraphs is the need to run an indexer node to test subgraphs locally. Setting it up [is not difficult](https://medium.com/subsquid/indexing-uniswap-v3-with-subsquid-vs-the-graph-first-impressions-68a5216d107b#51b7), but it does take some time.

On the other hand, Subsquid is slightly harder for the super simple tasks like my event tracker *simply because you have to actually write some code*. It is easy, but you actually need to do it and that's more difficult than generating everything and then just removing the unnecessary parts. A simple code generation tool would crack this issue with ease, and Subsquid devs are already developing it. Template scaffolding functionality has been added to the `@subsquid/cli` package in form of the [`sqd init` command](https://docs.subsquid.io/quickstart/quickstart-ethereum/#step-1-scaffold-from-a-template). At the moment it does nothing but create a folder and populate it with files from a github template, but in a few weeks it will be capable of generating simple indexers that scrape events and method calls data based on ABI. A prototype of the code generation tool is [already available](https://github.com/subsquid/squid-abi-template).

In return for the modest increase in development complexity, Subsquid offered - in my case - roughly an elevenfold decrease in sync time, compared to a subgraph using a public ETH node. The difference is not as dramatic as [it is with heavier indexers like the Uniswap one](https://medium.com/subsquid/indexing-uniswap-v3-with-subsquid-vs-the-graph-first-impressions-68a5216d107b#eeb5), as my small subgraph manages to sync in under two hours - a time that is acceptable in practice. Still, going from that to under ten minutes is certainly convenient.

All of this applies to the case when the developer knows where their contract is and what its ABI looks like. For a newbie like me figuring out the ABI of someone else's contract was [a challenge far exceeding the development of indexers in its difficulty](https://medium.com/subsquid/developing-blockchain-indexers-part-1-a-newbie-investigation-of-smart-contracts-38e140db6a2e).

[^1]: A more modern way to do this would be with `sqd init evm-tutorial --template evm; cd evm-tutorial; git init; git add --all`. I discuss `sqd init` in more details in [Conclusions](link-to-conclusions).

[^2]: For those who want to avoid installing global `npm` packages systemwide and cluttering their systems thus, [this Stackoverflow answer](https://stackoverflow.com/a/13021677/3512356) contains good instructions on how to install such packages into home folder.

[^3]: The interested can find my instructions for setting up a graph indexer node [here](https://github.com/abernatskiy/squiddyPosts/blob/main/uniswap-v3-comparison/main.md#graph-node). In my experience it is about as difficult as making the simple subgraph I describe here. If you already have a node and would like to restore it to its original state before deploying a new subgraph, simply drop its database: the node does not keep any data outside of it.
