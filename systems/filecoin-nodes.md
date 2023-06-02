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
