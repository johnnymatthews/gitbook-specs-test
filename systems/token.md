# Token

## [Minting Model](https://spec.filecoin.io/#section-systems.filecoin\_token.minting\_model) <a href="#section-systems.filecoin_token.minting_model" id="section-systems.filecoin_token.minting_model"></a>

Many blockchains mint tokens based on a simple exponential decay model. Under this model, block rewards are highest in the beginning, and miner participation is often the lowest, so mining generates many tokens per unit of work early in the networkʼs life, then rapidly decreases.

Over many cryptoeconomic simulations, it became clear that the simple exponential decay model would encourage short-term behavior around network launch with an unhealthy effect on the Filecoin Economy. Specifically, it would incentivize storage miners to over-invest in hardware for the sealing stage of mining to onboard storage as quickly as possible. It would be profitable to exit the network after exhausting these early rewards, even if it resulted in losing client data. This would harm the network: clients would lose data and have less access to long-term storage, and miners would have little incentive to contribute more resources to the network. Additionally, this would result in the majority of network subsidies being paid based wholly on timing, rather than actual storage (and hence value) provided to the network.

To encourage consistent storage onboarding and investment in long-term storage, not just rapid sealing, Filecoin introduces the concept of a network baseline. Instead of minting tokens based purely on elapsed time, block rewards instead scale up as total storage power on the network increases. This preserves the shape of the original exponential decay model, but softens it in the earliest days of the network. Once the network reaches the baseline, the cumulative block reward issued is identical to a simple exponential decay model, but if the network does not pass the pre-established threshold, a portion of block rewards are deferred. The overall result is that Filecoin rewards to miners more closely match the utility they, and the network as a whole, provide to clients.

Specifically, a hybrid exponential minting mechanism is introduced with a proportion of the reward coming from simple exponential decay, “Simple Minting” and the other proportion from network baseline, “Baseline Minting”. The total reward per epoch will be the sum of the two rewards. Mining Filecoin should be even more profitable with this mechanism. Simple minting allocation disproportionately rewards early miners and provides counter pressure to shocks. Baseline minting allocation mints more tokens when more value for the network has been created. More tokens are minted to facilitate greater trade when the network can unlock a greater potential. This should lead to increased creation of value for the network and lower risk of minting filecoin too quickly.

The protocol allocates 30% of Storage Mining Allocation in Simple Minting and the remaining 70% in Baseline Minting. 30% of Simple Minting can provide counter forces in the event of shocks. Baseline capacity can start from a smaller percentage of worldʼs storage today, grow at a rapid rate, and catch up to a higher but still reasonable percentage of worldʼs storage in the future. As such, the network baseline will start from 1EiB (which is less than 0.01% of the worldʼs storage today) and grow at an annual rate of 200% (higher than the usual world storage annual growth rate at 40%). The community can come together to slow down the rate of growth when the network is providing 1-10% of the worldʼs storage.

There are many features that will make passing the baseline more efficient and economical and unleash a greater share of baseline minting. The community can come together to collectively achieve these goals:

* More performant Proof of Replication algorithms, with lower on chain footprint, faster verification time, cheaper hardware requirement, different security assumptions, resulting in sectors with longer lifetime and enabling sector upgrades without reseal.
* A more scalable consensus algorithm that can provide greater throughput and handle larger volume with shorter finality.
* More deal functionalities that allow sectors to last for longer.

Lastly, it is important to note that while the block reward incentivizes participation, it cannot be treated as a resource to be exploited. It is a common pool of subsidies that seeds and grows the network to benefit the economy and participants. An example of different stages of the economy and different sources of subsidies is illustrated in the following Figure.

<figure><img src="https://spec.filecoin.io/systems/filecoin_token/minting_model/final-stages-of-economy.jpg" alt="Filecoin Economy Stages" height="100%"><figcaption><p><a href="https://spec.filecoin.io/#figure-filecoin-economy-stages">Figure: Filecoin Economy Stages</a> </p></figcaption></figure>

## [Block Reward Minting](https://spec.filecoin.io/#section-systems.filecoin\_token.block\_reward\_minting) <a href="#section-systems.filecoin_token.block_reward_minting" id="section-systems.filecoin_token.block_reward_minting"></a>

In this section, we provide the mathematical specification for Simple Minting, Baseline Minting and Block Reward Issuance. We will provide the details and key mathematical properties for the above concepts.

### [**Economic parameters**](https://spec.filecoin.io/#section-systems.filecoin\_token.block\_reward\_minting.economic-parameters)

* `M∞M∞​` is the total asymptotic number of tokens to be emitted as storage-mining block rewards. Per the [Token Allocation spec](https://spec.filecoin.io/#section-\_index.systems\_\_filecoin\_token\_\_token\_allocation), `M∞:=55%⋅FIL_BASE=0.55⋅2×109FIL=1.1×109FILM∞​:=55%⋅FIL_BASE=0.55⋅2×109FIL=1.1×109FIL`. The dimension of the `M∞M∞​` quantity is tokens.
* `λλ` is the “simple exponential decay” minting rate corresponding to a 6-year half-life. The meaning of “simple exponential decay” is that the total minted supply at time `tt` is `M∞⋅(1−e−λt)M∞​⋅(1−e−λt)`, so the specification of `λλ` in symbols becomes the equation `1−e−λ⋅6yr=121−e−λ⋅6yr=21​`. Note that a “year” is poorly defined. The simplified definition of `1yr:=365d1yr:=365d` was agreed upon for Filecoin. Of course, `1d=86400s1d=86400s`, so `1yr=31536000s1yr=31536000s`. We can solve this equation as

λ=ln⁡26yr=ln⁡2189216000s≈3.663258818×10−9Hzλ=6yrln2​=189216000sln2​≈3.663258818×10−9Hz

The dimension of the `λλ` quantity is `time−1−1`.

* `γγ` is the mixture between baseline and simple minting. A `γγ` value of 1.0 corresponds to pure baseline minting, while a `γγ` value of 0.0 corresponds to pure simple minting. We currently use `γ:=0.7γ:=0.7`. The `γγ` quantity is dimensionless.
* `b(t)b(t)` is the baseline function, which was designed as an exponential

b(t)=b0⋅egtb(t)=b0​⋅egt

where

* `b0b0​` is the “initial baseline”. The dimension of the `b0b0​` quantity is information.
* `gg` is related to the baseline’s “annual growth rate” (`gaga​`) by the equation `exp⁡(g⋅1yr)=1+gaexp(g⋅1yr)=1+ga​`, which has the solution

g=ln⁡(1+ga)31536000s.g=31536000sln(1+ga​)​.

While `gaga​` is dimensionless, the dimension of the `gg` quantity is `time−1−1`.

The dimension of the `b(t)b(t)` quantity is information.

### [**Simple Minting**](https://spec.filecoin.io/#section-systems.filecoin\_token.block\_reward\_minting.simple-minting)

* `M∞BM∞B​` is the total number of tokens to be emitted via baseline minting: `M∞B=M∞⋅γM∞B​=M∞​⋅γ`. Correspondingly, `M∞SM∞S​` is the total asymptotic number of tokens to be emitted via simple minting: `M∞S=M∞⋅(1−γ)M∞S​=M∞​⋅(1−γ)`. Of course, `M∞B+M∞S=M∞M∞B​+M∞S​=M∞​`.
* `MS(t)MS​(t)` is the total number of tokens that should ideally have been emitted by simple minting up until time `tt`. It is defined as `MS(t)=M∞S⋅(1−e−λt)MS​(t)=M∞S​⋅(1−e−λt)`. It is easy to verify that `lim⁡t→∞MS(t)=M∞Slimt→∞​MS​(t)=M∞S​`.

Note that `MS(t)MS​(t)` is easy to calculate, and can be determined quite independently of the network’s state. (This justifies the name “simple minting”.)

### [**Baseline Minting**](https://spec.filecoin.io/#section-systems.filecoin\_token.block\_reward\_minting.baseline-minting)

To define `MB(t)MB​(t)` (which is the number of tokens that should be emitted up until time `tt` by baseline minting), we must introduce a number of auxiliary variables, some of which depend on network state.

* `R(t)R(t)` is the instantaneous network raw-byte power (the total amount of bytes among all active sectors) at time `tt`. This quantity is state-dependent—it depends on the activities of miners on the network (specifically: commitment, expiration, faulting, and termination of sectors). The dimension of the `R(t)R(t)` quantity is information.
* `R‾(t)R(t)` is the capped network raw-byte power, defined as `R‾(t):=min⁡{b(t),R(t)}R(t):=min{b(t),R(t)}`. Its dimension is also information.
* `R‾Σ(t)RΣ​(t)` is the cumulative capped raw-byte power, defined as `R‾Σ(t):=∫0tR‾(x) dxRΣ​(t):=∫0t​R(x)dx`. The dimension of `RΣ‾(t)RΣ​​(t)` is `information⋅⋅time` (a dimension often referred to as “spacetime”).
* `θ(t)θ(t)` is the “effective network time”, and is defined as the solution to the equation

∫0θ(t)b(x) dx=∫0tR‾(x) dx=R‾Σ(t)∫0θ(t)​b(x)dx=∫0t​R(x)dx=RΣ​(t)

By plugging in the definition of `b(x)b(x)` and evaluating the integral, we can solve for a closed form of `θ(t)θ(t)` as follows:

∫0θ(t)b(x) dx=b0g(egθ(t)−1)=R‾Σ(t)∫0θ(t)​b(x)dx=gb0​​(egθ(t)−1)=RΣ​(t) θ(t)=1gln⁡(gR‾Σ(t)b0+1)θ(t)=g1​ln(b0​gRΣ​(t)​+1)

* `MB(t)MB​(t)` is defined similarly to `MS(t)MS​(t)`, just with `θ(t)θ(t)` in place of `tt` and `M∞BM∞B​` in place of `M∞SM∞S​`:

MB(t)=M∞B⋅(1−e−λθ(t))MB​(t)=M∞B​⋅(1−e−λθ(t))

### [**Block Reward Issuance**](https://spec.filecoin.io/#section-systems.filecoin\_token.block\_reward\_minting.block-reward-issuance)

* `M(t)M(t)`, the total number of tokens to be emitted as expected block rewards up until time `tt`, is defined as the sum of simple and baseline minting:

M(t)=MS(t)+MB(t)M(t)=MS​(t)+MB​(t)

Now we have defined a continuous target trajectory for _cumulative_ minting. But minting actually occurs _incrementally_, and also in _discrete_ increments. Periodically, a “tipset” is formed consisting of multiple winners, each of which receives an equal, finite amount of reward. A single miner may win multiple times, but may only submit one block and may still receive rewards _as if_ they submitted multiple winning blocks. The mechanism by which multiple wins are rewarded is multiplication by a variable called `WinCount`, so we refer to the finite quantity minted and awarded for each win as “reward per `WinCount`” or “per win reward”.

* `ττ` is the duration of an “epoch” or “round” (these are synonymous). Per the [spec](https://spec.filecoin.io/#glossary\_\_epoch), `τ=30sτ=30s`. The dimension of `ττ` is time.
* `EE` is a parameter which determines the expected number of wins per round. While `EE` could be considered dimensionless, it useful to give it a dimension of “wins”. In Filecoin, the value of `EE` is 5.
* `W(n)W(n)` is the total number of wins by all miners in the tipset during round `nn`. This also has dimension “wins”. For each `nn`, `W(n)W(n)` is a random variable with the independent identical distribution `Poisson(E)Poisson(E)`.
* `w(n)w(n)` is the “reward per `WinCount`” or “per win reward” for round `nn`. It is defined by:

w(n)=max⁡{M(nτ+τ)−M(nτ),0}Ew(n)=Emax{M(nτ+τ)−M(nτ),0}​

The dimension of W(n)W(n) is `tokens⋅⋅wins−1−1`.

* While `M(t)M(t)` is a continuous target for minted supply, the discrete and random amount of tokens which have been minted as of time `tt` is

m(t)=∑k=0⌊t/τ⌋−1w(k)W(k)m(t)=k=0∑⌊t/τ⌋−1​w(k)W(k)

`m(t)m(t)` depends on past values of both `W(n)W(n)` and `R(nτ)R(nτ)`.

## [Token Allocation](https://spec.filecoin.io/#section-systems.filecoin\_token.token\_allocation) <a href="#section-systems.filecoin_token.token_allocation" id="section-systems.filecoin_token.token_allocation"></a>

Filecoinʼs token distribution is broken down as follows. A maximum of 2,000,000,000 FIL will ever be created, referred to as `FIL_BASE`. Of the Filecoin genesis block allocation, 10% of `FIL_BASE` were allocated for fundraising, of which 7.5% were sold in the 2017 token sale, and the 2.5% remaining were allocated for ecosystem development and potential future fundraising. 15% of `FIL_BASE` were allocated to Protocol Labs (including 4.5% for the PL team & contributors), and 5% were allocated to the Filecoin Foundation. The other 70% of all tokens were allocated to miners, as mining rewards, “for providing data storage service, maintaining the blockchain, distributing data, running contracts, and more.” There are multiple types of mining that these rewards will support over time; therefore, this allocation has been subdivided to cover different mining activities. A pie chart reflecting the FIL token allocation is shown in the following Figure.

<figure><img src="https://spec.filecoin.io/systems/filecoin_token/token_allocation/filtokenallocation.png" alt="Filecoin Token Allocation" height="100%"><figcaption><p><a href="https://spec.filecoin.io/#figure-filecoin-token-allocation">Figure: Filecoin Token Allocation</a> </p></figcaption></figure>

**Storage Mining Allocation.** At network launch, the only mining group with allocated incentives will be storage miners. This is the earliest group of miners, and the one responsible for maintaining the core functionality of the protocol. Therefore, this group has been allocated the largest amount of mining rewards. 55% of `FIL_BASE` (78.6% of mining rewards) is allocated to storage mining. This will cover primarily block rewards, which reward maintaining the blockchain, running actor code, and subsidizing reliable and useful storage. This amount will also cover early storage mining rewards, such as rewards in the SpaceRace competition and other potential types of storage miner initialization, such as faucets.

**Mining Reserve.** The Filecoin ecosystem must ensure incentives exist for all types of miners (e.g. retrieval miners, repair miners, and including future unknown types of miners) to support a robust economy. In order to ensure the network can provide incentives for these other types of miners, 15% of `FIL_BASE` (21.4% of mining rewards) have been set aside as a Mining Reserve. It will be up to the community to determine in the future how to distribute those tokens, through Filecoin improvement proposals (FIPs) or similar decentralized decision making processes. For example, the community might decide to create rewards for retrieval mining or other types of mining-related activities. The Filecoin Network, like all blockchain networks and open source projects, will continue to evolve, adapt, and overcome challenges for many years. Reserving these tokens provides future flexibility for miners and the ecosystem as a whole. Other types of mining, like retrieval mining, are not yet subsidized and yet are very important to the Filecoin Economy; Arguably, those uses may need a larger percentage of mining rewards. As years pass and the network evolves, it will be up to the community to decide whether this reserve is enough, or whether to make adjustments with unmined tokens.

**Market Cap.** Various communities estimate the size of cryptocurrency and token networks using different analogous measures of market capitalization. The most sensible token supply for such calculations is `FIL_CirculatingSupply`, because unmined, unvested, locked, and burnt funds are not circulating or tradeable in the economy. Any calculations using larger measures such as `FIL_BASE` are likely to be erroneously inflated and not to be believed.

**Total Burnt Funds.** Some filecoin are burned to fund on-chain computations and bandwidth as network message fees, in addition to those burned in penalties for storage faults and consensus faults, creating long-term deflationary pressure on the token. Accompanying the network message fees is the priority fee that is not burned, but goes to the block-producing miners for including a message.

| **Parameter**                 | **Value**                                                                                                                              | **Description**                                                                                                                            |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
|                               |                                                                                                                                        |                                                                                                                                            |
| `FIL_BASE`                    | 2,000,000,000 FIL                                                                                                                      | The maximum amount of FIL that will ever be created.                                                                                       |
| `FIL_MiningReserveAlloc`      | 300,000,000 FIL                                                                                                                        | Tokens reserved for funding mining to support growth of the Filecoin Economy, whose future usage will be decided by the Filecoin community |
| `FIL_StorageMiningAlloc`      | 1,100,000,000 FIL                                                                                                                      | The amount of FIL allocated to storage miners through block rewards, network initialization                                                |
| `FIL_Vested`                  | <p>Sum of genesis <code>MultisigActors.</code><br><code>AmountUnlocked</code></p>                                                      | Total amount of FIL that is vested from genesis allocation.                                                                                |
| `FIL_StorageMined`            | <p><code>RewardActor.</code><br><code>TotalStoragePowerReward</code></p>                                                               | The amount of FIL that has been mined by storage miners                                                                                    |
| `FIL_Locked`                  | `TotalPledgeCollateral` + `TotalProviderDealCollateral` + `TotalClientDealCollateral` + `TotalPendingDealPayment` + `OtherLockedFunds` | The amount of FIL locked as part of mining, deals, and other mechanisms.                                                                   |
| `FIL_CirculatingSupply`       | `FIL_Vested` + `FIL_Mined` - `TotalBurntFunds` - `FIL_Locked`                                                                          | The amount of FIL circulating and tradeable in the economy. The basis for Market Cap calculations.                                         |
| `TotalBurntFunds`             | <p><code>BurntFundsActor.</code><br><code>Balance</code></p>                                                                           | Total FIL burned as part of penalties and on-chain computations.                                                                           |
| `TotalPledgeCollateral`       | <p><code>StoragePowerActor.</code><br><code>TotalPledgeCollateral</code></p>                                                           | Total FIL locked as pledge collateral in all miners.                                                                                       |
| `TotalProviderDealCollateral` | <p><code>StorageMarketActor.</code><br><code>TotalProviderDealCollateral</code></p>                                                    | Total FIL locked as provider deal collateral                                                                                               |
| `TotalClientDealCollateral`   | <p><code>StorageMarketActor.</code><br><code>TotalClientDealColateral</code></p>                                                       | Total FIL locked as client deal collateral                                                                                                 |
| `TotalPendingDealPayment`     | <p><code>StorageMarketActor.</code><br><code>TotalPendingDealPayment</code></p>                                                        | Total FIL locked as pending client deal payment                                                                                            |

## [Payment Channels](https://spec.filecoin.io/#section-systems.filecoin\_token.payment\_channels) <a href="#section-systems.filecoin_token.payment_channels" id="section-systems.filecoin_token.payment_channels"></a>

Payment channels are generally used as a mechanism to increase the scalability of blockchains and enable users to transact without involving (i.e., publishing their transactions on) the blockchain, which: i) increases the load of the system, and ii) incurs gas costs for the user. Payment channels generally use a smart contract as an agreement between the two participants. In the Filecoin blockchain Payment Channels are realised by the `paychActor`.

The goal of the Payment Channel Actor specified here is to enable a series of off-chain microtransactions for applications built on top of Filecoin to be reconciled on-chain at a later time with fewer messages that involve the blockchain. Payment channels are already used in the Retrieval Market of the Filecoin Network, but their applicability is not constrained within this use-case only. Hence, here, we provide a detailed description of Payment Channels in the Filecoin network and then describe how Payment Channels are used in the specific case of the Filecoin Retrieval Market.

The payment channel actor can be used to open long-lived, flexible payment channels between users. Filecoin payment channels are _uni-directional_ and can be funded by adding to their balance. Given the context of _uni-directional_ payment channels, we define the **payment channel sender** as the party that receives some service, creates the channel, deposits funds and _sends_ payments (hence the term _payment channel sender_). The _payment channel recipient_, on the other hand is defined as the party that provides services and _receives payment_ for the services delivered (hence the term _payment channel recipient_). The fact that payment channels are uni-directional means that only the payment channel sender can add funds and only the recipient can receive funds. Payment channels are identified by a unique address, as is the case with all Filecoin actors.

The payment channel state structure looks like this:

```go
// A given payment channel actor is established by From (the receipent of a service)
// to enable off-chain microtransactions to To (the provider of a service) to be reconciled
// and tallied on chain.
type State struct {
	// Channel owner, who has created and funded the actor - the channel sender
	From addr.Address
	// Recipient of payouts from channel
	To addr.Address

	// Amount successfully redeemed through the payment channel, paid out on `Collect()`
	ToSend abi.TokenAmount

	// Height at which the channel can be `Collected`
	SettlingAt abi.ChainEpoch
	// Height before which the channel `ToSend` cannot be collected
	MinSettleHeight abi.ChainEpoch

	// Collections of lane states for the channel, maintained in ID order.
	LaneStates []*LaneState
}
```

Before continuing with the details of the Payment Channel and its components and features, it is worth defining a few terms.

* Voucher: a signed message created by either of the two channel parties that updates the channel balance. To differentiate from the **payment channel sender/recipient**, we refer to the voucher parties as **voucher sender/recipient**, who might or might not be the same as the payment channel ones (i.e., the voucher sender might be either the payment channel recipient or the payment channel sender).
* Redeeming a voucher: the voucher MUST be submitted on-chain by the opposite party from the one that created it. Redeeming a voucher does not trigger movement of funds from the channel to the recipient’s account, but it does incur message/gas costs. A voucher can be redeemed at any time up to `Collect` (see below), as long as it has a higher `Nonce` than previously submitted vouchers.
* `UpdateChannelState`: this is the process by which a voucher is redeemed, i.e., a voucher is submitted (but not cashed-out) on-chain.
* `Settle`: this process starts closing the channel. It can be called by either the channel creator (sender) or the channel recipient.
* `Collect`: with this process funds are eventually transferred from the payment channel sender to the payment channel recipient. This process incurs message/gas costs.

### [**Vouchers**](https://spec.filecoin.io/#section-systems.filecoin\_token.payment\_channels.vouchers)

Traditionally, in order to transact through a Payment Channel, the payment channel parties send to each other signed messages that update the balance of the channel. In Filecoin, these signed messages are called _vouchers_.

Throughout the interaction between the two parties, the channel sender (`From` address) is sending vouchers to the recipient (`To` address). The `Value` included in the voucher indicates the value available for the receiving party to _redeem_. The `Value` is based on the service that the _payment channel recipient_ has provided to the _payment channel sender_. Either the _payment channel recipient_ or the _payment channel sender_ can `Update` the balance of the channel and the balance `ToSend` to the _payment channel recipient_ (using a voucher), but the `Update` (i.e., the voucher) has to be accepted by the other party before funds can be collected. Furthermore, the voucher has to be redeemed by the opposite party from the one that issued the voucher. The _payment channel recipient_ can choose to `Collect` this balance at any time incurring the corresponding gas cost.

Redeeming a voucher does not transfer funds from the payment channel to the recipient’s account. Instead, redeeming a voucher attests that some service of worth `Value` has been provided by the payment channel recipient to the payment channel sender. It is not until the whole payment channel is _collected_ that the funds are dispatched to the recipient’s account.

This is the structure of the voucher:

```go
// A voucher can be created and sent by any of the two parties. The `To` payment channel address can redeem the voucher and then `Collect` the funds.
type SignedVoucher struct {
	// ChannelAddr is the address of the payment channel this signed voucher is valid for
	ChannelAddr addr.Address
	// TimeLockMin sets a min epoch before which the voucher cannot be redeemed
	TimeLockMin abi.ChainEpoch
	// TimeLockMax sets a max epoch beyond which the voucher cannot be redeemed
	// TimeLockMax set to 0 means no timeout
	TimeLockMax abi.ChainEpoch
	// (optional) The SecretPreImage is used by `To` to validate
	SecretPreimage []byte
	// (optional) Extra can be specified by `From` to add a verification method to the voucher
	Extra *ModVerifyParams
	// Specifies which lane the Voucher is added to (will be created if does not exist)
	Lane uint64
	// Nonce is set by `From` to prevent redemption of stale vouchers on a lane
	Nonce uint64
	// Amount voucher can be redeemed for
	Amount big.Int
	// (optional) MinSettleHeight can extend channel MinSettleHeight if needed
	MinSettleHeight abi.ChainEpoch

	// (optional) Set of lanes to be merged into `Lane`
	Merges []Merge

	// Sender's signature over the voucher
	Signature *crypto.Signature
}
```

Over the course of a transaction cycle, each participant in the payment channel can send `Voucher`s to the other participant.

For instance, if the payment channel sender (`From` address) has sent to the payment channel recipient (`To` address) the following three vouchers `(voucher_val, voucher_nonce)` for a lane with 100 FIL to be redeemed: (10, 1), (20, 2), (30, 3), then the recipient could choose to redeem (30, 3) bringing the lane’s value to 70 (100 - 30) and cancelling the preceding vouchers, i.e., they would not be able to redeem (10, 1) or (20, 2) anymore. However, they could redeem (20, 2), that is, 20 FIL, and then follow up with (30, 3) to redeem the remaining 10 FIL later.

It is worth highlighting that while the `Nonce` is a strictly increasing value to denote the sequence of vouchers issued within the remit of a payment channel, the `Value` is not a strictly increasing value. Decreasing `Value` (although expected rarely) can be realized in cases of refunds that need to flow in the direction from the payment channel recipient to the payment channel sender. This can be the case when some bits arrive corrupted in the case of file retrieval, for instance.

Vouchers are signed by the party that creates them and are authenticated using a (`Secret`, `PreImage`) pair provided by the paying party (channel sender). If the `PreImage` is indeed a pre-image of the `Secret` when used as input to some given algorithm (typically a one-way function like a hash), the `Voucher` is valid. The `Voucher` itself contains the `PreImage` but not the `Secret` (communicated separately to the receiving party). This enables multi-hop payments since an intermediary cannot redeem a voucher on their own. Vouchers can also be used to update the minimum height at which a channel will be settled (i.e., closed), or have `TimeLock`s to prevent voucher recipients from redeeming them too early. A channel can also have a `MinCloseHeight` to prevent it being closed prematurely (e.g. before the payment channel recipient has collected funds) by the payment channel creator/sender.

Once their transactions have completed, either party can choose to `Settle` (i.e., close) the channel. There is a 12hr period after `Settle` during which either party can submit any outstanding vouchers. Once the vouchers are submitted, either party can then call `Collect`. This will send the payment channel recipient the `ToPay` amount from the channel, and the channel sender (`From` address) will be refunded the remaining balance in the channel (if any).

### [**Lanes**](https://spec.filecoin.io/#section-systems.filecoin\_token.payment\_channels.lanes)

In addition, payment channels in Filecoin can be split into `lane`s created as part of updating the channel state with a payment `voucher`. Each lane has an associated `nonce` and amount of tokens it can be `redeemed` for. Lanes can be thought of as transactions for several different services provided by the channel recipient to the channel sender. The `nonce` plays the role of a sequence number of vouchers within a given lane, where a voucher with a higher nonce replaces a voucher with a lower nonce.

Payment channel lanes allow for a lot of accounting between parties to be done off-chain and reconciled via single updates to the payment channel. The multiple lanes enable two parties to use a single payment channel to adjudicate multiple independent sets of payments.

One example of such accounting is _merging of lanes_. When a pair of channel sender-recipient nodes have a payment channel established between them with many lanes, the channel recipient will have to pay gas cost for each one of the lanes in order to `Collect` funds. Merging of lanes allow the channel recipient to send a “merge” request to the channel sender to request merging of (some of the) lanes and consolidate the funds. This way, the recipient can reduce the overall gas cost. As an incentive for the channel sender to accept the merge lane request, the channel recipient can ask for a lower total value to balance out the gas cost. For instance, if the recipient has collected vouchers worth of 10 FIL from two lanes, say 5 from each, and the gas cost of submitting the vouchers for these funds is 2, then it can ask for 9 from the creator if the latter accepts to merge the two lanes. This way, the channel sender pays less overall for the services it received and the channel recipient pays less gas cost to submit the voucher for the services they provided.

### [**Lifecycle of a Payment Channel**](https://spec.filecoin.io/#section-systems.filecoin\_token.payment\_channels.lifecycle-of-a-payment-channel)

Summarising, we have the following sequence:

1. Two parties agree to a series of transactions (for instance as part of file retrieval) with one party paying the other party up to some _total_ sum of Filecoin over time. This is part of the deal-phase, it takes place off-chain and does not (at this stage) involve payment channels.
2. The Payment Channel Actor is used, called by the payment channel sender (who is the recipient of some service, e.g., file in case of file retrieval) to create the payment channel and deposit funds.
3. Any of the two parties can create vouchers to send to the other party.
4. The voucher recipient saves the voucher locally. Each voucher has to be submitted by the opposite party from the one that created the voucher.
5. Either immediately or later, the voucher recipient “redeems” the voucher by submitting it to the chain, calling `UpdateChannelState`
6. The channel sender or the channel recipient `Settle` the payment channel.
7. 12-hour period to close the channel begins.
8. If any of the two parties have outstanding (i.e., non-redeemed) vouchers, they should now submit the vouchers to the chain (there should be the option of this being done automatically). If the channel recipient so desires, they should send a “merge lanes” request to the sender.
9. 12-hour period ends.
10. Either the channel sender or the channel recipient calls `Collect`.
11. Funds are transferred to the channel recipient’s account and any unclaimed balance goes back to channel sender.

### [**Payment Channels as part of the Filecoin Retrieval**](https://spec.filecoin.io/#section-systems.filecoin\_token.payment\_channels.payment-channels-as-part-of-the-filecoin-retrieval)

Payment Channels are used in the Filecoin [Retrieval Market](https://spec.filecoin.io/#section-systems.filecoin\_markets.retrieval\_market) to enable efficient off-chain payments and accounting between parties for what is expected to be a series of microtransactions, as these occur during data retrieval.

In particular, given that there is no proving method provided for the act of sending data from a provider (miner) to a client, there is no trust anchor between the two. Therefore, in order to avoid mis-behaviour, Filecoin is making use of payment channels in order to realise a step-wise “data transfer <-> payment” relationship between the data provider and the client (data receiver). Clients issue requests for data that miners are responding to. The miner is entitled to ask for interim payments, the volume-oriented interval for which is agreed in the Deal phase. In order to facilitate this process, the Filecoin client is creating a payment channel once the provider has agreed on the proposed deal. The client should also lock monetary value in the payment channel equal to the amount needed for retrieval of the entire block of data requested. Every time a provider is completing transfer of the pre-specified amount of data, they can request a payment. The client responds to this request with a voucher which the provider can redeem (immediately or later), as per the process described earlier.

[Example: Payment Channel Implementation ](https://spec.filecoin.io/#example-payment-channel-implementation)

```go
package paychmgr

import (
	"context"
	"fmt"

	"github.com/ipfs/go-cid"
	"golang.org/x/xerrors"

	"github.com/filecoin-project/go-address"
	cborutil "github.com/filecoin-project/go-cbor-util"
	"github.com/filecoin-project/go-state-types/big"

	"github.com/filecoin-project/lotus/api"
	"github.com/filecoin-project/lotus/chain/actors"
	"github.com/filecoin-project/lotus/chain/actors/builtin/paych"
	"github.com/filecoin-project/lotus/chain/types"
	"github.com/filecoin-project/lotus/lib/sigs"
)

// insufficientFundsErr indicates that there are not enough funds in the
// channel to create a voucher
type insufficientFundsErr interface {
	Shortfall() types.BigInt
}

type ErrInsufficientFunds struct {
	shortfall types.BigInt
}

func newErrInsufficientFunds(shortfall types.BigInt) *ErrInsufficientFunds {
	return &ErrInsufficientFunds{shortfall: shortfall}
}

func (e *ErrInsufficientFunds) Error() string {
	return fmt.Sprintf("not enough funds in channel to cover voucher - shortfall: %d", e.shortfall)
}

func (e *ErrInsufficientFunds) Shortfall() types.BigInt {
	return e.shortfall
}

type laneState struct {
	redeemed big.Int
	nonce    uint64
}

func (ls laneState) Redeemed() (big.Int, error) {
	return ls.redeemed, nil
}

func (ls laneState) Nonce() (uint64, error) {
	return ls.nonce, nil
}

// channelAccessor is used to simplify locking when accessing a channel
type channelAccessor struct {
	from address.Address
	to   address.Address

	// chctx is used by background processes (eg when waiting for things to be
	// confirmed on chain)
	chctx         context.Context
	sa            *stateAccessor
	api           managerAPI
	store         *Store
	lk            *channelLock
	fundsReqQueue []*fundsReq
	msgListeners  msgListeners
}

func newChannelAccessor(pm *Manager, from address.Address, to address.Address) *channelAccessor {
	return &channelAccessor{
		from:         from,
		to:           to,
		chctx:        pm.ctx,
		sa:           pm.sa,
		api:          pm.pchapi,
		store:        pm.store,
		lk:           &channelLock{globalLock: &pm.lk},
		msgListeners: newMsgListeners(),
	}
}

func (ca *channelAccessor) messageBuilder(ctx context.Context, from address.Address) (paych.MessageBuilder, error) {
	nwVersion, err := ca.api.StateNetworkVersion(ctx, types.EmptyTSK)
	if err != nil {
		return nil, err
	}

	av, err := actors.VersionForNetwork(nwVersion)
	if err != nil {
		return nil, err
	}
	return paych.Message(av, from), nil
}

func (ca *channelAccessor) getChannelInfo(addr address.Address) (*ChannelInfo, error) {
	ca.lk.Lock()
	defer ca.lk.Unlock()

	return ca.store.ByAddress(addr)
}

func (ca *channelAccessor) outboundActiveByFromTo(from, to address.Address) (*ChannelInfo, error) {
	ca.lk.Lock()
	defer ca.lk.Unlock()

	return ca.store.OutboundActiveByFromTo(from, to)
}

// createVoucher creates a voucher with the given specification, setting its
// nonce, signing the voucher and storing it in the local datastore.
// If there are not enough funds in the channel to create the voucher, returns
// the shortfall in funds.
func (ca *channelAccessor) createVoucher(ctx context.Context, ch address.Address, voucher paych.SignedVoucher) (*api.VoucherCreateResult, error) {
	ca.lk.Lock()
	defer ca.lk.Unlock()

	// Find the channel for the voucher
	ci, err := ca.store.ByAddress(ch)
	if err != nil {
		return nil, xerrors.Errorf("failed to get channel info by address: %w", err)
	}

	// Set the voucher channel
	sv := &voucher
	sv.ChannelAddr = ch

	// Get the next nonce on the given lane
	sv.Nonce = ca.nextNonceForLane(ci, voucher.Lane)

	// Sign the voucher
	vb, err := sv.SigningBytes()
	if err != nil {
		return nil, xerrors.Errorf("failed to get voucher signing bytes: %w", err)
	}

	sig, err := ca.api.WalletSign(ctx, ci.Control, vb)
	if err != nil {
		return nil, xerrors.Errorf("failed to sign voucher: %w", err)
	}
	sv.Signature = sig

	// Store the voucher
	if _, err := ca.addVoucherUnlocked(ctx, ch, sv, types.NewInt(0)); err != nil {
		// If there are not enough funds in the channel to cover the voucher,
		// return a voucher create result with the shortfall
		var ife insufficientFundsErr
		if xerrors.As(err, &ife) {
			return &api.VoucherCreateResult{
				Shortfall: ife.Shortfall(),
			}, nil
		}

		return nil, xerrors.Errorf("failed to persist voucher: %w", err)
	}

	return &api.VoucherCreateResult{Voucher: sv, Shortfall: types.NewInt(0)}, nil
}

func (ca *channelAccessor) nextNonceForLane(ci *ChannelInfo, lane uint64) uint64 {
	var maxnonce uint64
	for _, v := range ci.Vouchers {
		if v.Voucher.Lane == lane {
			if v.Voucher.Nonce > maxnonce {
				maxnonce = v.Voucher.Nonce
			}
		}
	}

	return maxnonce + 1
}

func (ca *channelAccessor) checkVoucherValid(ctx context.Context, ch address.Address, sv *paych.SignedVoucher) (map[uint64]paych.LaneState, error) {
	ca.lk.Lock()
	defer ca.lk.Unlock()

	return ca.checkVoucherValidUnlocked(ctx, ch, sv)
}

func (ca *channelAccessor) checkVoucherValidUnlocked(ctx context.Context, ch address.Address, sv *paych.SignedVoucher) (map[uint64]paych.LaneState, error) {
	if sv.ChannelAddr != ch {
		return nil, xerrors.Errorf("voucher ChannelAddr doesn't match channel address, got %s, expected %s", sv.ChannelAddr, ch)
	}

	// check voucher is unlocked
	if sv.Extra != nil {
		return nil, xerrors.Errorf("voucher is Message Locked")
	}
	if sv.TimeLockMax != 0 {
		return nil, xerrors.Errorf("voucher is Max Time Locked")
	}
	if sv.TimeLockMin != 0 {
		return nil, xerrors.Errorf("voucher is Min Time Locked")
	}
	if len(sv.SecretPreimage) != 0 {
		return nil, xerrors.Errorf("voucher is Hash Locked")
	}

	// Load payment channel actor state
	act, pchState, err := ca.sa.loadPaychActorState(ctx, ch)
	if err != nil {
		return nil, err
	}

	// Load channel "From" account actor state
	f, err := pchState.From()
	if err != nil {
		return nil, err
	}

	from, err := ca.api.ResolveToKeyAddress(ctx, f, nil)
	if err != nil {
		return nil, err
	}

	// verify voucher signature
	vb, err := sv.SigningBytes()
	if err != nil {
		return nil, err
	}

	// TODO: technically, either party may create and sign a voucher.
	// However, for now, we only accept them from the channel creator.
	// More complex handling logic can be added later
	if err := sigs.Verify(sv.Signature, from, vb); err != nil {
		return nil, err
	}

	// Check the voucher against the highest known voucher nonce / value
	laneStates, err := ca.laneState(pchState, ch)
	if err != nil {
		return nil, err
	}

	// If the new voucher nonce value is less than the highest known
	// nonce for the lane
	ls, lsExists := laneStates[sv.Lane]
	if lsExists {
		n, err := ls.Nonce()
		if err != nil {
			return nil, err
		}

		if sv.Nonce <= n {
			return nil, fmt.Errorf("nonce too low")
		}

		// If the voucher amount is less than the highest known voucher amount
		r, err := ls.Redeemed()
		if err != nil {
			return nil, err
		}
		if sv.Amount.LessThanEqual(r) {
			return nil, fmt.Errorf("voucher amount is lower than amount for voucher with lower nonce")
		}
	}

	// Total redeemed is the total redeemed amount for all lanes, including
	// the new voucher
	// eg
	//
	// lane 1 redeemed:            3
	// lane 2 redeemed:            2
	// voucher for lane 1:         5
	//
	// Voucher supersedes lane 1 redeemed, therefore
	// effective lane 1 redeemed:  5
	//
	// lane 1:  5
	// lane 2:  2
	//          -
	// total:   7
	totalRedeemed, err := ca.totalRedeemedWithVoucher(laneStates, sv)
	if err != nil {
		return nil, err
	}

	// Total required balance must not exceed actor balance
	if act.Balance.LessThan(totalRedeemed) {
		return nil, newErrInsufficientFunds(types.BigSub(totalRedeemed, act.Balance))
	}

	if len(sv.Merges) != 0 {
		return nil, fmt.Errorf("dont currently support paych lane merges")
	}

	return laneStates, nil
}

func (ca *channelAccessor) checkVoucherSpendable(ctx context.Context, ch address.Address, sv *paych.SignedVoucher, secret []byte) (bool, error) {
	ca.lk.Lock()
	defer ca.lk.Unlock()

	recipient, err := ca.getPaychRecipient(ctx, ch)
	if err != nil {
		return false, err
	}

	ci, err := ca.store.ByAddress(ch)
	if err != nil {
		return false, err
	}

	// Check if voucher has already been submitted
	submitted, err := ci.wasVoucherSubmitted(sv)
	if err != nil {
		return false, err
	}
	if submitted {
		return false, nil
	}

	mb, err := ca.messageBuilder(ctx, recipient)
	if err != nil {
		return false, err
	}

	mes, err := mb.Update(ch, sv, secret)
	if err != nil {
		return false, err
	}

	ret, err := ca.api.Call(ctx, mes, nil)
	if err != nil {
		return false, err
	}

	if ret.MsgRct.ExitCode != 0 {
		return false, nil
	}

	return true, nil
}

func (ca *channelAccessor) getPaychRecipient(ctx context.Context, ch address.Address) (address.Address, error) {
	_, state, err := ca.api.GetPaychState(ctx, ch, nil)
	if err != nil {
		return address.Address{}, err
	}

	return state.To()
}

func (ca *channelAccessor) addVoucher(ctx context.Context, ch address.Address, sv *paych.SignedVoucher, minDelta types.BigInt) (types.BigInt, error) {
	ca.lk.Lock()
	defer ca.lk.Unlock()

	return ca.addVoucherUnlocked(ctx, ch, sv, minDelta)
}

func (ca *channelAccessor) addVoucherUnlocked(ctx context.Context, ch address.Address, sv *paych.SignedVoucher, minDelta types.BigInt) (types.BigInt, error) {
	ci, err := ca.store.ByAddress(ch)
	if err != nil {
		return types.BigInt{}, err
	}

	// Check if the voucher has already been added
	for _, v := range ci.Vouchers {
		eq, err := cborutil.Equals(sv, v.Voucher)
		if err != nil {
			return types.BigInt{}, err
		}
		if eq {
			// Ignore the duplicate voucher.
			log.Warnf("AddVoucher: voucher re-added")
			return types.NewInt(0), nil
		}

	}

	// Check voucher validity
	laneStates, err := ca.checkVoucherValidUnlocked(ctx, ch, sv)
	if err != nil {
		return types.NewInt(0), err
	}

	// The change in value is the delta between the voucher amount and
	// the highest previous voucher amount for the lane
	laneState, exists := laneStates[sv.Lane]
	redeemed := big.NewInt(0)
	if exists {
		redeemed, err = laneState.Redeemed()
		if err != nil {
			return types.NewInt(0), err
		}
	}

	delta := types.BigSub(sv.Amount, redeemed)
	if minDelta.GreaterThan(delta) {
		return delta, xerrors.Errorf("addVoucher: supplied token amount too low; minD=%s, D=%s; laneAmt=%s; v.Amt=%s", minDelta, delta, redeemed, sv.Amount)
	}

	ci.Vouchers = append(ci.Vouchers, &VoucherInfo{
		Voucher: sv,
	})

	if ci.NextLane <= sv.Lane {
		ci.NextLane = sv.Lane + 1
	}

	return delta, ca.store.putChannelInfo(ci)
}

func (ca *channelAccessor) submitVoucher(ctx context.Context, ch address.Address, sv *paych.SignedVoucher, secret []byte) (cid.Cid, error) {
	ca.lk.Lock()
	defer ca.lk.Unlock()

	ci, err := ca.store.ByAddress(ch)
	if err != nil {
		return cid.Undef, err
	}

	has, err := ci.hasVoucher(sv)
	if err != nil {
		return cid.Undef, err
	}

	// If the channel has the voucher
	if has {
		// Check that the voucher hasn't already been submitted
		submitted, err := ci.wasVoucherSubmitted(sv)
		if err != nil {
			return cid.Undef, err
		}
		if submitted {
			return cid.Undef, xerrors.Errorf("cannot submit voucher that has already been submitted")
		}
	}

	mb, err := ca.messageBuilder(ctx, ci.Control)
	if err != nil {
		return cid.Undef, err
	}

	msg, err := mb.Update(ch, sv, secret)
	if err != nil {
		return cid.Undef, err
	}

	smsg, err := ca.api.MpoolPushMessage(ctx, msg, nil)
	if err != nil {
		return cid.Undef, err
	}

	// If the channel didn't already have the voucher
	if !has {
		// Add the voucher to the channel
		ci.Vouchers = append(ci.Vouchers, &VoucherInfo{
			Voucher: sv,
		})
	}

	// Mark the voucher and any lower-nonce vouchers as having been submitted
	err = ca.store.MarkVoucherSubmitted(ci, sv)
	if err != nil {
		return cid.Undef, err
	}

	return smsg.Cid(), nil
}

func (ca *channelAccessor) allocateLane(ch address.Address) (uint64, error) {
	ca.lk.Lock()
	defer ca.lk.Unlock()

	return ca.store.AllocateLane(ch)
}

func (ca *channelAccessor) listVouchers(ctx context.Context, ch address.Address) ([]*VoucherInfo, error) {
	ca.lk.Lock()
	defer ca.lk.Unlock()

	// TODO: just having a passthrough method like this feels odd. Seems like
	// there should be some filtering we're doing here
	return ca.store.VouchersForPaych(ch)
}

// laneState gets the LaneStates from chain, then applies all vouchers in
// the data store over the chain state
func (ca *channelAccessor) laneState(state paych.State, ch address.Address) (map[uint64]paych.LaneState, error) {
	// TODO: we probably want to call UpdateChannelState with all vouchers to be fully correct
	//  (but technically dont't need to)

	laneCount, err := state.LaneCount()
	if err != nil {
		return nil, err
	}

	// Note: we use a map instead of an array to store laneStates because the
	// client sets the lane ID (the index) and potentially they could use a
	// very large index.
	laneStates := make(map[uint64]paych.LaneState, laneCount)
	err = state.ForEachLaneState(func(idx uint64, ls paych.LaneState) error {
		laneStates[idx] = ls
		return nil
	})
	if err != nil {
		return nil, err
	}

	// Apply locally stored vouchers
	vouchers, err := ca.store.VouchersForPaych(ch)
	if err != nil && err != ErrChannelNotTracked {
		return nil, err
	}

	for _, v := range vouchers {
		for range v.Voucher.Merges {
			return nil, xerrors.Errorf("paych merges not handled yet")
		}

		// Check if there is an existing laneState in the payment channel
		// for this voucher's lane
		ls, ok := laneStates[v.Voucher.Lane]

		// If the voucher does not have a higher nonce than the existing
		// laneState for this lane, ignore it
		if ok {
			n, err := ls.Nonce()
			if err != nil {
				return nil, err
			}
			if v.Voucher.Nonce < n {
				continue
			}
		}

		// Voucher has a higher nonce, so replace laneState with this voucher
		laneStates[v.Voucher.Lane] = laneState{v.Voucher.Amount, v.Voucher.Nonce}
	}

	return laneStates, nil
}

// Get the total redeemed amount across all lanes, after applying the voucher
func (ca *channelAccessor) totalRedeemedWithVoucher(laneStates map[uint64]paych.LaneState, sv *paych.SignedVoucher) (big.Int, error) {
	// TODO: merges
	if len(sv.Merges) != 0 {
		return big.Int{}, xerrors.Errorf("dont currently support paych lane merges")
	}

	total := big.NewInt(0)
	for _, ls := range laneStates {
		r, err := ls.Redeemed()
		if err != nil {
			return big.Int{}, err
		}
		total = big.Add(total, r)
	}

	lane, ok := laneStates[sv.Lane]
	if ok {
		// If the voucher is for an existing lane, and the voucher nonce
		// is higher than the lane nonce
		n, err := lane.Nonce()
		if err != nil {
			return big.Int{}, err
		}

		if sv.Nonce > n {
			// Add the delta between the redeemed amount and the voucher
			// amount to the total
			r, err := lane.Redeemed()
			if err != nil {
				return big.Int{}, err
			}

			delta := big.Sub(sv.Amount, r)
			total = big.Add(total, delta)
		}
	} else {
		// If the voucher is *not* for an existing lane, just add its
		// value (implicitly a new lane will be created for the voucher)
		total = big.Add(total, sv.Amount)
	}

	return total, nil
}

func (ca *channelAccessor) settle(ctx context.Context, ch address.Address) (cid.Cid, error) {
	ca.lk.Lock()
	defer ca.lk.Unlock()

	ci, err := ca.store.ByAddress(ch)
	if err != nil {
		return cid.Undef, err
	}

	mb, err := ca.messageBuilder(ctx, ci.Control)
	if err != nil {
		return cid.Undef, err
	}
	msg, err := mb.Settle(ch)
	if err != nil {
		return cid.Undef, err
	}
	smgs, err := ca.api.MpoolPushMessage(ctx, msg, nil)
	if err != nil {
		return cid.Undef, err
	}

	ci.Settling = true
	err = ca.store.putChannelInfo(ci)
	if err != nil {
		log.Errorf("Error marking channel as settled: %s", err)
	}

	return smgs.Cid(), err
}

func (ca *channelAccessor) collect(ctx context.Context, ch address.Address) (cid.Cid, error) {
	ca.lk.Lock()
	defer ca.lk.Unlock()

	ci, err := ca.store.ByAddress(ch)
	if err != nil {
		return cid.Undef, err
	}

	mb, err := ca.messageBuilder(ctx, ci.Control)
	if err != nil {
		return cid.Undef, err
	}

	msg, err := mb.Collect(ch)
	if err != nil {
		return cid.Undef, err
	}

	smsg, err := ca.api.MpoolPushMessage(ctx, msg, nil)
	if err != nil {
		return cid.Undef, err
	}

	return smsg.Cid(), nil
}
```

[Example: SignedVoucher ](https://spec.filecoin.io/#example-signedvoucher)A voucher is sent by `From` to `To` off-chain in order to enable `To` to redeem payments on-chain in the future

```go
type SignedVoucher struct {
	// ChannelAddr is the address of the payment channel this signed voucher is valid for
	ChannelAddr addr.Address
	// TimeLockMin sets a min epoch before which the voucher cannot be redeemed
	TimeLockMin abi.ChainEpoch
	// TimeLockMax sets a max epoch beyond which the voucher cannot be redeemed
	// TimeLockMax set to 0 means no timeout
	TimeLockMax abi.ChainEpoch
	// (optional) The SecretPreImage is used by `To` to validate
	SecretPreimage []byte
	// (optional) Extra can be specified by `From` to add a verification method to the voucher
	Extra *ModVerifyParams
	// Specifies which lane the Voucher merges into (will be created if does not exist)
	Lane uint64
	// Nonce is set by `From` to prevent redemption of stale vouchers on a lane
	Nonce uint64
	// Amount voucher can be redeemed for
	Amount big.Int
	// (optional) MinSettleHeight can extend channel MinSettleHeight if needed
	MinSettleHeight abi.ChainEpoch

	// (optional) Set of lanes to be merged into `Lane`
	Merges []Merge

	// Sender's signature over the voucher
	Signature *crypto.Signature
}
```

[Example: Payment Channel Actor ](https://spec.filecoin.io/#example-payment-channel-actor)

```go
package paych

import (
	"bytes"

	addr "github.com/filecoin-project/go-address"
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/big"
	"github.com/filecoin-project/go-state-types/cbor"
	"github.com/filecoin-project/go-state-types/exitcode"
	paych0 "github.com/filecoin-project/specs-actors/actors/builtin/paych"
	paych2 "github.com/filecoin-project/specs-actors/v2/actors/builtin/paych"

	"github.com/ipfs/go-cid"

	"github.com/filecoin-project/specs-actors/v7/actors/builtin"
	"github.com/filecoin-project/specs-actors/v7/actors/runtime"
	"github.com/filecoin-project/specs-actors/v7/actors/util/adt"
)

const (
	ErrChannelStateUpdateAfterSettled = exitcode.FirstActorSpecificExitCode + iota
)

type Actor struct{}

func (a Actor) Exports() []interface{} {
	return []interface{}{
		builtin.MethodConstructor: a.Constructor,
		2:                         a.UpdateChannelState,
		3:                         a.Settle,
		4:                         a.Collect,
	}
}

func (a Actor) Code() cid.Cid {
	return builtin.PaymentChannelActorCodeID
}

func (a Actor) State() cbor.Er {
	return new(State)
}

var _ runtime.VMActor = Actor{}

//type ConstructorParams struct {
//	From addr.Address // Payer
//	To   addr.Address // Payee
//}
type ConstructorParams = paych0.ConstructorParams

// Constructor creates a payment channel actor. See State for meaning of params.
func (pca *Actor) Constructor(rt runtime.Runtime, params *ConstructorParams) *abi.EmptyValue {
	// Only InitActor can create a payment channel actor. It creates the actor on
	// behalf of the payer/payee.
	rt.ValidateImmediateCallerType(builtin.InitActorCodeID)

	// check that both parties are capable of signing vouchers
	to, err := pca.resolveAccount(rt, params.To)
	builtin.RequireNoErr(rt, err, exitcode.Unwrap(err, exitcode.ErrIllegalState), "failed to resolve to address: %s", params.To)
	from, err := pca.resolveAccount(rt, params.From)
	builtin.RequireNoErr(rt, err, exitcode.Unwrap(err, exitcode.ErrIllegalState), "failed to resolve from address: %s", params.From)

	emptyArr, err := adt.MakeEmptyArray(adt.AsStore(rt), LaneStatesAmtBitwidth)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to create empty array")
	emptyArrCid, err := emptyArr.Root()
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to persist empty array")

	st := ConstructState(from, to, emptyArrCid)
	rt.StateCreate(st)

	return nil
}

// Resolves an address to a canonical ID address and requires it to address an account actor.
func (pca *Actor) resolveAccount(rt runtime.Runtime, raw addr.Address) (addr.Address, error) {
	resolved, err := builtin.ResolveToIDAddr(rt, raw)
	if err != nil {
		return addr.Undef, exitcode.ErrIllegalState.Wrapf("failed to resolve address %v: %w", raw, err)
	}

	codeCID, ok := rt.GetActorCodeCID(resolved)
	if !ok {
		return addr.Undef, exitcode.ErrIllegalArgument.Wrapf("no code for address %v", resolved)
	}
	if codeCID != builtin.AccountActorCodeID {
		return addr.Undef, exitcode.ErrForbidden.Wrapf("actor %v must be an account (%v), was %v", raw,
			builtin.AccountActorCodeID, codeCID)
	}

	return resolved, nil
}

////////////////////////////////////////////////////////////////////////////////
// Payment Channel state operations
////////////////////////////////////////////////////////////////////////////////

// type UpdateChannelStateParams struct {
// 	Sv     SignedVoucher
// 	Secret []byte
// }
type UpdateChannelStateParams = paych2.UpdateChannelStateParams

// A voucher is sent by `From` to `To` off-chain in order to enable
// `To` to redeem payments on-chain in the future
//type SignedVoucher struct {
//	// ChannelAddr is the address of the payment channel this signed voucher is valid for
//	ChannelAddr addr.Address
//	// TimeLockMin sets a min epoch before which the voucher cannot be redeemed
//	TimeLockMin abi.ChainEpoch
//	// TimeLockMax sets a max epoch beyond which the voucher cannot be redeemed
//	// TimeLockMax set to 0 means no timeout
//	TimeLockMax abi.ChainEpoch
//	// (optional) The SecretPreImage is used by `To` to validate
//	SecretPreimage []byte
//	// (optional) Extra can be specified by `From` to add a verification method to the voucher.
//	Extra *ModVerifyParams
//	// Specifies which lane the Voucher merges into (will be created if does not exist)
//	Lane uint64
//	// Nonce is set by `From` to prevent redemption of stale vouchers on a lane
//	Nonce uint64
//	// Amount voucher can be redeemed for
//	Amount big.Int
//	// (optional) MinSettleHeight can extend channel MinSettleHeight if needed
//	MinSettleHeight abi.ChainEpoch
//
//	// (optional) Set of lanes to be merged into `Lane`
//	Merges []Merge
//
//	// Sender's signature over the voucher
//	Signature *crypto.Signature
//}
type SignedVoucher = paych0.SignedVoucher

// Modular Verification method
//type ModVerifyParams struct {
//	// Actor on which to invoke the method.
//	Actor addr.Address
//	// Method to invoke.
//	Method abi.MethodNum
//	// Pre-serialized method parameters.
//	Params []byte
//}
type ModVerifyParams = paych0.ModVerifyParams

// Specifies which `Lane`s to be merged with what `Nonce` on channelUpdate
//type Merge struct {
//	Lane  uint64
//	Nonce uint64
//}
type Merge = paych0.Merge

func (pca Actor) UpdateChannelState(rt runtime.Runtime, params *UpdateChannelStateParams) *abi.EmptyValue {
	var st State
	rt.StateReadonly(&st)

	// both parties must sign voucher: one who submits it, the other explicitly signs it
	rt.ValidateImmediateCallerIs(st.From, st.To)
	var signer addr.Address
	if rt.Caller() == st.From {
		signer = st.To
	} else {
		signer = st.From
	}
	sv := params.Sv

	if sv.Signature == nil {
		rt.Abortf(exitcode.ErrIllegalArgument, "voucher has no signature")
	}

	if st.SettlingAt != 0 && rt.CurrEpoch() >= st.SettlingAt {
		rt.Abortf(ErrChannelStateUpdateAfterSettled, "no vouchers can be processed after SettlingAt epoch")
	}

	if len(params.Secret) > MaxSecretSize {
		rt.Abortf(exitcode.ErrIllegalArgument, "secret must be at most 256 bytes long")
	}

	vb, err := sv.SigningBytes()
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "failed to serialize signedvoucher")

	err = rt.VerifySignature(*sv.Signature, signer, vb)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "voucher signature invalid")

	pchAddr := rt.Receiver()
	svpchIDAddr, found := rt.ResolveAddress(sv.ChannelAddr)
	if !found {
		rt.Abortf(exitcode.ErrIllegalArgument, "voucher payment channel address %s does not resolve to an ID address", sv.ChannelAddr)
	}
	if pchAddr != svpchIDAddr {
		rt.Abortf(exitcode.ErrIllegalArgument, "voucher payment channel address %s does not match receiver %s", svpchIDAddr, pchAddr)
	}

	if rt.CurrEpoch() < sv.TimeLockMin {
		rt.Abortf(exitcode.ErrIllegalArgument, "cannot use this voucher yet!")
	}

	if sv.TimeLockMax != 0 && rt.CurrEpoch() > sv.TimeLockMax {
		rt.Abortf(exitcode.ErrIllegalArgument, "this voucher has expired!")
	}

	if sv.Amount.Sign() < 0 {
		rt.Abortf(exitcode.ErrIllegalArgument, "voucher amount must be non-negative, was %v", sv.Amount)
	}

	if len(sv.SecretPreimage) > 0 {
		hashedSecret := rt.HashBlake2b(params.Secret)
		if !bytes.Equal(hashedSecret[:], sv.SecretPreimage) {
			rt.Abortf(exitcode.ErrIllegalArgument, "incorrect secret!")
		}
	}

	if sv.Extra != nil {

		code := rt.Send(
			sv.Extra.Actor,
			sv.Extra.Method,
			builtin.CBORBytes(sv.Extra.Data),
			abi.NewTokenAmount(0),
			&builtin.Discard{},
		)
		builtin.RequireSuccess(rt, code, "spend voucher verification failed")
	}

	rt.StateTransaction(&st, func() {
		laneFound := true

		lstates, err := adt.AsArray(adt.AsStore(rt), st.LaneStates, LaneStatesAmtBitwidth)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load lanes")

		// Find the voucher lane, creating if necessary.
		laneId := sv.Lane
		laneState := findLane(rt, lstates, sv.Lane)

		if laneState == nil {
			laneState = &LaneState{
				Redeemed: big.Zero(),
				Nonce:    0,
			}
			laneFound = false
		}

		if laneFound {
			if laneState.Nonce >= sv.Nonce {
				rt.Abortf(exitcode.ErrIllegalArgument, "voucher has an outdated nonce, existing nonce: %d, voucher nonce: %d, cannot redeem",
					laneState.Nonce, sv.Nonce)
			}
		}

		// The next section actually calculates the payment amounts to update the payment channel state
		// 1. (optional) sum already redeemed value of all merging lanes
		redeemedFromOthers := big.Zero()
		for _, merge := range sv.Merges {
			if merge.Lane == sv.Lane {
				rt.Abortf(exitcode.ErrIllegalArgument, "voucher cannot merge lanes into its own lane")
			}

			otherls := findLane(rt, lstates, merge.Lane)
			if otherls == nil {
				rt.Abortf(exitcode.ErrIllegalArgument, "voucher specifies invalid merge lane %v", merge.Lane)
				return // makes linters happy
			}

			if otherls.Nonce >= merge.Nonce {
				rt.Abortf(exitcode.ErrIllegalArgument, "merged lane in voucher has outdated nonce, cannot redeem")
			}

			redeemedFromOthers = big.Add(redeemedFromOthers, otherls.Redeemed)
			otherls.Nonce = merge.Nonce
			err = lstates.Set(merge.Lane, otherls)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to store lane %d", merge.Lane)
		}

		// 2. To prevent double counting, remove already redeemed amounts (from
		// voucher or other lanes) from the voucher amount
		laneState.Nonce = sv.Nonce
		balanceDelta := big.Sub(sv.Amount, big.Add(redeemedFromOthers, laneState.Redeemed))
		// 3. set new redeemed value for merged-into lane
		laneState.Redeemed = sv.Amount

		newSendBalance := big.Add(st.ToSend, balanceDelta)

		// 4. check operation validity
		if newSendBalance.LessThan(big.Zero()) {
			rt.Abortf(exitcode.ErrIllegalArgument, "voucher would leave channel balance negative")
		}
		if newSendBalance.GreaterThan(rt.CurrentBalance()) {
			rt.Abortf(exitcode.ErrIllegalArgument, "not enough funds in channel to cover voucher")
		}

		// 5. add new redemption ToSend
		st.ToSend = newSendBalance

		// update channel settlingAt and MinSettleHeight if delayed by voucher
		if sv.MinSettleHeight != 0 {
			if st.SettlingAt != 0 && st.SettlingAt < sv.MinSettleHeight {
				st.SettlingAt = sv.MinSettleHeight
			}
			if st.MinSettleHeight < sv.MinSettleHeight {
				st.MinSettleHeight = sv.MinSettleHeight
			}
		}

		err = lstates.Set(laneId, laneState)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to store lane", laneId)

		st.LaneStates, err = lstates.Root()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save lanes")
	})
	return nil
}

func (pca Actor) Settle(rt runtime.Runtime, _ *abi.EmptyValue) *abi.EmptyValue {
	var st State
	rt.StateTransaction(&st, func() {
		rt.ValidateImmediateCallerIs(st.From, st.To)

		if st.SettlingAt != 0 {
			rt.Abortf(exitcode.ErrIllegalState, "channel already settling")
		}

		st.SettlingAt = rt.CurrEpoch() + SettleDelay
		if st.SettlingAt < st.MinSettleHeight {
			st.SettlingAt = st.MinSettleHeight
		}
	})
	return nil
}

func (pca Actor) Collect(rt runtime.Runtime, _ *abi.EmptyValue) *abi.EmptyValue {
	var st State
	rt.StateReadonly(&st)
	rt.ValidateImmediateCallerIs(st.From, st.To)

	if st.SettlingAt == 0 || rt.CurrEpoch() < st.SettlingAt {
		rt.Abortf(exitcode.ErrForbidden, "payment channel not settling or settled")
	}

	// send ToSend to "To"
	codeTo := rt.Send(
		st.To,
		builtin.MethodSend,
		nil,
		st.ToSend,
		&builtin.Discard{},
	)
	builtin.RequireSuccess(rt, codeTo, "Failed to send funds to `To`")

	// the remaining balance will be returned to "From" upon deletion.
	rt.DeleteActor(st.From)

	return nil
}

// Returns the insertion index for a lane ID, with the matching lane state if found, or nil.
func findLane(rt runtime.Runtime, ls *adt.Array, id uint64) *LaneState {
	if id > MaxLane {
		rt.Abortf(exitcode.ErrIllegalArgument, "maximum lane ID is 2^63-1")
	}

	var out LaneState
	found, err := ls.Get(id, &out)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load lane %d", id)

	if !found {
		return nil
	}

	return &out
}
```

## [Multisig Wallet & Actor](https://spec.filecoin.io/#section-systems.filecoin\_token.multisig) <a href="#section-systems.filecoin_token.multisig" id="section-systems.filecoin_token.multisig"></a>

The Multisig actor is a single actor representing a group of Signers. Signers may be external users, other Multisigs, or even the Multisig itself. There should be a maximum of 256 signers in a multisig wallet. In case more signers are needed, then the multisigs should be combined into a tree.

The implementation of the Multisig Actor can be found [here](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/multisig/multisig\_actor.go).

The Multisig Actor statuses can be found [here](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/multisig/multisig\_state.go).
