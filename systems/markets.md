# Markets

Filecoin is a consensus protocol, a data-storage platform, and a marketplace for storing and retrieving data. There are two major components to Filecoin markets, the storage market and the retrieval market. While storage and retrieval negotiations for both the storage and the retrieval markets are taking place primarily _off the blockchain_ (at least in the current version of Filecoin), storage deals made in the storage market will be published on-chain and will be enforced by the protocol. Storage deal negotiation and order matching are expected to happen off-chain in the first version of Filecoin. Retrieval deals are also negotiated off-chain and executed with micropayments between transacting parties in payment channels.

Even though most of the market actions happen off the blockchain, there are on-chain invariants that create economic structure for network success and allow for positive emergent behavior. You can read more about the relationship between on-chain deals and storage power in [Storage Power Consensus](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.storage\_power\_consensus).

### [Storage Market in Filecoin](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market) <a href="#section-systems.filecoin_markets.storage_market" id="section-systems.filecoin_markets.storage_market"></a>

Storage Market subsystem is the data entry point into the network. Storage miners only earn power from data stored in a storage deal and all deals live on the Filecoin network. Specific deal negotiation process happens off chain, clients and miners enter a storage deal after an agreement has been reached and post storage deals on the Filecoin network to earn block rewards and get paid for storing the data in the storage deal. A deal is only valid when it is posted on chain with signatures from both parties and at the time of posting, there are sufficient balances for both parties locked up to honor the deal in terms of deal price and deal collateral.

#### [**Terminology**](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market.terminology)

* **StorageClient** - The party that wants to make a deal to store data
* **StorageProvider** - The party that will store the data in exchange for payment. A storage miner.
* **StorageMarketActor** - The on-chain component of deals. The StorageMarketActor is analogous to an escrow and a ledger for all deals made.
* **StorageAsk** - The current price and parameters a miner is currently offering for storage (analogous to an Ask in a financial market)
* **StorageDealProposal** - A proposal for a storage deal, signed only by the - `Storage client`
* **StorageDeal** - A storage deal proposal with a counter signature from the Provider, which then goes on-chain.

#### [**Deal Flow**](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market.deal-flow)

The lifecycle for a deal within the storage market contains distinct phases:

1. **Discovery** - The client identifies miners and determines their current asks.
2. **Negotiation** (out of band) - Both parties come to an agreement about the terms of the deal, each party commits funds to the deal and data is transferred from the client to the provider.
3. **Publishing** - The deal is published on chain, making the storage provider publicly accountable for the deal.
4. **Handoff** - Once the deal is published, it is handed off and handled by the Storage Mining Subsystem. The Storage Mining Subsystem will add the data corresponding to the deal to a sector, seal the sector, and tell the Storage Market Actor that the deal is in a sector, thereby marking the deal as active.

From that point on, the deal is handled by the Storage Mining Subsystem, which communicates with the Storage Market Actor in order to process deal payments. See [Storage Mining Subsystem](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining) for more details.

The following diagram outlines the phases of deal flow within the storage market in detail:

<figure><img src="https://spec.filecoin.io/_gen/diagrams/systems/filecoin_markets/storage_market/storage_market_flow.svg?1639060809" alt="" width="563"><figcaption></figcaption></figure>

### [**Discovery**](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market.discovery)

Discovery is the client process of identifying storage providers (i.e. a miner) who (subject to agreement on the deal’s terms) are offering to store the client’s data. There are many ways which a client can use to identify a provider to store their data. The list below outlines the minimum discovery services a filecoin implementation MUST provide. As the network evolves, third parties may build systems that supplement or enhance these services.

Discovery involves identifying providers and determining their current `StorageAsk`. The steps are as follows:

1. A client queries the chain to retrieve a list of Storage Miner Actors who have registerd as miners with the StoragePowerActor.
2. A client may perform additional queries to each Storage Miner Actor to determine their properties. Among others, these properties can include worker address, sector size, libp2p Multiaddress etc.
3. Once the client identifies potentially suitable providers, it sends a direct libp2p message using the `Storage Query Protocol` to get each potential provider’s current `StorageAsk`.
4. Miners respond on the `AskProtocol` with a signed version of their current `StorageAsk`.

A `StorageAsk` contains all the properties that a client will need to determine if a given provider will meet its needs for storage at this moment. Providers should update their asks frequently to ensure the information they are providing to clients is up to date.

### [**Negotiation**](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market.negotiation)

Negotiation is the out-of-band process during which a storage client and a storage provider come to an agreement about a storage deal and reach the point where a deal is published on chain.

Negotiation begins once a client has discovered a miner whose `StorageAsk` meets their desired criteria. The _recommended_ order of operations for negotiating and publishing a deal is as follows:

1. In order to propose a storage deal, the `StorageClient` calculates the piece commitment (`CommP`) for the data it intends to store. This is neccesary so that the `StorageProvider` can verify that the data the `StorageClient` sends to be stored matches the `CommP` in the `StorageDealProposal`. For more detail about the relationship between payloads, pieces, and `CommP` see [Piece](https://spec.filecoin.io/#section-systems.filecoin\_files.piece).
2. Before sending a proposal to the provider, the `StorageClient` adds funds for a deal, as necessary, to the `StorageMarketActor` (by calling `AddBalance`).
3. The `StorageClient` now creates a `StorageDealProposal` and sends the proposal and the CID for the root of the data payload to be stored to the `StorageProvider` using the `Storage Deal Protocol`.

From this point onwards, execution moves to the `StorageProvider`.

4. The `StorageProvider` inspects the deal to verify that the deal’s parameters match its own internal criteria (such as price, piece size, deal duration, etc). The `StorageProvider` rejects the proposal if the parameters don’t match its own criteria by sending a rejection to the client over the `Storage Deal Protocol`.
5. The `StorageProvider` queries the `StorageMarketActor` to verify the `StorageClient` has deposited enough funds to make the deal (i.e. the client’s balance is greater than the total storage price) and rejects the proposal if it hasn’t.
6. If all criteria are met, the `StorageProvider` responds using the `Storage Deal Protocol` to indicate an intent to accept the deal.

From this point onwards execution moves back to the `StorageClient`.

7. The `StorageClient` opens a push request for the payload data using the `Data Transfer Module`, and sends the request to the provider along with a voucher containing the CID for the `StorageDealProposal`.
8. The `StorageProvider` checks the voucher and verifies that the CID matches the storage deal proposal it has received and verified but not put on chain already. If so, it accepts the data transfer request from the `StorageClient`.
9. The `Data Transfer Module` now transfers the payload data to be stored from the `StorageClient` to the `StorageProvider` using `GraphSync`.
10. Once complete, the `Data Transfer Module` notifies the `StorageProvider`.
11. The `StorageProvider` recalculates the piece commitment (`CommP`) from the data transfer that just completed and verifies it matches the piece commitment in the `StorageDealProposal`.

### [**Publishing**](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market.publishing)

Data is now transferred, both parties have agreed, and it’s time to publish the deal. Given that the counter signature on a deal proposal is a standard message signature by the provider and the signed deal is an on-chain message, it is usually the `StorageProvider` that publishes the deal. However, if `StorageProvider` decides to send this signed on-chain message to the client before calling `PublishStorageDeal` then the client can publish the deal on-chain. The client’s funds are not locked until the deal is published and a published deal that is not activated within some pre-defined window will result in an on-chain penalty.

12. First, the `StorageProvider` adds collateral for the deal as needed to the `StorageMarketActor` (using `AddBalance`).
13. Then, the `StorageProvider` prepares and signs the on-chain `StorageDeal` message with the `StorageDealProposal` signed by the client and its own signature. It can now either send this message back to the client or call `PublishStorageDeals` on the `StorageMarketActor` to publish the deal. It is recommended for `StorageProvider` to send back the signed message before `PublishStorageDeals` is called.
14. After calling `PublishStorageDeals`, the `StorageProvider` sends a message to the `StorageClient` on the `Storage Deal Protocol` with the CID of the message that it is putting on chain for convenience.
15. If all goes well, the `StorageMarketActor` responds with an on-chain `DealID` for the published deal.

Finally, the `StorageClient` verifies the deal.

16. The `StorageClient` queries the node for the CID of the message published on chain (sent by the provider). It then inspects the message parameters to make sure they match the previously agreed deal.

### [**Handoff**](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market.handoff)

Now that a deal is published, it needs to be stored, sealed, and proven in order for the provider to be paid. See Storage Deal for more information about how deal payments are made. These later stages of a deal are handled by the [Storage Mining Subsystem](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining). So the final task for the Storage Market is to handoff to the Storage Mining Subsystem.

1. The `StorageProvider` writes the serialized, padded piece to a shared [Filestore](https://spec.filecoin.io/#section-systems.filecoin\_files.file.filestore).
2. The `StorageProvider` calls `HandleStorageDeal` on the `StorageMiner` with the published `StorageDeal` and filestore path (in Go this is the `io.Reader`).

A note regarding the order of operations: the only requirement to publish a storage deal with the `StorageMarketActor` is that the `StorageDealProposal` is signed by the `StorageClient`, the publish message is signed by the `StorageProvider`, and both parties have deposited adequate funds/collateral in the `StorageMarketActor`. As such, it’s not required that the steps listed above happen in this exact order. However, the above order is _recommended_ because it generally minimizes the ability of either party to act maliciously.

### [**Data Representation in the Storage Market**](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market.data-representation-in-the-storage-market)

Data submitted to the Filecoin network go through several transformations before they come to the format at which the `StorageProvider` stores it. Here we provide a summary of these transformations.

1. When a piece of data, or file is submitted to Filecoin (in some raw system format) it is transformed into a _UnixFS DAG style data representation_ (in case it is not in this format already, e.g., from IPFS-based applications). The hash that represents the root of the IPLD DAG of the UnixFS file is the _Payload CID_, which is used in the Retrieval Market.
2. In order to make a _Filecoin Piece_ the UnixFS IPLD DAG is serialised into a .car file, which is also raw bytes.
3. The resulting .car file is _padded_ with some extra data.
4. The next step is to calculate the Merkle root out of the hashes of individual Pieces. The resulting root of the Merkle tree is the **Piece CID**. This is also referred to as **CommP**. Note that at this stage the data is still unsealed.
5. At this point, the Piece is included in a Sector together with data from other deals. The `StorageProvider` then calculates Merkle root for all the Pieces inside the sector. The root of this tree is _CommD_. This is the _unsealed sector CID_.
6. The `StorageProvider` is then sealing the sector and the root of the resulting Merkle root is the _CommR_.

The following data types are unique to the Storage Market:

[Example: Storage Market Data Types ](https://spec.filecoin.io/#example-storage-market-data-types)

```go
package storagemarket

import (
	"fmt"
	"time"

	"github.com/ipfs/go-cid"
	logging "github.com/ipfs/go-log/v2"
	"github.com/libp2p/go-libp2p-core/peer"
	ma "github.com/multiformats/go-multiaddr"
	cbg "github.com/whyrusleeping/cbor-gen"

	"github.com/filecoin-project/go-address"
	datatransfer "github.com/filecoin-project/go-data-transfer"
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/crypto"
	"github.com/filecoin-project/specs-actors/actors/builtin/market"

	"github.com/filecoin-project/go-fil-markets/filestore"
)

var log = logging.Logger("storagemrkt")

//go:generate cbor-gen-for --map-encoding ClientDeal MinerDeal Balance SignedStorageAsk StorageAsk DataRef ProviderDealState DealStages DealStage Log

// DealProtocolID is the ID for the libp2p protocol for proposing storage deals.
const OldDealProtocolID = "/fil/storage/mk/1.0.1"
const DealProtocolID = "/fil/storage/mk/1.1.0"

// AskProtocolID is the ID for the libp2p protocol for querying miners for their current StorageAsk.
const OldAskProtocolID = "/fil/storage/ask/1.0.1"
const AskProtocolID = "/fil/storage/ask/1.1.0"

// DealStatusProtocolID is the ID for the libp2p protocol for querying miners for the current status of a deal.
const OldDealStatusProtocolID = "/fil/storage/status/1.0.1"
const DealStatusProtocolID = "/fil/storage/status/1.1.0"

// Balance represents a current balance of funds in the StorageMarketActor.
type Balance struct {
	Locked    abi.TokenAmount
	Available abi.TokenAmount
}

// StorageAsk defines the parameters by which a miner will choose to accept or
// reject a deal. Note: making a storage deal proposal which matches the miner's
// ask is a precondition, but not sufficient to ensure the deal is accepted (the
// storage provider may run its own decision logic).
type StorageAsk struct {
	// Price per GiB / Epoch
	Price         abi.TokenAmount
	VerifiedPrice abi.TokenAmount

	MinPieceSize abi.PaddedPieceSize
	MaxPieceSize abi.PaddedPieceSize
	Miner        address.Address
	Timestamp    abi.ChainEpoch
	Expiry       abi.ChainEpoch
	SeqNo        uint64
}

// SignedStorageAsk is an ask signed by the miner's private key
type SignedStorageAsk struct {
	Ask       *StorageAsk
	Signature *crypto.Signature
}

// SignedStorageAskUndefined represents the empty value for SignedStorageAsk
var SignedStorageAskUndefined = SignedStorageAsk{}

// StorageAskOption allows custom configuration of a storage ask
type StorageAskOption func(*StorageAsk)

// MinPieceSize configures a minimum piece size of a StorageAsk
func MinPieceSize(minPieceSize abi.PaddedPieceSize) StorageAskOption {
	return func(sa *StorageAsk) {
		sa.MinPieceSize = minPieceSize
	}
}

// MaxPieceSize configures maxiumum piece size of a StorageAsk
func MaxPieceSize(maxPieceSize abi.PaddedPieceSize) StorageAskOption {
	return func(sa *StorageAsk) {
		sa.MaxPieceSize = maxPieceSize
	}
}

// StorageAskUndefined represents an empty value for StorageAsk
var StorageAskUndefined = StorageAsk{}

// MinerDeal is the local state tracked for a deal by a StorageProvider
type MinerDeal struct {
	market.ClientDealProposal
	ProposalCid           cid.Cid
	AddFundsCid           *cid.Cid
	PublishCid            *cid.Cid
	Miner                 peer.ID
	Client                peer.ID
	State                 StorageDealStatus
	PiecePath             filestore.Path
	MetadataPath          filestore.Path
	SlashEpoch            abi.ChainEpoch
	FastRetrieval         bool
	Message               string
	FundsReserved         abi.TokenAmount
	Ref                   *DataRef
	AvailableForRetrieval bool

	DealID       abi.DealID
	CreationTime cbg.CborTime

	TransferChannelId *datatransfer.ChannelID
	SectorNumber      abi.SectorNumber

	InboundCAR string
}

// NewDealStages creates a new DealStages object ready to be used.
// EXPERIMENTAL; subject to change.
func NewDealStages() *DealStages {
	return &DealStages{}
}

// DealStages captures a timeline of the progress of a deal, grouped by stages.
// EXPERIMENTAL; subject to change.
type DealStages struct {
	// Stages contains an entry for every stage that the deal has gone through.
	// Each stage then contains logs.
	Stages []*DealStage
}

// DealStages captures data about the execution of a deal stage.
// EXPERIMENTAL; subject to change.
type DealStage struct {
	// Human-readable fields.
	// TODO: these _will_ need to be converted to canonical representations, so
	//  they are machine readable.
	Name             string
	Description      string
	ExpectedDuration string

	// Timestamps.
	// TODO: may be worth adding an exit timestamp. It _could_ be inferred from
	//  the start of the next stage, or from the timestamp of the last log line
	//  if this is a terminal stage. But that's non-determistic and it relies on
	//  assumptions.
	CreatedTime cbg.CborTime
	UpdatedTime cbg.CborTime

	// Logs contains a detailed timeline of events that occurred inside
	// this stage.
	Logs []*Log
}

// Log represents a point-in-time event that occurred inside a deal stage.
// EXPERIMENTAL; subject to change.
type Log struct {
	// Log is a human readable message.
	//
	// TODO: this _may_ need to be converted to a canonical data model so it
	//  is machine-readable.
	Log string

	UpdatedTime cbg.CborTime
}

// GetStage returns the DealStage object for a named stage, or nil if not found.
//
// TODO: the input should be a strongly-typed enum instead of a free-form string.
// TODO: drop Get from GetStage to make this code more idiomatic. Return a
//  second ok boolean to make it even more idiomatic.
// EXPERIMENTAL; subject to change.
func (ds *DealStages) GetStage(stage string) *DealStage {
	if ds == nil {
		return nil
	}

	for _, s := range ds.Stages {
		if s.Name == stage {
			return s
		}
	}

	return nil
}

// AddStageLog adds a log to the specified stage, creating the stage if it
// doesn't exist yet.
// EXPERIMENTAL; subject to change.
func (ds *DealStages) AddStageLog(stage, description, expectedDuration, msg string) {
	if ds == nil {
		return
	}

	log.Debugf("adding log for stage <%s> msg <%s>", stage, msg)

	now := curTime()
	st := ds.GetStage(stage)
	if st == nil {
		st = &DealStage{
			CreatedTime: now,
		}
		ds.Stages = append(ds.Stages, st)
	}

	st.Name = stage
	st.Description = description
	st.ExpectedDuration = expectedDuration
	st.UpdatedTime = now
	if msg != "" && (len(st.Logs) == 0 || st.Logs[len(st.Logs)-1].Log != msg) {
		// only add the log if it's not a duplicate.
		st.Logs = append(st.Logs, &Log{msg, now})
	}
}

// AddLog adds a log inside the DealStages object of the deal.
// EXPERIMENTAL; subject to change.
func (d *ClientDeal) AddLog(msg string, a ...interface{}) {
	if len(a) > 0 {
		msg = fmt.Sprintf(msg, a...)
	}

	stage := DealStates[d.State]
	description := DealStatesDescriptions[d.State]
	expectedDuration := DealStatesDurations[d.State]

	d.DealStages.AddStageLog(stage, description, expectedDuration, msg)
}

// ClientDeal is the local state tracked for a deal by a StorageClient
type ClientDeal struct {
	market.ClientDealProposal
	ProposalCid       cid.Cid
	AddFundsCid       *cid.Cid
	State             StorageDealStatus
	Miner             peer.ID
	MinerWorker       address.Address
	DealID            abi.DealID
	DataRef           *DataRef
	Message           string
	DealStages        *DealStages
	PublishMessage    *cid.Cid
	SlashEpoch        abi.ChainEpoch
	PollRetryCount    uint64
	PollErrorCount    uint64
	FastRetrieval     bool
	FundsReserved     abi.TokenAmount
	CreationTime      cbg.CborTime
	TransferChannelID *datatransfer.ChannelID
	SectorNumber      abi.SectorNumber
}

// StorageProviderInfo describes on chain information about a StorageProvider
// (use QueryAsk to determine more specific deal parameters)
type StorageProviderInfo struct {
	Address    address.Address // actor address
	Owner      address.Address
	Worker     address.Address // signs messages
	SectorSize uint64
	PeerID     peer.ID
	Addrs      []ma.Multiaddr
}

// ProposeStorageDealResult returns the result for a proposing a deal
type ProposeStorageDealResult struct {
	ProposalCid cid.Cid
}

// ProposeStorageDealParams describes the parameters for proposing a storage deal
type ProposeStorageDealParams struct {
	Addr          address.Address
	Info          *StorageProviderInfo
	Data          *DataRef
	StartEpoch    abi.ChainEpoch
	EndEpoch      abi.ChainEpoch
	Price         abi.TokenAmount
	Collateral    abi.TokenAmount
	Rt            abi.RegisteredSealProof
	FastRetrieval bool
	VerifiedDeal  bool
}

const (
	// TTGraphsync means data for a deal will be transferred by graphsync
	TTGraphsync = "graphsync"

	// TTManual means data for a deal will be transferred manually and imported
	// on the provider
	TTManual = "manual"
)

// DataRef is a reference for how data will be transferred for a given storage deal
type DataRef struct {
	TransferType string
	Root         cid.Cid

	PieceCid     *cid.Cid              // Optional for non-manual transfer, will be recomputed from the data if not given
	PieceSize    abi.UnpaddedPieceSize // Optional for non-manual transfer, will be recomputed from the data if not given
	RawBlockSize uint64                // Optional: used as the denominator when calculating transfer %
}

// ProviderDealState represents a Provider's current state of a deal
type ProviderDealState struct {
	State         StorageDealStatus
	Message       string
	Proposal      *market.DealProposal
	ProposalCid   *cid.Cid
	AddFundsCid   *cid.Cid
	PublishCid    *cid.Cid
	DealID        abi.DealID
	FastRetrieval bool
}

func curTime() cbg.CborTime {
	now := time.Now()
	return cbg.CborTime(time.Unix(0, now.UnixNano()).UTC())
}
```

Details about `StorageDealProposal` and `StorageDeal` (which are used in the Storage Market and elsewhere) specifically can be found in Storage Deal.

[**Protocols**](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market.protocols)

> **Name**: Storage Query Protocol\
> **Protocol ID**: `/fil/<network-name>/storage/ask/1.0.1`

Request: CBOR Encoded AskProtocolRequest Data Structure Response: CBOR Encoded AskProtocolResponse Data Structure

> **Name**: Storage Deal Protocol\
> **Protocol ID**: `/fil/<network-name>/storage/mk/1.0.1`

Request: CBOR Encoded DealProtocolRequest Data Structure Response: CBOR Encoded DealProtocolResponse Data Structure

[**Storage Provider**](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market.storage-provider)

The `StorageProvider` is a module that handles incoming queries for Asks and proposals for Deals from a `StorageClient`. It also tracks deals as they move through the deal flow, handling off chain actions during the negotiation phases of the deal and ultimately telling the `StorageMarketActor` to publish on chain. The `StorageProvider`’s last action is to handoff a published deal for storage and sealing to the Storage Mining Subsystem. Note that any address registered as a `StorageMarketParticipant` with the `StorageMarketActor` can be used with the `StorageClient`.

It is worth highlighting that a single participant can be a `StorageClient`, `StorageProvider`, or both at the same time.

Because most of what a Storage Provider does is respond to actions initiated by a `StorageClient`, most of its public facing methods relate to getting current status on deals, as opposed to initiating new actions. However, a user of the `StorageProvider` module can update the current Ask for the provider.

[Example: Storage Market Provider ](https://spec.filecoin.io/#example-storage-market-provider)

```go
package storagemarket

import (
	"context"
	"io"

	"github.com/ipfs/go-cid"

	"github.com/filecoin-project/go-state-types/abi"

	"github.com/filecoin-project/go-fil-markets/shared"
)

// ProviderSubscriber is a callback that is run when events are emitted on a StorageProvider
type ProviderSubscriber func(event ProviderEvent, deal MinerDeal)

// StorageProvider provides an interface to the storage market for a single
// storage miner.
type StorageProvider interface {

	// Start initializes deal processing on a StorageProvider and restarts in progress deals.
	// It also registers the provider with a StorageMarketNetwork so it can receive incoming
	// messages on the storage market's libp2p protocols
	Start(ctx context.Context) error

	// OnReady registers a listener for when the provider comes on line
	OnReady(shared.ReadyFunc)

	// Stop terminates processing of deals on a StorageProvider
	Stop() error

	// SetAsk configures the storage miner's ask with the provided prices (for unverified and verified deals),
	// duration, and options. Any previously-existing ask is replaced.
	SetAsk(price abi.TokenAmount, verifiedPrice abi.TokenAmount, duration abi.ChainEpoch, options ...StorageAskOption) error

	// GetAsk returns the storage miner's ask, or nil if one does not exist.
	GetAsk() *SignedStorageAsk

	// ListLocalDeals lists deals processed by this storage provider
	ListLocalDeals() ([]MinerDeal, error)

	// AddStorageCollateral adds storage collateral
	AddStorageCollateral(ctx context.Context, amount abi.TokenAmount) error

	// GetStorageCollateral returns the current collateral balance
	GetStorageCollateral(ctx context.Context) (Balance, error)

	// ImportDataForDeal manually imports data for an offline storage deal
	ImportDataForDeal(ctx context.Context, propCid cid.Cid, data io.Reader) error

	// SubscribeToEvents listens for events that happen related to storage deals on a provider
	SubscribeToEvents(subscriber ProviderSubscriber) shared.Unsubscribe

	RetryDealPublishing(propCid cid.Cid) error
}
```

[**Storage Client**](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market.storage-client)

The `StorageClient` is a module that discovers miners, determines their asks, and proposes deals to `StorageProviders`. It also tracks deals as they move through the deal flow. Note that any address registered as a `StorageMarketParticipant` with the `StorageMarketActor` can be used with the `StorageClient`.

Recall that a single participant can be a `StorageClient`, `StorageProvider`, or both at the same time.

[Example: Storage Market Client ](https://spec.filecoin.io/#example-storage-market-client)

```go
package storagemarket

import (
	"context"

	"github.com/ipfs/go-cid"
	bstore "github.com/ipfs/go-ipfs-blockstore"

	"github.com/filecoin-project/go-address"
	"github.com/filecoin-project/go-state-types/abi"

	"github.com/filecoin-project/go-fil-markets/shared"
)

type PayloadCID = cid.Cid

// BlockstoreAccessor is used by the storage market client to get a
// blockstore when needed, concretely to send the payload to the provider.
// This abstraction allows the caller to provider any blockstore implementation:
// a CARv2 file, an IPFS blockstore, or something else.
//
// They key is a payload CID because this is the unique top-level key of a
// client-side data import.
type BlockstoreAccessor interface {
	Get(PayloadCID) (bstore.Blockstore, error)
	Done(PayloadCID) error
}

// ClientSubscriber is a callback that is run when events are emitted on a StorageClient
type ClientSubscriber func(event ClientEvent, deal ClientDeal)

// StorageClient is a client interface for making storage deals with a StorageProvider
type StorageClient interface {

	// Start initializes deal processing on a StorageClient and restarts
	// in progress deals
	Start(ctx context.Context) error

	// OnReady registers a listener for when the client comes on line
	OnReady(shared.ReadyFunc)

	// Stop ends deal processing on a StorageClient
	Stop() error

	// ListProviders queries chain state and returns active storage providers
	ListProviders(ctx context.Context) (<-chan StorageProviderInfo, error)

	// ListLocalDeals lists deals initiated by this storage client
	ListLocalDeals(ctx context.Context) ([]ClientDeal, error)

	// GetLocalDeal lists deals that are in progress or rejected
	GetLocalDeal(ctx context.Context, cid cid.Cid) (ClientDeal, error)

	// GetAsk returns the current ask for a storage provider
	GetAsk(ctx context.Context, info StorageProviderInfo) (*StorageAsk, error)

	// GetProviderDealState queries a provider for the current state of a client's deal
	GetProviderDealState(ctx context.Context, proposalCid cid.Cid) (*ProviderDealState, error)

	// ProposeStorageDeal initiates deal negotiation with a Storage Provider
	ProposeStorageDeal(ctx context.Context, params ProposeStorageDealParams) (*ProposeStorageDealResult, error)

	// GetPaymentEscrow returns the current funds available for deal payment
	GetPaymentEscrow(ctx context.Context, addr address.Address) (Balance, error)

	// AddStorageCollateral adds storage collateral
	AddPaymentEscrow(ctx context.Context, addr address.Address, amount abi.TokenAmount) error

	// SubscribeToEvents listens for events that happen related to storage deals on a provider
	SubscribeToEvents(subscriber ClientSubscriber) shared.Unsubscribe
}
```

#### [Storage Market On-Chain Components](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market) <a href="#section-systems.filecoin_markets.onchain_storage_market" id="section-systems.filecoin_markets.onchain_storage_market"></a>

[**Storage Deals**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage-deals)

There are two types of deals in Filecoin markets, storage deals and retrieval deals. Storage deals are recorded on the blockchain and enforced by the protocol. Retrieval deals are off chain and enabled by a micropayment channel between transacting parties (see [Retrieval Market](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market) for more information).

The lifecycle of a Storage Deal touches several major subsystems, components, and protocols in Filecoin.

This section describes the storage deal data type and provides a technical outline of the deal flow in terms of how all the components interact with each other, as well as the functions they call. For more detail on the off-chain parts of the storage market see the [Storage Market section](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market).

[**Data Types**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.data-types)

[Example: ](https://spec.filecoin.io/#example-)

```go
package market

import (
	market0 "github.com/filecoin-project/specs-actors/actors/builtin/market"
)

//var PieceCIDPrefix = cid.Prefix{
//	Version:  1,
//	Codec:    cid.FilCommitmentUnsealed,
//	MhType:   mh.SHA2_256_TRUNC254_PADDED,
//	MhLength: 32,
//}
var PieceCIDPrefix = market0.PieceCIDPrefix

// Note: Deal Collateral is only released and returned to clients and miners
// when the storage deal stops counting towards power. In the current iteration,
// it will be released when the sector containing the storage deals expires,
// even though some storage deals can expire earlier than the sector does.
// Collaterals are denominated in PerEpoch to incur a cost for self dealing or
// minimal deals that last for a long time.
// Note: ClientCollateralPerEpoch may not be needed and removed pending future confirmation.
// There will be a Minimum value for both client and provider deal collateral.
//type DealProposal struct {
//	PieceCID     cid.Cid `checked:"true"` // Checked in validateDeal, CommP
//	PieceSize    abi.PaddedPieceSize
//	VerifiedDeal bool
//	Client       addr.Address
//	Provider     addr.Address
//
//	// Label is an arbitrary client chosen label to apply to the deal
//	// TODO: Limit the size of this: https://github.com/filecoin-project/specs-actors/issues/897
//	Label string
//
//	// Nominal start epoch. Deal payment is linear between StartEpoch and EndEpoch,
//	// with total amount StoragePricePerEpoch * (EndEpoch - StartEpoch).
//	// Storage deal must appear in a sealed (proven) sector no later than StartEpoch,
//	// otherwise it is invalid.
//	StartEpoch           abi.ChainEpoch
//	EndEpoch             abi.ChainEpoch
//	StoragePricePerEpoch abi.TokenAmount
//
//	ProviderCollateral abi.TokenAmount
//	ClientCollateral   abi.TokenAmount
//}
type DealProposal = market0.DealProposal

// ClientDealProposal is a DealProposal signed by a client
// type ClientDealProposal struct {
// 	Proposal        DealProposal
// 	ClientSignature crypto.Signature
// }
type ClientDealProposal = market0.ClientDealProposal
```

[**Storage Market Actor**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_market\_actor)

The `StorageMarketActor` is responsible for processing and managing on-chain deals. This is also the entry point of all storage deals and data into the system. It maintains a mapping of `StorageDealID` to `StorageDeal` and keeps track of locked balances of `StorageClient` and `StorageProvider`. When a deal is posted on chain through the `StorageMarketActor`, it will first check if both transacting parties have sufficient balances locked up and include the deal on chain.

[**`StorageMarketActor` implementation**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_market\_actor.storagemarketactor-implementation)

[Example: ](https://spec.filecoin.io/#example-)

```go
package market

import (
	"sort"

	addr "github.com/filecoin-project/go-address"
	"github.com/filecoin-project/go-bitfield"
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/big"
	"github.com/filecoin-project/go-state-types/cbor"
	"github.com/filecoin-project/go-state-types/exitcode"
	rtt "github.com/filecoin-project/go-state-types/rt"
	market0 "github.com/filecoin-project/specs-actors/actors/builtin/market"
	"github.com/ipfs/go-cid"
	cbg "github.com/whyrusleeping/cbor-gen"
	"golang.org/x/xerrors"

	"github.com/filecoin-project/specs-actors/v7/actors/builtin"
	"github.com/filecoin-project/specs-actors/v7/actors/builtin/power"
	"github.com/filecoin-project/specs-actors/v7/actors/builtin/reward"
	"github.com/filecoin-project/specs-actors/v7/actors/builtin/verifreg"
	"github.com/filecoin-project/specs-actors/v7/actors/runtime"
	"github.com/filecoin-project/specs-actors/v7/actors/util/adt"
)

type Actor struct{}

type Runtime = runtime.Runtime

func (a Actor) Exports() []interface{} {
	return []interface{}{
		builtin.MethodConstructor: a.Constructor,
		2:                         a.AddBalance,
		3:                         a.WithdrawBalance,
		4:                         a.PublishStorageDeals,
		5:                         a.VerifyDealsForActivation,
		6:                         a.ActivateDeals,
		7:                         a.OnMinerSectorsTerminate,
		8:                         a.ComputeDataCommitment,
		9:                         a.CronTick,
	}
}

func (a Actor) Code() cid.Cid {
	return builtin.StorageMarketActorCodeID
}

func (a Actor) IsSingleton() bool {
	return true
}

func (a Actor) State() cbor.Er {
	return new(State)
}

var _ runtime.VMActor = Actor{}

////////////////////////////////////////////////////////////////////////////////
// Actor methods
////////////////////////////////////////////////////////////////////////////////

func (a Actor) Constructor(rt Runtime, _ *abi.EmptyValue) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.SystemActorAddr)

	st, err := ConstructState(adt.AsStore(rt))
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to create state")
	rt.StateCreate(st)
	return nil
}

//type WithdrawBalanceParams struct {
//	ProviderOrClientAddress addr.Address
//	Amount                  abi.TokenAmount
//}
type WithdrawBalanceParams = market0.WithdrawBalanceParams

// Attempt to withdraw the specified amount from the balance held in escrow.
// If less than the specified amount is available, yields the entire available balance.
// Returns the amount withdrawn.
func (a Actor) WithdrawBalance(rt Runtime, params *WithdrawBalanceParams) *abi.TokenAmount {
	if params.Amount.LessThan(big.Zero()) {
		rt.Abortf(exitcode.ErrIllegalArgument, "negative amount %v", params.Amount)
	}

	nominal, recipient, approvedCallers := escrowAddress(rt, params.ProviderOrClientAddress)
	// for providers -> only corresponding owner or worker can withdraw
	// for clients -> only the client i.e the recipient can withdraw
	rt.ValidateImmediateCallerIs(approvedCallers...)

	amountExtracted := abi.NewTokenAmount(0)
	var st State
	rt.StateTransaction(&st, func() {
		msm, err := st.mutator(adt.AsStore(rt)).withEscrowTable(WritePermission).
			withLockedTable(WritePermission).build()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load state")

		// The withdrawable amount might be slightly less than nominal
		// depending on whether or not all relevant entries have been processed
		// by cron
		minBalance, err := msm.lockedTable.Get(nominal)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to get locked balance")

		ex, err := msm.escrowTable.SubtractWithMinimum(nominal, params.Amount, minBalance)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to subtract from escrow table")

		err = msm.commitState()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to flush state")

		amountExtracted = ex
	})
	code := rt.Send(recipient, builtin.MethodSend, nil, amountExtracted, &builtin.Discard{})
	builtin.RequireSuccess(rt, code, "failed to send funds")
	return &amountExtracted
}

// Deposits the received value into the balance held in escrow.
func (a Actor) AddBalance(rt Runtime, providerOrClientAddress *addr.Address) *abi.EmptyValue {
	msgValue := rt.ValueReceived()
	builtin.RequireParam(rt, msgValue.GreaterThan(big.Zero()), "balance to add must be greater than zero")

	// only signing parties can add balance for client AND provider.
	rt.ValidateImmediateCallerType(builtin.CallerTypesSignable...)

	nominal, _, _ := escrowAddress(rt, *providerOrClientAddress)

	var st State
	rt.StateTransaction(&st, func() {
		msm, err := st.mutator(adt.AsStore(rt)).withEscrowTable(WritePermission).
			withLockedTable(WritePermission).build()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load state")

		err = msm.escrowTable.Add(nominal, msgValue)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to add balance to escrow table")
		err = msm.commitState()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to flush state")
	})
	return nil
}

// type PublishStorageDealsParams struct {
// 	Deals []ClientDealProposal
// }
type PublishStorageDealsParams = market0.PublishStorageDealsParams

type PublishStorageDealsReturn struct {
	IDs        []abi.DealID
	ValidDeals bitfield.BitField
}

// Publish a new set of storage deals (not yet included in a sector).
func (a Actor) PublishStorageDeals(rt Runtime, params *PublishStorageDealsParams) *PublishStorageDealsReturn {
	// Deal message must have a From field identical to the provider of all the deals.
	// This allows us to retain and verify only the client's signature in each deal proposal itself.
	rt.ValidateImmediateCallerType(builtin.CallerTypesSignable...)
	if len(params.Deals) == 0 {
		rt.Abortf(exitcode.ErrIllegalArgument, "empty deals parameter")
	}

	// All deals should have the same provider so get worker once
	providerRaw := params.Deals[0].Proposal.Provider
	provider, ok := rt.ResolveAddress(providerRaw)
	if !ok {
		rt.Abortf(exitcode.ErrNotFound, "failed to resolve provider address %v", providerRaw)
	}

	codeID, ok := rt.GetActorCodeCID(provider)
	builtin.RequireParam(rt, ok, "no codeId for address %v", provider)
	if !codeID.Equals(builtin.StorageMinerActorCodeID) {
		rt.Abortf(exitcode.ErrIllegalArgument, "deal provider is not a StorageMinerActor")
	}

	caller := rt.Caller()
	_, worker, controllers := builtin.RequestMinerControlAddrs(rt, provider)
	callerOk := caller == worker
	for _, controller := range controllers {
		if callerOk {
			break
		}
		callerOk = caller == controller
	}
	if !callerOk {
		rt.Abortf(exitcode.ErrForbidden, "caller %v is not worker or control address of provider %v", caller, provider)
	}
	resolvedAddrs := make(map[addr.Address]addr.Address, len(params.Deals))
	baselinePower := requestCurrentBaselinePower(rt)
	networkRawPower, networkQAPower := requestCurrentNetworkPower(rt)

	// Drop invalid deals
	var st State
	proposalCidLookup := make(map[cid.Cid]struct{})
	validProposalCids := make([]cid.Cid, 0)
	validDeals := make([]ClientDealProposal, 0, len(params.Deals))
	totalClientLockup := make(map[addr.Address]abi.TokenAmount)
	totalProviderLockup := abi.NewTokenAmount(0)

	validInputBf := bitfield.New()
	rt.StateReadonly(&st)
	msm, err := st.mutator(adt.AsStore(rt)).withPendingProposals(ReadOnlyPermission).
		withEscrowTable(ReadOnlyPermission).withLockedTable(ReadOnlyPermission).build()
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load state")
	for di, deal := range params.Deals {
		/*
			drop malformed deals
		*/
		if err := validateDeal(rt, deal, networkRawPower, networkQAPower, baselinePower); err != nil {
			rt.Log(rtt.INFO, "invalid deal %d: %s", di, err)
			continue
		}
		if deal.Proposal.Provider != provider && deal.Proposal.Provider != providerRaw {
			rt.Log(rtt.INFO, "invalid deal %d: cannot publish deals from multiple providers in one batch", di)
			continue
		}
		client, ok := rt.ResolveAddress(deal.Proposal.Client)
		if !ok {
			rt.Log(rtt.INFO, "invalid deal %d: failed to resolve proposal.Client address %v for deal ", di, deal.Proposal.Client)
			continue
		}

		/*
			drop deals with insufficient lock up to cover costs
		*/
		if _, ok := totalClientLockup[client]; !ok {
			totalClientLockup[client] = abi.NewTokenAmount(0)
		}
		totalClientLockup[client] = big.Sum(totalClientLockup[client], deal.Proposal.ClientBalanceRequirement())
		clientBalanceOk, err := msm.balanceCovered(client, totalClientLockup[client])
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to check client balance coverage")
		if !clientBalanceOk {
			rt.Log(rtt.INFO, "invalid deal: %d: insufficient client funds to cover proposal cost", di)
			continue
		}
		totalProviderLockup = big.Sum(totalProviderLockup, deal.Proposal.ProviderCollateral)
		providerBalanceOk, err := msm.balanceCovered(provider, totalProviderLockup)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to check provider balance coverage")
		if !providerBalanceOk {
			rt.Log(rtt.INFO, "invalid deal: %d: insufficient provider funds to cover proposal cost", di)
			continue
		}

		/*
			drop duplicate deals
		*/
		// Normalise provider and client addresses in the proposal stored on chain.
		// Must happen after signature verification and before taking cid.
		deal.Proposal.Provider = provider
		resolvedAddrs[deal.Proposal.Client] = client
		deal.Proposal.Client = client

		pcid, err := deal.Proposal.Cid()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "failed to take cid of proposal %d", di)

		// check proposalCids for duplication within message batch
		// check state PendingProposals for duplication across messages
		duplicateInState, err := msm.pendingDeals.Has(abi.CidKey(pcid))
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to check for existence of deal proposal")
		_, duplicateInMessage := proposalCidLookup[pcid]
		if duplicateInState || duplicateInMessage {
			rt.Log(rtt.INFO, "invalid deal %d: cannot publish duplicate deal proposal %s", di)
			continue
		}

		/*
			check VerifiedClient allowed cap and deduct PieceSize from cap
			drop deals with a DealSize that cannot be fully covered by VerifiedClient's available DataCap
		*/
		if deal.Proposal.VerifiedDeal {
			code := rt.Send(
				builtin.VerifiedRegistryActorAddr,
				builtin.MethodsVerifiedRegistry.UseBytes,
				&verifreg.UseBytesParams{
					Address:  client,
					DealSize: big.NewIntUnsigned(uint64(deal.Proposal.PieceSize)),
				},
				abi.NewTokenAmount(0),
				&builtin.Discard{},
			)
			if code.IsError() {
				rt.Log(rtt.INFO, "invalid deal %d: failed to acquire datacap exitcode: %d", di, code)
				continue
			}
		}

		// update valid deal state
		proposalCidLookup[pcid] = struct{}{}
		validProposalCids = append(validProposalCids, pcid)
		validDeals = append(validDeals, deal)
		validInputBf.Set(uint64(di))
	}

	validDealCount, err := validInputBf.Count()
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to count valid deals in bitfield")
	builtin.RequirePredicate(rt, len(validDeals) == len(validProposalCids), exitcode.ErrIllegalState,
		"%d valid deals but %d valid proposal cids", len(validDeals), len(validProposalCids))
	builtin.RequirePredicate(rt, uint64(len(validDeals)) == validDealCount, exitcode.ErrIllegalState,
		"%d valid deals but validDealCount=%d", len(validDeals), validDealCount)
	builtin.RequireParam(rt, validDealCount > 0, "All deal proposals invalid")

	var newDealIds []abi.DealID
	rt.StateTransaction(&st, func() {
		msm, err := st.mutator(adt.AsStore(rt)).withPendingProposals(WritePermission).
			withDealProposals(WritePermission).withDealsByEpoch(WritePermission).withEscrowTable(WritePermission).
			withLockedTable(WritePermission).build()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load state")

		// All storage dealProposals will be added in an atomic transaction; this operation will be unrolled if any of them fails.
		// This should only fail on programmer error because all expected invalid conditions should be filtered in the first set of checks.
		for vdi, validDeal := range validDeals {
			err := msm.lockClientAndProviderBalances(&validDeal.Proposal)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to lock balance")

			id := msm.generateStorageDealID()

			pcid := validProposalCids[vdi]
			err = msm.pendingDeals.Put(abi.CidKey(pcid))
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to set pending deal")

			err = msm.dealProposals.Set(id, &validDeal.Proposal)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to set deal")

			// We randomize the first epoch for when the deal will be processed so an attacker isn't able to
			// schedule too many deals for the same tick.
			processEpoch := GenRandNextEpoch(validDeal.Proposal.StartEpoch, id)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to generate random process epoch")

			err = msm.dealsByEpoch.Put(processEpoch, id)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to set deal ops by epoch")

			newDealIds = append(newDealIds, id)
		}
		err = msm.commitState()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to flush state")
	})

	return &PublishStorageDealsReturn{
		IDs:        newDealIds,
		ValidDeals: validInputBf,
	}
}

// Changed since v2:
// - Array of sectors rather than just one
// - Removed SectorStart (which is unknown at call time)
type VerifyDealsForActivationParams struct {
	Sectors []SectorDeals
}

type SectorDeals struct {
	SectorExpiry abi.ChainEpoch
	DealIDs      []abi.DealID
}

// Changed since v2:
// - Array of sectors weights
type VerifyDealsForActivationReturn struct {
	Sectors []SectorWeights
}

type SectorWeights struct {
	DealSpace          uint64         // Total space in bytes of submitted deals.
	DealWeight         abi.DealWeight // Total space*time of submitted deals.
	VerifiedDealWeight abi.DealWeight // Total space*time of submitted verified deals.
}

// Computes the weight of deals proposed for inclusion in a number of sectors.
// Deal weight is defined as the sum, over all deals in the set, of the product of deal size and duration.
//
// This method performs some light validation on the way in order to fail early if deals can be
// determined to be invalid for the proposed sector properties.
// Full deal validation is deferred to deal activation since it depends on the activation epoch.
func (a Actor) VerifyDealsForActivation(rt Runtime, params *VerifyDealsForActivationParams) *VerifyDealsForActivationReturn {
	rt.ValidateImmediateCallerType(builtin.StorageMinerActorCodeID)
	minerAddr := rt.Caller()
	currEpoch := rt.CurrEpoch()

	var st State
	rt.StateReadonly(&st)
	store := adt.AsStore(rt)

	proposals, err := AsDealProposalArray(store, st.Proposals)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deal proposals")

	weights := make([]SectorWeights, len(params.Sectors))
	for i, sector := range params.Sectors {
		// Pass the current epoch as the activation epoch for validation.
		// The sector activation epoch isn't yet known, but it's still more helpful to fail now if the deal
		// is so late that a sector activating now couldn't include it.
		dealWeight, verifiedWeight, dealSpace, err := validateAndComputeDealWeight(proposals, sector.DealIDs, minerAddr, sector.SectorExpiry, currEpoch)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to validate deal proposals for activation")

		weights[i] = SectorWeights{
			DealSpace:          dealSpace,
			DealWeight:         dealWeight,
			VerifiedDealWeight: verifiedWeight,
		}
	}

	return &VerifyDealsForActivationReturn{
		Sectors: weights,
	}
}

//type ActivateDealsParams struct {
//	DealIDs      []abi.DealID
//	SectorExpiry abi.ChainEpoch
//}
type ActivateDealsParams = market0.ActivateDealsParams

// Verify that a given set of storage deals is valid for a sector currently being ProveCommitted,
// update the market's internal state accordingly.
func (a Actor) ActivateDeals(rt Runtime, params *ActivateDealsParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerType(builtin.StorageMinerActorCodeID)
	minerAddr := rt.Caller()
	currEpoch := rt.CurrEpoch()

	var st State
	store := adt.AsStore(rt)

	// Update deal dealStates.
	rt.StateTransaction(&st, func() {
		_, _, _, err := ValidateDealsForActivation(&st, store, params.DealIDs, minerAddr, params.SectorExpiry, currEpoch)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to validate dealProposals for activation")

		msm, err := st.mutator(adt.AsStore(rt)).withDealStates(WritePermission).
			withPendingProposals(ReadOnlyPermission).withDealProposals(ReadOnlyPermission).build()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load state")

		for _, dealID := range params.DealIDs {
			// This construction could be replaced with a single "update deal state" state method, possibly batched
			// over all deal ids at once.
			_, found, err := msm.dealStates.Get(dealID)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to get state for dealId %d", dealID)
			if found {
				rt.Abortf(exitcode.ErrIllegalArgument, "deal %d already included in another sector", dealID)
			}

			proposal, err := getDealProposal(msm.dealProposals, dealID)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to get dealId %d", dealID)

			propc, err := proposal.Cid()
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to calculate proposal CID")

			has, err := msm.pendingDeals.Has(abi.CidKey(propc))
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to get pending proposal %v", propc)

			if !has {
				rt.Abortf(exitcode.ErrIllegalState, "tried to activate deal that was not in the pending set (%s)", propc)
			}

			err = msm.dealStates.Set(dealID, &DealState{
				SectorStartEpoch: currEpoch,
				LastUpdatedEpoch: epochUndefined,
				SlashEpoch:       epochUndefined,
			})
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to set deal state %d", dealID)
		}

		err = msm.commitState()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to flush state")
	})

	return nil
}

type SectorDataSpec struct {
	DealIDs    []abi.DealID
	SectorType abi.RegisteredSealProof
}

type ComputeDataCommitmentParams struct {
	Inputs []*SectorDataSpec
}

type ComputeDataCommitmentReturn struct {
	CommDs []cbg.CborCid
}

func (a Actor) ComputeDataCommitment(rt Runtime, params *ComputeDataCommitmentParams) *ComputeDataCommitmentReturn {
	rt.ValidateImmediateCallerType(builtin.StorageMinerActorCodeID)

	var st State
	rt.StateReadonly(&st)
	proposals, err := AsDealProposalArray(adt.AsStore(rt), st.Proposals)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deal dealProposals")
	commDs := make([]cbg.CborCid, len(params.Inputs))
	for i, commInput := range params.Inputs {
		pieces := make([]abi.PieceInfo, 0)
		for _, dealID := range commInput.DealIDs {
			deal, err := getDealProposal(proposals, dealID)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to get dealId %d", dealID)

			pieces = append(pieces, abi.PieceInfo{
				PieceCID: deal.PieceCID,
				Size:     deal.PieceSize,
			})
		}
		commD, err := rt.ComputeUnsealedSectorCID(commInput.SectorType, pieces)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "failed to compute unsealed sectorCID: %s", err)
		commDs[i] = (cbg.CborCid)(commD)
	}
	return &ComputeDataCommitmentReturn{
		CommDs: commDs,
	}
}

//type OnMinerSectorsTerminateParams struct {
//	Epoch   abi.ChainEpoch
//	DealIDs []abi.DealID
//}
type OnMinerSectorsTerminateParams = market0.OnMinerSectorsTerminateParams

// Terminate a set of deals in response to their containing sector being terminated.
// Slash provider collateral, refund client collateral, and refund partial unpaid escrow
// amount to client.
func (a Actor) OnMinerSectorsTerminate(rt Runtime, params *OnMinerSectorsTerminateParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerType(builtin.StorageMinerActorCodeID)
	minerAddr := rt.Caller()

	var st State
	rt.StateTransaction(&st, func() {
		msm, err := st.mutator(adt.AsStore(rt)).withDealStates(WritePermission).
			withDealProposals(ReadOnlyPermission).build()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deal state")

		for _, dealID := range params.DealIDs {
			deal, found, err := msm.dealProposals.Get(dealID)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to get deal proposal %v", dealID)
			// The deal may have expired and been deleted before the sector is terminated.
			// Log the dealID for the dealProposal and continue execution for other deals
			if !found {
				rt.Log(rtt.INFO, "couldn't find deal %d", dealID)
				continue
			}
			builtin.RequireState(rt, deal.Provider == minerAddr, "caller %v is not the provider %v of deal %v",
				minerAddr, deal.Provider, dealID)

			// do not slash expired deals
			if deal.EndEpoch <= params.Epoch {
				rt.Log(rtt.INFO, "deal %d expired, not slashing", dealID)
				continue
			}

			state, found, err := msm.dealStates.Get(dealID)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to get deal state %v", dealID)
			if !found {
				// A deal with a proposal but no state is not activated, but then it should not be
				// part of a sector that is terminating.
				rt.Abortf(exitcode.ErrIllegalArgument, "no state for deal %v", dealID)
			}

			// if a deal is already slashed, we don't need to do anything here.
			if state.SlashEpoch != epochUndefined {
				rt.Log(rtt.INFO, "deal %d already slashed", dealID)
				continue
			}

			// mark the deal for slashing here.
			// actual releasing of locked funds for the client and slashing of provider collateral happens in CronTick.
			state.SlashEpoch = params.Epoch

			err = msm.dealStates.Set(dealID, state)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to set deal state %v", dealID)
		}

		err = msm.commitState()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to flush state")
	})
	return nil
}

func (a Actor) CronTick(rt Runtime, _ *abi.EmptyValue) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.CronActorAddr)
	amountSlashed := big.Zero()

	var timedOutVerifiedDeals []*DealProposal

	var st State
	rt.StateTransaction(&st, func() {
		updatesNeeded := make(map[abi.ChainEpoch][]abi.DealID)

		msm, err := st.mutator(adt.AsStore(rt)).withDealStates(WritePermission).
			withLockedTable(WritePermission).withEscrowTable(WritePermission).withDealsByEpoch(WritePermission).
			withDealProposals(WritePermission).withPendingProposals(WritePermission).build()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load state")

		for i := st.LastCron + 1; i <= rt.CurrEpoch(); i++ {
			err = msm.dealsByEpoch.ForEach(i, func(dealID abi.DealID) error {
				deal, err := getDealProposal(msm.dealProposals, dealID)
				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to get dealId %d", dealID)

				dcid, err := deal.Cid()
				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to calculate CID for proposal %v", dealID)

				state, found, err := msm.dealStates.Get(dealID)
				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to get deal state")

				// deal has been published but not activated yet -> terminate it as it has timed out
				if !found {
					// Not yet appeared in proven sector; check for timeout.
					builtin.RequireState(rt, rt.CurrEpoch() >= deal.StartEpoch, "deal %d processed before start epoch %d",
						dealID, deal.StartEpoch)

					slashed := msm.processDealInitTimedOut(rt, deal)
					if !slashed.IsZero() {
						amountSlashed = big.Add(amountSlashed, slashed)
					}
					if deal.VerifiedDeal {
						timedOutVerifiedDeals = append(timedOutVerifiedDeals, deal)
					}

					// Delete the proposal (but not state, which doesn't exist).
					err = msm.dealProposals.Delete(dealID)
					builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to delete deal proposal %d", dealID)

					err = msm.pendingDeals.Delete(abi.CidKey(dcid))
					builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to delete pending proposal %d (%v)", dealID, dcid)
					return nil
				}

				// if this is the first cron tick for the deal, it should be in the pending state.
				if state.LastUpdatedEpoch == epochUndefined {
					pdErr := msm.pendingDeals.Delete(abi.CidKey(dcid))
					builtin.RequireNoErr(rt, pdErr, exitcode.ErrIllegalState, "failed to delete pending proposal %v", dcid)
				}

				slashAmount, nextEpoch, removeDeal := msm.updatePendingDealState(rt, state, deal, rt.CurrEpoch())
				builtin.RequireState(rt, slashAmount.GreaterThanEqual(big.Zero()), "computed negative slash amount %v for deal %d", slashAmount, dealID)

				if removeDeal {
					builtin.RequireState(rt, nextEpoch == epochUndefined, "removed deal %d should have no scheduled epoch (got %d)", dealID, nextEpoch)
					amountSlashed = big.Add(amountSlashed, slashAmount)

					// Delete proposal and state simultaneously.
					err = msm.dealStates.Delete(dealID)
					builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to delete deal state %d", dealID)
					err = msm.dealProposals.Delete(dealID)
					builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to delete deal proposal %d", dealID)
				} else {
					builtin.RequireState(rt, nextEpoch > rt.CurrEpoch(), "continuing deal %d next epoch %d should be in future", dealID, nextEpoch)
					builtin.RequireState(rt, slashAmount.IsZero(), "continuing deal %d should not be slashed", dealID)

					// Update deal's LastUpdatedEpoch in DealStates
					state.LastUpdatedEpoch = rt.CurrEpoch()
					err = msm.dealStates.Set(dealID, state)
					builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to set deal state")

					updatesNeeded[nextEpoch] = append(updatesNeeded[nextEpoch], dealID)
				}

				return nil
			})
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to iterate deal ops")

			err = msm.dealsByEpoch.RemoveAll(i)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to delete deal ops for epoch %v", i)
		}

		// Iterate changes in sorted order to ensure that loads/stores
		// are deterministic. Otherwise, we could end up charging an
		// inconsistent amount of gas.
		changedEpochs := make([]abi.ChainEpoch, 0, len(updatesNeeded))
		for epoch := range updatesNeeded { //nolint:nomaprange
			changedEpochs = append(changedEpochs, epoch)
		}

		sort.Slice(changedEpochs, func(i, j int) bool { return changedEpochs[i] < changedEpochs[j] })

		for _, epoch := range changedEpochs {
			err = msm.dealsByEpoch.PutMany(epoch, updatesNeeded[epoch])
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to reinsert deal IDs for epoch %v", epoch)
		}

		st.LastCron = rt.CurrEpoch()

		err = msm.commitState()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to flush state")
	})

	for _, d := range timedOutVerifiedDeals {
		code := rt.Send(
			builtin.VerifiedRegistryActorAddr,
			builtin.MethodsVerifiedRegistry.RestoreBytes,
			&verifreg.RestoreBytesParams{
				Address:  d.Client,
				DealSize: big.NewIntUnsigned(uint64(d.PieceSize)),
			},
			abi.NewTokenAmount(0),
			&builtin.Discard{},
		)

		if !code.IsSuccess() {
			rt.Log(rtt.ERROR, "failed to send RestoreBytes call to the VerifReg actor for timed-out verified deal, client: %s, dealSize: %v, "+
				"provider: %v, got code %v", d.Client, d.PieceSize, d.Provider, code)
		}
	}

	if !amountSlashed.IsZero() {
		e := rt.Send(builtin.BurntFundsActorAddr, builtin.MethodSend, nil, amountSlashed, &builtin.Discard{})
		builtin.RequireSuccess(rt, e, "expected send to burnt funds actor to succeed")
	}

	return nil
}

func GenRandNextEpoch(startEpoch abi.ChainEpoch, dealID abi.DealID) abi.ChainEpoch {
	offset := abi.ChainEpoch(uint64(dealID) % uint64(DealUpdatesInterval))
	q := builtin.NewQuantSpec(DealUpdatesInterval, 0)
	prevDay := q.QuantizeDown(startEpoch)
	if prevDay+offset >= startEpoch {
		return prevDay + offset
	}
	nextDay := q.QuantizeUp(startEpoch)
	return nextDay + offset
}

//
// Exported functions
//

// Validates a collection of deal dealProposals for activation, and returns their combined weight,
// split into regular deal weight and verified deal weight.
func ValidateDealsForActivation(
	st *State, store adt.Store, dealIDs []abi.DealID, minerAddr addr.Address, sectorExpiry, currEpoch abi.ChainEpoch,
) (big.Int, big.Int, uint64, error) {
	proposals, err := AsDealProposalArray(store, st.Proposals)
	if err != nil {
		return big.Int{}, big.Int{}, 0, xerrors.Errorf("failed to load dealProposals: %w", err)
	}

	return validateAndComputeDealWeight(proposals, dealIDs, minerAddr, sectorExpiry, currEpoch)
}

////////////////////////////////////////////////////////////////////////////////
// Checks
////////////////////////////////////////////////////////////////////////////////

func validateAndComputeDealWeight(proposals *DealArray, dealIDs []abi.DealID, minerAddr addr.Address,
	sectorExpiry abi.ChainEpoch, sectorActivation abi.ChainEpoch) (big.Int, big.Int, uint64, error) {

	seenDealIDs := make(map[abi.DealID]struct{}, len(dealIDs))
	totalDealSpace := uint64(0)
	totalDealSpaceTime := big.Zero()
	totalVerifiedSpaceTime := big.Zero()
	for _, dealID := range dealIDs {
		// Make sure we don't double-count deals.
		if _, seen := seenDealIDs[dealID]; seen {
			return big.Int{}, big.Int{}, 0, exitcode.ErrIllegalArgument.Wrapf("deal ID %d present multiple times", dealID)
		}
		seenDealIDs[dealID] = struct{}{}

		proposal, found, err := proposals.Get(dealID)
		if err != nil {
			return big.Int{}, big.Int{}, 0, xerrors.Errorf("failed to load deal %d: %w", dealID, err)
		}
		if !found {
			return big.Int{}, big.Int{}, 0, exitcode.ErrNotFound.Wrapf("no such deal %d", dealID)
		}
		if err = validateDealCanActivate(proposal, minerAddr, sectorExpiry, sectorActivation); err != nil {
			return big.Int{}, big.Int{}, 0, xerrors.Errorf("cannot activate deal %d: %w", dealID, err)
		}

		// Compute deal weight
		totalDealSpace += uint64(proposal.PieceSize)
		dealSpaceTime := DealWeight(proposal)
		if proposal.VerifiedDeal {
			totalVerifiedSpaceTime = big.Add(totalVerifiedSpaceTime, dealSpaceTime)
		} else {
			totalDealSpaceTime = big.Add(totalDealSpaceTime, dealSpaceTime)
		}
	}
	return totalDealSpaceTime, totalVerifiedSpaceTime, totalDealSpace, nil
}

func validateDealCanActivate(proposal *DealProposal, minerAddr addr.Address, sectorExpiration, sectorActivation abi.ChainEpoch) error {
	if proposal.Provider != minerAddr {
		return exitcode.ErrForbidden.Wrapf("proposal has provider %v, must be %v", proposal.Provider, minerAddr)
	}
	if sectorActivation > proposal.StartEpoch {
		return exitcode.ErrIllegalArgument.Wrapf("proposal start epoch %d has already elapsed at %d", proposal.StartEpoch, sectorActivation)
	}
	if proposal.EndEpoch > sectorExpiration {
		return exitcode.ErrIllegalArgument.Wrapf("proposal expiration %d exceeds sector expiration %d", proposal.EndEpoch, sectorExpiration)
	}
	return nil
}

func validateDeal(rt Runtime, deal ClientDealProposal, networkRawPower, networkQAPower, baselinePower abi.StoragePower) error {
	if err := dealProposalIsInternallyValid(rt, deal); err != nil {
		return xerrors.Errorf("Invalid deal proposal %w", err)
	}

	proposal := deal.Proposal

	if len(proposal.Label) > DealMaxLabelSize {
		return xerrors.Errorf("deal label can be at most %d bytes, is %d", DealMaxLabelSize, len(proposal.Label))
	}

	if err := proposal.PieceSize.Validate(); err != nil {
		return xerrors.Errorf("proposal piece size is invalid: %w", err)
	}

	if !proposal.PieceCID.Defined() {
		return xerrors.Errorf("proposal PieceCid undefined")
	}

	if proposal.PieceCID.Prefix() != PieceCIDPrefix {
		return xerrors.Errorf("proposal PieceCID had wrong prefix")
	}

	if proposal.EndEpoch <= proposal.StartEpoch {
		return xerrors.Errorf("proposal end before proposal start")
	}

	if rt.CurrEpoch() > proposal.StartEpoch {
		return xerrors.Errorf("Deal start epoch has already elapsed")
	}

	minDuration, maxDuration := DealDurationBounds(proposal.PieceSize)
	if proposal.Duration() < minDuration || proposal.Duration() > maxDuration {
		return xerrors.Errorf("Deal duration out of bounds")
	}

	minPrice, maxPrice := DealPricePerEpochBounds(proposal.PieceSize, proposal.Duration())
	if proposal.StoragePricePerEpoch.LessThan(minPrice) || proposal.StoragePricePerEpoch.GreaterThan(maxPrice) {
		return xerrors.Errorf("Storage price out of bounds")
	}

	minProviderCollateral, maxProviderCollateral := DealProviderCollateralBounds(proposal.PieceSize, proposal.VerifiedDeal,
		networkRawPower, networkQAPower, baselinePower, rt.TotalFilCircSupply())
	if proposal.ProviderCollateral.LessThan(minProviderCollateral) || proposal.ProviderCollateral.GreaterThan(maxProviderCollateral) {
		return xerrors.Errorf("Provider collateral out of bounds")
	}

	minClientCollateral, maxClientCollateral := DealClientCollateralBounds(proposal.PieceSize, proposal.Duration())
	if proposal.ClientCollateral.LessThan(minClientCollateral) || proposal.ClientCollateral.GreaterThan(maxClientCollateral) {
		return xerrors.Errorf("Client collateral out of bounds")
	}
	return nil
}

//
// Helpers
//

// Resolves a provider or client address to the canonical form against which a balance should be held, and
// the designated recipient address of withdrawals (which is the same, for simple account parties).
func escrowAddress(rt Runtime, address addr.Address) (nominal addr.Address, recipient addr.Address, approved []addr.Address) {
	// Resolve the provided address to the canonical form against which the balance is held.
	nominal, ok := rt.ResolveAddress(address)
	if !ok {
		rt.Abortf(exitcode.ErrIllegalArgument, "failed to resolve address %v", address)
	}

	codeID, ok := rt.GetActorCodeCID(nominal)
	if !ok {
		rt.Abortf(exitcode.ErrIllegalArgument, "no code for address %v", nominal)
	}

	if codeID.Equals(builtin.StorageMinerActorCodeID) {
		// Storage miner actor entry; implied funds recipient is the associated owner address.
		ownerAddr, workerAddr, _ := builtin.RequestMinerControlAddrs(rt, nominal)
		return nominal, ownerAddr, []addr.Address{ownerAddr, workerAddr}
	}

	return nominal, nominal, []addr.Address{nominal}
}

func getDealProposal(proposals *DealArray, dealID abi.DealID) (*DealProposal, error) {
	proposal, found, err := proposals.Get(dealID)
	if err != nil {
		return nil, xerrors.Errorf("failed to load proposal: %w", err)
	}
	if !found {
		return nil, exitcode.ErrNotFound.Wrapf("no such deal %d", dealID)
	}

	return proposal, nil
}

// Requests the current epoch target block reward from the reward actor.
func requestCurrentBaselinePower(rt Runtime) abi.StoragePower {
	var ret reward.ThisEpochRewardReturn
	code := rt.Send(builtin.RewardActorAddr, builtin.MethodsReward.ThisEpochReward, nil, big.Zero(), &ret)
	builtin.RequireSuccess(rt, code, "failed to check epoch baseline power")
	return ret.ThisEpochBaselinePower
}

// Requests the current network total power and pledge from the power actor.
func requestCurrentNetworkPower(rt Runtime) (rawPower, qaPower abi.StoragePower) {
	var pwr power.CurrentTotalPowerReturn
	code := rt.Send(builtin.StoragePowerActorAddr, builtin.MethodsPower.CurrentTotalPower, nil, big.Zero(), &pwr)
	builtin.RequireSuccess(rt, code, "failed to check current power")
	return pwr.RawBytePower, pwr.QualityAdjPower
}
```

[**`StorageMarketActorState` implementation**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_market\_actor.storagemarketactorstate-implementation)

**Storage Market Actor Statuses**

[Example: ](https://spec.filecoin.io/#example-)

```go
package market

import (
	"bytes"

	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/big"
	"github.com/filecoin-project/go-state-types/exitcode"
	"github.com/ipfs/go-cid"
	xerrors "golang.org/x/xerrors"

	"github.com/filecoin-project/specs-actors/v7/actors/builtin"
	"github.com/filecoin-project/specs-actors/v7/actors/util/adt"
)

const epochUndefined = abi.ChainEpoch(-1)

// BalanceLockingReason is the reason behind locking an amount.
type BalanceLockingReason int

const (
	ClientCollateral BalanceLockingReason = iota
	ClientStorageFee
	ProviderCollateral
)

// Bitwidth of AMTs determined empirically from mutation patterns and projections of mainnet data.
const ProposalsAmtBitwidth = 5
const StatesAmtBitwidth = 6

type State struct {
	// Proposals are deals that have been proposed and not yet cleaned up after expiry or termination.
	Proposals cid.Cid // AMT[DealID]DealProposal
	// States contains state for deals that have been activated and not yet cleaned up after expiry or termination.
	// After expiration, the state exists until the proposal is cleaned up too.
	// Invariant: keys(States) ⊆ keys(Proposals).
	States cid.Cid // AMT[DealID]DealState

	// PendingProposals tracks dealProposals that have not yet reached their deal start date.
	// We track them here to ensure that miners can't publish the same deal proposal twice
	PendingProposals cid.Cid // Set[DealCid]

	// Total amount held in escrow, indexed by actor address (including both locked and unlocked amounts).
	EscrowTable cid.Cid // BalanceTable

	// Amount locked, indexed by actor address.
	// Note: the amounts in this table do not affect the overall amount in escrow:
	// only the _portion_ of the total escrow amount that is locked.
	LockedTable cid.Cid // BalanceTable

	NextID abi.DealID

	// Metadata cached for efficient iteration over deals.
	DealOpsByEpoch cid.Cid // SetMultimap, HAMT[epoch]Set
	LastCron       abi.ChainEpoch

	// Total Client Collateral that is locked -> unlocked when deal is terminated
	TotalClientLockedCollateral abi.TokenAmount
	// Total Provider Collateral that is locked -> unlocked when deal is terminated
	TotalProviderLockedCollateral abi.TokenAmount
	// Total storage fee that is locked in escrow -> unlocked when payments are made
	TotalClientStorageFee abi.TokenAmount
}

func ConstructState(store adt.Store) (*State, error) {
	emptyProposalsArrayCid, err := adt.StoreEmptyArray(store, ProposalsAmtBitwidth)
	if err != nil {
		return nil, xerrors.Errorf("failed to create empty array: %w", err)
	}
	emptyStatesArrayCid, err := adt.StoreEmptyArray(store, StatesAmtBitwidth)
	if err != nil {
		return nil, xerrors.Errorf("failed to create empty states array: %w", err)
	}

	emptyPendingProposalsMapCid, err := adt.StoreEmptyMap(store, builtin.DefaultHamtBitwidth)
	if err != nil {
		return nil, xerrors.Errorf("failed to create empty map: %w", err)
	}
	emptyDealOpsHamtCid, err := StoreEmptySetMultimap(store, builtin.DefaultHamtBitwidth)
	if err != nil {
		return nil, xerrors.Errorf("failed to create empty multiset: %w", err)
	}
	emptyBalanceTableCid, err := adt.StoreEmptyMap(store, adt.BalanceTableBitwidth)
	if err != nil {
		return nil, xerrors.Errorf("failed to create empty balance table: %w", err)
	}

	return &State{
		Proposals:        emptyProposalsArrayCid,
		States:           emptyStatesArrayCid,
		PendingProposals: emptyPendingProposalsMapCid,
		EscrowTable:      emptyBalanceTableCid,
		LockedTable:      emptyBalanceTableCid,
		NextID:           abi.DealID(0),
		DealOpsByEpoch:   emptyDealOpsHamtCid,
		LastCron:         abi.ChainEpoch(-1),

		TotalClientLockedCollateral:   abi.NewTokenAmount(0),
		TotalProviderLockedCollateral: abi.NewTokenAmount(0),
		TotalClientStorageFee:         abi.NewTokenAmount(0),
	}, nil
}

////////////////////////////////////////////////////////////////////////////////
// Deal state operations
////////////////////////////////////////////////////////////////////////////////

func (m *marketStateMutation) updatePendingDealState(rt Runtime, state *DealState, deal *DealProposal, epoch abi.ChainEpoch) (amountSlashed abi.TokenAmount, nextEpoch abi.ChainEpoch, removeDeal bool) {
	amountSlashed = abi.NewTokenAmount(0)

	everUpdated := state.LastUpdatedEpoch != epochUndefined
	everSlashed := state.SlashEpoch != epochUndefined

	builtin.RequireState(rt, !everUpdated || (state.LastUpdatedEpoch <= epoch), "deal updated at future epoch %d", state.LastUpdatedEpoch)

	// This would be the case that the first callback somehow triggers before it is scheduled to
	// This is expected not to be able to happen
	if deal.StartEpoch > epoch {
		return amountSlashed, epochUndefined, false
	}

	paymentEndEpoch := deal.EndEpoch
	if everSlashed {
		builtin.RequireState(rt, epoch >= state.SlashEpoch, "current epoch less than deal slash epoch %d", state.SlashEpoch)
		builtin.RequireState(rt, state.SlashEpoch <= deal.EndEpoch, "deal slash epoch %d after deal end %d", state.SlashEpoch, deal.EndEpoch)
		paymentEndEpoch = state.SlashEpoch
	} else if epoch < paymentEndEpoch {
		paymentEndEpoch = epoch
	}

	paymentStartEpoch := deal.StartEpoch
	if everUpdated && state.LastUpdatedEpoch > paymentStartEpoch {
		paymentStartEpoch = state.LastUpdatedEpoch
	}

	numEpochsElapsed := paymentEndEpoch - paymentStartEpoch

	{
		// Process deal payment for the elapsed epochs.
		totalPayment := big.Mul(big.NewInt(int64(numEpochsElapsed)), deal.StoragePricePerEpoch)

		// the transfer amount can be less than or equal to zero if a deal is slashed before or at the deal's start epoch.
		if totalPayment.GreaterThan(big.Zero()) {
			err := m.transferBalance(deal.Client, deal.Provider, totalPayment)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to transfer %v from %v to %v",
				totalPayment, deal.Client, deal.Provider)
		}
	}

	if everSlashed {
		// unlock client collateral and locked storage fee
		paymentRemaining, err := dealGetPaymentRemaining(deal, state.SlashEpoch)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to compute remaining payment")

		// unlock remaining storage fee
		err = m.unlockBalance(deal.Client, paymentRemaining, ClientStorageFee)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to unlock remaining client storage fee")

		// unlock client collateral
		err = m.unlockBalance(deal.Client, deal.ClientCollateral, ClientCollateral)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to unlock client collateral")

		// slash provider collateral
		amountSlashed = deal.ProviderCollateral
		err = m.slashBalance(deal.Provider, amountSlashed, ProviderCollateral)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "slashing balance")
		return amountSlashed, epochUndefined, true
	}

	if epoch >= deal.EndEpoch {
		m.processDealExpired(rt, deal, state)
		return amountSlashed, epochUndefined, true
	}

	// We're explicitly not inspecting the end epoch and may process a deal's expiration late, in order to prevent an outsider
	// from loading a cron tick by activating too many deals with the same end epoch.
	nextEpoch = epoch + DealUpdatesInterval

	return amountSlashed, nextEpoch, false
}

// Deal start deadline elapsed without appearing in a proven sector.
// Slash a portion of provider's collateral, and unlock remaining collaterals
// for both provider and client.
func (m *marketStateMutation) processDealInitTimedOut(rt Runtime, deal *DealProposal) abi.TokenAmount {
	if err := m.unlockBalance(deal.Client, deal.TotalStorageFee(), ClientStorageFee); err != nil {
		rt.Abortf(exitcode.ErrIllegalState, "failure unlocking client storage fee: %s", err)
	}
	if err := m.unlockBalance(deal.Client, deal.ClientCollateral, ClientCollateral); err != nil {
		rt.Abortf(exitcode.ErrIllegalState, "failure unlocking client collateral: %s", err)
	}

	amountSlashed := CollateralPenaltyForDealActivationMissed(deal.ProviderCollateral)
	amountRemaining := big.Sub(deal.ProviderBalanceRequirement(), amountSlashed)

	if err := m.slashBalance(deal.Provider, amountSlashed, ProviderCollateral); err != nil {
		rt.Abortf(exitcode.ErrIllegalState, "failed to slash balance: %s", err)
	}

	if err := m.unlockBalance(deal.Provider, amountRemaining, ProviderCollateral); err != nil {
		rt.Abortf(exitcode.ErrIllegalState, "failed to unlock deal provider balance: %s", err)
	}

	return amountSlashed
}

// Normal expiration. Unlock collaterals for both provider and client.
func (m *marketStateMutation) processDealExpired(rt Runtime, deal *DealProposal, state *DealState) {
	builtin.RequireState(rt, state.SectorStartEpoch != epochUndefined, "sector start epoch undefined")

	// Note: payment has already been completed at this point (_rtProcessDealPaymentEpochsElapsed)
	err := m.unlockBalance(deal.Provider, deal.ProviderCollateral, ProviderCollateral)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed unlocking deal provider balance")

	err = m.unlockBalance(deal.Client, deal.ClientCollateral, ClientCollateral)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed unlocking deal client balance")
}

func (m *marketStateMutation) generateStorageDealID() abi.DealID {
	ret := m.nextDealId
	m.nextDealId = m.nextDealId + abi.DealID(1)
	return ret
}

////////////////////////////////////////////////////////////////////////////////
// State utility functions
////////////////////////////////////////////////////////////////////////////////

func dealProposalIsInternallyValid(rt Runtime, proposal ClientDealProposal) error {
	// Note: we do not verify the provider signature here, since this is implicit in the
	// authenticity of the on-chain message publishing the deal.
	buf := bytes.Buffer{}
	err := proposal.Proposal.MarshalCBOR(&buf)
	if err != nil {
		return xerrors.Errorf("proposal signature verification failed to marshal proposal: %w", err)
	}
	err = rt.VerifySignature(proposal.ClientSignature, proposal.Proposal.Client, buf.Bytes())
	if err != nil {
		return xerrors.Errorf("signature proposal invalid: %w", err)
	}
	return nil
}

func dealGetPaymentRemaining(deal *DealProposal, slashEpoch abi.ChainEpoch) (abi.TokenAmount, error) {
	if slashEpoch > deal.EndEpoch {
		return big.Zero(), xerrors.Errorf("deal slash epoch %d after end epoch %d", slashEpoch, deal.EndEpoch)
	}

	// Payments are always for start -> end epoch irrespective of when the deal is slashed.
	if slashEpoch < deal.StartEpoch {
		slashEpoch = deal.StartEpoch
	}

	durationRemaining := deal.EndEpoch - slashEpoch
	if durationRemaining < 0 {
		return big.Zero(), xerrors.Errorf("deal remaining duration negative: %d", durationRemaining)
	}

	return big.Mul(big.NewInt(int64(durationRemaining)), deal.StoragePricePerEpoch), nil
}

// MarketStateMutationPermission is the mutation permission on a state field
type MarketStateMutationPermission int

const (
	// Invalid means NO permission
	Invalid MarketStateMutationPermission = iota
	// ReadOnlyPermission allows reading but not mutating the field
	ReadOnlyPermission
	// WritePermission allows mutating the field
	WritePermission
)

type marketStateMutation struct {
	st    *State
	store adt.Store

	proposalPermit MarketStateMutationPermission
	dealProposals  *DealArray

	statePermit MarketStateMutationPermission
	dealStates  *DealMetaArray

	escrowPermit MarketStateMutationPermission
	escrowTable  *adt.BalanceTable

	pendingPermit MarketStateMutationPermission
	pendingDeals  *adt.Set

	dpePermit    MarketStateMutationPermission
	dealsByEpoch *SetMultimap

	lockedPermit                  MarketStateMutationPermission
	lockedTable                   *adt.BalanceTable
	totalClientLockedCollateral   abi.TokenAmount
	totalProviderLockedCollateral abi.TokenAmount
	totalClientStorageFee         abi.TokenAmount

	nextDealId abi.DealID
}

func (s *State) mutator(store adt.Store) *marketStateMutation {
	return &marketStateMutation{st: s, store: store}
}

func (m *marketStateMutation) build() (*marketStateMutation, error) {
	if m.proposalPermit != Invalid {
		proposals, err := AsDealProposalArray(m.store, m.st.Proposals)
		if err != nil {
			return nil, xerrors.Errorf("failed to load deal proposals: %w", err)
		}
		m.dealProposals = proposals
	}

	if m.statePermit != Invalid {
		states, err := AsDealStateArray(m.store, m.st.States)
		if err != nil {
			return nil, xerrors.Errorf("failed to load deal state: %w", err)
		}
		m.dealStates = states
	}

	if m.lockedPermit != Invalid {
		lt, err := adt.AsBalanceTable(m.store, m.st.LockedTable)
		if err != nil {
			return nil, xerrors.Errorf("failed to load locked table: %w", err)
		}
		m.lockedTable = lt
		m.totalClientLockedCollateral = m.st.TotalClientLockedCollateral.Copy()
		m.totalClientStorageFee = m.st.TotalClientStorageFee.Copy()
		m.totalProviderLockedCollateral = m.st.TotalProviderLockedCollateral.Copy()
	}

	if m.escrowPermit != Invalid {
		et, err := adt.AsBalanceTable(m.store, m.st.EscrowTable)
		if err != nil {
			return nil, xerrors.Errorf("failed to load escrow table: %w", err)
		}
		m.escrowTable = et
	}

	if m.pendingPermit != Invalid {
		pending, err := adt.AsSet(m.store, m.st.PendingProposals, builtin.DefaultHamtBitwidth)
		if err != nil {
			return nil, xerrors.Errorf("failed to load pending proposals: %w", err)
		}
		m.pendingDeals = pending
	}

	if m.dpePermit != Invalid {
		dbe, err := AsSetMultimap(m.store, m.st.DealOpsByEpoch, builtin.DefaultHamtBitwidth, builtin.DefaultHamtBitwidth)
		if err != nil {
			return nil, xerrors.Errorf("failed to load deals by epoch: %w", err)
		}
		m.dealsByEpoch = dbe
	}

	m.nextDealId = m.st.NextID

	return m, nil
}

func (m *marketStateMutation) withDealProposals(permit MarketStateMutationPermission) *marketStateMutation {
	m.proposalPermit = permit
	return m
}

func (m *marketStateMutation) withDealStates(permit MarketStateMutationPermission) *marketStateMutation {
	m.statePermit = permit
	return m
}

func (m *marketStateMutation) withEscrowTable(permit MarketStateMutationPermission) *marketStateMutation {
	m.escrowPermit = permit
	return m
}

func (m *marketStateMutation) withLockedTable(permit MarketStateMutationPermission) *marketStateMutation {
	m.lockedPermit = permit
	return m
}

func (m *marketStateMutation) withPendingProposals(permit MarketStateMutationPermission) *marketStateMutation {
	m.pendingPermit = permit
	return m
}

func (m *marketStateMutation) withDealsByEpoch(permit MarketStateMutationPermission) *marketStateMutation {
	m.dpePermit = permit
	return m
}

func (m *marketStateMutation) commitState() error {
	var err error
	if m.proposalPermit == WritePermission {
		if m.st.Proposals, err = m.dealProposals.Root(); err != nil {
			return xerrors.Errorf("failed to flush deal dealProposals: %w", err)
		}
	}

	if m.statePermit == WritePermission {
		if m.st.States, err = m.dealStates.Root(); err != nil {
			return xerrors.Errorf("failed to flush deal states: %w", err)
		}
	}

	if m.lockedPermit == WritePermission {
		if m.st.LockedTable, err = m.lockedTable.Root(); err != nil {
			return xerrors.Errorf("failed to flush locked table: %w", err)
		}
		m.st.TotalClientLockedCollateral = m.totalClientLockedCollateral.Copy()
		m.st.TotalProviderLockedCollateral = m.totalProviderLockedCollateral.Copy()
		m.st.TotalClientStorageFee = m.totalClientStorageFee.Copy()
	}

	if m.escrowPermit == WritePermission {
		if m.st.EscrowTable, err = m.escrowTable.Root(); err != nil {
			return xerrors.Errorf("failed to flush escrow table: %w", err)
		}
	}

	if m.pendingPermit == WritePermission {
		if m.st.PendingProposals, err = m.pendingDeals.Root(); err != nil {
			return xerrors.Errorf("failed to flush pending deals: %w", err)
		}
	}

	if m.dpePermit == WritePermission {
		if m.st.DealOpsByEpoch, err = m.dealsByEpoch.Root(); err != nil {
			return xerrors.Errorf("failed to flush deals by epoch: %w", err)
		}
	}

	m.st.NextID = m.nextDealId
	return nil
}
```

**Storage Market Actor Balance states and mutations**

[Example: ](https://spec.filecoin.io/#example-)

```go
package market

import (
	addr "github.com/filecoin-project/go-address"
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/big"
	"github.com/filecoin-project/go-state-types/exitcode"
	"golang.org/x/xerrors"
)

func (m *marketStateMutation) lockClientAndProviderBalances(proposal *DealProposal) error {
	if err := m.maybeLockBalance(proposal.Client, proposal.ClientBalanceRequirement()); err != nil {
		return xerrors.Errorf("failed to lock client funds: %w", err)
	}
	if err := m.maybeLockBalance(proposal.Provider, proposal.ProviderCollateral); err != nil {
		return xerrors.Errorf("failed to lock provider funds: %w", err)
	}

	m.totalClientLockedCollateral = big.Add(m.totalClientLockedCollateral, proposal.ClientCollateral)
	m.totalClientStorageFee = big.Add(m.totalClientStorageFee, proposal.TotalStorageFee())
	m.totalProviderLockedCollateral = big.Add(m.totalProviderLockedCollateral, proposal.ProviderCollateral)
	return nil
}

func (m *marketStateMutation) unlockBalance(addr addr.Address, amount abi.TokenAmount, lockReason BalanceLockingReason) error {
	if amount.LessThan(big.Zero()) {
		return xerrors.Errorf("unlock negative amount %v", amount)
	}

	err := m.lockedTable.MustSubtract(addr, amount)
	if err != nil {
		return xerrors.Errorf("subtracting from locked balance: %w", err)
	}

	switch lockReason {
	case ClientCollateral:
		m.totalClientLockedCollateral = big.Sub(m.totalClientLockedCollateral, amount)
	case ClientStorageFee:
		m.totalClientStorageFee = big.Sub(m.totalClientStorageFee, amount)
	case ProviderCollateral:
		m.totalProviderLockedCollateral = big.Sub(m.totalProviderLockedCollateral, amount)
	}

	return nil
}

// move funds from locked in client to available in provider
func (m *marketStateMutation) transferBalance(fromAddr addr.Address, toAddr addr.Address, amount abi.TokenAmount) error {
	if amount.LessThan(big.Zero()) {
		return xerrors.Errorf("transfer negative amount %v", amount)
	}
	if err := m.escrowTable.MustSubtract(fromAddr, amount); err != nil {
		return xerrors.Errorf("subtract from escrow: %w", err)
	}
	if err := m.unlockBalance(fromAddr, amount, ClientStorageFee); err != nil {
		return xerrors.Errorf("subtract from locked: %w", err)
	}
	if err := m.escrowTable.Add(toAddr, amount); err != nil {
		return xerrors.Errorf("add to escrow: %w", err)
	}
	return nil
}

func (m *marketStateMutation) slashBalance(addr addr.Address, amount abi.TokenAmount, reason BalanceLockingReason) error {
	if amount.LessThan(big.Zero()) {
		return xerrors.Errorf("negative amount to slash: %v", amount)
	}

	if err := m.escrowTable.MustSubtract(addr, amount); err != nil {
		return xerrors.Errorf("subtract from escrow: %v", err)
	}

	return m.unlockBalance(addr, amount, reason)
}

func (m *marketStateMutation) maybeLockBalance(addr addr.Address, amount abi.TokenAmount) error {
	if amount.LessThan(big.Zero()) {
		return xerrors.Errorf("cannot lock negative amount %v", amount)
	}

	prevLocked, err := m.lockedTable.Get(addr)
	if err != nil {
		return xerrors.Errorf("failed to get locked balance: %w", err)
	}

	escrowBalance, err := m.escrowTable.Get(addr)
	if err != nil {
		return xerrors.Errorf("failed to get escrow balance: %w", err)
	}

	if big.Add(prevLocked, amount).GreaterThan(escrowBalance) {
		return exitcode.ErrInsufficientFunds.Wrapf("insufficient balance for addr %s: escrow balance %s < locked %s + required %s",
			addr, escrowBalance, prevLocked, amount)
	}

	if err := m.lockedTable.Add(addr, amount); err != nil {
		return xerrors.Errorf("failed to add locked balance: %w", err)
	}
	return nil
}

// Return true when the funds in escrow for the input address can cover an additional lockup of amountToLock
func (m *marketStateMutation) balanceCovered(addr addr.Address, amountToLock abi.TokenAmount) (bool, error) {
	prevLocked, err := m.lockedTable.Get(addr)
	if err != nil {
		return false, xerrors.Errorf("failed to get locked balance: %w", err)
	}
	escrowBalance, err := m.escrowTable.Get(addr)
	if err != nil {
		return false, xerrors.Errorf("failed to get escrow balance: %w", err)
	}
	return big.Add(prevLocked, amountToLock).LessThanEqual(escrowBalance), nil
}
```

[**Storage Deal Collateral**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_market\_actor.storage-deal-collateral)

Apart from [Initial Pledge Collateral and Block Reward Collateral](https://spec.filecoin.io/#section-systems.filecoin\_mining.miner\_collaterals) discussed earlier, the third form of collateral is provided by the storage provider to _collateralize deals_, is called _Storage Deal Collateral_ and is held in the `StorageMarketActor`.

There is a minimum amount of collateral required by the protocol to provide a minimum level of guarantee, which is agreed upon by the storage provider and client off-chain. However, miners can offer a higher deal collateral to imply a higher level of service and reliability to potential clients. Given the increased stakes, clients may associate additional provider deal collateral beyond the minimum with an increased likelihood that their data will be reliably stored.

Provider deal collateral is only slashed when a sector is terminated before the deal expires. If a miner enters Temporary Fault for a sector and later recovers from it, no deal collateral will be slashed.

This collateral is returned to the storage provider when all deals in the sector successfully conclude. Upon graceful deal expiration, storage providers must wait for finality number of epochs (as defined in [Finality](https://spec.filecoin.io/#section-algorithms.expected\_consensus.finality-in-ec)) before being able to withdraw their `StorageDealCollateral` from the `StorageMarketActor`.

```
MinimumProviderDealCollateral=1%×FILCirculatingSupply×DealRawBytemax(NetworkBaseline,NetworkRawBytePower)MinimumProviderDealCollateral=1%×FILCirculatingSupply×max(NetworkBaseline,NetworkRawBytePower)DealRawByte​
```

[**Storage Deal Flow**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_deal\_flow)

<figure><img src="https://spec.filecoin.io/_gen/diagrams/systems/filecoin_markets/onchain_storage_market/diagrams/deal-flow.svg?1639060809" alt="Deal Flow Sequence Diagram" height="100%"><figcaption><p><a href="https://spec.filecoin.io/#figure-deal-flow-sequence-diagram">Figure: Deal Flow Sequence Diagram</a> </p></figcaption></figure>

[**Add Storage Deal and Power**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_deal\_flow.add-storage-deal-and-power)

1. `StorageClient` and `StorageProvider` call `StorageMarketActor.AddBalance` to deposit funds into Storage Market.
   * `StorageClient` and `StorageProvider` can call `WithdrawBalance` before any deal is made.
2. `StorageClient` and `StorageProvider` negotiate a deal off chain. `StorageClient` sends a `StorageDealProposal` to a `StorageProvider`.
   * `StorageProvider` verifies the `StorageDeal` by checking: - the address and signature of the `StorageClient`, - the proposal’s `StartEpoch` is after the current Epoch, - (tentative) the `StorageClient` did not call withdraw in the last _X_ epochs (`WithdrawBalance` should take at least _X_ epochs) - _X_ is currently set to 0, but the setting will be re-considered in the near future. - both `StorageProvider` and `StorageClient` have sufficient available balances in `StorageMarketActor`.
3. `StorageProvider` signs the `StorageDealProposal` by constructing an on-chain message.
   * `StorageProvider` calls `PublishStorageDeals` in `StorageMarketActor` to publish this on-chain message which will generate a `DealID` for each `StorageDeal` and store a mapping from `DealID` to `StorageDeal`. However, the deals are not active at this point.
     * As a backup, `StorageClient` may call `PublishStorageDeals` with the `StorageDeal`, to activate the deal if they can obtain the signed on-chain message from `StorageProvider`.
     * It is possible for either `StorageProvider` or `StorageClient` to try to enter into two deals simultaneously with funds available only for one. Only the first deal to commit to the chain will clear, the second will fail with error `errorcode.InsufficientFunds`.
   * `StorageProvider` calls `HandleStorageDeal` in `StorageMiningSubsystem` which will then add the `StorageDeal` into a `Sector`.

[**Sealing sectors**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_deal\_flow.sealing-sectors)

4. Once a miner finishes packing a `Sector`, it generates a `SectorPreCommitInfo` and calls `PreCommitSector` or `PreCommitSectorBatch` with a `PreCommitDeposit`. It must call `ProveCommitSector` or `ProveCommitAggregate with` SectorProveCommitInfo`within some bound to recover the deposit. Initial pledge will then be required at time of`ProveCommit`. Initial Pledge is usually higher than` PreCommitDeposit`. Recovered` PreCommitDeposit`will count towards Initial Pledge and miners only need to top up additional funds at`ProveCommit`. Excess` PreCommitDeposit`, when it is greater than Initial Pledge, will be returned to the miner. An expired` PreCommit`message will result in`PreCommitDeposit`being burned. All Sectors have an explicit expiration epoch declared during`PreCommit`. For sectors with deals, all deals must expire before sector expiration. The Miner gains power for this particular sector upon successful` ProveCommit\`. For more details on the Sectors and the different types of deals that can be included in a Sector refer to the [Sector section](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector).

[**Prove Storage**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_deal\_flow.prove-storage)

5. Miners have to prove that they hold unique copies of Sectors by submitting proofs according to the [Proof of SpaceTime](https://spec.filecoin.io/#section-algorithms.pos.post) algorithm. Miners have to prove all their Sectors in regular time intervals in order for the system to guarantee that they indeed store the data they committed to store in the deal phase.

[**Declare and Recover Faults**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_deal\_flow.declare-and-recover-faults)

6. Miners can call `DeclareFaults` to mark certain Sectors as faulty to avoid paying Sector Fault Detection Fee. Power associated with the sector will be removed at fault declaration.
7. Miners can call `DeclareFaultsRecovered` to mark previously faulty sector as recovered. Power will be restored when recovered sectors pass WindowPoSt checks successfully.
8. A sector pays a Sector Fault Fee for every proving period during which it is marked as faulty.

[**Skipped Faults**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_deal\_flow.skipped-faults)

9. After a WindowPoSt deadline opens, a miner can mark one of their sectors as faulty and exempted by WindowPoSt checks, hence Skipped Faults. This could avoid paying a Sector Fault Detection Fee on the whole partition.

[**Detected Faults**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_deal\_flow.detected-faults)

10. If a partition misses a WindowPoSt submission deadline, all previously non-faulty sectors in the partition are detected as faulty and a Fault Detection Fee is charged.

[**Sector Expiration**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_deal\_flow.sector-expiration)

11. Sector expires when its expiration epoch is reached and sector expiration epoch must be greater than the expiration epoch of all its deals.

[**Sector Termination**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_deal\_flow.sector-termination)

12. Termination of a sector can be triggered in two ways. One when sector remains faulty for 42 consecutive days and the other when a miner initiates a termination by calling `TerminateSectors`. In both cases, a `TerminationFee` is penalized, which is in principle equivalent to how much the sector has earned so far. Miners are also penalized for the `DealCollateral` that the sector contains and remaining `DealPayment` will be returned to clients.

[**Deal Payment and slashing**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_deal\_flow.deal-payment-and-slashing)

13. Deal payment and slashing are evaluated lazily through `updatePendingDealState` called at `CronTick`.

[**Storage Deal States**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_deal\_states)

All on-chain economic activities in Filecoin start with the storage deal. This section aims to explain different states of a storage deal and their relationship with other concepts in the protocol such as Power, Payment, and Collaterals.

A deal has the following states:

* `Unpublished`: the deal has yet to be posted on chain.
* `Published`: the deal has been published and accepted by the chain but is not yet active as the sector containing the deal has not been proven.
* `Active`: the deal has been proven and not yet expired.
* `Deleted`: the deal has expired or the sector containing the deal has been terminated because of faults.

Note that `Unpublished` and `Deleted` states are not tracked on chain. To reduce on-chain footprint, an `OnChainDeal` struct is created when a deal is published and it keeps track of a `LastPaymentEpoch` which defaults to -1 when a deal is in the `Published` state. A deal transitions into the `Active` state when `LastPaymentEpoch` is positive.

The following describes how a deal transitions between its different states. These states in the list below are **on-chain states** understood by the actor/VM logic.

* `Unpublished -> Published`: this is triggered by `StorageMarketActor.PublishStorageDeals` which validates new storage deals, locks necessary funds, generates deal IDs, and registers the storage deals in `StorageMarketActor`.
* `Published -> Deleted`: this is triggered by `StorageMinerActor.ProveCommitSector` or `StorageMinerActor.ProveCommitAggregate` during InteractivePoRep when the elapsed number of epochs between PreCommit and ProveCommit messages exceeds `MAX_PROVE_COMMIT_SECTOR_EPOCH`. The ProveCommit message will also trigger garbage collection on the list of published storage deals.
* `Published -> Active`: this is triggered by `ActivateStorageDeals` after successful `StorageMinerActor.ProveCommitSector` or `StorageMinerActor.ProveCommitAggregate`. It is okay for the StorageDeal to have already started (i.e. for `StartEpoch` to have passed) at this point but it must not have expired.
* `Active -> Deleted`: this can happen under the following conditions:
  * The deal itself has expired. This is triggered by `StorageMinerActorCode._submitPowerReport` which is called whenever a PoSt is submitted. Power associated with the deal will be lost, collaterals returned, and all remaining storage fees unlocked (allowing miners to call `WithdrawBalance` successfully).
  * The sector containing the deal has expired. This is triggered by `StorageMinerActorCode._submitPowerReport` which is called whenver a PoSt is submitted. Power associated with the deals in the sector will be lost, collaterals returned, and all remaining storage fees unlocked.
  * The sector containing the active deal has been terminated. This is triggered by `StorageMinerActor._submitFaultReport` for `TerminatedFaults`. No storage deal collateral will be slashed on fault declaration or detection, only on termination. A terminated fault is triggered when a sector is in the `Failing` state for `MAX_CONSECUTIVE_FAULTS` consecutive proving periods.

Given the **onchain deal states and their transitions** discussed above, below is a description of the relationships between **onchain deal states** and other economic states and activities in the protocol.

* `Power`: only payload data in an Active storage deal counts towards power.
* `Deal Payment`: happens on `_onSuccessfulPoSt` and at deal/sector expiration through `_submitPowerReport`, paying out `StoragePricePerEpoch` for each epoch since the last PoSt.
* `Deal Collateral`: no storage deal collateral will be slashed for `NewDeclaredFaults` and `NewDetectedFaults` but instead some pledge collateral will be slashed given these faults' impact on consensus power. In the event of `NewTerminatedFaults`, all storage deal collateral and some pledge collateral will be slashed. Provider and client storage deal collaterals will be returned when a deal or a sector has expired. If a sector recovers from `Failing` within the `MAX_CONSECUTIVE_FAULTS` threshold, deals in that sector are still considered active. However, miners may need to top up pledge collateral when they try to `RecoverFaults` given the earlier slashing.

<figure><img src="https://spec.filecoin.io/_gen/diagrams/systems/filecoin_markets/onchain_storage_market/diagrams/deal-payment.svg?1639060809" alt="Deal States Sequence Diagram" height="100%"><figcaption><p><a href="https://spec.filecoin.io/#figure-deal-states-sequence-diagram">Figure: Deal States Sequence Diagram</a> </p></figcaption></figure>

[**Faults**](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.faults)

There are two main categories of faults in the Filecoin network:

1. [Storage or Sector Faults](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector.sector-faults) that relate with the failure to store files agreed in a deal previously due to a hardware error or malicious behaviour, and
2. [Consensus Faults](https://spec.filecoin.io/#section-algorithms.expected\_consensus.consensus-faults) that relate to a miner trying deviate from the protocol in order to gain more power than their storage deserves.

Please refer to the corresponding sections for more details.

Both Storage and Consensus Faults come with penalties that slash the miner’s collateral. See more details on the different types of collaterals in the [Miner Collaterals](https://spec.filecoin.io/#section-systems.filecoin\_mining.miner\_collaterals).

#### [Retrieval Market in Filecoin](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market) <a href="#section-systems.filecoin_markets.retrieval_market" id="section-systems.filecoin_markets.retrieval_market"></a>

[**Components**](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market.components)

The `retrieval market` refers to the process of negotiating deals for a provider to serve stored data to a client. It should be highlighted that the negotiation process for the retrieval happens primarily off-chain. It is only some parts of it (mostly relating to redeeming vouchers from payment channels) that involve interaction with the blockchain.

The main components are as follows:

* A [payment channel actor](https://spec.filecoin.io/#section-systems.filecoin\_token.payment\_channels.payment-channel-actor)
* A protocol for making queries
* A Data Transfer subsystem and protocol used to query retrieval miners and initiate retrieval deals
* A chain-based content routing interface
* A client module to query retrieval miners and initiate deals for retrieval
* A provider module to respond to queries and deal proposals

The retrieval market operate by piggybacking on the Data Transfer system and Graphsync to handle transfer and verification, to support arbitrary selectors, and to reduce round trips. The retrieval market can support sending arbitrary payload CIDs & selectors within a piece.

The Data Transfer System is augmented accordingly to support pausing/resuming and sending intermediate vouchers to facilitate this.

[**Deal Flow in the Retrieval Market**](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market.deal-flow-in-the-retrieval-market)

<figure><img src="https://spec.filecoin.io/_gen/diagrams/systems/filecoin_markets/retrieval_market/retrieval_flow_v1.svg?1639060809" alt="Retrieval Flow" height="100%"><figcaption><p><a href="https://spec.filecoin.io/#figure-retrieval-flow">Figure: Retrieval Flow</a> </p></figcaption></figure>

The Filecoin Retrieval Market protocol for proposing and accepting a deal works as follows:

* The client finds a provider of a given piece with `FindProviders()`.
* The client queries a provider to see if it meets its retrieval criteria (via Query Protocol)
* The client schedules a `Data Transfer Pull Request` passing the `RetrievalDealProposal` as a voucher.
* The provider validates the proposal and rejects it if it is invalid.
* If the proposal is valid, the provider responds with an accept message and begins monitoring the data transfer process.
* The client creates a payment channel as necessary and a “lane” and ensures there are enough funds in the channel.
* The provider unseals the sector as necessary.
* The provider monitors data transfer as it sends blocks over the protocol, until it requires payment.
* When the provider requires payment, it pauses the data transfer and sends a request for payment as an intermediate voucher.
* The client receives the request for payment.
* The client creates and stores a payment voucher off-chain.
* The client responds to the provider with a reference to the payment voucher, sent as an intermediate voucher (i.e., acknowledging receipt of a part of the data and channel or lane value).
* The provider validates the voucher sent by the client and saves it to be redeemed on-chain later
* The provider resumes sending data and requesting intermediate payments.
* The process continues until the end of the data transfer.

Some extra notes worth making with regard to the above process are as follows:

* The payment channel is created by the client.
* The payment channel is created when the provider accepts the deal, unless an open payment channel already exists between the given client and provider.
* The vouchers are also created by the client and (a reference/identifier to these vouchers is) sent to the provider.
* The payment indicated in the voucher is not taken out of the payment channel funds upon creation and exchange of vouchers between the client and the provider.
* In order for money to be transferred to the provider’s payment channel side, the provider has to _redeem_ the voucher
* In order for money to be taken out of the payment channel, the provider has to submit the voucher on-chain and `Collect` the funds.
* Both redeeming and collecting vouchers/funds can be done at any time during the data transfer, but redeeming vouchers and collecting funds involves the blockchain, which further means that it incurs gas cost.
* Once the data transfer is complete, the client or provider may Settle the channel. There is then a 12hr period within which the provider has to submit the redeemed vouchers on-chain in order to collect the funds. Once the 12hr period is complete, the client may collect any unclaimed funds from the channel, and the provider loses the funds for vouchers they did not submit.
* The provider can ask for a small payment ahead of the transfer, before they start unsealing data. The payment is meant to support the providers' computational cost of unsealing the first chunk of data (where chunk is the agreed step-wise data transfer). This process is needed in order to avoid clients from carrying out a DoS attack, according to which they start several deals and cause the provider to engage a large amount of computational resources.

[**Bootstrapping Trust**](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market.bootstrapping-trust)

Neither the client nor the provider have any specific reason to trust each other. Therefore, trust is established indirectly by payments for a retrieval deal done _incrementally_. This is achieved by sending vouchers as the data transfer progresses.

Trust establishment proceeds as follows:

* When the deal is created, client & provider agree to a “payment interval” in bytes, which is the _minimum_ amount of data the provider will send before each required increment.
* They also agree to a “payment interval increment”. This means that the interval will increase by this value after each successful transfer and payment, as trust develops between client and provider.
* Example:
  * If my “payment interval” is 1000, and my “payment interval increase” is 300, then:
  * The provider must send at least 1000 bytes before they require any payment (they may end up sending slightly more because block boundaries are uneven).
  * The client must pay (i.e., issue a voucher) for all bytes sent when the provider requests payment, provided that the provider has sent at least 1000 bytes.
  * The provider now must send at least 1300 bytes before they request payment again.
  * The client must pay (i.e., issue subsequent vouchers) for all bytes it has not yet paid for when the provider requests payment, assuming it has received at least 1300 bytes since last payment.
  * The process continues until the end of the retrieval, when the last payment will simply be for the remainder of bytes.

[**Data Representation in the Retrieval Market**](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market.data-representation-in-the-retrieval-market)

The retrieval market works based on the Payload CID. The PayloadCID is the hash that represents the root of the IPLD DAG of the UnixFS version of the file. At this stage the file is a raw system file with IPFS-style representation. In order for a client to request for some data under the retrieval market, they have to know the PayloadCID. It is important to highlight that PayloadCIDs are not stored or registered on-chain.

[Example: Retrieval Market - Common Data Types ](https://spec.filecoin.io/#example-retrieval-market---common-data-types)

```go
package retrievalmarket

import (
	"bytes"
	"errors"
	"fmt"

	"github.com/ipfs/go-cid"
	"github.com/ipld/go-ipld-prime"
	"github.com/ipld/go-ipld-prime/codec/dagcbor"
	"github.com/libp2p/go-libp2p-core/peer"
	"github.com/libp2p/go-libp2p-core/protocol"
	cbg "github.com/whyrusleeping/cbor-gen"
	"golang.org/x/xerrors"

	"github.com/filecoin-project/go-address"
	datatransfer "github.com/filecoin-project/go-data-transfer"
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/big"
	"github.com/filecoin-project/specs-actors/actors/builtin/paych"

	"github.com/filecoin-project/go-fil-markets/piecestore"
)

//go:generate cbor-gen-for --map-encoding Query QueryResponse DealProposal DealResponse Params QueryParams DealPayment ClientDealState ProviderDealState PaymentInfo RetrievalPeer Ask

// QueryProtocolID is the protocol for querying information about retrieval
// deal parameters
const QueryProtocolID = protocol.ID("/fil/retrieval/qry/1.0.0")

// OldQueryProtocolID is the old query protocol for tuple structs
const OldQueryProtocolID = protocol.ID("/fil/retrieval/qry/0.0.1")

// Unsubscribe is a function that unsubscribes a subscriber for either the
// client or the provider
type Unsubscribe func()

// PaymentInfo is the payment channel and lane for a deal, once it is setup
type PaymentInfo struct {
	PayCh address.Address
	Lane  uint64
}

// ClientDealState is the current state of a deal from the point of view
// of a retrieval client
type ClientDealState struct {
	DealProposal
	StoreID *uint64
	// Set when the data transfer is started
	ChannelID            *datatransfer.ChannelID
	LastPaymentRequested bool
	AllBlocksReceived    bool
	TotalFunds           abi.TokenAmount
	ClientWallet         address.Address
	MinerWallet          address.Address
	PaymentInfo          *PaymentInfo
	Status               DealStatus
	Sender               peer.ID
	TotalReceived        uint64
	Message              string
	BytesPaidFor         uint64
	CurrentInterval      uint64
	PaymentRequested     abi.TokenAmount
	FundsSpent           abi.TokenAmount
	UnsealFundsPaid      abi.TokenAmount
	WaitMsgCID           *cid.Cid // the CID of any message the client deal is waiting for
	VoucherShortfall     abi.TokenAmount
	LegacyProtocol       bool
}

func (deal *ClientDealState) NextInterval() uint64 {
	return deal.Params.NextInterval(deal.CurrentInterval)
}

// ProviderDealState is the current state of a deal from the point of view
// of a retrieval provider
type ProviderDealState struct {
	DealProposal
	StoreID uint64

	ChannelID       *datatransfer.ChannelID
	PieceInfo       *piecestore.PieceInfo
	Status          DealStatus
	Receiver        peer.ID
	TotalSent       uint64
	FundsReceived   abi.TokenAmount
	Message         string
	CurrentInterval uint64
	LegacyProtocol  bool
}

func (deal *ProviderDealState) IntervalLowerBound() uint64 {
	return deal.Params.IntervalLowerBound(deal.CurrentInterval)
}

func (deal *ProviderDealState) NextInterval() uint64 {
	return deal.Params.NextInterval(deal.CurrentInterval)
}

// Identifier provides a unique id for this provider deal
func (pds ProviderDealState) Identifier() ProviderDealIdentifier {
	return ProviderDealIdentifier{Receiver: pds.Receiver, DealID: pds.ID}
}

// ProviderDealIdentifier is a value that uniquely identifies a deal
type ProviderDealIdentifier struct {
	Receiver peer.ID
	DealID   DealID
}

func (p ProviderDealIdentifier) String() string {
	return fmt.Sprintf("%v/%v", p.Receiver, p.DealID)
}

// RetrievalPeer is a provider address/peer.ID pair (everything needed to make
// deals for with a miner)
type RetrievalPeer struct {
	Address  address.Address
	ID       peer.ID // optional
	PieceCID *cid.Cid
}

// QueryResponseStatus indicates whether a queried piece is available
type QueryResponseStatus uint64

const (
	// QueryResponseAvailable indicates a provider has a piece and is prepared to
	// return it
	QueryResponseAvailable QueryResponseStatus = iota

	// QueryResponseUnavailable indicates a provider either does not have or cannot
	// serve the queried piece to the client
	QueryResponseUnavailable

	// QueryResponseError indicates something went wrong generating a query response
	QueryResponseError
)

// QueryItemStatus (V1) indicates whether the requested part of a piece (payload or selector)
// is available for retrieval
type QueryItemStatus uint64

const (
	// QueryItemAvailable indicates requested part of the piece is available to be
	// served
	QueryItemAvailable QueryItemStatus = iota

	// QueryItemUnavailable indicates the piece either does not contain the requested
	// item or it cannot be served
	QueryItemUnavailable

	// QueryItemUnknown indicates the provider cannot determine if the given item
	// is part of the requested piece (for example, if the piece is sealed and the
	// miner does not maintain a payload CID index)
	QueryItemUnknown
)

// QueryParams - V1 - indicate what specific information about a piece that a retrieval
// client is interested in, as well as specific parameters the client is seeking
// for the retrieval deal
type QueryParams struct {
	PieceCID *cid.Cid // optional, query if miner has this cid in this piece. some miners may not be able to respond.
	//Selector                   ipld.Node // optional, query if miner has this cid in this piece. some miners may not be able to respond.
	//MaxPricePerByte            abi.TokenAmount    // optional, tell miner uninterested if more expensive than this
	//MinPaymentInterval         uint64    // optional, tell miner uninterested unless payment interval is greater than this
	//MinPaymentIntervalIncrease uint64    // optional, tell miner uninterested unless payment interval increase is greater than this
}

// Query is a query to a given provider to determine information about a piece
// they may have available for retrieval
type Query struct {
	PayloadCID  cid.Cid // V0
	QueryParams         // V1
}

// QueryUndefined is a query with no values
var QueryUndefined = Query{}

// NewQueryV0 creates a V0 query (which only specifies a payload)
func NewQueryV0(payloadCID cid.Cid) Query {
	return Query{PayloadCID: payloadCID}
}

// NewQueryV1 creates a V1 query (which has an optional pieceCID)
func NewQueryV1(payloadCID cid.Cid, pieceCID *cid.Cid) Query {
	return Query{
		PayloadCID: payloadCID,
		QueryParams: QueryParams{
			PieceCID: pieceCID,
		},
	}
}

// QueryResponse is a miners response to a given retrieval query
type QueryResponse struct {
	Status        QueryResponseStatus
	PieceCIDFound QueryItemStatus // V1 - if a PieceCID was requested, the result
	//SelectorFound   QueryItemStatus // V1 - if a Selector was requested, the result

	Size uint64 // Total size of piece in bytes
	//ExpectedPayloadSize uint64 // V1 - optional, if PayloadCID + selector are specified and miner knows, can offer an expected size

	PaymentAddress             address.Address // address to send funds to -- may be different than miner addr
	MinPricePerByte            abi.TokenAmount
	MaxPaymentInterval         uint64
	MaxPaymentIntervalIncrease uint64
	Message                    string
	UnsealPrice                abi.TokenAmount
}

// QueryResponseUndefined is an empty QueryResponse
var QueryResponseUndefined = QueryResponse{}

// PieceRetrievalPrice is the total price to retrieve the piece (size * MinPricePerByte + UnsealedPrice)
func (qr QueryResponse) PieceRetrievalPrice() abi.TokenAmount {
	return big.Add(big.Mul(qr.MinPricePerByte, abi.NewTokenAmount(int64(qr.Size))), qr.UnsealPrice)
}

// PayloadRetrievalPrice is the expected price to retrieve just the given payload
// & selector (V1)
//func (qr QueryResponse) PayloadRetrievalPrice() abi.TokenAmount {
//	return types.BigMul(qr.MinPricePerByte, types.NewInt(qr.ExpectedPayloadSize))
//}

// IsTerminalError returns true if this status indicates processing of this deal
// is complete with an error
func IsTerminalError(status DealStatus) bool {
	return status == DealStatusDealNotFound ||
		status == DealStatusFailing ||
		status == DealStatusRejected
}

// IsTerminalSuccess returns true if this status indicates processing of this deal
// is complete with a success
func IsTerminalSuccess(status DealStatus) bool {
	return status == DealStatusCompleted
}

// IsTerminalStatus returns true if this status indicates processing of a deal is
// complete (either success or error)
func IsTerminalStatus(status DealStatus) bool {
	return IsTerminalError(status) || IsTerminalSuccess(status)
}

// Params are the parameters requested for a retrieval deal proposal
type Params struct {
	Selector                *cbg.Deferred // V1
	PieceCID                *cid.Cid
	PricePerByte            abi.TokenAmount
	PaymentInterval         uint64 // when to request payment
	PaymentIntervalIncrease uint64
	UnsealPrice             abi.TokenAmount
}

func (p Params) SelectorSpecified() bool {
	return p.Selector != nil && !bytes.Equal(p.Selector.Raw, cbg.CborNull)
}

func (p Params) IntervalLowerBound(currentInterval uint64) uint64 {
	intervalSize := p.PaymentInterval
	var lowerBound uint64
	var target uint64
	for target < currentInterval {
		lowerBound = target
		target += intervalSize
		intervalSize += p.PaymentIntervalIncrease
	}
	return lowerBound
}

func (p Params) NextInterval(currentInterval uint64) uint64 {
	intervalSize := p.PaymentInterval
	var nextInterval uint64
	for nextInterval <= currentInterval {
		nextInterval += intervalSize
		intervalSize += p.PaymentIntervalIncrease
	}
	return nextInterval
}

// NewParamsV0 generates parameters for a retrieval deal, which is always a whole piece deal
func NewParamsV0(pricePerByte abi.TokenAmount, paymentInterval uint64, paymentIntervalIncrease uint64) Params {
	return Params{
		PricePerByte:            pricePerByte,
		PaymentInterval:         paymentInterval,
		PaymentIntervalIncrease: paymentIntervalIncrease,
		UnsealPrice:             big.Zero(),
	}
}

// NewParamsV1 generates parameters for a retrieval deal, including a selector
func NewParamsV1(pricePerByte abi.TokenAmount, paymentInterval uint64, paymentIntervalIncrease uint64, sel ipld.Node, pieceCid *cid.Cid, unsealPrice abi.TokenAmount) (Params, error) {
	var buffer bytes.Buffer

	if sel == nil {
		return Params{}, xerrors.New("selector required for NewParamsV1")
	}

	err := dagcbor.Encode(sel, &buffer)
	if err != nil {
		return Params{}, xerrors.Errorf("error encoding selector: %w", err)
	}

	return Params{
		Selector:                &cbg.Deferred{Raw: buffer.Bytes()},
		PieceCID:                pieceCid,
		PricePerByte:            pricePerByte,
		PaymentInterval:         paymentInterval,
		PaymentIntervalIncrease: paymentIntervalIncrease,
		UnsealPrice:             unsealPrice,
	}, nil
}

// DealID is an identifier for a retrieval deal (unique to a client)
type DealID uint64

func (d DealID) String() string {
	return fmt.Sprintf("%d", d)
}

// DealProposal is a proposal for a new retrieval deal
type DealProposal struct {
	PayloadCID cid.Cid
	ID         DealID
	Params
}

// Type method makes DealProposal usable as a voucher
func (dp *DealProposal) Type() datatransfer.TypeIdentifier {
	return "RetrievalDealProposal/1"
}

// DealProposalUndefined is an undefined deal proposal
var DealProposalUndefined = DealProposal{}

// DealResponse is a response to a retrieval deal proposal
type DealResponse struct {
	Status DealStatus
	ID     DealID

	// payment required to proceed
	PaymentOwed abi.TokenAmount

	Message string
}

// Type method makes DealResponse usable as a voucher result
func (dr *DealResponse) Type() datatransfer.TypeIdentifier {
	return "RetrievalDealResponse/1"
}

// DealResponseUndefined is an undefined deal response
var DealResponseUndefined = DealResponse{}

// DealPayment is a payment for an in progress retrieval deal
type DealPayment struct {
	ID             DealID
	PaymentChannel address.Address
	PaymentVoucher *paych.SignedVoucher
}

// Type method makes DealPayment usable as a voucher
func (dr *DealPayment) Type() datatransfer.TypeIdentifier {
	return "RetrievalDealPayment/1"
}

// DealPaymentUndefined is an undefined deal payment
var DealPaymentUndefined = DealPayment{}

var (
	// ErrNotFound means a piece was not found during retrieval
	ErrNotFound = errors.New("not found")

	// ErrVerification means a retrieval contained a block response that did not verify
	ErrVerification = errors.New("Error when verify data")
)

type Ask struct {
	PricePerByte            abi.TokenAmount
	UnsealPrice             abi.TokenAmount
	PaymentInterval         uint64
	PaymentIntervalIncrease uint64
}

// ShortfallErorr is an error that indicates a short fall of funds
type ShortfallError struct {
	shortfall abi.TokenAmount
}

// NewShortfallError returns a new error indicating a shortfall of funds
func NewShortfallError(shortfall abi.TokenAmount) error {
	return ShortfallError{shortfall}
}

// Shortfall returns the numerical value of the shortfall
func (se ShortfallError) Shortfall() abi.TokenAmount {
	return se.shortfall
}
func (se ShortfallError) Error() string {
	return fmt.Sprintf("Inssufficient Funds. Shortfall: %s", se.shortfall.String())
}

// ChannelAvailableFunds provides information about funds in a channel
type ChannelAvailableFunds struct {
	// ConfirmedAmt is the amount of funds that have been confirmed on-chain
	// for the channel
	ConfirmedAmt abi.TokenAmount
	// PendingAmt is the amount of funds that are pending confirmation on-chain
	PendingAmt abi.TokenAmount
	// PendingWaitSentinel can be used with PaychGetWaitReady to wait for
	// confirmation of pending funds
	PendingWaitSentinel *cid.Cid
	// QueuedAmt is the amount that is queued up behind a pending request
	QueuedAmt abi.TokenAmount
	// VoucherRedeemedAmt is the amount that is redeemed by vouchers on-chain
	// and in the local datastore
	VoucherReedeemedAmt abi.TokenAmount
}

// PricingInput provides input parameters required to price a retrieval deal.
type PricingInput struct {
	// PayloadCID is the cid of the payload to retrieve.
	PayloadCID cid.Cid
	// PieceCID is the cid of the Piece from which the Payload will be retrieved.
	PieceCID cid.Cid
	// PieceSize is the size of the Piece from which the payload will be retrieved.
	PieceSize abi.UnpaddedPieceSize
	// Client is the peerID of the retrieval client.
	Client peer.ID
	// VerifiedDeal is true if there exists a verified storage deal for the PayloadCID.
	VerifiedDeal bool
	// Unsealed is true if there exists an unsealed sector from which we can retrieve the given payload.
	Unsealed bool
	// CurrentAsk is the current configured ask in the ask-store.
	CurrentAsk Ask
}
```

[**Retrieval Peer Resolver**](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market.retrieval\_peer\_resolver)

The `peer resolver` is a content routing interface to discover retrieval miners that have a given Piece.

It can be backed by both a local store of previous storage deals or by querying the chain.

```go
// PeerResolver is an interface for looking up providers that may have a piece
type PeerResolver interface {
	GetPeers(payloadCID cid.Cid) ([]RetrievalPeer, error) // TODO: channel
}
```

[**Retrieval Protocols**](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market.retrieval\_protocols)

The `retrieval market` is implemented using the following `libp2p` service.

> **Name**: Query Protocol **Protocol ID**: `/fil/<network-name>/retrieval/qry/1.0.0`

Request: CBOR Encoded RetrievalQuery Data Structure Response: CBOR Encoded RetrievalQueryResponse Data Structure

[**Retrieval Client**](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market.retrieval\_client)

[**Client Dependencies**](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market.retrieval\_client.client-dependencies)

The Retrieval Client Depends On The Following Dependencies

* **Host**: A libp2p host (set setup the libp2p protocols)
* **Filecoin Node**: A node implementation to query the chain for pieces and to setup and manage payment channels
* **BlockStore**: Same as one used by data transfer module
* **Data Transfer**: Module used for transferring payload. Writes to the blockstore.

[Example: Retrieval Client API ](https://spec.filecoin.io/#example-retrieval-client-api)

```go
package retrievalmarket

import (
	"context"

	"github.com/ipfs/go-cid"
	bstore "github.com/ipfs/go-ipfs-blockstore"

	"github.com/filecoin-project/go-address"
	"github.com/filecoin-project/go-state-types/abi"

	"github.com/filecoin-project/go-fil-markets/shared"
)

type PayloadCID = cid.Cid

// BlockstoreAccessor is used by the retrieval market client to get a
// blockstore when needed, concretely to store blocks received from the provider.
// This abstraction allows the caller to provider any blockstore implementation:
// a CARv2 file, an IPFS blockstore, or something else.
type BlockstoreAccessor interface {
	Get(DealID, PayloadCID) (bstore.Blockstore, error)
	Done(DealID) error
}

// ClientSubscriber is a callback that is registered to listen for retrieval events
type ClientSubscriber func(event ClientEvent, state ClientDealState)

type RetrieveResponse struct {
	DealID      DealID
	CarFilePath string
}

// RetrievalClient is a client interface for making retrieval deals
type RetrievalClient interface {

	// NextID generates a new deal ID.
	NextID() DealID

	// Start initializes the client by running migrations
	Start(ctx context.Context) error

	// OnReady registers a listener for when the client comes on line
	OnReady(shared.ReadyFunc)

	// Find Providers finds retrieval providers who may be storing a given piece
	FindProviders(payloadCID cid.Cid) []RetrievalPeer

	// Query asks a provider for information about a piece it is storing
	Query(
		ctx context.Context,
		p RetrievalPeer,
		payloadCID cid.Cid,
		params QueryParams,
	) (QueryResponse, error)

	// Retrieve retrieves all or part of a piece with the given retrieval parameters
	Retrieve(
		ctx context.Context,
		id DealID,
		payloadCID cid.Cid,
		params Params,
		totalFunds abi.TokenAmount,
		p RetrievalPeer,
		clientWallet address.Address,
		minerWallet address.Address,
	) (DealID, error)

	// SubscribeToEvents listens for events that happen related to client retrievals
	SubscribeToEvents(subscriber ClientSubscriber) Unsubscribe

	// V1

	// TryRestartInsufficientFunds attempts to restart any deals stuck in the insufficient funds state
	// after funds are added to a given payment channel
	TryRestartInsufficientFunds(paymentChannel address.Address) error

	// CancelDeal attempts to cancel an inprogress deal
	CancelDeal(id DealID) error

	// GetDeal returns a given deal by deal ID, if it exists
	GetDeal(dealID DealID) (ClientDealState, error)

	// ListDeals returns all deals
	ListDeals() (map[DealID]ClientDealState, error)
}
```

[**Retrieval Provider (Miner)**](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market.retrieval\_provider)

[**Provider Dependencies**](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market.retrieval\_provider.provider-dependencies)

The Retrieval Provider depends on the following dependencies

* **Host**: A libp2p host (set setup the libp2p protocols)
* **Filecoin Node**: A node implementation to query the chain for pieces and to setup and manage payment channels
* **StorageMining Subsystem**: For unsealing sectors
* **BlockStore**: Same as one used by data transfer module
* **Data Transfer**: Module used for transferring payload. Reads from the blockstore.

[Example: Retrieval Provider API ](https://spec.filecoin.io/#example-retrieval-provider-api)

```go
package retrievalmarket

import (
	"context"

	"github.com/filecoin-project/go-fil-markets/shared"
)

// ProviderSubscriber is a callback that is registered to listen for retrieval events on a provider
type ProviderSubscriber func(event ProviderEvent, state ProviderDealState)

// RetrievalProvider is an interface by which a provider configures their
// retrieval operations and monitors deals received and process
type RetrievalProvider interface {
	// Start begins listening for deals on the given host
	Start(ctx context.Context) error

	// OnReady registers a listener for when the provider comes on line
	OnReady(shared.ReadyFunc)

	// Stop stops handling incoming requests
	Stop() error

	// SetAsk sets the retrieval payment parameters that this miner will accept
	SetAsk(ask *Ask)

	// GetAsk returns the retrieval providers pricing information
	GetAsk() *Ask

	// SubscribeToEvents listens for events that happen related to client retrievals
	SubscribeToEvents(subscriber ProviderSubscriber) Unsubscribe

	ListDeals() map[ProviderDealIdentifier]ProviderDealState
}

// AskStore is an interface which provides access to a persisted retrieval Ask
type AskStore interface {
	GetAsk() *Ask
	SetAsk(ask *Ask) error
}
```

[**Retrieval Deal Status**](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market.deal\_status)

[Example: ](https://spec.filecoin.io/#example-)

```go
package retrievalmarket

import "fmt"

// DealStatus is the status of a retrieval deal returned by a provider
// in a DealResponse
type DealStatus uint64

const (
	// DealStatusNew is a deal that nothing has happened with yet
	DealStatusNew DealStatus = iota

	// DealStatusUnsealing means the provider is unsealing data
	DealStatusUnsealing

	// DealStatusUnsealed means the provider has finished unsealing data
	DealStatusUnsealed

	// DealStatusWaitForAcceptance means we're waiting to hear back if the provider accepted our deal
	DealStatusWaitForAcceptance

	// DealStatusPaymentChannelCreating is the status set while waiting for the
	// payment channel creation to complete
	DealStatusPaymentChannelCreating

	// DealStatusPaymentChannelAddingFunds is the status when we are waiting for funds
	// to finish being sent to the payment channel
	DealStatusPaymentChannelAddingFunds

	// DealStatusAccepted means a deal has been accepted by a provider
	// and its is ready to proceed with retrieval
	DealStatusAccepted

	// DealStatusFundsNeededUnseal means a deal has been accepted by a provider
	// and payment is needed to unseal the data
	DealStatusFundsNeededUnseal

	// DealStatusFailing indicates something went wrong during a retrieval,
	// and we are cleaning up before terminating with an error
	DealStatusFailing

	// DealStatusRejected indicates the provider rejected a client's deal proposal
	// for some reason
	DealStatusRejected

	// DealStatusFundsNeeded indicates the provider needs a payment voucher to
	// continue processing the deal
	DealStatusFundsNeeded

	// DealStatusSendFunds indicates the client is now going to send funds because we reached the threshold of the last payment
	DealStatusSendFunds

	// DealStatusSendFundsLastPayment indicates the client is now going to send final funds because
	// we reached the threshold of the final payment
	DealStatusSendFundsLastPayment

	// DealStatusOngoing indicates the provider is continuing to process a deal
	DealStatusOngoing

	// DealStatusFundsNeededLastPayment indicates the provider needs a payment voucher
	// in order to complete a deal
	DealStatusFundsNeededLastPayment

	// DealStatusCompleted indicates a deal is complete
	DealStatusCompleted

	// DealStatusDealNotFound indicates an update was received for a deal that could
	// not be identified
	DealStatusDealNotFound

	// DealStatusErrored indicates a deal has terminated in an error
	DealStatusErrored

	// DealStatusBlocksComplete indicates that all blocks have been processed for the piece
	DealStatusBlocksComplete

	// DealStatusFinalizing means the last payment has been received and
	// we are just confirming the deal is complete
	DealStatusFinalizing

	// DealStatusCompleting is just an inbetween state to perform final cleanup of
	// complete deals
	DealStatusCompleting

	// DealStatusCheckComplete is used for when the provided completes without a last payment
	// requested cycle, to verify we have received all blocks
	DealStatusCheckComplete

	// DealStatusCheckFunds means we are looking at the state of funding for the channel to determine
	// if more money is incoming
	DealStatusCheckFunds

	// DealStatusInsufficientFunds indicates we have depleted funds for the retrieval payment channel
	// - we can resume after funds are added
	DealStatusInsufficientFunds

	// DealStatusPaymentChannelAllocatingLane is the status when we are making a lane for this channel
	DealStatusPaymentChannelAllocatingLane

	// DealStatusCancelling means we are cancelling an inprogress deal
	DealStatusCancelling

	// DealStatusCancelled means a deal has been cancelled
	DealStatusCancelled

	// DealStatusRetryLegacy means we're attempting the deal proposal for a second time using the legacy datatype
	DealStatusRetryLegacy

	// DealStatusWaitForAcceptanceLegacy means we're waiting to hear the results on the legacy protocol
	DealStatusWaitForAcceptanceLegacy

	// DealStatusClientWaitingForLastBlocks means that the provider has told
	// the client that all blocks were sent for the deal, and the client is
	// waiting for the last blocks to arrive. This should only happen when
	// the deal price per byte is zero (if it's not zero the provider asks
	// for final payment after sending the last blocks).
	DealStatusClientWaitingForLastBlocks

	// DealStatusPaymentChannelAddingInitialFunds means that a payment channel
	// exists from an earlier deal between client and provider, but we need
	// to add funds to the channel for this particular deal
	DealStatusPaymentChannelAddingInitialFunds

	// DealStatusErroring means that there was an error and we need to
	// do some cleanup before moving to the error state
	DealStatusErroring

	// DealStatusRejecting means that the deal was rejected and we need to do
	// some cleanup before moving to the rejected state
	DealStatusRejecting

	// DealStatusDealNotFoundCleanup means that the deal was not found and we
	// need to do some cleanup before moving to the not found state
	DealStatusDealNotFoundCleanup

	// DealStatusFinalizingBlockstore means that all blocks have been received,
	// and the blockstore is being finalized
	DealStatusFinalizingBlockstore
)

// DealStatuses maps deal status to a human readable representation
var DealStatuses = map[DealStatus]string{
	DealStatusNew:                              "DealStatusNew",
	DealStatusUnsealing:                        "DealStatusUnsealing",
	DealStatusUnsealed:                         "DealStatusUnsealed",
	DealStatusWaitForAcceptance:                "DealStatusWaitForAcceptance",
	DealStatusPaymentChannelCreating:           "DealStatusPaymentChannelCreating",
	DealStatusPaymentChannelAddingFunds:        "DealStatusPaymentChannelAddingFunds",
	DealStatusAccepted:                         "DealStatusAccepted",
	DealStatusFundsNeededUnseal:                "DealStatusFundsNeededUnseal",
	DealStatusFailing:                          "DealStatusFailing",
	DealStatusRejected:                         "DealStatusRejected",
	DealStatusFundsNeeded:                      "DealStatusFundsNeeded",
	DealStatusSendFunds:                        "DealStatusSendFunds",
	DealStatusSendFundsLastPayment:             "DealStatusSendFundsLastPayment",
	DealStatusOngoing:                          "DealStatusOngoing",
	DealStatusFundsNeededLastPayment:           "DealStatusFundsNeededLastPayment",
	DealStatusCompleted:                        "DealStatusCompleted",
	DealStatusDealNotFound:                     "DealStatusDealNotFound",
	DealStatusErrored:                          "DealStatusErrored",
	DealStatusBlocksComplete:                   "DealStatusBlocksComplete",
	DealStatusFinalizing:                       "DealStatusFinalizing",
	DealStatusCompleting:                       "DealStatusCompleting",
	DealStatusCheckComplete:                    "DealStatusCheckComplete",
	DealStatusCheckFunds:                       "DealStatusCheckFunds",
	DealStatusInsufficientFunds:                "DealStatusInsufficientFunds",
	DealStatusPaymentChannelAllocatingLane:     "DealStatusPaymentChannelAllocatingLane",
	DealStatusCancelling:                       "DealStatusCancelling",
	DealStatusCancelled:                        "DealStatusCancelled",
	DealStatusRetryLegacy:                      "DealStatusRetryLegacy",
	DealStatusWaitForAcceptanceLegacy:          "DealStatusWaitForAcceptanceLegacy",
	DealStatusClientWaitingForLastBlocks:       "DealStatusWaitingForLastBlocks",
	DealStatusPaymentChannelAddingInitialFunds: "DealStatusPaymentChannelAddingInitialFunds",
	DealStatusErroring:                         "DealStatusErroring",
	DealStatusRejecting:                        "DealStatusRejecting",
	DealStatusDealNotFoundCleanup:              "DealStatusDealNotFoundCleanup",
	DealStatusFinalizingBlockstore:             "DealStatusFinalizingBlockstore",
}

func (s DealStatus) String() string {
	str, ok := DealStatuses[s]
	if ok {
		return str
	}
	return fmt.Sprintf("DealStatusUnknown - %d", s)
}
```
