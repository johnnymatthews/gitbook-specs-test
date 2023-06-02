# Filecoin nodes

This section starts by discussing the concept of Filecoin Nodes. Although different node types in the Lotus implementation of Filecoin are less strictly defined than in other blockchain networks, there are different properties and features that different types of nodes should implement. In short, nodes are defined based on the set of services they provide.

In this section we also discuss issues related to storage of system files in Filecoin nodes. Note that by storage in this section we do not refer to the storage that a node commits for mining in the network, but rather the local storage repositories that it needs to have available for keys and IPLD data among other things.

In this section we are also discussing the network interface and how nodes find and connect with each other, how they interact and propagate messages using libp2p, as well as how to set the node’s clock.

## Node Types

Nodes in the Filecoin network are primarily identified in terms of the services they provide. The type of node, therefore, depends on which services a node provides. A basic set of services in the Filecoin network include:

* chain verification
* storage market client
* storage market provider
* retrieval market client
* retrieval market provider
* storage mining

Any node participating in the Filecoin network should provide the _chain verification_ service as a minimum. Depending on which extra services a node provides on top of chain verification, it gets the corresponding functionality and Node Type “label”.

Nodes can be realized with a repository (directory) in the host in a one-to-one relationship - that is, one repo belongs to a single node. That said, one host can implement multiple Filecoin nodes by having the corresponding repositories.

A Filecoin implementation can support the following subsystems, or types of nodes:

* **Chain Verifier Node:** this is the minimum functionality that a node needs to have in order to participate in the Filecoin network. This type of node cannot play an active role in the network, unless it implements **Client Node** functionality, described below. A Chain Verifier Node must synchronise the chain (ChainSync) when it first joins the network to reach current consensus. From then on, the node must constantly be fetching any addition to the chain (i.e., receive the latest blocks) and validate them to reach consensus state.
* **Client Node:** this type of node builds on top of the **Chain Verifier Node** and must be implemented by any application that is building on the Filecoin network. This can be thought of as the main infrastructure node (at least as far as interaction with the blockchain is concerned) of applications such as exchanges or decentralised storage applications building on Filecoin. The node should implement the _storage market and retrieval market client_ services. The client node should interact with the Storage and Retrieval Markets and be able to do Data Transfers through the Data Transfer Module.
* **Retrieval Miner Node:** this node type is extending the **Chain Verifier Node** to add _retrieval miner_ functionality, that is, participate in the retrieval market. As such, this node type needs to implement the _retrieval market provider_ service and be able to do Data Transfers through the Data Transfer Module.
* **Storage Miner Node:** this type of node must implement all of the required functionality for validating, creating and adding blocks to extend the blockchain. It should implement the chain verification, storage mining and storage market provider services and be able to do Data Transfers through the Data Transfer Module.

### **Node Interface**

The Lotus implementation of the Node Interface can be found [here](https://github.com/filecoin-project/lotus/blob/master/node/repo/interface.go).

### **Chain Verifier Node**

```go
type ChainVerifierNode interface {
  FilecoinNode

  systems.Blockchain
}
```

The Lotus implementation of the Chain Verifier Node can be found [here](https://github.com/filecoin-project/lotus/blob/master/node/impl/full.go).

### **Client Node**

```go
type ClientNode struct {
  FilecoinNode

  systems.Blockchain
  markets.StorageMarketClient
  markets.RetrievalMarketClient
  markets.DataTransfers
}
```

The Lotus implementation of the Client Node can be found [here](https://github.com/filecoin-project/lotus/blob/master/node/impl/client/client.go).

### **Storage Miner Node**

```go
type StorageMinerNode interface {
  FilecoinNode

  systems.Blockchain
  systems.Mining
  markets.StorageMarketProvider
  markets.DataTransfers
}
```

The Lotus implementation of the Storage Miner Node can be found [here](https://github.com/filecoin-project/lotus/blob/master/node/impl/storminer.go).

### **Retrieval Miner Node**

```go
type RetrievalMinerNode interface {
  FilecoinNode

  blockchain.Blockchain
  markets.RetrievalMarketProvider
  markets.DataTransfers
}
```

### **Relayer Node**

```go
type RelayerNode interface {
  FilecoinNode

  blockchain.MessagePool
}
```

### **Node Configuration**

The Lotus implementation of Filecoin Node configuration values can be found [here](https://github.com/filecoin-project/lotus/blob/master/node/config/def.go).

## Node repository

The Filecoin node repository is simply local storage for system and chain data. It is an abstraction of the data which any functional Filecoin node needs to store locally in order to run correctly.

The repository is accessible to the node’s systems and subsystems and can be compartmentalized from the node’s `FileStore`.

The repository stores the node’s keys, the IPLD data structures of stateful objects as well as the node configuration settings.

The Lotus implementation of the FileStore Repository can be found [here](https://github.com/filecoin-project/lotus/blob/master/node/repo/fsrepo.go).

### **Key Store**

The `Key Store` is a fundamental abstraction in any full Filecoin node used to store the keypairs associated with a given miner’s address (see actual definition further down) and distinct workers (should the miner choose to run multiple workers).

Node security depends in large part on keeping these keys secure. To that end we strongly recommend: 1) keeping keys separate from all subsystems, 2) using a separate key store to sign requests as required by other subsystems, and 3) keeping those keys that are not used as part of mining in cold storage.

Filecoin storage miners rely on three main components:

* **The storage miner **_**actor**_** address** is uniquely assigned to a given storage miner actor upon calling `registerMiner()` in the Storage Power Consensus Subsystem. In effect, the storage miner does not have an address itself, but is rather identified by the address of the actor it is tied to. This is a unique identifier for a given storage miner to which its power and other keys will be associated. The `actor value` specifies the address of an already created miner actor.
* **The owner keypair** is provided by the miner ahead of registration and its public key associated with the miner address. The owner keypair can be used to administer a miner and withdraw funds.
* **The worker keypair** is the public key associated with the storage miner actor address. It can be chosen and changed by the miner. The worker keypair is used to sign blocks and may also be used to sign other messages. It must be a BLS keypair given its use as part of the [Verifiable Random Function](https://spec.filecoin.io/#section-algorithms.crypto.vrf).

Multiple storage miner actors can share one owner public key or likewise a worker public key.

The process for changing the worker keypairs on-chain (i.e. the worker Key associated with a storage miner actor) is specified in [Storage Miner Actor](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.storage\_miner\_actor). Note that this is a two-step process. First, a miner stages a change by sending a message to the chain. Then, the miner confirms the key change after the randomness lookback time. Finally, the miner will begin signing blocks with the new key after an additional randomness lookback time. This delay exists to prevent adaptive key selection attacks.

Key security is of utmost importance in Filecoin, as is also the case with keys in every blockchain. **Failure to securely store and use keys or exposure of private keys to adversaries can result in the adversary having access to the miner’s funds.**

### **IPLD Store**

InterPlanetary Linked Data (IPLD) is a set of libraries which allow for the interoperability of content-addressed data structures across different distributed systems and protocols. It provides a fundamental ‘common language’ to primitive cryptographic hashing, enabling data structures to be verifiably referenced and retrieved between two independent protocols. For example, a user can reference an IPFS directory in an Ethereum transaction or smart contract.

The IPLD Store of a Filecoin Node is local storage for hash-linked data.

IPLD is fundamentally comprised of three layers:

* the Block Layer, which focuses on block formats and addressing, how blocks can advertise or self-describe their codec
* the Data Model Layer, which defines a set of required types that need to be included in any implementation - discussed in more detail below.
* the Schema Layer, which allows for extension of the Data Model to interact with more complex structures without the need for custom translation abstractions.

Further details about IPLD can be found in its [specification](https://github.com/ipld/specs).

#### **The Data Model**

At its core, IPLD defines a [Data Model](https://github.com/ipld/specs/blob/master/data-model-layer/data-model.md) for representing data. The Data Model is designed for practical implementation across a wide variety of programming languages, while maintaining usability for content-addressed data and a broad range of generalized tools that interact with that data.

The Data Model includes a range of standard primitive types (or “kinds”), such as booleans, integers, strings, nulls and byte arrays, as well as two recursive types: lists and maps. Because IPLD is designed for content-addressed data, it also includes a “link” primitive in its Data Model. In practice, links use the [CID](https://github.com/multiformats/cid) specification. IPLD data is organized into “blocks”, where a block is represented by the raw, encoded data and its content-address, or CID. Every content-addressable chunk of data can be represented as a block, and together, blocks can form a coherent graph, or [Merkle DAG](https://docs.ipfs.io/guides/concepts/merkle-dag/).

Applications interact with IPLD via the Data Model, and IPLD handles marshalling and unmarshalling via a suite of codecs. IPLD codecs may support the complete Data Model or part of the Data Model. Two codecs that support the complete Data Model are [DAG-CBOR](https://github.com/ipld/specs/blob/master/block-layer/codecs/dag-cbor.md) and [DAG-JSON](https://github.com/ipld/specs/blob/master/block-layer/codecs/dag-json.md). These codecs are respectively based on the CBOR and JSON serialization formats but include formalizations that allow them to encapsulate the IPLD Data Model (including its link type) and additional rules that create a strict mapping between any set of data and it’s respective content address (or hash digest). These rules include the mandating of particular ordering of keys when encoding maps, or the sizing of integer types when stored.

#### **IPLD in Filecoin**

IPLD is used in two ways in the Filecoin network:

* All system datastructures are stored using DAG-CBOR (an IPLD codec). DAG-CBOR is a more strict subset of CBOR with a predefined tagging scheme, designed for storage, retrieval and traversal of hash-linked data DAGs. As compared to CBOR, DAG-CBOR can guarantee determinism.
* Files and data stored on the Filecoin network are also stored using various IPLD codecs (not necessarily DAG-CBOR).

IPLD provides a consistent and coherent abstraction above data that allows Filecoin to build and interact with complex, multi-block data structures, such as HAMT and AMT. Filecoin uses the DAG-CBOR codec for the serialization and deserialization of its data structures and interacts with that data using the IPLD Data Model, upon which various tools are built. IPLD Selectors can also be used to address specific nodes within a linked data structure.

#### IPLD Stores

The Filecoin network relies primarily on two distinct IPLD GraphStores:

* One `ChainStore` which stores the blockchain, including block headers, associated messages, etc.
* One `StateStore` which stores the payload state from a given blockchain, or the `stateTree` resulting from all block messages in a given chain being applied to the genesis state by the [Filecoin VM](https://spec.filecoin.io/#section-systems.filecoin\_vm).

The `ChainStore` is downloaded by a node from their peers during the bootstrapping phase of [Chain Sync](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.chainsync) and is stored by the node thereafter. It is updated on every new block reception, or if the node syncs to a new best chain.

The `StateStore` is computed through the execution of all block messages in a given `ChainStore` and is stored by the node thereafter. It is updated with every new incoming block’s processing by the [VM Interpreter](https://spec.filecoin.io/#section-systems.filecoin\_vm.interpreter), and referenced accordingly by new blocks produced atop it in the [block header’s](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.struct.block) `ParentState` field.

## Network Interface <a href="#section-systems.filecoin_nodes.network" id="section-systems.filecoin_nodes.network"></a>

Filecoin nodes use several protocols of the libp2p networking stack for peer discovery, peer routing and block and message propagation. Libp2p is a modular networking stack for peer-to-peer networks. It includes several protocols and mechanisms to enable efficient, secure and resilient peer-to-peer communication. Libp2p nodes open connections with one another and mount different protocols or streams over the same connection. In the initial handshake, nodes exchange the protocols that each of them supports and all Filecoin related protocols will be mounted under `/fil/...` protocol identifiers.

The complete specification of libp2p can be found at [https://github.com/libp2p/specs](https://github.com/libp2p/specs). Here is the list of libp2p protocols used by Filecoin.

* **Graphsync:** Graphsync is a protocol to synchronize graphs across peers. It is used to reference, address, request and transfer blockchain and user data between Filecoin nodes. The [draft specification of GraphSync](https://github.com/ipld/specs/blob/master/block-layer/graphsync/graphsync.md) provides more details on the concepts, the interfaces and the network messages used by GraphSync. There are no Filecoin-specific modifications to the protocol id.
* **Gossipsub:** Block headers and messages are propagating through the Filecoin network using a gossip-based pubsub protocol acronymed _GossipSub_. As is traditionally the case with pubsub protocols, nodes subscribe to topics and receive messages published on those topics. When nodes receive messages from a topic they are subscribed to, they run a validation process and i) pass the message to the application, ii) forward the message further to nodes they know off being subscribed to the same topic. Furthermore, v1.1 version of GossipSub, which is the one used in Filecoin is enhanced with security mechanisms that make the protocol resilient against security attacks. The [GossipSub Specification](https://github.com/libp2p/specs/tree/master/pubsub/gossipsub) provides all the protocol details pertaining to its design and implementation, as well as specific settings for the protocols parameters. There have been no filecoin specific modifications to the protocol id. However the topic identifiers MUST be of the form `fil/blocks/<network-name>` and `fil/msgs/<network-name>`
* **Kademlia DHT:** The Kademlia DHT is a distributed hash table with a logarithmic bound on the maximum number of lookups for a particular node. In the Filecoin network, the Kademlia DHT is used primarily for peer discovery and peer routing. In particular, when a node wants to store data in the Filecoin network, they get a list of miners and their node information. This node information includes (among other things) the PeerID of the miner. In order to connect to the miner and exchange data, the node that wants to store data in the network has to find the Multiaddress of the miner, which they do by queering the DHT. The [libp2p Kad DHT Specification](https://github.com/libp2p/go-libp2p-kad-dht) provides implementation details of the DHT structure. For the Filecoin network, the protocol id must be of the form `fil/<network-name>/kad/1.0.0`.
* **Bootstrap List:** This is a list of nodes that a new node attempts to connect to upon joining the network. The list of bootstrap nodes and their addresses are defined by the users (i.e., applications).
* **Peer Exchange:** This protocol is the realisation of the peer discovery process discussed above at Kademlia DHT. It enables peers to find information and addresses of other peers in the network by interfacing with the DHT and create and issue queries for the peers they want to connect to.

## Clock <a href="#section-systems.filecoin_nodes.clock" id="section-systems.filecoin_nodes.clock"></a>

Filecoin assumes weak clock synchrony amongst participants in the system. That is, the system relies on participants having access to a globally synchronized clock (tolerating some bounded offset).

Filecoin relies on this system clock in order to secure consensus. Specifically, the clock is necessary to support validation rules that prevent block producers from mining blocks with a future timestamp and running leader elections more frequently than the protocol allows.

### **Clock uses**

The Filecoin system clock is used:

* by syncing nodes to validate that incoming blocks were mined in the appropriate epoch given their timestamp (see [Block Validation](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.struct.block.block-syntax-validation)). This is possible because the system clock maps all times to a unique epoch number totally determined by the start time in the genesis block.
* by syncing nodes to drop blocks coming from a future epoch
* by mining nodes to maintain protocol liveness by allowing participants to try leader election in the next round if no one has produced a block in the current round (see [Storage Power Consensus](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.storage\_power\_consensus)).

In order to allow miners to do the above, the system clock must:

1. Have low enough offset relative to other nodes so that blocks are not mined in epochs considered future epochs from the perspective of other nodes (those blocks should not be validated until the proper epoch/time as per [validation rules](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.struct.block.block-semantic-validation)).
2. Set epoch number on node initialization equal to `epoch = Floor[(current_time - genesis_time) / epoch_time]`

It is expected that other subsystems will register to a `NewRound()` event from the clock subsystem.

### **Clock Requirements**

Clocks used as part of the Filecoin protocol should be kept in sync, with offset less than 1 second so as to enable appropriate validation.

Computer-grade crystals can be expected to deviate by [1ppm](https://www.hindawi.com/journals/jcnc/2008/583162/) (i.e. 1 microsecond every second, or 0.6 seconds per week). Therefore, in order to respect the requirement above:

* Nodes SHOULD run an NTP daemon (e.g. timesyncd, ntpd, chronyd) to keep their clocks synchronized to one or more reliable external references.
  * We recommend the following sources:
    * **`pool.ntp.org`** ( [details](https://www.ntppool.org/en/use.html))
    * `time.cloudflare.com:1234` ( [details](https://www.cloudflare.com/time/))
    * `time.google.com` ( [details](https://developers.google.com/time))
    * `time.nist.gov` ( [details](https://tf.nist.gov/tf-cgi/servers.cgi))
* Larger mining operations MAY consider using local NTP/PTP servers with GPS references and/or frequency-stable external clocks for improved timekeeping.

Mining operations have a strong incentive to prevent their clock skewing ahead more than one epoch to keep their block submissions from being rejected. Likewise they have an incentive to prevent their clocks skewing behind more than one epoch to avoid partitioning themselves off from the synchronized nodes in the network.
