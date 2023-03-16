---
title: "Scaling Magic Eden: Part 2"
seoTitle: "Scaling Magic Eden (Part 2)"
seoDescription: "How Magic Eden scaled its marketplace on Solana using Geyser, Kafka, Redis, and AWS Aurora (Part 2)"
datePublished: Thu Mar 16 2023 01:30:52 GMT+0000 (Coordinated Universal Time)
cuid: clfafq1h7000209lb4aqv4s8h
slug: scaling-magic-eden-part-2
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1678930072453/1f44460f-7f26-4a25-9801-af4eb29f65fe.jpeg
tags: solana, nft, geyser

---

In the [last post](https://eng.magiceden.dev/scaling-magic-eden-part-1), we discussed motivations and options for building your own indexer for Solana

[Geyser](https://docs.solana.com/developing/plugins/geyser-plugins) has emerged as a popular mechanism for indexing on Solana, and has dramatically improved our site reliability and speed - becoming almost 30x faster from chain to site!

### **Solana Validators**

Solana validators are nodes in the network responsible for verifying transactions and maintaining the blockchain's integrity. They perform critical functions such as participating in the network's consensus algorithm, providing RPC (Remote Procedure Call) endpoints, and staking SOL tokens to secure the network. Validators play a crucial role in ensuring the network's stability and security by continuously monitoring the blockchain and ensuring that it remains in consensus.

Solana validators run on beefy hardware to keep up with the network’s high throughput. They require at least 12 CPU cores, 128GB of RAM, and an SSD hard drive with a minimum capacity of 1.5TB, to ensure fast transaction processing and participation in the consensus algorithm. These requirements contribute to the platform's high-performance capabilities and ability to handle a large volume of transactions efficiently.

In comparison, the typical hardware requirements for an Ethereum node are much lower, with a minimum of 4 CPU cores, 4-8GB of RAM, and an SSD hard drive with a minimum capacity of 256 GB. However, the exact specifications can vary depending on the specific use case and the network's current state.

### **What is Geyser?**

Geyser, [introduced in September 2021](https://github.com/solana-labs/solana/pull/20047), is a validator plugin mechanism through which the information about accounts, slots, blocks, and transactions can be transmitted to an external sink. Common sinks include external data stores such as relational databases, NoSQL databases or Kafka.

While it requires running your own validator, the Geyser interface allows you to open up the fire-hose of all updates on Solana.

This beats fetching data from RPC by orders of magnitude, in part because there is no need for egress from your backend’s network! Geyser enables you to dump updates wherever they need to go, without the unnecessary network overhead.

Account updates are of particular interest to us, because they are not performantly accessible via RPC. The [Geyser interface](https://docs.rs/solana-geyser-plugin-interface/latest/solana_geyser_plugin_interface/geyser_plugin_interface/trait.GeyserPlugin.html#method.update_slot_status) includes a method you can implement called ***update\_account***. This method is called for each updated account, along with metadata like slot and whether the update came from the validator’s snapshot, or a recent on-chain update.

```rust
 fn update_account(
        &mut self,
        account: ReplicaAccountInfoVersions,
        slot: u64,
        is_startup: bool,
    ) -> Result<()>
```

### **Geyser Considerations**

**Running a Validator**

Running your own validator is the most operationally complex part of setting up Geyser-based indexing. It’s gotten easier over the years, with a [strong community of validators on Discord](https://discord.com/channels/428295358100013066/837340113067049050) and increasingly stable releases from the Solana Foundation.

As we discussed, running a full voting Solana Validator [requires some hefty hardware](https://docs.solana.com/running-validator/validator-reqs). If you’d like to operate your validator as an RPC node (serving read/write requests to the network), bump those specs to 256GB RAM with 16 cores. Most dApp teams are not in a position to conveniently run bare-metal, and the network cost to pipe data can be exorbitant.

Fortunately for us - Geyser-only validators have lower hardware requirements compared to voting or RPC nodes, which makes them more manageable to operate on cloud computing platforms such as AWS or GCP.

We’ll go over our validator setup in detail in a later post.

**Handling Throughput**

Ingesting all accounts provided through *update\_account* would result in thousands of updates per second. Scaling a data sink to this amount of throughput is unnecessary for most dApps, as most only care about a small subset of on-chain programs. Implementing a set of account filters in your plugin is recommended to reduce your egress.

The Solana Labs Postgres Plugin has an [AccountSelector](https://github.com/solana-labs/solana-accountsdb-plugin-postgres) which provides a great example of flexible account filtering:

![](https://lh6.googleusercontent.com/Dx8qvcs8OWWUr4h72E25nqmu3PGCIaOvvl1PRUhsu9xwesHtq3c4_KWM7cj0JuCB2K59Tvhqp35ZSH1FSLveB5WMu9Q85QTlu73M-8vPW9DytkLhcAhyYLzvASdBEbKDIr1HebYvdRdyL6_R50Hcxik align="left")

It's worth noting that the configuration is loaded during validator startup, and any changes to the configuration require a restart of the validator. This restart can take up to an hour, which can be inconvenient if the configuration is updated frequently. To avoid this downtime and maintain flexibility with your configuration, incorporating configuration hot-reload into your plugin is recommended.

Some implementations (like [Holaplex’s plugin](https://github.com/holaplex/indexer-geyser-plugin)) provide additional flexibility by fetching detailed configuration (like a [token whitelist](https://raw.githubusercontent.com/solana-labs/token-list/main/src/tokens/solana.tokenlist.json)) from an external server at startup.

**Commitment Status**

Solana has 3 states for a slot:

*Processed -* The highest slot of the heaviest fork processed by the node. Ledger state at this slot is not derived from a confirmed or finalized block, but if multiple forks are present, is from the fork the validator believes is most likely to finalize.

*Confirmed -* Block has received a super-majority of ledger votes.

*Finalized (Rooted) -* Block has become a root of the chain, meaning all branches on the network are descended from this block.

![](https://lh6.googleusercontent.com/76oqY4BMnTBJStfonj2Tm9eS3bclWHHCltNFvdAGBnwbhKxL3AMJDab7gQXRno30pqz6vxzU7smJu8MTWIpbjmrOS6OuPeGRZtr-g3q2FlwIWaD6Kwhgwp4kiv9oslNDTK48JXK710P2YCTQAaMAnNI align="left")

Geyser emits account and transaction updates as soon as they are processed, which is beneficial for end-to-end indexing speed. However, there is a small chance (5-10%) that a processed slot may be [skipped](https://docs.solana.com/terminology#skipped-slot), so it's essential for data sinks to be aware of this and handle it appropriately.

There's a trade-off to consider here - waiting for a block to be finalized before displaying the updated state to a user can result in a delay of up to 30 seconds behind the chain, while displaying new state at confirmed may only delay it by a few seconds once a transaction has been submitted to the network, with a negligible chance of being reverted.

![](https://lh3.googleusercontent.com/11cvH0ceT35hC0ZcLO8evadVYwXUSsxff0Or-pJTpK5CFnJYyNN-8Z58vWH5e1ew0U98uj-xdtSa-eLtW3x-LeHKPKZ_Nd8pCXdbWU5oUcpYzf5gE5hzuOghjppmeNZgt7tPYMp4pFkYoc33xkt2NgU align="left")

We’ll go over our chosen solution for handling commitment status in a following post.

**Transactions Handling Considerations**

In addition to the [*account\_update*](https://docs.rs/solana-geyser-plugin-interface/latest/solana_geyser_plugin_interface/geyser_plugin_interface/trait.GeyserPlugin.html#method.update_account) method, Geyser also provides a [*notify\_transaction*](https://docs.rs/solana-geyser-plugin-interface/latest/solana_geyser_plugin_interface/geyser_plugin_interface/trait.GeyserPlugin.html#method.notify_transaction), which similarly signals a new processed transaction. This can be quite useful if you’d like to fully remove your dependencies from RPC providers - with account state and transactions, you have all the information you need to power any read-dApp you’d need to build (beyond actually broadcasting transactions to the network).

This may not be as critical as ingesting account state from Geyser, however, since your backend can use any RPC provider’s [Subscription Websocket API](https://docs.solana.com/api/websocket#blocksubscribe) to subscribe for new blocks, without placing additional reliability concerns/load on your account-indexing infrastructure.

### **Popular OSS Geyser Plugins**

[Solana Labs Postgres Plugin](https://github.com/solana-labs/solana-accountsdb-plugin-postgres)

This plugin was the original proof-of-concept built by the Solana Labs team and writes transactions, account, block, and slot updates directly to Postgres.

It showcases some good examples of filtering down transactions/accounts based on configuration mounted to the validator. There’s also functionality to store [historic account data](https://github.com/solana-labs/solana-accountsdb-plugin-postgres#capture-historical-account-data) in Postgres, allowing for a look back on older values of accounts.

[Blockdaemon Kafka Plugin](https://github.com/Blockdaemon/solana-accountsdb-plugin-kafka)

This plugin is a great example of writing account updates to Kafka. It uses the [librdkafka](https://github.com/confluentinc/librdkafka) library to [buffer Kafka events](https://github.com/Blockdaemon/solana-accountsdb-plugin-kafka#buffering), which can be quite useful when Kafka is experiencing temporary issues.

Writing to a message broker like Kafka decouples your validator from your indexers, allowing for:

1. Better performance - Geyser will not be blocked by how quickly we can index updates
    
2. Increase Reliability - Message consumer can handle database failures downstream
    
3. Throttling - Can smooth out large bursts of messages in times of high TPS load
    

[Mango GRPC Stream](https://github.com/ckamm/solana-accountsdb-connector)

One of the more mature open-source plugins. This runs a GRPC server and streams updates to any consumer that connects.

It also comes with stateless “connectors” that will startup and subscribe for updates from Geyser based on connector-specific configuration, deserialize account data and write to a table of your choice.

[Holaplex RabbitMq](https://github.com/holaplex/indexer-geyser-plugin)

Another great example of writing to a message broker.

This comes with additional flavors of filters for accounts like owners, account public keys, token mint addresses, and validator startup messages (which make it easy to backfill a new set of accounts to your database)

**Conclusion**

In conclusion - Geyser is a powerful tool for your indexing infrastructure to allow your dApps to be as fast and performant as possible. In our next post - we’ll dive deeper into how we integrated Geyser into our backend.