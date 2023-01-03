Indexing Uniswap-v3 with Subsquid vs The Graph: first impressions
================================================================

Accessing blockchains directly can be tedious. Typically, it involves setting up a node, then waiting for it to download and verify the on-chain data - *all* the on-chain data. This can take hours or days and requires powerful hardware. Only once this process has been completed does it become possible to inquire about the current network consensus.

To free end-users from having to do this, Web3 application developers make systems that, in real-time, scrape only the most relevant parts of on-chain data, making the data readily available through regular web APIs. Such systems are called *blockchain indexers* or just indexers for short.

Here I will compare my initial impressions of setting up indexers for [Uniswap](https://app.uniswap.org/) data built with two popular frameworks: the up-and-coming [Subsquid](https://subsquid.io) and [TheGraph](https://thegraph.com), currently the most popular solution in the space. In particular, I will be looking at local (as opposed to "cloud") deployment of the indexers, a setup that is useful for development, iteration, and experimentation.

**Disclaimer: this post is sponsored by Subsquid.** I will, however, try to keep things as objective as possible and highlight the pros and cons of both frameworks.

**Prerequisites:** a Linux system, familiarity with command line, basic knowledge of Docker. Having a synced Ethereum beacon node helps a lot.  
**Build dependencies:** common build system (make, gcc), NodeJS. For The Graph additionally yarn, rust, clang and 8Gb of RAM.  
**Difficulty:** intermediate.

## Subsquid

The [Subsquid](https://subsquid.io) indexer, or "squid", for Uniswap v3 is available at [github.com/subsquid/uniswap-squid](https://github.com/subsquid/uniswap-squid). Squids are regular Node.js applications and they do not require a specialized environment to run. This is in contrast to The Graph's indexers, which run within the nodes of that network.

### Architecture

Squids ingest blockchain data from two sources:

1. [archives](https://docs.subsquid.io/archives/), services that ingest a wide range of data for specific blockchains, filter it and forward it to squids, and
2. directly from blockchain RPC endpoints.

Most archives are open source, meaning that app creators can easily run them on their own infrastructure. However, the Ethereum archive used by the Uniswap squid is still in alpha and has yet to be unveiled. Archive setup, therefore, is outside the scope of this article, but I will address some related concerns when I discuss scalability at [Corners that were and weren't cut](#corners-that-were-and-werent-cut).

### Setup

The procedure is straightforward. The instructions from the README just work:
```bash
$ git clone https://github.com/subsquid/uniswap-squid
$ cd uniswap-squid
$ npm ci # installs dependencies
$ npm run build
```

### Database

The next step spins up a PostgreSQL database for blockchain data caching. One small caveat here: a database Docker container stores the data in a volume automatically created at the Docker data folder (usually `/var/lib/docker`). If that folder happens to be on a small partition, the filesystem will fill up quickly. After a bit of trial and error I found that about 30Gb of free space is necessary. I moved the data folder to a drive that had enough space and ran
```bash
$ make up
```
The container shows up as `uniswap-squid_db_1` in `docker ps`:
![Image of the container showing up](/uniswap-v3-comparison/squid/workingDatabase.png)

### Ethereum node

Aside from the database, the squid requires access to an Ethereum RPC endpoint over websocket [^1]. It is trivial to set one up if you have a private Ethereum beacon node, but at the time of writing mine was still syncing. To skip the wait, I registered with a bunch of cloud endpoint providers and had a bit of an adventure with their services.
<details>
	<summary>Nodereal turned out to be the only compatible service.</summary>

> I went through most of the services marked as "Working" on [this list](https://ethereumnodes.com). Surprisingly not all of them provide RPC over websocket, with many opting to only serve HTTPS. Four services did not ask for a credit card and had websocket access, and three of those ([Rivet](https://rivet.cloud/), [Alchemy](https://alchemyapi.io/) and [Infura](https://infura.io/)) seem to serve API versions incompatible with the Uniswap squid (yes, even Infura). That left me with [Nodereal](https://nodereal.io/meganode) as the only option.
</details>

### Run the project

Once I had my websocket link all I had to do was pass it to the squid via the `CHAIN_NODE` variable in `.env` file
```
CHAIN_NODE=wss://eth-mainnet.nodereal.io/ws/v1/<token>
```
and run
```bash
$ make process
```
At this point the squid began ingesting the network data.

## The Graph

[The Graph](https://thegraph.com) is a blockchain indexing framework that runs on its own decentralized computational platform. Once an indexer (called "subgraph" in The Graph terminology) is deployed, any node of the network can choose to execute it. Such nodes will get rewarded in native GRT tokens based on the number of API queries that they process.

### Architecture

The data is initially ingested when a node of The Graph running a subgraph queries the RPC of a node of a target network (such as Ethereum). However, once the subgraph is deployed to The Graph network, its code and the data it indexed is distributed among other nodes via [IPFS](https://ipfs.tech). Thus, the process of indexing the whole required block range - *syncing* - only happens once for every deployed subgraph.

### Setup

I began by cloning the Uniswap v3 subgraph repo and installing dependencies:
```bash
$ git clone https://github.com/Uniswap/v3-subgraph
$ cd v3-subgraph
$ yarn install
```
Then I inspected the output of `npm run` and found that there are two scripts for building the subgraph:

```bash
$ npm codegen
$ npm build

```

### IPFS

Subsequent steps require IPFS and a working node of The Graph:

```bash
$ npm run 
Scripts available in uniswap-v3-subgraph@1.0.0 via `npm run-script`:
  ...
  create-local
    graph create ianlapham/uniswap-v3 --node http://127.0.0.1:8020
  deploy-local
    graph deploy ianlapham/uniswap-v3 --debug --ipfs http://localhost:5001 --node http://127.0.0.1:8020
  ...
```

I've set up IPFS using instructions for my Linux distro. One thing that I found lacking is a good way to test a running IPFS gateway. Here is a one-liner that helps with that:
```bash
$ ipfs cat bafybeiemxf5abjwjbikoz4mc3a3dla6ual3jsgpdr4cjr3oz3evfyavhwq/wiki/Beatles.html | grep h1 | sed -e 's/.*>\(.*\)<.*/\1/'
```
It retrieves a page from the IPFS Wikipedia mirror and extracts its title. The output should read `The Beatles`.

### Graph Node
The source code of the Graph node can be found at [github.com/graphprotocol/graph-node](https://github.com/graphprotocol/graph-node). Aside from IPFS, the node requires a database and an Ethereum endpoint to run. I began by making a database, this time without containerization - as described in the repo readme:

```bash
$ cd ..
$ initdb -D .postgres
$ pg_ctl -D .postgres -l logfile start
$ createdb graph-node
```

This produces a new `.postgres` folder, starts a PostgreSQL daemon, and creates a new database called `graph-node`. I also created a new database superuser called `subgraph`:

```bash
$ psql -U <my system login> -d graph-node
psql (14.5)
Type "help" for help.

graph-node=# CREATE ROLE subgraph WITH LOGIN;
graph-node=# \password subgraph
...
graph-node=# ALTER USER subgraph WITH SUPERUSER;
```

Building `graph-node` is straightforward:

```bash
$ git clone https://github.com/graphprotocol/graph-node
$ cd graph-node
$ cargo build
```

The last command requires `clang`, a somewhat unusual build dependency. Once I got it installed the build process finished successfully.

### Ethereum node

To run, the node needs access to an Ethereum RPC endpoint:

```bash
$ cargo run -p graph-node --release -- \
  --postgres-url postgresql://subgraph:<subgraph db user password>@localhost:5432/graph-node \
  --ethereum-rpc mainnet:<ethereum rpc endpoint url> \
  --ipfs 127.0.0.1:5001
```

Only HTTPS and local RPC are supported. Again, I didn't have a private node, so I looked up a free cloud service I could use.
<details>
	<summary>Alchemy ended up working.</summary>

> I began with Infura and it worked, but its daily limit of 100k requests was exhausted just minutes after deploying the subgraph. The next thing I tried was [Alchemy](https://alchemyapi.io/) with a generous 300M-free-requests-per-month limit. It worked.

</details>

Immediately after I got the node up it began syncing. This pre-deployment sync took just a few minutes, but sent about 30 to 50k requests to the Ethereum endpoint.

At this point I had everything in place to deploy the Uniswap subgraph. In a new terminal I navigated to the `v3-uniswap` folder and renamed the subgraph at `package.json` to avoid name collisions:

```bash
$ cd ../v3-uniswap
$ nano package.json
...
- "create-local": "graph create ianlapham/uniswap-v3 --node http://127.0.0.1:8020",
- "deploy-local": "graph deploy ianlapham/uniswap-v3 --debug --ipfs http://localhost:5001 --node http://127.0.0.1:8020",
+ "create-local": "graph create abernatskiy/uniswap-v3 --node http://127.0.0.1:8020",
+ "deploy-local": "graph deploy abernatskiy/uniswap-v3 --debug --ipfs http://localhost:5001 --node http://127.0.0.1:8020
...
```

Running

```bash
$ npm run create-local
$ npm run deploy-local
```

### Grafting

I got `subgraph validation error: [the graft base is invalid: deployment not found: QmPrb5mvZj3ycUugZgwLWCvK93jfXfhvfjRXrFk4tRmyCX]`. That is how I learned that some subgraphs can reuse the data of previously deployed subgraphs in a procedure called *grafting*. In case of the uniswap v3 subgraph it attempted to reuse the data of some subgraph known as `QmPrb5mvZj3ycUugZgwLWCvK93jfXfhvfjRXrFk4tRmyCX` on IPFS, up to a certain block. Grafting is [not recommended for use in production](https://thegraph.com/docs/en/developing/creating-a-subgraph/#grafting-onto-existing-subgraphs). I assumed that the subgraph should be capable of doing the indexing from scratch and removed the `graft` section from `subgraph.yaml`:

```bash
$ nano subgraph.yaml
...
- graft:
-   base: QmPrb5mvZj3ycUugZgwLWCvK93jfXfhvfjRXrFk4tRmyCX
-   block: 14292820
...
```
then re-ran `npm run deploy-local` and got the subgraph to start syncing.

## Syncing performance

### Subsquid

Uniswap blinked into existence at block 12369621 of the Ethereum blockchain and the current block is 15932386. Consequently, both of our indexers have about 3.5 million blocks to go through in search of the relevant data.

The squid starts fast, offering a 235 blocks-per-second rate and the corresponding ETA of 4.2 hours, but within minutes the block rate dropped to about 40-50 blocks-per-second and hovers there indefinitely. It didn't take long for me to figure out why once I saw the RPC stats:

![Nodereal screenshot](/uniswap-v3-comparison/squid/noderealThrottle.png)

The number of RPC calls that the squid makes is less than a thousand per hour, which is exceptionally low as we'll see in a bit. However, each of these calls takes a whooping 3 seconds (!) on average. Assuming that syncing executes the calls synchronously, these calls account for at least a half of the total execution time.

So what causes the enormous response times? Throttling is unlikely to be the reason: according to [Nodereal documentation](https://docs.nodereal.io/docs/cups-rate-limit), the API begins to return error responses if the requests consume too much resources. That would show up as a drop in success rate. The most likely explanation is that the squid just does a lot of work in each call, causing the RPC endpoint to take its time before answering.

The resulting estimated time to complete the sync is around 22 hours. With a more powerful node serving RPC it should be possible to slash that time in half, at the very least. Developers of the squid report syncing in under four hours.

### The Graph

In contrast, the subgraph proceeds much slower. At the time of this writing it's been running for 48 hours and processed just about 45 thousands blocks, resulting in a roughly 0.27 block-per-second rate. This is just about three times faster than the current block production rate of one block every 12 seconds. If I leave the subgraph syncing it will complete in about seven to eight months, assuming a constant block production rate by the network.

This figure is for syncing from scratch, which is not what the subgraph devs intended: a graft was provided to speed up the process for blocks up to 14292820. But even if I enabled the graft and its data transferred over IPFS instantaneously, syncing would still take about three and a half months.

Admittedly, the machine I used to run the subgraph is rather slow by modern standards. It is an Intel Xeon X3330 running at 2.66GHz with a SATA-connected consumer-grade SSD. It also runs a syncing Ethereum node, which takes up a substantial portion of IO bandwidth of the said SSD. However, neither CPU usage (as reported by `htop` with hyperthreading disabled) not disk IO utilization (as reported by `iostat -x 1`) are stuck at 100%, suggesting that network IO is at least partially responsible for the slow sync rate.

Looking at the RPC endpoint stats two things jump out: the huge number of requests and how lightweight they are.
![Alchemy screenshot](/uniswap-v3-comparison/subgraph/alchemyScreenshot.png)
The subgraph needed about 3M requests to process 45k blocks, with the median response time just below 10 ms. Distributions of server response times tend to have a positive [skew](https://en.wikipedia.org/wiki/Skewness), so it is fairly safe to assume that the mean response time is at least as long as the median. This places the estimate of IO wait time roughly at 8 hours, or 17% of all execution time. And that is a lower estimate: given that the subgraph talks to the RPC endpoint in a lot of short HTTPS requests the actual figure might be several times greater. The speedup that can be achieved by reducing these wait times is certainly substantial, but I can't estimate it until my Ethereum node finishes syncing.

## Corners that were and weren't cut

The difference in sync performance between the two frameworks is, on the surface, drastic. Comparing the average block rates we get that the squid syncs about 200 times faster. That translates into spending hours instead of weeks on a sync, a crucial advantage for rapid development.

Does that mean that every developer should choose Subsquid over The Graph to avoid the syncing hell? Not necessarily.

First, the figures for syncing rates I obtained are uncertain. It very well might be that the subgraph would sync 10-20 times faster with a more appropriate setup. Still, it is hard to imagine that any speedup will close the gap of two orders of magnitude between the frameworks. It is very likely that even at its best the subgraph would be at least ten times slower than the squid.

It is, however, important to keep in mind that while these longer sync times will impact the development, they will not have a drastic effect on scalability once the indexer is deployed. The data scraped from the blockchain is shared between the nodes running the subgraph and ideally the full sync only runs once. Reusing data between the deployments is also possible to some extent through the grafting mechanism.

So the only clear disadvantage of The Graph due to its long sync times has to do with development. And while this is not insignificant, some developers may find it an appropriate concession for gaining access to the free market-based scaling mechanisms of The Graph. After all, this is a "fire-and-forget" type of solution. These are as attractive as they are effective, and the market-based scaling mechanism of The Graph has certainly demonstrated that it is effective in some applications, by surviving its use by Uniswap if nothing else.

However, there are certain disadvantages to this approach. Free markets are prone to volatility, so if you paid X gwei for your API endpoints today there is no guarantee that tomorrow you won't have to pay 2X for the same number of requests. Another issue is latency arising from the reactive nature of market feedback. If the number of API requests grows rapidly, the network of indexers will need some time to adjust. This might work fine for a decentralized exchange, but it is a whole different story if you're selling football tickets.

For developers who want a more fine-grained control of their infrastructure, Subsquid is a superior solution. It also seems to offer superior syncing performance, which helps with development. 
One could ask if does it retain its performance as it scales, but making more squids and distributing the load among them is actually trivial. Similar to TheGraph, it suffices to sync a squid once, and then it can easily be cloned.

There is, however, one aspect of Subsquid's architecture that might raise eyebrows - archives. In the words of Subsquid lead developer Eldar Gabdullin:

> An Archive is a service which ingests blockchain data and makes it easy to access and query. 
>
> For example, from the whole chain one might be interested only in a particular call to a particular contract. 
>
> The Archive service allows to quickly and efficiently select only the relevant parts, rather than slowly grinding through each block via direct RPC calls to blockchain nodes.

Use of archives is a wonderful architectural solution performance-wise. The removal of the duplicated effort makes Subsquid indexers a much more efficient network in terms of computation per request served. It is the key to achieving the superior syncing performance I observed. But it also means that indexers depend on another, potentially centralized service.

With most archives right now developers have a choice of either using free archives provided by the Subsquid team or running their own archive. Archives are open-source and can be easily set up, but they do require some time to sync and some computational resources to service squids. For developers seeking to control their entire off-chain infrastructure these might be of concern.

I originally contacted Eldar to get a better idea of what running an Ethereum archive looks like. According to him, sync performance of this yet-to-be-unveiled piece of software is mainly limited by the performance of the Ethereum RPC endpoint and its connection to the archive. His setup with a single Ethereum node takes about 48 hours to sync. Interestingly, the archive is capable of utilizing multiple nodes in parallel with sync time reducing linearly. Once the archive is done synchronizing it will require one CPU and 300Mb of RAM for every synchronizing squid it serves.

I did not ask Eldar about how much archive time each synchronized squid requires to maintain synchronization, but judging by the fact that Subsquid is still able to maintain its army of archives - not much. A single archive should suffice for most projects, even those on the bigger side.

Currently, Subsquid archives are undergoing architectural streamlining, with the end goal of them being able to share the scraped data over IPFS. This will further simplify their deployment and use.

## Conclusions

The very obvious main lesson of this exercise is to **bring your own node**. Useful indexers typically have to go through a lot of blockchain data while syncing, and that implies heavy usage of the RPC endpoint. Endpoints provided by free cloud services are not up to the task.

There is, however, a significant difference in how the two frameworks use this resource. The squid I tested sent a small number of extremely computationally intensive requests, while the subgraph spammed lightweight requests at a very high rate. Consequently, preferred hardware depends on the choice of framework.

The uniswap-v3 squid would sync well using an RPC endpoint running on a separate machine, as long as it is capable of handling heavy requests quickly. Additional studies are required to figure out exactly what it would take, but my guess would be that a powerful CPU is a must. Another potential bottleneck to consider is the IO bandwidth of the disk where the RPC node keeps its chain data. It is, however, unclear whether the pattern of "heavy requests in low quantity" would apply to other squids. I might look into this in future posts.

To achieve reasonable sync times on the Uniswap-v3 subgraph, one would need to run both the RPC node and the Graph node on the same machine and have them communicate over a loopback interface. The Graph's documentation calls this ["the speediest way to get mainnet or testnet data"](https://github.com/graphprotocol/graph-node/blob/master/docs/getting-started.md#232-local-geth-or-parity-node), so it probably applies to other subgraphs. This exercise does not clarify where the next bottleneck would be, but to play safe I would suggest a machine with 4+ cores and two separate SSDs - one for blockchain data and one for the database used by the Graph indexer node.

As for what to choose when building a scalable application, it really boils down to whether you prefer the market-based scaling system of the Graph or a more straightforward approach offered by Subsquid. But beware - if you do choose the Graph your initial sync times will be high. Plan accordingly.

[^1]: Support for HTTPS ETH endpoints has been added to `@subsquid/evm-processor` package since I wrote this. If did this again, I could simply upgrade this dependency and use any of the free services I mention as incompatible. However, the squid still makes extremely heavy RPC calls and many services will refuse to process these. I successfully used the aforementioned Nodereal and, with less stability, Alchemy.
