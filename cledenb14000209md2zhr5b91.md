# Scaling Magic Eden: Part 1

As the leading cross-chain NFT platform for Solana, Ethereum, and Polygon, Magic Eden handles a significant amount of traffic daily, both through our core website as well as our [API that we expose to the community](https://api.magiceden.dev/). In 2021, Magic Eden processed over 1.5x more transactions than [OpenSea](https://opensea.io/), with each tx reflecting on the site in a matter of 1-3 seconds.

We recently underwent a redesign of our Solana indexing system this year and are proud to make the following improvements:

1. 10x faster wallet and collections page loads
    
2. 10x faster e2e latency (duration from a user seeing a transaction succeed to when our site reflects this information)
    
3. Significant cost savings
    

### Wins

Our Wallet page saw one of the biggest speedups getting over 10x faster

![](https://lh3.googleusercontent.com/7qBzFCiUx7jGji0DP6V_dcGfCwzFFg97AO-vVy9Wp6Tw13vOZH-kkvVNTFu_wfDFleD4Sq_SOtY9mRDVqTA1JNRh92LbCb5-NDzSeXjGcBiiwB7drTwdLnQ4FT4WuHUln5Yfo5mibnCM3-Pn-7LefvQ align="left")

Overall p99 of our API across all endpoints improved dramatically, tying up fewer server resources

![](https://lh4.googleusercontent.com/Do0ztxvwTnb9lnpbh8kuZ0d-7AgpPze7gSdhf-jt5_nGw5-2-jFQv1Wos3sUUqAQFrPg7ljILiPcM4vYWYYDbm3v7GD1N_l_ImauxLWw_N2yBpRu9QV1Ki0iWBIdV-Qm7WBSQcIGkAifpmzfi_JwE28 align="left")

The time for our internal systems to reflect the most recent state of Solana improved 30x, usually below 2s end-to-end:

![](https://lh5.googleusercontent.com/XA_7Z-4lQSPFRhq2uLM2PzE0W8SB0MIuSkTkqp5P7bsnGU-40uY4B-vn1hFYAImieHAHlCJUYkiLSbOId8IBUPCmZ_onV0Nhby4cXx6C63oaN9Hmj4H5aYsPPHY9c0ylp8AN-9uNnGEa__YBu86pG_g align="left")

Recently, we’ve hit up to 4000 queries per second across all of our APIs. Our APIs for fetching NFTs for specific collections hit 100qps and 500qps at peak. Our system handles 300 transactions a second at peak, with transactions reflecting on the site from the blockchain within seconds.

We’ll go into the details of the architecture behind these improvements in subsequent posts, but let’s set the table on Solana development.

### **Web3 Development**

*How Web3 development differs from traditional web development*

There are plenty of in-depth descriptions of the difference between a traditional web app and a dApp (an application built on top of the blockchain). Some of my favorites include [Web2 vs Web3 by Drift](https://www.drift.trade/blog/web2-vs-web3),  [Web2 vs Web3 by the Ethereum Org](https://ethereum.org/en/developers/docs/web2-vs-web3/)**,** and [Architecture of a Web3 App](https://www.preethikasireddy.com/post/the-architecture-of-a-web-3-0-application).

For now, we’ll focus on the largest difference: the *application data source*.

In a traditional web application, the core data for the application will be stored in a database managed by the application developer. The application developer has full control over the data modeling, table schemas, and workloads. The application developer can decide the choice of database, and optimizations like indexing, normalization, and caching based on the workloads they expect their app to take. All applications reads and writes pass through the same stack and database.

In web3 - that core data is stored in a decentralized, globally available, permissionless database called a blockchain. All application writes that are initiated by the user go to the blockchain, not the developer’s database. The developer can deploy new programs to that blockchain, but the fundamental properties of being distributed and general-purpose can bog down a vanilla web app without additional optimization.

To recap:

![](https://lh4.googleusercontent.com/7pDW1QkFZNx4Y6XYWXUBOnrrhW0dNwexaELFZeSAF-eNvMuswh3YkmX27svEfkOTaDFZn1yX62AuZ2D3DyJXWpmtNwESpL6CTZamQ1GUQ33ECVb79yQVLS1-kORwd-JWkPsCSxsd7AXNpSqvoLDBs54 align="left")

<table><tbody><tr><td colspan="1" rowspan="1"><p></p></td><td colspan="1" rowspan="1"><p><strong>Reads</strong>&nbsp;</p></td><td colspan="1" rowspan="1"><p><strong>Writes</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p><strong>WebApp</strong></p></td><td colspan="1" rowspan="1"><p>App Database&nbsp;</p></td><td colspan="1" rowspan="1"><p>App Database&nbsp;</p></td></tr><tr><td colspan="1" rowspan="1"><p><strong>dApp (no backend)&nbsp;</strong></p></td><td colspan="1" rowspan="1"><p>Blockchain <a target="_blank" rel="noopener noreferrer nofollow" href="https://www.alchemy.com/overviews/rpc-node" style="pointer-events: none">RPC Provider</a></p></td><td colspan="1" rowspan="1"><p>Blockchain <a target="_blank" rel="noopener noreferrer nofollow" href="https://www.alchemy.com/overviews/rpc-node" style="pointer-events: none">RPC Provider</a></p></td></tr><tr><td colspan="1" rowspan="1"><p><strong>dApp (custom backend)</strong>&nbsp;</p></td><td colspan="1" rowspan="1"><p>App Server</p></td><td colspan="1" rowspan="1"><p>Blockchain <a target="_blank" rel="noopener noreferrer nofollow" href="https://www.alchemy.com/overviews/rpc-node" style="pointer-events: none">RPC Provider</a></p></td></tr></tbody></table>

**Example Web3 dApp**

Let’s say you want to build a dApp that allows users to stake Solana with your program.

This will involve setting up your frontend - typical HTML, CSS, Javascript, potentially with a frontend framework like React. You’d then write your smart contract ([Anchor](https://www.anchor-lang.com/) is popular) and then deploy it to the chain. The final step is to integrate your app to write and read from the Solana blockchain via [web3js](https://docs.solana.com/developing/clients/javascript-api), and deploy!

At this point, you’ve managed to build your dApp, get it deployed, and start serving users! You didn’t even need to deploy a backend server. RPC providers, backed by the blockchain, handle your read/write traffic and get you functional quickly.

Things are going well, but as you scale your user base, you hit a wall with performance. It becomes challenging to add new analytical features like a leaderboard for most prolific stakers. You check the network log and notice that [*getProgramAccounts*](https://docs.solana.com/developing/clients/jsonrpc-api#getprogramaccounts) takes an incredibly long time to fetch the [token accounts](https://lorisleiva.com/owning-digital-assets-in-solana/how-nfts-are-represented-in-solana#mints-and-tokens) relevant to the users on your site!

**Solana Storage Model**

Solana is a write-optimized chain, meaning it is designed to prioritize the speed and efficiency of transactions - write operations that modify the state of the blockchain. Solana achieves this by using a unique proof-of-history (PoH) consensus algorithm, which allows it to process thousands of transactions per second. Its core design principles are performance, scalability, and reliability.

Solana’s [account model](https://solanacookbook.com/core-concepts/accounts.html#account-model) is fairly simple - optimized for fast reads by account and owner addresses. Most information for each account is stored in a *data* field as a serialized byte array:

![](https://lh3.googleusercontent.com/e7W6uK9Jl7HqozSbwJVqIVxPdD4bhEySSh7_XC9SoBzsJWWXeMyHqiYu8NjmxU9D8p8n6hhzWQ-6BJ-mosa9-RMY0_WHH9YqC3PwL2KEskUq69qmWJB1xxI7qFvnGCDCh1Rp2CEG6n45_PMndqB8a2M align="left")

This storage makes it difficult to query for any data contained by each account since the data is serialized and unindexed!

So if we’d like to execute a query for all tokens a user owns, we’d need to query *getProgramAccounts* for the [Token Program](https://pencilflip.medium.com/solanas-token-program-explained-de0ddce29714), filtering down for tokens owned by the user’s wallet address. This sounds convenient, but the underlying database does not index the Token owner field, because it is in the serialized byte array *data* field. This means we must scan over all accounts owned by the token program, an O(n) scan! And N is in the hundreds of millions in this case.

Many other common Solana use cases rely on this call pattern:

1. Enumerate all NFTs in a collection
    
2. Get all listings on Magic Eden for a specific user
    
3. Find the rarest NFT in a user’s wallet
    
    ![](https://lh6.googleusercontent.com/s6zb4hyOeW0W8HymYosNGMpitcl0c7m61UAUCNjdiub5Oa8SgOKeUPBuoWSV1w-Fs52wtDqDih-INat4o3FQSi47q76zZlzzwQQ61W8wAFdqtlH5VelFry4evEyjCeczvCPy-eI49Y5l_BE6khqwJMU align="left")
    

There’s room for optimization here - accounts should be indexed in a way specific to the schemas of the serialized data they contain. Solana’s RocksDB does not do this for efficiency/abstraction - it’s reminiscent of how blocks are laid out on disc in a typical file system.

**Custom Indexing of Solana data**

So we’re now at the point in our web3 dApp when we want to bootstrap a backend database that will store and index data in a deserialized format. This database will allow our application to quickly find the data we want for our front end. Going back to our need to fetch all tokens that a user has staked, we’ll now have a new endpoint:

```sql
GET /app/user_wallet/:wallet_address/staked
```

Which will hit our database w/ a query similar to

```sql
SELECT token_amount 
FROM app_stakes 
WHERE wallet = :wallet_address
```

Since we’ve indexed the *wallet* column of the *app\_stakes* table, the lookup becomes O(1)! We no longer need to wait for our target RPC provider to iterate through all token accounts to find our user’s stake!

### **Other Self-Indexing Motivations**

There are other reasons we may want to index on-chain data ourselves:

**Denormalization**

A common pattern in database design is [denormalization](https://www.techtarget.com/searchdatamanagement/definition/denormalization) - increasing redundancy of our stored data to avoid joins and provide faster, specialized reads for various use cases. On Solana - certain entities we’re interested in tracking may have multiple accounts that constitute a single entity.

Let’s take an NFT for example - NFTs on Solana are constituted by a MintAccount, TokenAccount, a Metadata account, and additional off-chain data.

[![](https://lh6.googleusercontent.com/yyKowk46Ufvzpanz6kxpLgILrz7zAadLGCkz3CUv_iBLuReSwlXEu1nr7G-1g3OyVetKxeAr_Nq_91ogGVUpkmCSqTnmVYo1TRYf74UAqzx4bNyGcx4neDfivg8dcbuffIVQM2oTB1ZpPmpYstTEDEg align="left")](https://www.google.com/url?q=https://lorisleiva.com/owning-digital-assets-in-solana/how-nfts-are-represented-in-solana%23nonfungible&sa=D&source=docs&ust=1676935356806798&usg=AOvVaw21PSzVZxVH-zHZhnE-R1Xg)

Fetching all of this data via RPC in a front end would be arduous. Indexing all accounts in our DB as separate tables means reads are still not as efficient as possible, since we require joins. Denormalizing this info into a single “NFT” table makes reads faster and more efficient.

**Denormalization**

Indexing data ourselves allows us to support custom aggregations. Both [OLTP](https://en.wikipedia.org/wiki/Online_transaction_processing) (like Postgres, and Cockroach)  and [OLAP](https://en.wikipedia.org/wiki/Online_analytical_processing) (like Pinot) databases could be great options here to allow fast reads for aggregations, like the token-staker leaderboard:

```sql
SELECT wallet, SUM(*) as stakedTokens
FROM app_stakes
GROUP BY wallet
ORDER BY stakedTokens  
LIMIT 10
```

**Indexing via RPC**

A common practice in the Solana ecosystem to maintain a database reflecting on-chain data is by utilizing RPC nodes to backfill the entities we’re interested in, then process transactions for those entities to pick up new mutations as they come in.

Solana RPC nodes come with a [Subscription WebSocket API](https://docs.solana.com/developing/clients/jsonrpc-api#subscription-websocket) which consumers can subscribe for updates to various updates.

We can use the [slotSubscribe](https://docs.solana.com/developing/clients/jsonrpc-api#slotsubscribe) web socket to get updated when the network has produced new blocks, then [getBlock](https://docs.solana.com/developing/clients/jsonrpc-api#getblock) to retrieve all transactions in the block for that slot, filter down to the transactions for accounts we’re interested in, then fetch all recent account data for the accounts that we’ve observed to have changed.

![](https://lh4.googleusercontent.com/oW8UrH-mR4ST794nh8QDqDT7pyQlG6DlhuY5oYWzYuBT2HIxKIRXf1FwPnReYWfoNJYfhmL49aXghefFCopkAhKjYSQtyIAbLn6IXW6WCwinEbcPEd-kOuGQnCiopwbUmWwxlAcoVqJbhpDvYPZ5apI align="left")

This method works reasonably well and is extensible to new transaction types and new account types we may see.

However, there can be some drawbacks if we’re interested in knowing the up-to-date state of accounts on-chain...

**RPC Performance**

The same performance issues we were having in our dApp with RPC calls like *getProgramAccounts* are still happening - we’ve just moved our RPC call site from the frontend to our backend. We’ve unlocked efficient reads for our users, but all data must travel over the wire, and we’re bound to the same RPC interface on the indexing path as we were calling before.

Backfills still require *getProgramAccounts*, and account updates must be fetched.

Imagine streaming 100 GB of data in HTTP calls that return 100kb at a time - this is what’s happening when fetching (and backfilling) account data from RPC

In addition, RPC nodes generally serve multiple functions beyond read traffic. Many RPC nodes submit client transactions to the network and participate in consensus. These supplemental responsibilities slow down RPC nodes and stop them from excelling at serving read traffic, and can end up falling behind the network.

**Pagination**

RPC nodes don’t support pagination on most endpoints. This means that APIs have limited scalability as the underlying data size grows, making endpoints slower to load and more resource intensive.

### **Enter Geyser**

This is where [Geyser](https://docs.solana.com/developing/plugins/geyser-plugins) - a validator plugin for data streaming - comes in. We’ll discuss Geyser details, the motivations behind integrating it into your stack, and some popular plugins in the next post.