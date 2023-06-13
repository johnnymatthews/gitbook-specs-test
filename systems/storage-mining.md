# Storage mining

The Storage Mining System is the part of the Filecoin Protocol that deals with storing Client’s data and producing proof artifacts that demonstrate correct storage behavior.

Storage Mining is one of the most central parts of the Filecoin protocol overall, as it provides all the required consensus algorithms based on proven _storage power_ in the network. Miners are selected to mine blocks and extend the blockchain based on the storage power that they have committed to the network. Storage is added in unit of sectors and sectors are promises to the network that some storage will remain for a promised duration. In order to participate in Storage Mining, the storage miners have to: i) Add storage to the system, and ii) Prove that they maintain a copy of the data they have agreed to throughout the sector’s lifetime.

Storing data and producing proofs is a complex, highly optimizable process, with lots of tunable choices. Miners should explore the design space to arrive at something that (a) satisfies protocol and network-wide constraints, (b) satisfies clients' requests and expectations (as expressed in `Deals`), and (c) gives them the most cost-effective operation. This part of the Filecoin Spec primarily describes in detail what MUST and SHOULD happen here, and leaves ample room for various optimizations for implementers, miners, and users to make. In some parts, we describe algorithms that could be replaced by other, more optimized versions, but in those cases it is important that the **protocol constraints** are satisfied. The **protocol constraints** are spelled out in clear detail. It is up to implementers who deviate from the algorithms presented here to ensure their modifications satisfy those constraints, especially those relating to protocol security.

## [Sector](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector) <a href="#section-systems.filecoin_mining.sector" id="section-systems.filecoin_mining.sector"></a>

Sectors are the basic units of storage on Filecoin. They have standard sizes, as well as well-defined time-increments for commitments. The size of a sector balances security concerns against usability. A sectorʼs lifetime is determined in the storage market, and sets the promised duration of the sector.

In the first iteration of the protocol, 32GiB and 64GiB sectors are supported. Maximum sector lifetime is determined by the proof algorithm. Maximum sector lifetime is initially 18 months. A sector naturally expires when it reaches the end of its lifetime. Additionally, the miner can extend the lifetime of their sectors. Rewards are earned and collaterals recovered when the miner fulfils their commitment.

Individual deals are formed when a storage miner and client are matched on Filecoinʼs storage market. The protocol does not distinguish miners matching with real clients from miners generating self-deals. However, **committed capacity** is a construction that is introduced to make self-dealing unnecessary and economically irrational. In earlier designs of the network, only sectors filled with deals increased the minerʼs likelihood of winning the block reward. This led to the expectation that miners would attack and exploit the network by playing the role of both storage provider and client, creating a malicious self-deal.

If a sector is only partially full of deals, the network considers the remainder to be _committed capacity_. Similarly, sectors with no deals are called committed capacity sectors; miners are rewarded for proving to the network that they are pledging storage capacity and are encouraged to find clients who need storage. When a miner finds storage demand, they can upgrade their committed capacity sectors to earn additional revenue in the form of a deal fee from paying clients. More details on how to add storage and upgrade sectors in [Adding Storage](https://spec.filecoin.io/#section-systems.filecoin\_mining.adding\_storage).

Committed capacity sectors improve minersʼ incentives to store client data, but they donʼt solve the problem entirely. Storing real client files adds some operational overhead for storage miners. In certain circumstances – for example, if a miner values block rewards far more than deal fees – miners might still choose to ignore client data entirely and simply store committed capacity to increase their storage power as rapidly as possible in pursuit of block rewards. This would make Filecoin less useful and limit clientsʼ ability to store data on the network. Filecoin addresses this issue by introducing the concept of verified clients. Verified clients are certified by a decentralized network of verifiers. Once verified, they can post a predetermined amount of verified client deal data to the storage market, set by the size of their DataCap. Sectors with verified client deals are awarded more storage power – and therefore more block rewards – than sectors without. This provides storage miners with an additional incentive to store client data.

Verification is not intended to be scarce – it will be very easy to acquire for anyone with real data to store on Filecoin. Even though verifiers may allocate verified client DataCaps liberally (yet responsibly and transparently) to make onboarding easier, the overall effect should be a dramatic increase in the proportion of useful data stored on Filecoin.

Once a sector is full (either with client data or as committed capacity), the unsealed sector is combined by a proving tree into a single root `UnsealedSectorCID`. The sealing process then encodes (using CBOR) an unsealed sector into a sealed sector, with the root `SealedSectorCID`.

This diagram shows the composition of an unsealed sector and a sealed sector.

<figure><img src="https://spec.filecoin.io/systems/filecoin_mining/sector/sectors.png" alt="Unsealed Sectors and Sealed Sectors" height="100%"><figcaption><p><a href="https://spec.filecoin.io/#figure-unsealed-sectors-and-sealed-sectors">Figure: Unsealed Sectors and Sealed Sectors</a> </p></figcaption></figure>

**Sector Storage & Window PoSt**

The Lotus implementation of the Window PoSt scheduler can be found [here](https://github.com/filecoin-project/lotus/blob/master/storage/wdpost\_sched.go) and the actual execution of Window PoSt on a sector can be found [here](https://github.com/filecoin-project/lotus/blob/master/storage/wdpost\_run.go).

The Lotus block store implementation for sectors can be found [here](https://github.com/filecoin-project/lotus/blob/master/storage/sectorblocks/blocks.go).

### [**Sector Lifecycle**](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector.lifecycle)

Once the sector has been generated and the deal has been incorporated into the Filecoin blockchain, the storage miner begins generating Proofs-of-Spacetime (PoSt) on the sector, starting to potentially win block rewards and also earn storage fees. Parameters are set so that miners generate and capture more value if they guarantee that their sectors will be around for the duration of the original contract. However, some bounds are placed on a sectorʼs lifetime to improve the network performance.

In particular, as sectors of shorter lifetime are added, the networkʼs capacity can be bottlenecked. The reason is that the chainʼs bandwidth is consumed with new sectors only replacing expiring ones. As a result, a minimum sector lifetime of six months was introduced to more effectively utilize chain bandwidth and miners have the incentive to commit to sectors of longer lifetime. The maximum sector lifetime is limited by the security of the present proofs construction. For a given set of proofs and parameters, the security of Filecoinʼs Proof-of-Replication (PoRep) is expected to decrease as sector lifetimes increase.

It is reasonable to assume that miners enter the network by adding Committed Capacity sectors, that is, sectors that do not contain user data. Once miners agree storage deals with clients, they upgrade their sectors to Regular Sectors. Alternatively, if they find Verified Clients and agree a storage deal with them, they upgrade their sector accordingly. Depending on whether or not a sector includes a (verified) deal, the miner acquires the corresponding storage power in the network.

All sectors are expected to remain live until the end of their sector lifetime and early dropping of sectors will result in slashing. This is done to provide clients a certain level of guarantee on the reliability of their hosted data. Sector termination can comes with a corresponding _termination fee_.

As with every system it is expected that sectors will present faults. Although this might degrade the quality offered by the network, the reaction of the miner to the fault drives system decisions on whether or not the miner should be penalized. A miner can recover the faulty sector, let the system terminate the sector automatically after 42 days of faults, or proactively terminate the sector immediately in the case of unrecoverable data loss. In case of a faulty sector, a small penalty fee approximately equal to the block reward that the sector would win per day is applied. The fee is calculated per day of the sector being unavailable to the network, i.e. until the sector is recovered or terminated.

Miners can extend the lifetime of a sector at any time, though the sector will be expected to remain live until it has reached the end of the new sector lifetime. This can be done by submitting a `ExtendedSectorExpiration` message to the chain.

A sector can be in one of the following states.

| State          | Description                                                                                                                                           |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Precommitted` | Miner seals sector and submits `miner.PreCommitSector` or `miner.PreCommitSectorBatch`                                                                |
| `Committed`    | Miner generates a Seal proof and submits `miner.ProveCommitSector` or `miner.ProveCommitAggregate`                                                    |
| `Active`       | Miner generate valid PoSt proofs and timely submits `miner.SubmitWindowedPoSt`                                                                        |
| `Faulty`       | Miner fails to generate a proof (see Fault section)                                                                                                   |
| `Recovering`   | Miner declared a faulty sector via `miner.DeclareFaultRecovered`                                                                                      |
| `Terminated`   | Either sector is expired, or early terminated by a miner via `miner.TerminateSectors`, or was failed to be proven for 42 consecutive proving periods. |

### [**Sector Quality**](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector.sector-quality)

Given different sector contents, not all sectors have the same usefulness to the network. The notion of Sector Quality distinguishes between sectors with heuristics indicating the presence of valuable data. That distinction is used to allocate more subsidies to higher-quality sectors. To quantify the contribution of a sector to the consensus power of the network, some relevant parameters are described here.

* **Sector Spacetime:** This measurement is the sector size multiplied by its promised duration in byte-epochs.
* **Deal Weight:** This weight converts spacetime occupied by deals into consensus power. Deal weight of verified client deals in a sector is called Verified Deal Weight and will be greater than the regular deal weight.
* **Deal Quality Multiplier:** This factor is assigned to different deal types (committed capacity, regular deals, and verified client deals) to reward different content.
* **Sector Quality Multiplier:** Sector quality is assigned on Activation (the epoch when the miner starts proving theyʼre storing the file). The sector quality multiplier is computed as an average of deal quality multipliers (committed capacity, regular deals, and verified client deals), weighted by the amount of spacetime each type of deal occupies in the sector.

SectorQualityMultiplier=∑dealsDealWeight∗DealQualityMultiplierSectorSpaceTimeSectorQualityMultiplier=SectorSpaceTime∑deals​DealWeight∗DealQualityMultiplier​

* **Raw Byte Power:** This measurement is the size of a sector in bytes.
* **Quality-Adjusted Power:** This parameter measures the consensus power of stored data on the network, and is equal to Raw Byte Power multiplied by Sector Quality Multiplier.

The multipliers for committed capacity and regular deals are equal to make self dealing irrational in the current configuration of the protocol. In the future, it may make sense to pick different values, depending on other ways of preventing attacks becoming available.

The high quality multiplier and easy verification process for verified client deals facilitates decentralization of miner power. Unlike other proof-of-work-based protocols, like Bitcoin, central control of the network is not simply decided based on the resources that a new participant can bring. In Filecoin, accumulating control either requires significantly more resources or some amount of consent from verified clients, who must make deals with the centralized miners for them to increase their influence. Verified client mechanisms add a layer of social trust to a purely resource driven network. As long as the process is fair and transparent with accountability and bounded trust, abuse can be contained and minimized. A high sector quality multiplier is a very powerful leverage for clients to push storage providers to build features that will be useful to the network as a whole and increase the networkʼs long-term value. The verification process and DataCap allocation are meant to evolve over time as the community learns to automate and improve this process. An illustration of sectors with various contents and their respective sector qualities are shown in the following Figure.

<figure><img src="https://spec.filecoin.io/systems/filecoin_mining/sector/sector-quality/sector-quality.jpg" alt="Sector Quality" height="100%"><figcaption><p><a href="https://spec.filecoin.io/#figure-sector-quality">Figure: Sector Quality</a> </p></figcaption></figure>

**Sector Quality Adjusted Power** is a weighted average of the quality of its space and it is based on the size, duration and quality of its deals.

| Name                                | Description                                           |
| ----------------------------------- | ----------------------------------------------------- |
| QualityBaseMultiplier (QBM)         | Multiplier for power for storage without deals.       |
| DealWeightMultiplier (DWM)          | Multiplier for power for storage with deals.          |
| VerifiedDealWeightMultiplier (VDWM) | Multiplier for power for storage with verified deals. |

The formula for calculating Sector Quality Adjusted Power (or QAp, often referred to as power) makes use of the following factors:

* `dealSpaceTime`: sum of the `duration*size` of each deal
* `verifiedSpaceTime`: sum of the `duration*size` of each verified deal
* `baseSpaceTime` (spacetime without deals): `sectorSize*sectorDuration - dealSpaceTime - verifiedSpaceTime`

Based on these the average quality of a sector is:

avgQuality=baseSpaceTime∗QBM+dealSpaceTime∗DWM+verifiedSpaceTime∗VDWMsectorSize∗sectorDuration∗QBMavgQuality=sectorSize∗sectorDuration∗QBMbaseSpaceTime∗QBM+dealSpaceTime∗DWM+verifiedSpaceTime∗VDWM​

The _Sector Quality Adjusted Power_ is:

sectorQuality=avgQuality∗sizesectorQuality=avgQuality∗size

During `miner.PreCommitSector` and `miner.PreCommitSectorBatch`, the sector quality is calculated and stored in the sector information.

### [**Sector Sealing**](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector.sealing)

Before a Sector can be used, the Miner must _seal_ the Sector: encode the data in the Sector to prepare it for the proving process.

* **Unsealed Sector**: A Sector of raw data.
  * **UnsealedCID (CommD)**: The root hash of the Unsealed Sector’s merkle tree. Also called CommD, or “data commitment.”
* **Sealed Sector**: A Sector that has been encoded to prepare it for the proving process.
  * **SealedCID (CommR)**: The root hash of the Sealed Sector’s merkle tree. Also called CommR, or “replica commitment.”

Sealing a sector through Proof-of-Replication (PoRep) is a computation-intensive process that results in a unique encoding of the sector. Once data is sealed, storage miners: generate a proof; run a SNARK on the proof to compress it; and finally, submit the result of the compression to the blockchain as a certification of the storage commitment. Depending on the PoRep algorithm and protocol security parameters, cost profiles and performance characteristics vary and tradeoffs have to be made among sealing cost, security, onchain footprint, retrieval latency and so on. However, sectors can be sealed with commercial hardware and sealing cost is expected to decrease over time. The Filecoin Protocol will launch with Stacked Depth Robust (SDR) PoRep with a planned upgrade to Narrow Stacked Expander (NSE) PoRep with improvement in both cost and retrieval latency.

The Lotus-specific set of functions applied to the sealing of a sector can be found [here](https://github.com/filecoin-project/lotus/blob/master/storage/sealing.go).

#### [**Randomness**](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector.sealing.randomness)

Randomness is an important attribute that helps the network verify the integrity of Miners' stored data. Filecoin’s block creation process includes two types of randomness:

* [DRAND](https://spec.filecoin.io/#section-libraries.drand): Values pulled from a distributed random beacon
* VRF: The output of a _Verifiable Random Function_ (VRF), which takes the previous block’s VRF value and produces the current block’s VRF value.

Each block produced in Filecoin includes values pulled from these two sources of randomness.

When Miners submit proofs about their stored data, the proofs incorporate references to randomness added at specific epochs. Assuming these values were not able to be predicted ahead of time, this helps ensure that Miners generated proofs at a specific point in time.

There are two proof types. Each uses one of the two sources of randomness:

* Windowed PoSt: Uses Drand values
* Proof of Replication (PoRep): Uses VRF values

#### [**Drawing randomness for sector commitments**](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector.sealing.drawing-randomness-for-sector-commitments)

Tickets are used as input to calculation of the ReplicaID in order to tie Proofs-of-Replication to a given chain, thereby preventing long-range attacks (from another miner in the future trying to reuse SEALs).

The ticket has to be drawn from a finalized block in order to prevent the miner from potential losing storage (in case of a chain reorg) even though their storage is intact.

Verification should ensure that the ticket was drawn no farther back than necessary by the miner. We note that tickets can uniquely be associated with a given round in the protocol (lest a hash collision be found), but that the round number is explicited by the miner in `commitSector`.

We present precisely how ticket selection and verification should work. In the below, we use the following notation:

* `F`– Finality (number of rounds)
* `X`– round in which SEALing starts
* `Z`– round in which the SEAL appears (in a block)
* `Y`– round announced in the SEAL `commitSector` (should be X, but a miner could use any Y <= X), denoted by the ticket selection
* `T`– estimated time for SEAL, dependent on sector size
* `G = T + variance`– necessary flexibility to account for network delay and SEAL-time variance.

We expect Filecoin will be able to produce estimates for sector commitment time based on sector sizes, e.g.: `(estimate, variance) <--- SEALTime(sectors)` G and T will be selected using these.

**Picking a Ticket to Seal:** When starting to prepare a SEAL in round X, the miner should draw a ticket from X-F with which to compute the SEAL.

**Verifying a Seal’s ticket:** When verifying a SEAL in round Z, a verifier should ensure that the ticket used to generate the SEAL is found in the range of rounds `[Z-T-F-G, Z-T-F+G]`.

```
                               Prover
           ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
          │

          ▼
         X-F ◀───────F────────▶ X ◀──────────T─────────▶ Z
     -G   .  +G                 .                        .
  ───(┌───────┐)───────────────( )──────────────────────( )────────▶
      └───────┘                 '                        '        time
 [Z-T-F-G, Z-T-F+G]
          ▲

          └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
                              Verifier
```

Note that the prover here is submitting a message on chain (i.e. the SEAL). Using an older ticket than necessary to generate the SEAL is something the miner may do to gain more confidence about finality (since we are in a probabilistically final system). However it has a cost in terms of securing the chain in the face of long-range attacks (specifically, by mixing in chain randomness here, we ensure that an attacker going back a month in time to try and create their own chain would have to completely regenerate any and all sectors drawing randomness since to use for their fork’s power).

We break this down as follows:

* The miner should draw from `X-F`.
* The verifier wants to find what `X-F` should have been (to ensure the miner is not drawing from farther back) even though Y (i.e. the round of the ticket actually used) is an unverifiable value.
* Thus, the verifier will need to make an inference about what `X-F` is likely to have been based on:
  * (known) round in which the message is received (Z)
  * (known) finality value (F)
  * (approximate) SEAL time (T)
* Because T is an approximate value, and to account for network delay and variance in SEAL time across miners, the verifier allows for G offset from the assumed value of `X-F`: `Z-T-F`, hence verifying that the ticket is drawn from the range `[Z-T-F-G, Z-T-F+G]`.

In Practice, the Filecoin protocol will include a `MAX_SEAL_TIME` for each sector size and proof type.

### [**Sector Faults**](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector.sector-faults)

It is very important for storage providers to have a strong incentive to both report the failure to the chain and attempt recovery from the fault in order to uphold the storage guarantee for the networkʼs clients. Without this incentive, it is impossible to distinguish an honest minerʼs hardware failure from malicious behavior, which is necessary to treat miners fairly. The size of the fault fees depend on the severity of the failure and the rewards that the miner is expected to earn from the sector to make sure incentives are aligned. The two types of sector storage fault fees are:

* **Sector fault fee:** This fee is paid per sector per day while the sector is in a faulty state. This fee is not paid the first day the system detects the fault allowing a one day grace period for recovery without fee. The size of the sector fault fee is slightly more than the amount the sector is expected to earn per day in block rewards. If a sector remains faulty for more than 42 consecutive days, the sector will pay a termination fee and be _removed from the chain state_. As storage miner reliability increases above a reasonable threshold, the risk posed by these fees decreases rapidly.
* **Sector termination fee:** A sector can be terminated before its expiration through automatic faults or miner decisions. A termination fee is charged that is, in principle, equivalent to how much a sector has earned so far, up to a limit in order to avoid discouraging long sector lifetimes. In an active termination, the miner decides to stop mining and they pay a fee to leave. In a fault termination, a sector is in a faulty state for too long, and the chain terminates the deal, returns unpaid deal fees to the client and penalizes the miner. Termination fee is currently capped at 90 days worth of block reward that a sector will earn. Miners are responsible for deciding to comply with local regulations, and may sometimes need to accept a termination fee for complying with content laws. Many of the concepts and parameters above make use of the notion of “how much a sector would have earned in a day” in order to understand and align incentives for participants. This concept is robustly tracked and extrapolated on chain.

### [**Sector Recovery**](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector.sector-recovery)

Miners should try to recover faulty sectors in order to avoid paying the penalty, which is approximately equal to the block reward that the miner would receive from that sector. After fixing technical issues, the miner should call `RecoveryDeclaration` and produce the WindowPoSt challenge in order to regain the power from that sector.

Note that if a sector is in a faulty state for 42 consecutive days it will be terminated and the miner will receive a penalty. The miner can terminate the sector themselves by calling `TerminationDeclaration`, if they know that they cannot recover it, in which case they will receive a smaller penalty fee.

Both the `RecoveryDeclaration` and the `TerminationDeclaration` can be found in the [miner actor implementation](https://github.com/filecoin-project/specs-actors/blob/master/actors/builtin/miner/miner\_actor.go).

### [**Adding Storage**](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector.adding\_storage)

A Miner adds more storage in the form of Sectors. Adding more storage is a two-step process:

1. **PreCommitting a Sector**: A Miner publishes a Sector’s SealedCID, through `miner.PreCommitSector` of `miner.PreCommitSectorBatch`, and makes a deposit. The Sector is now registered to the Miner, and the Miner must ProveCommit the Sector or lose their deposit.
2. **ProveCommitting a Sector**: The Miner provides a Proof of Replication (PoRep) for the Sector through miner.ProveCommitSector or miner.ProveCommitAggregate. This proof must be submitted AFTER a delay (the InteractiveEpoch), and BEFORE PreCommit expiration.

This two-step process provides assurance that the Miner’s PoRep _actually proves_ that the Miner has replicated the Sector data and is generating proofs from it:

* ProveCommitments must happen AFTER the InteractiveEpoch (150 blocks after Sector PreCommit), as the randomness included at that epoch is used in the PoRep.
* ProveCommitments must happen BEFORE the PreCommit expiration, which is a boundary established to make sure Miners don’t have enough time to “fake” PoRep generation.

For each Sector successfully ProveCommitted, the Miner becomes responsible for continuously proving the existence of their Sectors' data. In return, the Miner is awarded storage power.

### [**Upgrading Sectors**](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector.adding\_storage)

Miners are granted storage power in exchange for the storage space they dedicate to Filecoin. Ideally, this storage space is used to store data on behalf of Clients, but there may not always be enough Clients to utilize all the space a Miner has to offer.

In order for a Miner to maximize storage power (and profit), they should take advantage of all available storage space immediately, _even before they find enough Clients to use this space_.

To facilitate this, there are _two types_ of Sectors that may be sealed and ProveCommitted:

* **Regular Sector**: A Sector that contains Client data
* **Committed Capacity (CC) Sector**: A Sector with no data (all zeroes)

Miners are free to coose which types of Sectors to store. CC sectors, in particular, allow Miners to immediately make use of existing disk space: earning storage power and a higher chance at producing a block. Miners can decide if they should upgrade their CC sectors to take client deals or continue proving CC sectors. Currently, CC sectors store randomness by default in client implementation, but this does not preclude miners from storing any type of useful data that increase their private utility in CC sectors (as long as it is legal). The protocol expects that new use-cases and diversity will emerge out of such behaviour.

To incentivize Miners to hoard storage space and dedicate it to Filecoin, CC Sectors have a unique capability: **they can be “upgraded” to Regular Sectors** (also called “replacing a CC Sector”).

Miners upgrade their ProveCommitted CC Sectors by PreCommitting a Regular Sector, and specifying that it should replace an existing CC Sector. Once the Regular Sector is successfully ProveCommitted, it will replace the existing CC Sector. If the newly ProveCommitted Regular sector contains a Verified Client deal, i.e., a deal with higher Sector Quality, then the miner’s storage power will increase accordingly.

Upgrading capacity currently involves resealing, that is, creating a unique representation of the new data included in the Sector through a computationally intensive process. Looking ahead, committed capacity upgrades should eventually be possible without a reseal. A succinct and publicly verifiable proof that the committed capacity has been correctly replaced with replicated data should achieve this goal. However, this mechanism must be fully specified to preserve the security and incentives of the network before it can be implemented and is, therefore, left as a future improvement.

## [Storage Miner](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining) <a href="#section-systems.filecoin_mining.storage_mining" id="section-systems.filecoin_mining.storage_mining"></a>

### [**Storage Mining Subsystem**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.storage-mining-subsystem)

The Filecoin Storage Mining Subsystem ensures a storage miner can effectively commit storage to the Filecoin protocol in order to both:

* Participate in the Filecoin [Storage Market](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market) by taking on client data and participating in storage deals.
* Participate in Filecoin [Storage Power Consensus](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.storage\_power\_consensus) by verifying and generating blocks to grow the Filecoin blockchain and earning block rewards and fees for doing so.

The above involves a number of steps to putting on and maintaining online storage, such as:

* Committing new storage (see [Sector](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector), [Sector Sealing](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector.sealing) and [PoRep](https://spec.filecoin.io/#section-algorithms.sdr))
* Continuously proving storage (see [WinningPoSt](https://spec.filecoin.io/#section-algorithms.expected\_consensus.winning-a-block) and Window PoSt)
* Declaring [storage faults](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector.sector-faults) and recovering from them.

### [**Filecoin Proofs**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.filecoin-proofs)

#### [**Proof of Replication**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.proof-of-replication)

A [Proof of Replication (PoRep)](https://spec.filecoin.io/#section-algorithms.sdr) is a proof that a Miner has correctly generated a unique _replica_ of some underlying data.

In practice, the underlying data is the _raw data_ contained in an Unsealed Sector, and a PoRep is a SNARK proof that the _sealing process_ was performed correctly to produce a Sealed Sector (See [Sealing a Sector](https://spec.filecoin.io/#section-\_index.Sealing-a-Sector)).

It is important to note that the replica should not only be unique to the miner, but also to the time when a miner has actually created the replica, i.e., sealed the sector. This means that if the same miner produces a sealed sector out of the same raw data twice, then this would count as a different replica.

When Miners commit to storing data, they must first produce a valid Proof of Replication.

#### [**Proof of Spacetime**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.proof-of-spacetime)

A [Proof of Spacetime (aka PoSt)](https://spec.filecoin.io/#section-algorithms.pos.post) is a long-term assurance of a Miner’s continuous storage of their Sectors' data. _This is not a single proof,_ but a collection of proofs the Miner has submitted over time. Periodically, a Miner must add to these proofs by submitting a **WindowPoSt**:

* Fundamentally, a WindowPoSt is a collection of merkle proofs over the underlying data in a Miner’s Sectors.
* WindowPoSts bundle proofs of various leaves across groups of Sectors (called **Partitions**).
* These proofs are submitted as a single SNARK.

The historical and ongoing submission of WindowPoSts creates assurance that the Miner has been storing, and continues to store the Sectors they agreed to store in the storage deal.

Once a Miner successfully adds and ProveCommits a Sector, the Sector is assigned to a Deadline: a specific window of time during which PoSts must be submitted. The day is broken up into 48 individual Deadlines of 30 minutes each, and ProveCommitted Sectors are assigned to one of these 48 Deadlines.

* PoSts may only be submitted for the currently-active Deadline. Deadlines are open for 30 minutes, starting from the Deadline’s “Open” epoch and ending at its “Close” epoch.
* PoSts must incorporate randomness pulled from a random beacon. This randomness becomes publicly available at the Deadline’s “Challenge” epoch, which is 20 epochs prior to its “Open” epoch.
* Deadlines also have a `FaultCutoff` epoch, 70 epochs prior to its “Open” epoch. After this epoch, Faults can no longer be declared for the Deadline’s Sectors.

### [**Miner Accounting**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.miner-accounting)

A Miner’s financial gain or loss is affected by the following three actions:

1. Miners deposit tokens to act as collateral for their PreCommitted and ProveCommitted Sectors
2. Miners earn tokens from block rewards, when they are elected to mine a new block and extend the blockchain.
3. Miners lose tokens if they fail to prove storage of a sector and are given penalties as a result.

#### [**Balance Requirements**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.balance-requirements)

A Miner’s token balance MUST cover ALL of the following:

* **PreCommit Deposits**: When a Miner PreCommits a Sector, they must supply a “precommit deposit” for the Sector, which acts as collateral. If the Sector is not ProveCommitted on time, this deposit is removed and burned.
* **Initial Pledge**: When a Miner ProveCommits a Sector, they must supply an “initial pledge” for the Sector, which acts as collateral. If the Sector is terminated, this deposit is removed and burned along with rewards earned by this sector up to a limit.
* **Locked Funds**: When a Miner receives tokens from block rewards, the tokens are locked and added to the Miner’s vesting table to be unlocked linearly over some future epochs.

#### [**Faults, Penalties and Fee Debt**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.faults-penalties-and-fee-debt)

**Faults**

A Sector’s PoSts must be submitted on time, or that Sector is marked “faulty.” There are three types of faults:

* **Declared Fault**: When the Miner explicitly declares a Sector “faulty” _before_ its Deadline’s FaultCutoff. Recall that `WindowPoSt` proofs are submitted per partition for a specific `ChallengeWindow`. A miner has to declare the sector as faulty before the `ChallengeWindow` for the particular partition opens. Until the sectors are recovered they will be masked from proofs in subsequent proving periods.
* **Detected Fault**: Partitions of sectors without PoSt proof verification records, which have not been declared faulty before the `FaultCutoff` epoch’s deadline are marked as detected faults.
* **Skipped Fault**: If a sector is currently in active or recovering state and has not been declared faulty before, but the miner’s PoSt submission does not include a proof for this sector, then this is a “skipped fault” sector (also referred to as “skipped undeclared fault”). In other words, when a miner submits PoSt proofs for a partition but does not include proofs for some sectors in the partition, then these sectors are in “skipped fault” state. This is in contrast to the “detected fault” state, where the miner does not submit a PoSt proof for any section in the partition at all. The skipped fault is helpful in case a sector becomes faulty after the `FaultCutoff` epoch.

Note that the “skipped fault” allows for sector-wise fault penalties, as compared to partition-wide faults and penalties, as is the case with “detected faults”.

**Deadlines**

A _deadline_ is a period of `WPoStChallengeWindow` epochs that divides a proving period. Sectors are assigned to a deadline on ProveCommit, by calling either `miner.ProveCommitSector`, or `miner.ProveCommitAggregate`, and will remain assigned to it throughout their lifetime. Recall that Sectors are also assigned to a partition.

A miner must submit a `miner.SubmitWindowedPoSt` for each deadline.

There are four relevant epochs associated to a deadline:

| Name          | Distance from `Open`      | Description                                                                                                                   |
| ------------- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `Open`        | `0`                       | Epoch from which a PoSt Proof for this deadline can be submitted.                                                             |
| `Close`       | `WPoStChallengeWindow`    | Epoch after which a PoSt Proof for this deadline will be rejected.                                                            |
| `FaultCutoff` | `-FaultDeclarationCutoff` | Epoch after which a `miner.DeclareFault` and `miner.DeclareFaultRecovered` for sectors in the upcoming deadline are rejected. |
| `Challenge`   | `-WPoStChallengeLookback` | Epoch at which the randomness for the challenges is available.                                                                |

**Fault Recovery**

Regardless of how a fault first becomes known (declared, skipped, detected), the sector stays faulty and is excluded from future proofs until the miner explicitly declares it recovered. The declaration of recovery restores the sector to the proving set at the start of the subsequent proving period. When a PoSt for a just-recovered sector is received, power for that sector is restored.

**Penalties**

A Miner may accrue penalties for many reasons:

* **PreCommit Expiry Penalty**: Occurs if a Miner fails to ProveCommit a PreCommitted Sector in time. This happens the first time that a miner declares that it proves a sector and falls into the PoRep consensus.
* **Undeclared Fault Penalty**: Occurs if a Miner fails to submit a PoSt for a Sector on time. Depending on whether the “Skipped Fault” option is implemented, this penalty applies to either a sector or a whole partition.
* **Declared Fault Penalty**: Occurs if a Miner fails to submit a PoSt for a Sector on time, but they declare the Sector faulty before the system finds out (in which case the fault falls in the “Undeclared Fault Penalty” above). **This penalty fee should be lower than the undeclared fault penalty**, in order to incentivize Miners to declare faults early.
* **Ongoing Fault Penalty**: Occurs every Proving Period a Miner fails to submit a PoSt for a Sector.
* **Termination Penalty**: Occurs if a Sector is terminated before its expiration.
* **Consensus Fault Penalty**: Occurs if a Miner commits a consensus fault and is reported.

When a Miner accrues penalties, the amount penalized is tracked as “Fee Debt.” If a Miner has Fee Debt, they are restricted from certain actions until the amount owed is paid off. Miners with Fee Debt may not:

* PreCommit new Sectors
* Declare faulty Sectors “recovered”
* Withdraw balance

Faults are implied to be “temporary” - that is, a Miner that temporarily loses internet connection may choose to declare some Sectors for their upcoming proving period as faulty, because the Miner knows they will eventually regain the ability to submit proofs for those Sectors. This declaration allows the Miner to still submit a valid proof for their Deadline (minus the faulty Sectors). This is very important for Miners, as missing a Deadline’s PoSt entirely incurs a high penalty.

### [**Storage Mining Cycle**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.mining\_cycle)

Block miners should constantly, on every epoch, be checking if they win the Secret Leader Election and in case they are elected, determine whether they can propose a block by running the Winning PoSt. Epochs are currently set to take 30 seconds, in order to account for Winning PoSt and network propagation around the world. The detailed steps for the above process can be found in the [Secret Leader Election](https://spec.filecoin.io/#section-algorithms.expected\_consensus.secret-leader-election) section.

Here we provide a detailed description of the mining cycle.

#### [**Active Miner Mining Cycle**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.mining\_cycle.active-miner-mining-cycle)

In order to mine blocks on the Filecoin blockchain a miner must be running [Block Validation](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.struct.block) at all times, keeping track of recent blocks received and the heaviest current chain (based on [Expected Consensus](https://spec.filecoin.io/#section-algorithms.expected\_consensus)).

With every new tipset, the miner can use their committed power to attempt to craft a new block.

For additional details around how consensus works in Filecoin, see [Expected Consensus](https://spec.filecoin.io/#section-algorithms.expected\_consensus). For the purposes of this section, there is a consensus protocol (Expected Consensus) that guarantees a fair process for determining what blocks have been generated in a round, whether a miner is eligible to mine a block, and other rules pertaining to the production of some artifacts required of valid blocks (e.g. Tickets, WinningPoSt).

[**Mining Cycle**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.mining\_cycle.mining-cycle)

After the chain has caught up to the current head using [ChainSync](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.chainsync), the mining process is as follows, (we go into more detail on epoch timing below):

* The node receives and transmits messages using the [Message Syncer](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.message\_pool.message\_syncer)
* At the same time the node receives blocks through [BlockSync](https://spec.filecoin.io/#section-algorithms.block\_sync).
  * Each block has an associated timestamp and epoch (quantized time window in which it was crafted)
  * Blocks are validated as they come in [block validation](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.struct.block)
* After an epoch’s “cutoff”, the miner should take all the valid blocks received for this epoch and assemble them into tipsets according to [Tipset validation rules](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.struct.tipset)
* The miner then attempts to mine atop the heaviest tipset (as calculated with [EC’s weight function](https://spec.filecoin.io/#section-algorithms.expected\_consensus.chain-selection)) using its smallest ticket to run leader election
  * The miner runs [Leader Election](https://spec.filecoin.io/#section-algorithms.expected\_consensus.secret-leader-election) using the most recent [random](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.storage\_power\_consensus.beacon-entries) output by a [drand](https://spec.filecoin.io/#section-libraries.drand) beacon.
    * if this yields a valid `ElectionProof`, the miner generates a new [ticket](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.storage\_power\_consensus.tickets) and winning PoSt for inclusion in the block.
    * the miner then assembles a new block (see “block creation” below) and waits until this epoch’s quantized timestamp to broadcast it

This process is repeated until either the [Leader Election](https://spec.filecoin.io/#section-algorithms.expected\_consensus.secret-leader-election) process yields a winning ticket (in EC) and the miner publishes a block or a new valid block comes in from the network.

At any height `H`, there are three possible situations:

* The miner is eligible to mine a block: they produce their block and propagate it. They then resume mining at the next height `H+1`.
* The miner is not eligible to mine a block but has received blocks: they form a Tipset with them and resume mining at the next height `H+1`.
* The miner is not eligible to mine a block and has received no blocks: prompted by their clock they run leader election again, incrementing the epoch number.

Anytime a miner receives new valid blocks, it should evaluate what is the heaviest Tipset it knows about and mine atop it.

[**Epoch Timing**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.mining\_cycle.epoch-timing)

<figure><img src="https://spec.filecoin.io/systems/filecoin_mining/storage_mining/mining_cycle/timing.png" alt="Mining Cycle Timing"><figcaption><p><a href="https://spec.filecoin.io/#figure-mining-cycle-timing">Figure: Mining Cycle Timing</a> </p></figcaption></figure>

The timing diagram above describes the sequence of block creation “mining”, propagation and reception.

This sequence of events applies only when the node is in the `CHAIN_FOLLOW` syncing mode. Nodes in other syncing modes do not mine blocks.

The upper row represents the conceptual consumption channel consisting of successive receiving periods `Rx` during which nodes validate incoming blocks. The lower row is the conceptual production channel made up of a period of mining `M` followed by a period of transmission `Tx` (which lasts long enough for blocks to propagate throughout the network). The lengths of the periods are not to scale.

The above diagram represents the important events within an epoch:

* **Epoch boundary**: change of current epoch. New blocks mined are mined in new epoch, and timestamped accordingly.
* **Epoch cutoff**: blocks from the prior epoch propagated on the network are no longer accepted. Miners can form a new tipset to mine on.

In an epoch, blocks are received and validated during `Rx` up to the prior epoch’s cutoff. At the cutoff, the miner computes the heaviest tipset from the blocks received during `Rx`, and uses it as the head to build on during the next mining period `M`. If mining is successful, the miner sets the block’s timestamp to the epoch boundary and waits until the boundary to release the block. While some blocks could be submitted a bit later, blocks are all transmitted during `Tx`, the transmission period.

The timing validation rules are as follows:

* Blocks whose timestamps are not exactly on the epoch boundary are rejected.
* Blocks received with a timestamp in the future are rejected.
* Blocks received after the cutoff are rejected.
  * Note that those blocks are not invalid, just not considered for the miner’s own tipset building. Tipsets received with such a block as a parent should be accepted.

In a fully synchronized network most of period `Rx` does not see any network traffic, only its beginning should. While there may be variance in operator mining time, most miners are expected to finish mining by the epoch boundary.

Let’s look at an example, both use a block-time of 30s, and a cutoff at 15s.

* `T = 0`: start of epoch n
* `T in [0, 15]`: miner A receives, validates and propagates incoming blocks. Valid blocks should have timestamp 0.
* `T = 15`: epoch cutoff for n-1, A assembles the heaviest tipset and starts mining atop it.
* `T = 25`: A successfully generates a block, sets its timestamp to 30, and waits until the epoch boundary (at 30) to release it.
* `T = 30`: start of epoch n + 1, A releases its block for epoch n.
* `T in [30, 45]`: A receives and validates incoming blocks, their timestamp is 30.
* `T = 45`: epoch cutoff for n, A forms tipsets and starts mining atop the heaviest.
* `T = 60`: start of epoch n + 2.
* `T in [60, 75]`: A receives and validates incoming blocks
* `T = 67`: A successfully generates a block, sets it timestamp to 60 and releases it.
* `T = 75`: epoch cutoff for n+1…

Above, in epoch n, A mines fast, in epoch n+1 A mines slow. So long as the miner’s block is between the epoch boundary and the cutoff, it will be accepted by other miners.

In practice miners should not be releasing blocks close to the epoch cutoff. Implementations may choose to locally randomize the exact time of the cutoff in order to prevent such behavior (while this means it may accept/reject blocks others do not, in practice this will not affect the miners submitting blocks on time).

#### [**Full Miner Lifecycle**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.mining\_cycle.full-miner-lifecycle)

[**Step 0: Registration and Market participation**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.mining\_cycle.step-0-registration-and-market-participation)

To initially become a miner, a miner first registers a new miner actor on-chain. This is done through the storage power actor’s `CreateStorageMiner` method. The call will then create a new miner actor instance and return its address.

The next step is to place one or more storage market asks on the market. This is done off-chain as part of storage market functions. A miner may create a single ask for their entire storage, or partition their storage up in some way with multiple asks (at potentially different prices).

After that, they need to make deals with clients and begin filling up sectors with data. For more information on making deals, see the [Storage Market](https://spec.filecoin.io/#section-systems.filecoin\_markets.storage\_market). The miner will need to put up storage deal collateral for the deals they have entered into.

When they have a full sector, they should seal it. This is done by invoking the [Sector Sealer](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_proving.sealer).

[Owner/Worker distinction](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.mining\_cycle.ownerworker-distinction)

The miner actor has two distinct ‘controller’ [addresses](https://spec.filecoin.io/#section-appendix.address). One is the worker, which is the address which will be responsible for doing all of the work, submitting proofs, committing new sectors, and all other day to day activities. The owner address is the address that created the miner, paid the collateral, and has block rewards paid out to it. The reason for the distinction is to allow different parties to fulfil the different roles. One example would be for the owner to be a multisig wallet, or a cold storage key, and the worker key to be a ‘hot wallet’ key.

[Changing Worker Addresses](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.mining\_cycle.changing-worker-addresses)

Note that any change to worker keys after registration must be appropriately delayed in relation to randomness lookback for SEALing data (see [this issue](https://github.com/filecoin-project/specs/issues/415)).

[**Step 1: Committing Sectors**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.mining\_cycle.step-1-committing-sectors)

When the miner has completed their first seal, they should post it on-chain using the [Storage Miner Actor’s](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.storage\_miner\_actor) `ProveCommitSector` function. The miner will need to put up [pledge collateral](https://spec.filecoin.io/#section-systems.filecoin\_mining.miner\_collaterals) in proportion to the amount of storage they commit on chain. The miner will now gain power for this particular sector upon successful `ProveCommitSector`.

You can read more about sectors [here](https://spec.filecoin.io/#section-systems.filecoin\_mining.sector) and how sector relates to power [here](https://spec.filecoin.io/#section-systems.filecoin\_blockchain.storage\_power\_consensus.on-power).

[**Step 2: Producing Blocks**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.mining\_cycle.step-2-producing-blocks)

Once the miner has power on the network, they are randomly chosen by the [“Secret Leader Election”](https://spec.filecoin.io/#section-algorithms.expected\_consensus.secret\_leader\_election) algorithm to mine and submit blocks proportionally to the power they hold, i.e., if a miner holds 3% of the overall network power they will be chosen in 3% of the cases. The winning miner is chosen by the system and the miner can prove that they were chosen by submitting an Election Proof.

When a miner is chosen to produce a block, they must submit a `WinningPoSt` proof. This process is as follows: an elected miner gets the randomness value through the DRAND randomness generator based on the current epoch and uses it to generate WinningPoSt.

WinningPoSt uses the randomness to select a sector for which the miner must generate a proof. If the miner is not able to generate this proof within some predefined amount of time, then they will not be able to create a block.

[**Faults**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.mining\_cycle.faults)

If a miner detects [Storage Faults](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.faults.storage-faults) among their sectors (any sort of storage failure that would prevent them from crafting a PoSt), they should declare these faults as discussed earlier.

The miner will be unable to craft valid PoSt proofs over faulty sectors, thereby reducing their chances of being able to create a valid block (i.e., adding a Winning PoSt). By declaring a fault, the miner will no longer be challenged on that sector, and will lose power accordingly.

[**Step 3: Deal/Sector Expiration**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.mining\_cycle.step-3-dealsector-expiration)

In order to stop mining, a miner must complete all of its storage deals. Once all deals in a sector have expired, the sector itself will expire thereby enabling the miner to remove the associated collateral from their account.

### [**Storage Miner Actor**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_mining.storage\_miner\_actor)

[Example: Storage Miner Actor State ](https://spec.filecoin.io/#example-storage-miner-actor-state)Balance of Miner Actor should be greater than or equal to the sum of PreCommitDeposits and LockedFunds. It is possible for balance to fall below the sum of PCD, LF and InitialPledgeRequirements, and this is a bad state (IP Debt) that limits a miner actor’s behavior (i.e. no balance withdrawals) Excess balance as computed by st.GetAvailableBalance will be withdrawable or usable for pre-commit deposit or pledge lock-up.

```go
type State struct {
	// Information not related to sectors.
	Info cid.Cid

	PreCommitDeposits abi.TokenAmount // Total funds locked as PreCommitDeposits
	LockedFunds       abi.TokenAmount // Total rewards and added funds locked in vesting table

	VestingFunds cid.Cid // VestingFunds (Vesting Funds schedule for the miner).

	FeeDebt abi.TokenAmount // Absolute value of debt this miner owes from unpaid fees

	InitialPledge abi.TokenAmount // Sum of initial pledge requirements of all active sectors

	// Sectors that have been pre-committed but not yet proven.
	PreCommittedSectors cid.Cid // Map, HAMT[SectorNumber]SectorPreCommitOnChainInfo

	// PreCommittedSectorsCleanUp maintains the state required to cleanup expired PreCommittedSectors.
	PreCommittedSectorsCleanUp cid.Cid // BitFieldQueue (AMT[Epoch]*BitField)

	// Allocated sector IDs. Sector IDs can never be reused once allocated.
	AllocatedSectors cid.Cid // BitField

	// Information for all proven and not-yet-garbage-collected sectors.
	//
	// Sectors are removed from this AMT when the partition to which the
	// sector belongs is compacted.
	Sectors cid.Cid // Array, AMT[SectorNumber]SectorOnChainInfo (sparse)

	// DEPRECATED. This field will change names and no longer be updated every proving period in a future upgrade
	// The first epoch in this miner's current proving period. This is the first epoch in which a PoSt for a
	// partition at the miner's first deadline may arrive. Alternatively, it is after the last epoch at which
	// a PoSt for the previous window is valid.
	// Always greater than zero, this may be greater than the current epoch for genesis miners in the first
	// WPoStProvingPeriod epochs of the chain; the epochs before the first proving period starts are exempt from Window
	// PoSt requirements.
	// Updated at the end of every period by a cron callback.
	ProvingPeriodStart abi.ChainEpoch

	// DEPRECATED. This field will be removed from state in a future upgrade.
	// Index of the deadline within the proving period beginning at ProvingPeriodStart that has not yet been
	// finalized.
	// Updated at the end of each deadline window by a cron callback.
	CurrentDeadline uint64

	// The sector numbers due for PoSt at each deadline in the current proving period, frozen at period start.
	// New sectors are added and expired ones removed at proving period boundary.
	// Faults are not subtracted from this in state, but on the fly.
	Deadlines cid.Cid

	// Deadlines with outstanding fees for early sector termination.
	EarlyTerminations bitfield.BitField

	// True when miner cron is active, false otherwise
	DeadlineCronActive bool
}
```

[Example: Storage Miner Actor ](https://spec.filecoin.io/#example-storage-miner-actor)

```go
package miner

import (
	"bytes"
	"encoding/binary"
	"fmt"
	"math"

	addr "github.com/filecoin-project/go-address"
	"github.com/filecoin-project/go-bitfield"
	"github.com/filecoin-project/go-state-types/abi"
	"github.com/filecoin-project/go-state-types/big"
	"github.com/filecoin-project/go-state-types/cbor"
	"github.com/filecoin-project/go-state-types/crypto"
	"github.com/filecoin-project/go-state-types/dline"
	"github.com/filecoin-project/go-state-types/exitcode"
	rtt "github.com/filecoin-project/go-state-types/rt"
	miner0 "github.com/filecoin-project/specs-actors/actors/builtin/miner"
	miner2 "github.com/filecoin-project/specs-actors/v2/actors/builtin/miner"
	miner3 "github.com/filecoin-project/specs-actors/v3/actors/builtin/miner"
	cid "github.com/ipfs/go-cid"
	cbg "github.com/whyrusleeping/cbor-gen"
	"golang.org/x/xerrors"

	"github.com/filecoin-project/specs-actors/v7/actors/builtin"
	"github.com/filecoin-project/specs-actors/v7/actors/builtin/market"
	"github.com/filecoin-project/specs-actors/v7/actors/builtin/power"
	"github.com/filecoin-project/specs-actors/v7/actors/builtin/reward"
	"github.com/filecoin-project/specs-actors/v7/actors/runtime"
	"github.com/filecoin-project/specs-actors/v7/actors/runtime/proof"
	. "github.com/filecoin-project/specs-actors/v7/actors/util"
	"github.com/filecoin-project/specs-actors/v7/actors/util/adt"
	"github.com/filecoin-project/specs-actors/v7/actors/util/smoothing"
)

type Runtime = runtime.Runtime

const (
	// The first 1000 actor-specific codes are left open for user error, i.e. things that might
	// actually happen without programming error in the actor code.
	//ErrToBeDetermined = exitcode.FirstActorSpecificExitCode + iota

	// The following errors are particular cases of illegal state.
	// They're not expected to ever happen, but if they do, distinguished codes can help us
	// diagnose the problem.
	ErrBalanceInvariantBroken = 1000
)

type Actor struct{}

func (a Actor) Exports() []interface{} {
	return []interface{}{
		builtin.MethodConstructor: a.Constructor,
		2:                         a.ControlAddresses,
		3:                         a.ChangeWorkerAddress,
		4:                         a.ChangePeerID,
		5:                         a.SubmitWindowedPoSt,
		6:                         a.PreCommitSector,
		7:                         a.ProveCommitSector,
		8:                         a.ExtendSectorExpiration,
		9:                         a.TerminateSectors,
		10:                        a.DeclareFaults,
		11:                        a.DeclareFaultsRecovered,
		12:                        a.OnDeferredCronEvent,
		13:                        a.CheckSectorProven,
		14:                        a.ApplyRewards,
		15:                        a.ReportConsensusFault,
		16:                        a.WithdrawBalance,
		17:                        a.ConfirmSectorProofsValid,
		18:                        a.ChangeMultiaddrs,
		19:                        a.CompactPartitions,
		20:                        a.CompactSectorNumbers,
		21:                        a.ConfirmUpdateWorkerKey,
		22:                        a.RepayDebt,
		23:                        a.ChangeOwnerAddress,
		24:                        a.DisputeWindowedPoSt,
		25:                        a.PreCommitSectorBatch,
		26:                        a.ProveCommitAggregate,
		27:                        a.ProveReplicaUpdates,
	}
}

func (a Actor) Code() cid.Cid {
	return builtin.StorageMinerActorCodeID
}

func (a Actor) State() cbor.Er {
	return new(State)
}

var _ runtime.VMActor = Actor{}

/////////////////
// Constructor //
/////////////////

// Storage miner actors are created exclusively by the storage power actor. In order to break a circular dependency
// between the two, the construction parameters are defined in the power actor.
type ConstructorParams = power.MinerConstructorParams

func (a Actor) Constructor(rt Runtime, params *ConstructorParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.InitActorAddr)

	checkControlAddresses(rt, params.ControlAddrs)
	checkPeerInfo(rt, params.PeerId, params.Multiaddrs)

	if !CanWindowPoStProof(params.WindowPoStProofType) {
		rt.Abortf(exitcode.ErrIllegalArgument, "proof type %d not allowed for new miner actors", params.WindowPoStProofType)
	}

	owner := resolveControlAddress(rt, params.OwnerAddr)
	worker := resolveWorkerAddress(rt, params.WorkerAddr)
	controlAddrs := make([]addr.Address, 0, len(params.ControlAddrs))
	for _, ca := range params.ControlAddrs {
		resolved := resolveControlAddress(rt, ca)
		controlAddrs = append(controlAddrs, resolved)
	}

	currEpoch := rt.CurrEpoch()
	offset, err := assignProvingPeriodOffset(rt.Receiver(), currEpoch, rt.HashBlake2b)
	builtin.RequireNoErr(rt, err, exitcode.ErrSerialization, "failed to assign proving period offset")
	periodStart := currentProvingPeriodStart(currEpoch, offset)
	builtin.RequireState(rt, periodStart <= currEpoch, "computed proving period start %d after current epoch %d", periodStart, currEpoch)
	deadlineIndex := currentDeadlineIndex(currEpoch, periodStart)
	builtin.RequireState(rt, deadlineIndex < WPoStPeriodDeadlines, "computed proving deadline index %d invalid", deadlineIndex)

	info, err := ConstructMinerInfo(owner, worker, controlAddrs, params.PeerId, params.Multiaddrs, params.WindowPoStProofType)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to construct initial miner info")
	infoCid := rt.StorePut(info)

	store := adt.AsStore(rt)
	state, err := ConstructState(store, infoCid, periodStart, deadlineIndex)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to construct state")
	rt.StateCreate(state)

	return nil
}

/////////////
// Control //
/////////////

// type GetControlAddressesReturn struct {
// 	Owner        addr.Address
// 	Worker       addr.Address
// 	ControlAddrs []addr.Address
// }
type GetControlAddressesReturn = miner2.GetControlAddressesReturn

func (a Actor) ControlAddresses(rt Runtime, _ *abi.EmptyValue) *GetControlAddressesReturn {
	rt.ValidateImmediateCallerAcceptAny()
	var st State
	rt.StateReadonly(&st)
	info := getMinerInfo(rt, &st)
	return &GetControlAddressesReturn{
		Owner:        info.Owner,
		Worker:       info.Worker,
		ControlAddrs: info.ControlAddresses,
	}
}

//type ChangeWorkerAddressParams struct {
//	NewWorker       addr.Address
//	NewControlAddrs []addr.Address
//}
type ChangeWorkerAddressParams = miner0.ChangeWorkerAddressParams

// ChangeWorkerAddress will ALWAYS overwrite the existing control addresses with the control addresses passed in the params.
// If a nil addresses slice is passed, the control addresses will be cleared.
// A worker change will be scheduled if the worker passed in the params is different from the existing worker.
func (a Actor) ChangeWorkerAddress(rt Runtime, params *ChangeWorkerAddressParams) *abi.EmptyValue {
	checkControlAddresses(rt, params.NewControlAddrs)

	newWorker := resolveWorkerAddress(rt, params.NewWorker)

	var controlAddrs []addr.Address
	for _, ca := range params.NewControlAddrs {
		resolved := resolveControlAddress(rt, ca)
		controlAddrs = append(controlAddrs, resolved)
	}

	var st State
	rt.StateTransaction(&st, func() {
		info := getMinerInfo(rt, &st)

		// Only the Owner is allowed to change the newWorker and control addresses.
		rt.ValidateImmediateCallerIs(info.Owner)

		// save the new control addresses
		info.ControlAddresses = controlAddrs

		// save newWorker addr key change request
		if newWorker != info.Worker && info.PendingWorkerKey == nil {
			info.PendingWorkerKey = &WorkerKeyChange{
				NewWorker:   newWorker,
				EffectiveAt: rt.CurrEpoch() + WorkerKeyChangeDelay,
			}
		}

		err := st.SaveInfo(adt.AsStore(rt), info)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "could not save miner info")
	})

	return nil
}

// Triggers a worker address change if a change has been requested and its effective epoch has arrived.
func (a Actor) ConfirmUpdateWorkerKey(rt Runtime, params *abi.EmptyValue) *abi.EmptyValue {
	var st State
	rt.StateTransaction(&st, func() {
		info := getMinerInfo(rt, &st)

		// Only the Owner is allowed to change the newWorker.
		rt.ValidateImmediateCallerIs(info.Owner)

		processPendingWorker(info, rt, &st)
	})

	return nil
}

// Proposes or confirms a change of owner address.
// If invoked by the current owner, proposes a new owner address for confirmation. If the proposed address is the
// current owner address, revokes any existing proposal.
// If invoked by the previously proposed address, with the same proposal, changes the current owner address to be
// that proposed address.
func (a Actor) ChangeOwnerAddress(rt Runtime, newAddress *addr.Address) *abi.EmptyValue {
	if newAddress.Empty() {
		rt.Abortf(exitcode.ErrIllegalArgument, "empty address")
	}
	if newAddress.Protocol() != addr.ID {
		rt.Abortf(exitcode.ErrIllegalArgument, "owner address must be an ID address")
	}
	var st State
	rt.StateTransaction(&st, func() {
		info := getMinerInfo(rt, &st)
		if rt.Caller() == info.Owner || info.PendingOwnerAddress == nil {
			// Propose new address.
			rt.ValidateImmediateCallerIs(info.Owner)
			info.PendingOwnerAddress = newAddress
		} else { // info.PendingOwnerAddress != nil
			// Confirm the proposal.
			// This validates that the operator can in fact use the proposed new address to sign messages.
			rt.ValidateImmediateCallerIs(*info.PendingOwnerAddress)
			if *newAddress != *info.PendingOwnerAddress {
				rt.Abortf(exitcode.ErrIllegalArgument, "expected confirmation of %v, got %v",
					info.PendingOwnerAddress, newAddress)
			}
			info.Owner = *info.PendingOwnerAddress
		}

		// Clear any resulting no-op change.
		if info.PendingOwnerAddress != nil && *info.PendingOwnerAddress == info.Owner {
			info.PendingOwnerAddress = nil
		}

		err := st.SaveInfo(adt.AsStore(rt), info)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save miner info")
	})
	return nil
}

//type ChangePeerIDParams struct {
//	NewID abi.PeerID
//}
type ChangePeerIDParams = miner0.ChangePeerIDParams

func (a Actor) ChangePeerID(rt Runtime, params *ChangePeerIDParams) *abi.EmptyValue {
	checkPeerInfo(rt, params.NewID, nil)

	var st State
	rt.StateTransaction(&st, func() {
		info := getMinerInfo(rt, &st)

		rt.ValidateImmediateCallerIs(append(info.ControlAddresses, info.Owner, info.Worker)...)

		info.PeerId = params.NewID
		err := st.SaveInfo(adt.AsStore(rt), info)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "could not save miner info")
	})
	return nil
}

//type ChangeMultiaddrsParams struct {
//	NewMultiaddrs []abi.Multiaddrs
//}
type ChangeMultiaddrsParams = miner0.ChangeMultiaddrsParams

func (a Actor) ChangeMultiaddrs(rt Runtime, params *ChangeMultiaddrsParams) *abi.EmptyValue {
	checkPeerInfo(rt, nil, params.NewMultiaddrs)

	var st State
	rt.StateTransaction(&st, func() {
		info := getMinerInfo(rt, &st)

		rt.ValidateImmediateCallerIs(append(info.ControlAddresses, info.Owner, info.Worker)...)

		info.Multiaddrs = params.NewMultiaddrs
		err := st.SaveInfo(adt.AsStore(rt), info)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "could not save miner info")
	})
	return nil
}

//////////////////
// WindowedPoSt //
//////////////////

//type PoStPartition struct {
//	// Partitions are numbered per-deadline, from zero.
//	Index uint64
//	// Sectors skipped while proving that weren't already declared faulty
//	Skipped bitfield.BitField
//}
type PoStPartition = miner0.PoStPartition

// Information submitted by a miner to provide a Window PoSt.
//type SubmitWindowedPoStParams struct {
//	// The deadline index which the submission targets.
//	Deadline uint64
//	// The partitions being proven.
//	Partitions []PoStPartition
//	// Array of proofs, one per distinct registered proof type present in the sectors being proven.
//	// In the usual case of a single proof type, this array will always have a single element (independent of number of partitions).
//	Proofs []proof.PoStProof
//	// The epoch at which these proofs is being committed to a particular chain.
//	// NOTE: This field should be removed in the future. See
//	// https://github.com/filecoin-project/specs-actors/issues/1094
//	ChainCommitEpoch abi.ChainEpoch
//	// The ticket randomness on the chain at the chain commit epoch.
//	ChainCommitRand abi.Randomness
//}
type SubmitWindowedPoStParams = miner0.SubmitWindowedPoStParams

// Invoked by miner's worker address to submit their fallback post
func (a Actor) SubmitWindowedPoSt(rt Runtime, params *SubmitWindowedPoStParams) *abi.EmptyValue {
	currEpoch := rt.CurrEpoch()
	store := adt.AsStore(rt)
	var st State

	// Verify that the miner has passed exactly 1 proof.
	if len(params.Proofs) != 1 {
		rt.Abortf(exitcode.ErrIllegalArgument, "expected exactly one proof, got %d", len(params.Proofs))
	}

	if !CanWindowPoStProof(params.Proofs[0].PoStProof) {
		rt.Abortf(exitcode.ErrIllegalArgument, "proof type %d not allowed", params.Proofs[0].PoStProof)
	}

	if params.Deadline >= WPoStPeriodDeadlines {
		rt.Abortf(exitcode.ErrIllegalArgument, "invalid deadline %d of %d", params.Deadline, WPoStPeriodDeadlines)
	}
	// Technically, ChainCommitRand should be _exactly_ 32 bytes. However:
	// 1. It's convenient to allow smaller slices when testing.
	// 2. Nothing bad will happen if the caller provides too little randomness.
	if len(params.ChainCommitRand) > abi.RandomnessLength {
		rt.Abortf(exitcode.ErrIllegalArgument, "expected at most %d bytes of randomness, got %d", abi.RandomnessLength, len(params.ChainCommitRand))
	}

	var postResult *PoStResult
	var info *MinerInfo
	rt.StateTransaction(&st, func() {
		info = getMinerInfo(rt, &st)
		maxProofSize, err := info.WindowPoStProofType.ProofSize()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to determine max window post proof size")

		rt.ValidateImmediateCallerIs(append(info.ControlAddresses, info.Owner, info.Worker)...)

		// Make sure the miner is using the correct proof type.
		if params.Proofs[0].PoStProof != info.WindowPoStProofType {
			rt.Abortf(exitcode.ErrIllegalArgument, "expected proof of type %d, got proof of type %d", info.WindowPoStProofType, params.Proofs[0])
		}

		// Make sure the proof size doesn't exceed the max. We could probably check for an exact match, but this is safer.
		if maxSize := maxProofSize * uint64(len(params.Partitions)); uint64(len(params.Proofs[0].ProofBytes)) > maxSize {
			rt.Abortf(exitcode.ErrIllegalArgument, "expected proof to be smaller than %d bytes", maxSize)
		}

		// Validate that the miner didn't try to prove too many partitions at once.
		submissionPartitionLimit := loadPartitionsSectorsMax(info.WindowPoStPartitionSectors)
		if uint64(len(params.Partitions)) > submissionPartitionLimit {
			rt.Abortf(exitcode.ErrIllegalArgument, "too many partitions %d, limit %d", len(params.Partitions), submissionPartitionLimit)
		}

		currDeadline := st.DeadlineInfo(currEpoch)
		// Check that the miner state indicates that the current proving deadline has started.
		// This should only fail if the cron actor wasn't invoked, and matters only in case that it hasn't been
		// invoked for a whole proving period, and hence the missed PoSt submissions from the prior occurrence
		// of this deadline haven't been processed yet.
		if !currDeadline.IsOpen() {
			rt.Abortf(exitcode.ErrIllegalState, "proving period %d not yet open at %d", currDeadline.PeriodStart, currEpoch)
		}

		// The miner may only submit a proof for the current deadline.
		if params.Deadline != currDeadline.Index {
			rt.Abortf(exitcode.ErrIllegalArgument, "invalid deadline %d at epoch %d, expected %d",
				params.Deadline, currEpoch, currDeadline.Index)
		}

		// Verify that the PoSt was committed to the chain at most WPoStChallengeLookback+WPoStChallengeWindow in the past.
		if params.ChainCommitEpoch < currDeadline.Challenge {
			rt.Abortf(exitcode.ErrIllegalArgument, "expected chain commit epoch %d to be after %d", params.ChainCommitEpoch, currDeadline.Challenge)
		}
		if params.ChainCommitEpoch >= currEpoch {
			rt.Abortf(exitcode.ErrIllegalArgument, "chain commit epoch %d must be less than the current epoch %d", params.ChainCommitEpoch, currEpoch)
		}
		// Verify the chain commit randomness.
		commRand := rt.GetRandomnessFromTickets(crypto.DomainSeparationTag_PoStChainCommit, params.ChainCommitEpoch, nil)
		if !bytes.Equal(commRand, params.ChainCommitRand) {
			rt.Abortf(exitcode.ErrIllegalArgument, "post commit randomness mismatched")
		}

		sectors, err := LoadSectors(store, st.Sectors)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load sectors")

		deadlines, err := st.LoadDeadlines(adt.AsStore(rt))
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadlines")

		deadline, err := deadlines.LoadDeadline(store, params.Deadline)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadline %d", params.Deadline)

		// Record proven sectors/partitions, returning updates to power and the final set of sectors
		// proven/skipped.
		//
		// NOTE: This function does not actually check the proofs but does assume that they're correct. Instead,
		// it snapshots the deadline's state and the submitted proofs at the end of the challenge window and
		// allows third-parties to dispute these proofs.
		//
		// While we could perform _all_ operations at the end of challenge window, we do as we can here to avoid
		// overloading cron.
		faultExpiration := currDeadline.Last() + FaultMaxAge
		postResult, err = deadline.RecordProvenSectors(store, sectors, info.SectorSize, QuantSpecForDeadline(currDeadline), faultExpiration, params.Partitions)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to process post submission for deadline %d", params.Deadline)

		// Make sure we actually proved something.

		provenSectors, err := bitfield.SubtractBitField(postResult.Sectors, postResult.IgnoredSectors)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to determine proven sectors for deadline %d", params.Deadline)

		noSectors, err := provenSectors.IsEmpty()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to determine if any sectors were proven", params.Deadline)
		if noSectors {
			// Abort verification if all sectors are (now) faults. There's nothing to prove.
			// It's not rational for a miner to submit a Window PoSt marking *all* non-faulty sectors as skipped,
			// since that will just cause them to pay a penalty at deadline end that would otherwise be zero
			// if they had *not* declared them.
			rt.Abortf(exitcode.ErrIllegalArgument, "cannot prove partitions with no active sectors")
		}

		// If we're not recovering power, record the proof for optimistic verification.
		if postResult.RecoveredPower.IsZero() {
			err = deadline.RecordPoStProofs(store, postResult.Partitions, params.Proofs)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to record proof for optimistic verification", params.Deadline)
		} else {
			// otherwise, check the proof
			sectorInfos, err := sectors.LoadForProof(postResult.Sectors, postResult.IgnoredSectors)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load sectors for post verification")

			err = verifyWindowedPost(rt, currDeadline.Challenge, sectorInfos, params.Proofs)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "window post failed")
		}

		err = deadlines.UpdateDeadline(store, params.Deadline, deadline)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to update deadline %d", params.Deadline)

		err = st.SaveDeadlines(store, deadlines)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save deadlines")
	})

	// Restore power for recovered sectors. Remove power for new faults.
	// NOTE: It would be permissible to delay the power loss until the deadline closes, but that would require
	// additional accounting state.
	// https://github.com/filecoin-project/specs-actors/issues/414
	requestUpdatePower(rt, postResult.PowerDelta)

	rt.StateReadonly(&st)
	err := st.CheckBalanceInvariants(rt.CurrentBalance())
	builtin.RequireNoErr(rt, err, ErrBalanceInvariantBroken, "balance invariants broken")

	return nil
}

// type DisputeWindowedPoStParams struct {
// 		Deadline  uint64
// 		PoStIndex uint64 // only one is allowed at a time to avoid loading too many sector infos.
// }
type DisputeWindowedPoStParams = miner3.DisputeWindowedPoStParams

func (a Actor) DisputeWindowedPoSt(rt Runtime, params *DisputeWindowedPoStParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerType(builtin.CallerTypesSignable...)
	reporter := rt.Caller()

	if params.Deadline >= WPoStPeriodDeadlines {
		rt.Abortf(exitcode.ErrIllegalArgument, "invalid deadline %d of %d", params.Deadline, WPoStPeriodDeadlines)
	}

	currEpoch := rt.CurrEpoch()

	// Note: these are going to be slightly inaccurate as time
	// will have moved on from when the post was actually
	// submitted.
	//
	// However, these are estimates _anyways_.
	epochReward := requestCurrentEpochBlockReward(rt)
	pwrTotal := requestCurrentTotalPower(rt)

	toBurn := abi.NewTokenAmount(0)
	toReward := abi.NewTokenAmount(0)
	pledgeDelta := abi.NewTokenAmount(0)
	powerDelta := NewPowerPairZero()
	var st State
	rt.StateTransaction(&st, func() {
		dlInfo := st.DeadlineInfo(currEpoch)
		if !deadlineAvailableForOptimisticPoStDispute(dlInfo.PeriodStart, params.Deadline, currEpoch) {
			rt.Abortf(exitcode.ErrForbidden, "can only dispute window posts during the dispute window (%d epochs after the challenge window closes)", WPoStDisputeWindow)
		}

		info := getMinerInfo(rt, &st)
		penalisedPower := NewPowerPairZero()
		store := adt.AsStore(rt)

		// Check proof
		{
			// Find the proving period start for the deadline in question.
			ppStart := dlInfo.PeriodStart
			if dlInfo.Index < params.Deadline {
				ppStart -= WPoStProvingPeriod
			}
			targetDeadline := NewDeadlineInfo(ppStart, params.Deadline, currEpoch)

			// Load the target deadline.
			deadlinesCurrent, err := st.LoadDeadlines(store)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadlines")

			dlCurrent, err := deadlinesCurrent.LoadDeadline(store, params.Deadline)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadline")

			// Take the post from the snapshot for dispute.
			// This operation REMOVES the PoSt from the snapshot so
			// it can't be disputed again. If this method fails,
			// this operation must be rolled back.
			partitions, proofs, err := dlCurrent.TakePoStProofs(store, params.PoStIndex)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load proof for dispute")

			// Load the partition info we need for the dispute.
			disputeInfo, err := dlCurrent.LoadPartitionsForDispute(store, partitions)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load partition info for dispute")
			// This includes power that is no longer active (e.g., due to sector terminations).
			// It must only be used for penalty calculations, not power adjustments.
			penalisedPower = disputeInfo.DisputedPower

			// Load sectors for the dispute.
			sectors, err := LoadSectors(store, st.Sectors)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load sectors array")

			sectorInfos, err := sectors.LoadForProof(disputeInfo.AllSectorNos, disputeInfo.IgnoredSectorNos)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load sectors to dispute window post")

			// Check proof, we fail if validation succeeds.
			err = verifyWindowedPost(rt, targetDeadline.Challenge, sectorInfos, proofs)
			if err == nil {
				rt.Abortf(exitcode.ErrIllegalArgument, "failed to dispute valid post")
				return
			}
			rt.Log(rtt.INFO, "successfully disputed: %s", err)

			// Ok, now we record faults. This always works because
			// we don't allow compaction/moving sectors during the
			// challenge window.
			//
			// However, some of these sectors may have been
			// terminated. That's fine, we'll skip them.
			faultExpirationEpoch := targetDeadline.Last() + FaultMaxAge
			powerDelta, err = dlCurrent.RecordFaults(store, sectors, info.SectorSize, QuantSpecForDeadline(targetDeadline), faultExpirationEpoch, disputeInfo.DisputedSectors)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to declare faults")

			err = deadlinesCurrent.UpdateDeadline(store, params.Deadline, dlCurrent)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to update deadline %d", params.Deadline)
			err = st.SaveDeadlines(store, deadlinesCurrent)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save deadlines")
		}

		// Penalties.
		{
			// Calculate the base penalty.
			penaltyBase := PledgePenaltyForInvalidWindowPoSt(
				epochReward.ThisEpochRewardSmoothed,
				pwrTotal.QualityAdjPowerSmoothed,
				penalisedPower.QA,
			)

			// Calculate the target reward.
			rewardTarget := RewardForDisputedWindowPoSt(info.WindowPoStProofType, penalisedPower)

			// Compute the target penalty by adding the
			// base penalty to the target reward. We don't
			// take reward out of the penalty as the miner
			// could end up receiving a substantial
			// portion of their fee back as a reward.
			penaltyTarget := big.Add(penaltyBase, rewardTarget)

			err := st.ApplyPenalty(penaltyTarget)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to apply penalty")
			penaltyFromVesting, penaltyFromBalance, err := st.RepayPartialDebtInPriorityOrder(store, currEpoch, rt.CurrentBalance())
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to pay debt")
			toBurn = big.Add(penaltyFromVesting, penaltyFromBalance)

			// Now, move as much of the target reward as
			// we can from the burn to the reward.
			toReward = big.Min(toBurn, rewardTarget)
			toBurn = big.Sub(toBurn, toReward)

			pledgeDelta = penaltyFromVesting.Neg()
		}
	})

	requestUpdatePower(rt, powerDelta)

	if !toReward.IsZero() {
		// Try to send the reward to the reporter.
		code := rt.Send(reporter, builtin.MethodSend, nil, toReward, &builtin.Discard{})

		// If we fail, log and burn the reward to make sure the balances remain correct.
		if !code.IsSuccess() {
			rt.Log(rtt.ERROR, "failed to send reward")
			toBurn = big.Add(toBurn, toReward)
		}
	}
	burnFunds(rt, toBurn, BurnMethodDisputeWindowedPoSt)
	notifyPledgeChanged(rt, pledgeDelta)
	rt.StateReadonly(&st)

	err := st.CheckBalanceInvariants(rt.CurrentBalance())
	builtin.RequireNoErr(rt, err, ErrBalanceInvariantBroken, "balance invariants broken")
	return nil
}

///////////////////////
// Sector Commitment //
///////////////////////

//type SectorPreCommitInfo struct {
//	SealProof       abi.RegisteredSealProof
//	SectorNumber    abi.SectorNumber
//	SealedCID       cid.Cid `checked:"true"` // CommR
//	SealRandEpoch   abi.ChainEpoch
//	DealIDs         []abi.DealID
//	Expiration      abi.ChainEpoch
//	ReplaceCapacity bool // Whether to replace a "committed capacity" no-deal sector (requires non-empty DealIDs)
//	// The committed capacity sector to replace, and it's deadline/partition location
//	ReplaceSectorDeadline  uint64
//	ReplaceSectorPartition uint64
//	ReplaceSectorNumber    abi.SectorNumber
//}
type PreCommitSectorParams = miner0.SectorPreCommitInfo

// Pledges to seal and commit a single sector.
// See PreCommitSectorBatch for details.
// This method may be deprecated and removed in the future.
func (a Actor) PreCommitSector(rt Runtime, params *PreCommitSectorParams) *abi.EmptyValue {
	// This is a direct method call to self, not a message send.
	batchParams := &PreCommitSectorBatchParams{Sectors: []miner0.SectorPreCommitInfo{*params}}
	a.PreCommitSectorBatch(rt, batchParams)
	return nil
}

type PreCommitSectorBatchParams struct {
	Sectors []miner0.SectorPreCommitInfo
}

// Pledges the miner to seal and commit some new sectors.
// The caller specifies sector numbers, sealed sector data CIDs, seal randomness epoch, expiration, and the IDs
// of any storage deals contained in the sector data. The storage deal proposals must be already submitted
// to the storage market actor.
// A pre-commitment may specify an existing committed-capacity sector that the committed sector will replace
// when proven.
// This method calculates the sector's power, locks a pre-commit deposit for the sector, stores information about the
// sector in state and waits for it to be proven or expire.
func (a Actor) PreCommitSectorBatch(rt Runtime, params *PreCommitSectorBatchParams) *abi.EmptyValue {
	currEpoch := rt.CurrEpoch()
	if len(params.Sectors) == 0 {
		rt.Abortf(exitcode.ErrIllegalArgument, "batch empty")
	} else if len(params.Sectors) > PreCommitSectorBatchMaxSize {
		rt.Abortf(exitcode.ErrIllegalArgument, "batch of %d too large, max %d", len(params.Sectors), PreCommitSectorBatchMaxSize)
	}

	// Check per-sector preconditions before opening state transaction or sending other messages.
	challengeEarliest := currEpoch - MaxPreCommitRandomnessLookback
	sectorsDeals := make([]market.SectorDeals, len(params.Sectors))
	sectorNumbers := bitfield.New()
	for i, precommit := range params.Sectors {
		// Bitfied.IsSet() is fast when there are only locally-set values.
		set, err := sectorNumbers.IsSet(uint64(precommit.SectorNumber))
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "error checking sector number")
		if set {
			rt.Abortf(exitcode.ErrIllegalArgument, "duplicate sector number %d", precommit.SectorNumber)
		}
		sectorNumbers.Set(uint64(precommit.SectorNumber))

		if !CanPreCommitSealProof(precommit.SealProof) {
			rt.Abortf(exitcode.ErrIllegalArgument, "unsupported seal proof type %v", precommit.SealProof)
		}
		if precommit.SectorNumber > abi.MaxSectorNumber {
			rt.Abortf(exitcode.ErrIllegalArgument, "sector number %d out of range 0..(2^63-1)", precommit.SectorNumber)
		}
		if !precommit.SealedCID.Defined() {
			rt.Abortf(exitcode.ErrIllegalArgument, "sealed CID undefined")
		}
		if precommit.SealedCID.Prefix() != SealedCIDPrefix {
			rt.Abortf(exitcode.ErrIllegalArgument, "sealed CID had wrong prefix")
		}
		if precommit.SealRandEpoch >= currEpoch {
			rt.Abortf(exitcode.ErrIllegalArgument, "seal challenge epoch %v must be before now %v", precommit.SealRandEpoch, rt.CurrEpoch())
		}
		if precommit.SealRandEpoch < challengeEarliest {
			rt.Abortf(exitcode.ErrIllegalArgument, "seal challenge epoch %v too old, must be after %v", precommit.SealRandEpoch, challengeEarliest)
		}

		// Require sector lifetime meets minimum by assuming activation happens at last epoch permitted for seal proof.
		// This could make sector maximum lifetime validation more lenient if the maximum sector limit isn't hit first.
		maxActivation := currEpoch + MaxProveCommitDuration[precommit.SealProof]
		validateExpiration(rt, maxActivation, precommit.Expiration, precommit.SealProof)

		if precommit.ReplaceCapacity {
			rt.Abortf(exitcode.SysErrForbidden, "cc upgrade through precommit discontinued, use lightweight cc upgrade instead")
		}

		sectorsDeals[i] = market.SectorDeals{
			SectorExpiry: precommit.Expiration,
			DealIDs:      precommit.DealIDs,
		}
	}

	// gather information from other actors
	rewardStats := requestCurrentEpochBlockReward(rt)
	pwrTotal := requestCurrentTotalPower(rt)
	dealWeights := requestDealWeights(rt, sectorsDeals)

	if len(dealWeights.Sectors) != len(params.Sectors) {
		rt.Abortf(exitcode.ErrIllegalState, "deal weight request returned %d records, expected %d",
			len(dealWeights.Sectors), len(params.Sectors))
	}

	store := adt.AsStore(rt)
	var st State
	var err error
	feeToBurn := abi.NewTokenAmount(0)
	var needsCron bool
	rt.StateTransaction(&st, func() {
		// Aggregate fee applies only when batching.
		if len(params.Sectors) > 1 {
			aggregateFee := AggregatePreCommitNetworkFee(len(params.Sectors), rt.BaseFee())
			// AggregateFee applied to fee debt to consolidate burn with outstanding debts
			err := st.ApplyPenalty(aggregateFee)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to apply penalty")
		}

		// available balance already accounts for fee debt so it is correct to call
		// this before RepayDebts. We would have to
		// subtract fee debt explicitly if we called this after.
		availableBalance, err := st.GetAvailableBalance(rt.CurrentBalance())
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to calculate available balance")
		feeToBurn = RepayDebtsOrAbort(rt, &st)

		info := getMinerInfo(rt, &st)
		rt.ValidateImmediateCallerIs(append(info.ControlAddresses, info.Owner, info.Worker)...)

		if ConsensusFaultActive(info, currEpoch) {
			rt.Abortf(exitcode.ErrForbidden, "pre-commit not allowed during active consensus fault")
		}

		chainInfos := make([]*SectorPreCommitOnChainInfo, len(params.Sectors))
		totalDepositRequired := big.Zero()
		cleanUpEvents := map[abi.ChainEpoch][]uint64{}
		dealCountMax := SectorDealsMax(info.SectorSize)
		for i, precommit := range params.Sectors {
			// Sector must have the same Window PoSt proof type as the miner's recorded seal type.
			sectorWPoStProof, err := precommit.SealProof.RegisteredWindowPoStProof()
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "failed to lookup Window PoSt proof type for sector seal proof %d", precommit.SealProof)
			if sectorWPoStProof != info.WindowPoStProofType {
				rt.Abortf(exitcode.ErrIllegalArgument, "sector Window PoSt proof type %d must match miner Window PoSt proof type %d (seal proof type %d)",
					sectorWPoStProof, info.WindowPoStProofType, precommit.SealProof)
			}

			if uint64(len(precommit.DealIDs)) > dealCountMax {
				rt.Abortf(exitcode.ErrIllegalArgument, "too many deals for sector %d > %d", len(precommit.DealIDs), dealCountMax)
			}

			// Ensure total deal space does not exceed sector size.
			dealWeight := dealWeights.Sectors[i]
			if dealWeight.DealSpace > uint64(info.SectorSize) {
				rt.Abortf(exitcode.ErrIllegalArgument, "deals too large to fit in sector %d > %d", dealWeight.DealSpace, info.SectorSize)
			}

			if precommit.ReplaceCapacity {
				validateReplaceSector(rt, &st, store, &precommit)
			}

			// Estimate the sector weight using the current epoch as an estimate for activation,
			// and compute the pre-commit deposit using that weight.
			// The sector's power will be recalculated when it's proven.
			duration := precommit.Expiration - currEpoch
			sectorWeight := QAPowerForWeight(info.SectorSize, duration, dealWeight.DealWeight, dealWeight.VerifiedDealWeight)
			depositReq := PreCommitDepositForPower(rewardStats.ThisEpochRewardSmoothed, pwrTotal.QualityAdjPowerSmoothed, sectorWeight)

			// Build on-chain record.
			chainInfos[i] = &SectorPreCommitOnChainInfo{
				Info:               SectorPreCommitInfo(precommit),
				PreCommitDeposit:   depositReq,
				PreCommitEpoch:     currEpoch,
				DealWeight:         dealWeight.DealWeight,
				VerifiedDealWeight: dealWeight.VerifiedDealWeight,
			}
			totalDepositRequired = big.Add(totalDepositRequired, depositReq)

			// Calculate pre-commit cleanup
			msd, ok := MaxProveCommitDuration[precommit.SealProof]
			if !ok {
				rt.Abortf(exitcode.ErrIllegalArgument, "no max seal duration set for proof type: %d", precommit.SealProof)
			}
			// PreCommitCleanUpDelay > 0 here is critical for the batch verification of proofs. Without it, if a proof arrived exactly on the
			// due epoch, ProveCommitSector would accept it, then the expiry event would remove it, and then
			// ConfirmSectorProofsValid would fail to find it.
			cleanUpBound := currEpoch + msd + ExpiredPreCommitCleanUpDelay
			cleanUpEvents[cleanUpBound] = append(cleanUpEvents[cleanUpBound], uint64(precommit.SectorNumber))
		}

		// Batch update actor state.
		if availableBalance.LessThan(totalDepositRequired) {
			rt.Abortf(exitcode.ErrInsufficientFunds, "insufficient funds %v for pre-commit deposit: %v", availableBalance, totalDepositRequired)
		}
		err = st.AddPreCommitDeposit(totalDepositRequired)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to add pre-commit deposit %v", totalDepositRequired)

		err = st.AllocateSectorNumbers(store, sectorNumbers, DenyCollisions)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to allocate sector ids %v", sectorNumbers)

		err = st.PutPrecommittedSectors(store, chainInfos...)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to write pre-committed sectors")

		err = st.AddPreCommitCleanUps(store, cleanUpEvents)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to add pre-commit expiry to queue")

		// Activate miner cron
		needsCron = !st.DeadlineCronActive
		st.DeadlineCronActive = true
	})

	burnFunds(rt, feeToBurn, BurnMethodPreCommitSectorBatch)
	rt.StateReadonly(&st)
	err = st.CheckBalanceInvariants(rt.CurrentBalance())
	builtin.RequireNoErr(rt, err, ErrBalanceInvariantBroken, "balance invariants broken")
	if needsCron {
		newDlInfo := st.DeadlineInfo(currEpoch)
		enrollCronEvent(rt, newDlInfo.Last(), &CronEventPayload{
			EventType: CronEventProvingDeadline,
		})
	}

	return nil
}

type ProveCommitAggregateParams struct {
	SectorNumbers  bitfield.BitField
	AggregateProof []byte
}

// Checks state of the corresponding sector pre-commitments and verifies aggregate proof of replication
// of these sectors. If valid, the sectors' deals are activated, sectors are assigned a deadline and charged pledge
// and precommit state is removed.
func (a Actor) ProveCommitAggregate(rt Runtime, params *ProveCommitAggregateParams) *abi.EmptyValue {
	aggSectorsCount, err := params.SectorNumbers.Count()
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to count aggregated sectors")
	if aggSectorsCount > MaxAggregatedSectors {
		rt.Abortf(exitcode.ErrIllegalArgument, "too many sectors addressed, addressed %d want <= %d", aggSectorsCount, MaxAggregatedSectors)
	} else if aggSectorsCount < MinAggregatedSectors {
		rt.Abortf(exitcode.ErrIllegalArgument, "too few sectors addressed, addressed %d want >= %d", aggSectorsCount, MinAggregatedSectors)
	}

	if uint64(len(params.AggregateProof)) > MaxAggregateProofSize {
		rt.Abortf(exitcode.ErrIllegalArgument, "sector prove-commit proof of size %d exceeds max size of %d",
			len(params.AggregateProof), MaxAggregateProofSize)
	}

	store := adt.AsStore(rt)
	var st State
	rt.StateReadonly(&st)

	info := getMinerInfo(rt, &st)
	rt.ValidateImmediateCallerIs(append(info.ControlAddresses, info.Owner, info.Worker)...)

	precommits, err := st.GetAllPrecommittedSectors(store, params.SectorNumbers)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to get precommits")

	// compute data commitments and validate each precommit
	computeDataCommitmentsInputs := make([]*market.SectorDataSpec, len(precommits))
	precommitsToConfirm := []*SectorPreCommitOnChainInfo{}
	for i, precommit := range precommits {
		msd, ok := MaxProveCommitDuration[precommit.Info.SealProof]
		if !ok {
			rt.Abortf(exitcode.ErrIllegalState, "no max seal duration for proof type: %d", precommit.Info.SealProof)
		}
		proveCommitDue := precommit.PreCommitEpoch + msd
		if rt.CurrEpoch() > proveCommitDue {
			rt.Log(rtt.WARN, "skipping commitment for sector %d, too late at %d, due %d", precommit.Info.SectorNumber, rt.CurrEpoch(), proveCommitDue)
		} else {
			precommitsToConfirm = append(precommitsToConfirm, precommit)
		}
		// All sealProof types should match
		if i >= 1 {
			prevSealProof := precommits[i-1].Info.SealProof
			builtin.RequireState(rt, prevSealProof == precommit.Info.SealProof, "aggregate contains mismatched seal proofs %d and %d", prevSealProof, precommit.Info.SealProof)
		}

		computeDataCommitmentsInputs[i] = &market.SectorDataSpec{
			SectorType: precommit.Info.SealProof,
			DealIDs:    precommit.Info.DealIDs,
		}
	}

	// compute shared verification inputs
	commDs := requestUnsealedSectorCIDs(rt, computeDataCommitmentsInputs...)
	svis := make([]proof.AggregateSealVerifyInfo, 0)
	receiver := rt.Receiver()
	minerActorID, err := addr.IDFromAddress(receiver)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "runtime provided non-ID receiver address %s", receiver)
	buf := new(bytes.Buffer)
	err = receiver.MarshalCBOR(buf)
	receiverBytes := buf.Bytes()
	builtin.RequireNoErr(rt, err, exitcode.ErrSerialization, "failed to marshal address for seal verification challenge")

	for i, precommit := range precommits {
		interactiveEpoch := precommit.PreCommitEpoch + PreCommitChallengeDelay
		if rt.CurrEpoch() <= interactiveEpoch {
			rt.Abortf(exitcode.ErrForbidden, "too early to prove sector %d", precommit.Info.SectorNumber)
		}

		svInfoRandomness := rt.GetRandomnessFromTickets(crypto.DomainSeparationTag_SealRandomness, precommit.Info.SealRandEpoch, receiverBytes)
		svInfoInteractiveRandomness := rt.GetRandomnessFromBeacon(crypto.DomainSeparationTag_InteractiveSealChallengeSeed, interactiveEpoch, receiverBytes)
		svi := proof.AggregateSealVerifyInfo{
			Number:                precommit.Info.SectorNumber,
			InteractiveRandomness: abi.InteractiveSealRandomness(svInfoInteractiveRandomness),
			Randomness:            abi.SealRandomness(svInfoRandomness),
			SealedCID:             precommit.Info.SealedCID,
			UnsealedCID:           commDs[i],
		}
		svis = append(svis, svi)
	}

	builtin.RequireState(rt, len(precommits) > 0, "bitfield non-empty but zero precommits read from state")
	sealProof := precommits[0].Info.SealProof
	err = rt.VerifyAggregateSeals(
		proof.AggregateSealVerifyProofAndInfos{
			Infos:          svis,
			Proof:          params.AggregateProof,
			Miner:          abi.ActorID(minerActorID),
			SealProof:      sealProof,
			AggregateProof: abi.RegisteredAggregationProof_SnarkPackV1,
		})
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "aggregate seal verify failed")

	rew := requestCurrentEpochBlockReward(rt)
	pwr := requestCurrentTotalPower(rt)

	confirmSectorProofsValid(rt, precommitsToConfirm, rew.ThisEpochBaselinePower, rew.ThisEpochRewardSmoothed, pwr.QualityAdjPowerSmoothed)

	// Compute and burn the aggregate network fee. We need to re-load the state as
	// confirmSectorProofsValid can change it.
	rt.StateReadonly(&st)
	aggregateFee := AggregateProveCommitNetworkFee(len(precommitsToConfirm), rt.BaseFee())
	unlockedBalance, err := st.GetUnlockedBalance(rt.CurrentBalance())
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to determine unlocked balance")
	if unlockedBalance.LessThan(aggregateFee) {
		rt.Abortf(exitcode.ErrInsufficientFunds,
			"remaining unlocked funds after prove-commit (%s) are insufficient to pay aggregation fee of %s",
			unlockedBalance, aggregateFee,
		)
	}
	burnFunds(rt, aggregateFee, BurnMethodProveCommitAggregate)

	err = st.CheckBalanceInvariants(rt.CurrentBalance())
	builtin.RequireNoErr(rt, err, ErrBalanceInvariantBroken, "balance invariants broken")

	return nil
}

//type ProveCommitSectorParams struct {
//	SectorNumber abi.SectorNumber
//	ReplicaProof        []byte
//}
type ProveCommitSectorParams = miner0.ProveCommitSectorParams

// Checks state of the corresponding sector pre-commitment, then schedules the proof to be verified in bulk
// by the power actor.
// If valid, the power actor will call ConfirmSectorProofsValid at the end of the same epoch as this message.
func (a Actor) ProveCommitSector(rt Runtime, params *ProveCommitSectorParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerAcceptAny()

	if params.SectorNumber > abi.MaxSectorNumber {
		rt.Abortf(exitcode.ErrIllegalArgument, "sector number greater than maximum")
	}

	store := adt.AsStore(rt)
	sectorNo := params.SectorNumber

	var st State
	rt.StateReadonly(&st)

	precommit, found, err := st.GetPrecommittedSector(store, sectorNo)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load pre-committed sector %v", sectorNo)
	if !found {
		rt.Abortf(exitcode.ErrNotFound, "no pre-committed sector %v", sectorNo)
	}

	maxProofSize, err := precommit.Info.SealProof.ProofSize()
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to determine max proof size for sector %v", sectorNo)
	if uint64(len(params.Proof)) > maxProofSize {
		rt.Abortf(exitcode.ErrIllegalArgument, "sector prove-commit proof of size %d exceeds max size of %d",
			len(params.Proof), maxProofSize)
	}

	msd, ok := MaxProveCommitDuration[precommit.Info.SealProof]
	if !ok {
		rt.Abortf(exitcode.ErrIllegalState, "no max seal duration for proof type: %d", precommit.Info.SealProof)
	}
	proveCommitDue := precommit.PreCommitEpoch + msd
	if rt.CurrEpoch() > proveCommitDue {
		rt.Abortf(exitcode.ErrIllegalArgument, "commitment proof for %d too late at %d, due %d", sectorNo, rt.CurrEpoch(), proveCommitDue)
	}

	svi := getVerifyInfo(rt, &SealVerifyStuff{
		SealedCID:           precommit.Info.SealedCID,
		InteractiveEpoch:    precommit.PreCommitEpoch + PreCommitChallengeDelay,
		SealRandEpoch:       precommit.Info.SealRandEpoch,
		Proof:               params.Proof,
		DealIDs:             precommit.Info.DealIDs,
		SectorNumber:        precommit.Info.SectorNumber,
		RegisteredSealProof: precommit.Info.SealProof,
	})

	code := rt.Send(
		builtin.StoragePowerActorAddr,
		builtin.MethodsPower.SubmitPoRepForBulkVerify,
		svi,
		abi.NewTokenAmount(0),
		&builtin.Discard{},
	)
	builtin.RequireSuccess(rt, code, "failed to submit proof for bulk verification")
	return nil
}

func (a Actor) ConfirmSectorProofsValid(rt Runtime, params *builtin.ConfirmSectorProofsParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.StoragePowerActorAddr)

	// This should be enforced by the power actor. We log here just in case
	// something goes wrong.
	if len(params.Sectors) > power.MaxMinerProveCommitsPerEpoch {
		rt.Log(rtt.WARN, "confirmed more prove commits in an epoch than permitted: %d > %d",
			len(params.Sectors), power.MaxMinerProveCommitsPerEpoch,
		)
	}

	var st State
	rt.StateReadonly(&st)
	store := adt.AsStore(rt)

	// This skips missing pre-commits.
	precommittedSectors, err := st.FindPrecommittedSectors(store, params.Sectors...)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load pre-committed sectors")

	confirmSectorProofsValid(rt, precommittedSectors, params.RewardBaselinePower, params.RewardSmoothed, params.QualityAdjPowerSmoothed)

	return nil
}

func confirmSectorProofsValid(rt Runtime, preCommits []*SectorPreCommitOnChainInfo, thisEpochBaselinePower big.Int,
	thisEpochRewardSmoothed smoothing.FilterEstimate, qualityAdjPowerSmoothed smoothing.FilterEstimate) {

	circulatingSupply := rt.TotalFilCircSupply()

	// 1. Activate deals, skipping pre-commits with invalid deals.
	//    - calls the market actor.
	// 2. Reschedule replacement sector expiration.
	//    - loads and saves sectors
	//    - loads and saves deadlines/partitions
	// 3. Add new sectors.
	//    - loads and saves sectors.
	//    - loads and saves deadlines/partitions
	//
	// Ideally, we'd combine some of these operations, but at least we have
	// a constant number of them.

	activation := rt.CurrEpoch()
	// Pre-commits for new sectors.
	var validPreCommits []*SectorPreCommitOnChainInfo
	for _, precommit := range preCommits {
		if len(precommit.Info.DealIDs) > 0 {
			// Check (and activate) storage deals associated to sector. Abort if checks failed.
			// TODO: we should batch these calls...
			// https://github.com/filecoin-project/specs-actors/issues/474
			code := rt.Send(
				builtin.StorageMarketActorAddr,
				builtin.MethodsMarket.ActivateDeals,
				&market.ActivateDealsParams{
					DealIDs:      precommit.Info.DealIDs,
					SectorExpiry: precommit.Info.Expiration,
				},
				abi.NewTokenAmount(0),
				&builtin.Discard{},
			)

			if code != exitcode.Ok {
				rt.Log(rtt.INFO, "failed to activate deals on sector %d, dropping from prove commit set", precommit.Info.SectorNumber)
				continue
			}
		}

		validPreCommits = append(validPreCommits, precommit)
	}

	// When all prove commits have failed abort early
	if len(validPreCommits) == 0 {
		rt.Abortf(exitcode.ErrIllegalArgument, "all prove commits failed to validate")
	}

	totalPledge := big.Zero()
	depositToUnlock := big.Zero()
	newSectors := make([]*SectorOnChainInfo, 0)
	newlyVested := big.Zero()
	var st State
	store := adt.AsStore(rt)
	rt.StateTransaction(&st, func() {
		info := getMinerInfo(rt, &st)

		newSectorNos := make([]abi.SectorNumber, 0, len(validPreCommits))
		for _, precommit := range validPreCommits {
			// compute initial pledge
			duration := precommit.Info.Expiration - activation
			// This should have been caught in precommit, but don't let other sectors fail because of it.
			if duration < MinSectorExpiration {
				rt.Log(rtt.WARN, "precommit %d has lifetime %d less than minimum. ignoring", precommit.Info.SectorNumber, duration, MinSectorExpiration)
				continue
			}
			pwr := QAPowerForWeight(info.SectorSize, duration, precommit.DealWeight, precommit.VerifiedDealWeight)

			dayReward := ExpectedRewardForPower(thisEpochRewardSmoothed, qualityAdjPowerSmoothed, pwr, builtin.EpochsInDay)
			// The storage pledge is recorded for use in computing the penalty if this sector is terminated
			// before its declared expiration.
			// It's not capped to 1 FIL, so can exceed the actual initial pledge requirement.
			storagePledge := ExpectedRewardForPower(thisEpochRewardSmoothed, qualityAdjPowerSmoothed, pwr, InitialPledgeProjectionPeriod)
			initialPledge := InitialPledgeForPower(pwr, thisEpochBaselinePower, thisEpochRewardSmoothed,
				qualityAdjPowerSmoothed, circulatingSupply)

			// Lower-bound the pledge by that of the sector being replaced.
			// Record the replaced age and reward rate for termination fee calculations.
			_, replacedAge, replacedDayReward := zeroReplacedSectorParameters()

			newSectorInfo := SectorOnChainInfo{
				SectorNumber:          precommit.Info.SectorNumber,
				SealProof:             precommit.Info.SealProof,
				SealedCID:             precommit.Info.SealedCID,
				DealIDs:               precommit.Info.DealIDs,
				Expiration:            precommit.Info.Expiration,
				Activation:            activation,
				DealWeight:            precommit.DealWeight,
				VerifiedDealWeight:    precommit.VerifiedDealWeight,
				InitialPledge:         initialPledge,
				ExpectedDayReward:     dayReward,
				ExpectedStoragePledge: storagePledge,
				ReplacedSectorAge:     replacedAge,
				ReplacedDayReward:     replacedDayReward,
			}

			depositToUnlock = big.Add(depositToUnlock, precommit.PreCommitDeposit)
			newSectors = append(newSectors, &newSectorInfo)
			newSectorNos = append(newSectorNos, newSectorInfo.SectorNumber)
			totalPledge = big.Add(totalPledge, initialPledge)
		}

		err := st.PutSectors(store, newSectors...)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to put new sectors")

		err = st.DeletePrecommittedSectors(store, newSectorNos...)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to delete precommited sectors")

		err = st.AssignSectorsToDeadlines(store, rt.CurrEpoch(), newSectors, info.WindowPoStPartitionSectors, info.SectorSize)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to assign new sectors to deadlines")

		// Unlock deposit for successful proofs, make it available for lock-up as initial pledge.
		err = st.AddPreCommitDeposit(depositToUnlock.Neg())
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to add pre-commit deposit %v", depositToUnlock.Neg())

		unlockedBalance, err := st.GetUnlockedBalance(rt.CurrentBalance())
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to calculate unlocked balance")
		if unlockedBalance.LessThan(totalPledge) {
			rt.Abortf(exitcode.ErrInsufficientFunds, "insufficient funds for aggregate initial pledge requirement %s, available: %s", totalPledge, unlockedBalance)
		}

		err = st.AddInitialPledge(totalPledge)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to add initial pledge %v", totalPledge)
		err = st.CheckBalanceInvariants(rt.CurrentBalance())
		builtin.RequireNoErr(rt, err, ErrBalanceInvariantBroken, "balance invariants broken")
	})

	// Request pledge update for activated sector.
	notifyPledgeChanged(rt, big.Sub(totalPledge, newlyVested))
}

//type CheckSectorProvenParams struct {
//	SectorNumber abi.SectorNumber
//}
type CheckSectorProvenParams = miner0.CheckSectorProvenParams

func (a Actor) CheckSectorProven(rt Runtime, params *CheckSectorProvenParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerAcceptAny()

	if params.SectorNumber > abi.MaxSectorNumber {
		rt.Abortf(exitcode.ErrIllegalArgument, "sector number out of range")
	}

	var st State
	rt.StateReadonly(&st)
	store := adt.AsStore(rt)
	sectorNo := params.SectorNumber

	if _, found, err := st.GetSector(store, sectorNo); err != nil {
		rt.Abortf(exitcode.ErrIllegalState, "failed to load proven sector %v", sectorNo)
	} else if !found {
		rt.Abortf(exitcode.ErrNotFound, "sector %v not proven", sectorNo)
	}
	return nil
}

/////////////////////////
// Sector Modification //
/////////////////////////

//type ExtendSectorExpirationParams struct {
//	Extensions []ExpirationExtension
//}
type ExtendSectorExpirationParams = miner0.ExtendSectorExpirationParams

//type ExpirationExtension struct {
//	Deadline      uint64
//	Partition     uint64
//	Sectors       bitfield.BitField
//	NewExpiration abi.ChainEpoch
//}
type ExpirationExtension = miner0.ExpirationExtension

// Changes the expiration epoch for a sector to a new, later one.
// The sector must not be terminated or faulty.
// The sector's power is recomputed for the new expiration.
func (a Actor) ExtendSectorExpiration(rt Runtime, params *ExtendSectorExpirationParams) *abi.EmptyValue {
	if uint64(len(params.Extensions)) > DeclarationsMax {
		rt.Abortf(exitcode.ErrIllegalArgument, "too many declarations %d, max %d", len(params.Extensions), DeclarationsMax)
	}

	// limit the number of sectors declared at once
	// https://github.com/filecoin-project/specs-actors/issues/416
	var sectorCount uint64
	for _, decl := range params.Extensions {
		if decl.Deadline >= WPoStPeriodDeadlines {
			rt.Abortf(exitcode.ErrIllegalArgument, "deadline %d not in range 0..%d", decl.Deadline, WPoStPeriodDeadlines)
		}
		count, err := decl.Sectors.Count()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument,
			"failed to count sectors for deadline %d, partition %d",
			decl.Deadline, decl.Partition,
		)
		if sectorCount > math.MaxUint64-count {
			rt.Abortf(exitcode.ErrIllegalArgument, "sector bitfield integer overflow")
		}
		sectorCount += count
	}
	if sectorCount > AddressedSectorsMax {
		rt.Abortf(exitcode.ErrIllegalArgument,
			"too many sectors for declaration %d, max %d",
			sectorCount, AddressedSectorsMax,
		)
	}

	currEpoch := rt.CurrEpoch()

	powerDelta := NewPowerPairZero()
	pledgeDelta := big.Zero()
	store := adt.AsStore(rt)
	var st State
	rt.StateTransaction(&st, func() {
		info := getMinerInfo(rt, &st)

		rt.ValidateImmediateCallerIs(append(info.ControlAddresses, info.Owner, info.Worker)...)

		deadlines, err := st.LoadDeadlines(adt.AsStore(rt))
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadlines")

		// Group declarations by deadline, and remember iteration order.
		// This should be merged with the iteration outside the state transaction.
		declsByDeadline := map[uint64][]*ExpirationExtension{}
		var deadlinesToLoad []uint64
		for i := range params.Extensions {
			// Take a pointer to the value inside the slice, don't
			// take a reference to the temporary loop variable as it
			// will be overwritten every iteration.
			decl := &params.Extensions[i]
			if _, ok := declsByDeadline[decl.Deadline]; !ok {
				deadlinesToLoad = append(deadlinesToLoad, decl.Deadline)
			}
			declsByDeadline[decl.Deadline] = append(declsByDeadline[decl.Deadline], decl)
		}

		sectors, err := LoadSectors(store, st.Sectors)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load sectors array")

		for _, dlIdx := range deadlinesToLoad {
			deadline, err := deadlines.LoadDeadline(store, dlIdx)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadline %d", dlIdx)

			partitions, err := deadline.PartitionsArray(store)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load partitions for deadline %d", dlIdx)

			quant := st.QuantSpecForDeadline(dlIdx)

			// Group modified partitions by epoch to which they are extended. Duplicates are ok.
			partitionsByNewEpoch := map[abi.ChainEpoch][]uint64{}
			// Remember iteration order of epochs.
			var epochsToReschedule []abi.ChainEpoch

			for _, decl := range declsByDeadline[dlIdx] {
				var partition Partition
				found, err := partitions.Get(decl.Partition, &partition)
				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadline %v partition %v", dlIdx, decl.Partition)
				if !found {
					rt.Abortf(exitcode.ErrNotFound, "no such deadline %v partition %v", dlIdx, decl.Partition)
				}

				oldSectors, err := sectors.Load(decl.Sectors)
				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load sectors in deadline %v partition %v", dlIdx, decl.Partition)
				newSectors := make([]*SectorOnChainInfo, len(oldSectors))
				for i, sector := range oldSectors {
					if !CanExtendSealProofType(sector.SealProof) {
						rt.Abortf(exitcode.ErrForbidden, "cannot extend expiration for sector %v with unsupported seal type %v",
							sector.SectorNumber, sector.SealProof)
					}
					// This can happen if the sector should have already expired, but hasn't
					// because the end of its deadline hasn't passed yet.
					if sector.Expiration < currEpoch {
						rt.Abortf(exitcode.ErrForbidden, "cannot extend expiration for expired sector %v, expired at %d, now %d",
							sector.SectorNumber,
							sector.Expiration,
							currEpoch,
						)
					}
					if decl.NewExpiration < sector.Expiration {
						rt.Abortf(exitcode.ErrIllegalArgument, "cannot reduce sector %v's expiration to %d from %d",
							sector.SectorNumber, decl.NewExpiration, sector.Expiration)
					}
					validateExpiration(rt, sector.Activation, decl.NewExpiration, sector.SealProof)

					// Remove "spent" deal weights
					newDealWeight := big.Div(
						big.Mul(sector.DealWeight, big.NewInt(int64(sector.Expiration-currEpoch))),
						big.NewInt(int64(sector.Expiration-sector.Activation)),
					)
					newVerifiedDealWeight := big.Div(
						big.Mul(sector.VerifiedDealWeight, big.NewInt(int64(sector.Expiration-currEpoch))),
						big.NewInt(int64(sector.Expiration-sector.Activation)),
					)

					newSector := *sector
					newSector.Expiration = decl.NewExpiration
					newSector.DealWeight = newDealWeight
					newSector.VerifiedDealWeight = newVerifiedDealWeight

					newSectors[i] = &newSector
				}

				// Overwrite sector infos.
				err = sectors.Store(newSectors...)
				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to update sectors %v", decl.Sectors)

				// Remove old sectors from partition and assign new sectors.
				partitionPowerDelta, partitionPledgeDelta, err := partition.ReplaceSectors(store, oldSectors, newSectors, info.SectorSize, quant)
				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to replace sector expirations at deadline %v partition %v", dlIdx, decl.Partition)

				powerDelta = powerDelta.Add(partitionPowerDelta)
				pledgeDelta = big.Add(pledgeDelta, partitionPledgeDelta) // expected to be zero, see note below.

				err = partitions.Set(decl.Partition, &partition)
				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save deadline %v partition %v", dlIdx, decl.Partition)

				// Record the new partition expiration epoch for setting outside this loop over declarations.
				prevEpochPartitions, ok := partitionsByNewEpoch[decl.NewExpiration]
				partitionsByNewEpoch[decl.NewExpiration] = append(prevEpochPartitions, decl.Partition)
				if !ok {
					epochsToReschedule = append(epochsToReschedule, decl.NewExpiration)
				}
			}

			deadline.Partitions, err = partitions.Root()
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save partitions for deadline %d", dlIdx)

			// Record partitions in deadline expiration queue
			for _, epoch := range epochsToReschedule {
				pIdxs := partitionsByNewEpoch[epoch]
				err := deadline.AddExpirationPartitions(store, epoch, pIdxs, quant)
				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to add expiration partitions to deadline %v epoch %v: %v",
					dlIdx, epoch, pIdxs)
			}

			err = deadlines.UpdateDeadline(store, dlIdx, deadline)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save deadline %d", dlIdx)
		}

		st.Sectors, err = sectors.Root()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save sectors")

		err = st.SaveDeadlines(store, deadlines)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save deadlines")
	})

	requestUpdatePower(rt, powerDelta)
	// Note: the pledge delta is expected to be zero, since pledge is not re-calculated for the extension.
	// But in case that ever changes, we can do the right thing here.
	notifyPledgeChanged(rt, pledgeDelta)
	return nil
}

//type TerminateSectorsParams struct {
//	Terminations []TerminationDeclaration
//}
type TerminateSectorsParams = miner0.TerminateSectorsParams

//type TerminationDeclaration struct {
//	Deadline  uint64
//	Partition uint64
//	Sectors   bitfield.BitField
//}
type TerminationDeclaration = miner0.TerminationDeclaration

//type TerminateSectorsReturn struct {
//	// Set to true if all early termination work has been completed. When
//	// false, the miner may choose to repeatedly invoke TerminateSectors
//	// with no new sectors to process the remainder of the pending
//	// terminations. While pending terminations are outstanding, the miner
//	// will not be able to withdraw funds.
//	Done bool
//}
type TerminateSectorsReturn = miner0.TerminateSectorsReturn

// Marks some sectors as terminated at the present epoch, earlier than their
// scheduled termination, and adds these sectors to the early termination queue.
// This method then processes up to AddressedSectorsMax sectors and
// AddressedPartitionsMax partitions from the early termination queue,
// terminating deals, paying fines, and returning pledge collateral. While
// sectors remain in this queue:
//
//  1. The miner will be unable to withdraw funds.
//  2. The chain will process up to AddressedSectorsMax sectors and
//     AddressedPartitionsMax per epoch until the queue is empty.
//
// The sectors are immediately ignored for Window PoSt proofs, and should be
// masked in the same way as faulty sectors. A miner may not terminate sectors in the
// current deadline or the next deadline to be proven.
//
// This function may be invoked with no new sectors to explicitly process the
// next batch of sectors.
func (a Actor) TerminateSectors(rt Runtime, params *TerminateSectorsParams) *TerminateSectorsReturn {
	// Note: this cannot terminate pre-committed but un-proven sectors.
	// They must be allowed to expire (and deposit burnt).

	if len(params.Terminations) > DeclarationsMax {
		rt.Abortf(exitcode.ErrIllegalArgument,
			"too many declarations when terminating sectors: %d > %d",
			len(params.Terminations), DeclarationsMax,
		)
	}

	toProcess := make(DeadlineSectorMap)
	for _, term := range params.Terminations {
		err := toProcess.Add(term.Deadline, term.Partition, term.Sectors)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument,
			"failed to process deadline %d, partition %d", term.Deadline, term.Partition,
		)
	}
	err := toProcess.Check(AddressedPartitionsMax, AddressedSectorsMax)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "cannot process requested parameters")

	var hadEarlyTerminations bool
	var st State
	store := adt.AsStore(rt)
	currEpoch := rt.CurrEpoch()
	powerDelta := NewPowerPairZero()
	rt.StateTransaction(&st, func() {
		hadEarlyTerminations = havePendingEarlyTerminations(rt, &st)

		info := getMinerInfo(rt, &st)
		rt.ValidateImmediateCallerIs(append(info.ControlAddresses, info.Owner, info.Worker)...)

		deadlines, err := st.LoadDeadlines(adt.AsStore(rt))
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadlines")

		// We're only reading the sectors, so there's no need to save this back.
		// However, we still want to avoid re-loading this array per-partition.
		sectors, err := LoadSectors(store, st.Sectors)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load sectors")

		err = toProcess.ForEach(func(dlIdx uint64, partitionSectors PartitionSectorMap) error {
			// If the deadline the current or next deadline to prove, don't allow terminating sectors.
			// We assume that deadlines are immutable when being proven.
			if !deadlineIsMutable(st.CurrentProvingPeriodStart(currEpoch), dlIdx, currEpoch) {
				rt.Abortf(exitcode.ErrIllegalArgument, "cannot terminate sectors in immutable deadline %d", dlIdx)
			}

			quant := st.QuantSpecForDeadline(dlIdx)

			deadline, err := deadlines.LoadDeadline(store, dlIdx)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadline %d", dlIdx)

			removedPower, err := deadline.TerminateSectors(store, sectors, currEpoch, partitionSectors, info.SectorSize, quant)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to terminate sectors in deadline %d", dlIdx)

			st.EarlyTerminations.Set(dlIdx)

			powerDelta = powerDelta.Sub(removedPower)

			err = deadlines.UpdateDeadline(store, dlIdx, deadline)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to update deadline %d", dlIdx)

			return nil
		})
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to walk sectors")

		err = st.SaveDeadlines(store, deadlines)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save deadlines")
	})

	epochReward := requestCurrentEpochBlockReward(rt)
	pwrTotal := requestCurrentTotalPower(rt)

	// Now, try to process these sectors.
	more := processEarlyTerminations(rt, epochReward.ThisEpochRewardSmoothed, pwrTotal.QualityAdjPowerSmoothed)
	if more && !hadEarlyTerminations {
		// We have remaining terminations, and we didn't _previously_
		// have early terminations to process, schedule a cron job.
		// NOTE: This isn't quite correct. If we repeatedly fill, empty,
		// fill, and empty, the queue, we'll keep scheduling new cron
		// jobs. However, in practice, that shouldn't be all that bad.
		scheduleEarlyTerminationWork(rt)
	}

	rt.StateReadonly(&st)
	err = st.CheckBalanceInvariants(rt.CurrentBalance())
	builtin.RequireNoErr(rt, err, ErrBalanceInvariantBroken, "balance invariants broken")

	requestUpdatePower(rt, powerDelta)
	return &TerminateSectorsReturn{Done: !more}
}

////////////
// Faults //
////////////

//type DeclareFaultsParams struct {
//	Faults []FaultDeclaration
//}
type DeclareFaultsParams = miner0.DeclareFaultsParams

//type FaultDeclaration struct {
//	// The deadline to which the faulty sectors are assigned, in range [0..WPoStPeriodDeadlines)
//	Deadline uint64
//	// Partition index within the deadline containing the faulty sectors.
//	Partition uint64
//	// Sectors in the partition being declared faulty.
//	Sectors bitfield.BitField
//}
type FaultDeclaration = miner0.FaultDeclaration

func (a Actor) DeclareFaults(rt Runtime, params *DeclareFaultsParams) *abi.EmptyValue {
	if len(params.Faults) > DeclarationsMax {
		rt.Abortf(exitcode.ErrIllegalArgument,
			"too many fault declarations for a single message: %d > %d",
			len(params.Faults), DeclarationsMax,
		)
	}

	toProcess := make(DeadlineSectorMap)
	for _, term := range params.Faults {
		err := toProcess.Add(term.Deadline, term.Partition, term.Sectors)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument,
			"failed to process deadline %d, partition %d", term.Deadline, term.Partition,
		)
	}
	err := toProcess.Check(AddressedPartitionsMax, AddressedSectorsMax)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "cannot process requested parameters")

	store := adt.AsStore(rt)
	var st State
	powerDelta := NewPowerPairZero()
	rt.StateTransaction(&st, func() {
		info := getMinerInfo(rt, &st)
		rt.ValidateImmediateCallerIs(append(info.ControlAddresses, info.Owner, info.Worker)...)

		deadlines, err := st.LoadDeadlines(store)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadlines")

		sectors, err := LoadSectors(store, st.Sectors)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load sectors array")

		currEpoch := rt.CurrEpoch()
		err = toProcess.ForEach(func(dlIdx uint64, pm PartitionSectorMap) error {
			targetDeadline, err := declarationDeadlineInfo(st.CurrentProvingPeriodStart(currEpoch), dlIdx, currEpoch)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "invalid fault declaration deadline %d", dlIdx)

			err = validateFRDeclarationDeadline(targetDeadline)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "failed fault declaration at deadline %d", dlIdx)

			deadline, err := deadlines.LoadDeadline(store, dlIdx)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadline %d", dlIdx)

			faultExpirationEpoch := targetDeadline.Last() + FaultMaxAge
			deadlinePowerDelta, err := deadline.RecordFaults(store, sectors, info.SectorSize, QuantSpecForDeadline(targetDeadline), faultExpirationEpoch, pm)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to declare faults for deadline %d", dlIdx)

			err = deadlines.UpdateDeadline(store, dlIdx, deadline)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to store deadline %d partitions", dlIdx)

			powerDelta = powerDelta.Add(deadlinePowerDelta)
			return nil
		})
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to iterate deadlines")

		err = st.SaveDeadlines(store, deadlines)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save deadlines")
	})

	// Remove power for new faulty sectors.
	// NOTE: It would be permissible to delay the power loss until the deadline closes, but that would require
	// additional accounting state.
	// https://github.com/filecoin-project/specs-actors/issues/414
	requestUpdatePower(rt, powerDelta)

	// Payment of penalty for declared faults is deferred to the deadline cron.
	return nil
}

//type DeclareFaultsRecoveredParams struct {
//	Recoveries []RecoveryDeclaration
//}
type DeclareFaultsRecoveredParams = miner0.DeclareFaultsRecoveredParams

//type RecoveryDeclaration struct {
//	// The deadline to which the recovered sectors are assigned, in range [0..WPoStPeriodDeadlines)
//	Deadline uint64
//	// Partition index within the deadline containing the recovered sectors.
//	Partition uint64
//	// Sectors in the partition being declared recovered.
//	Sectors bitfield.BitField
//}
type RecoveryDeclaration = miner0.RecoveryDeclaration

func (a Actor) DeclareFaultsRecovered(rt Runtime, params *DeclareFaultsRecoveredParams) *abi.EmptyValue {
	if len(params.Recoveries) > DeclarationsMax {
		rt.Abortf(exitcode.ErrIllegalArgument,
			"too many recovery declarations for a single message: %d > %d",
			len(params.Recoveries), DeclarationsMax,
		)
	}

	toProcess := make(DeadlineSectorMap)
	for _, term := range params.Recoveries {
		err := toProcess.Add(term.Deadline, term.Partition, term.Sectors)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument,
			"failed to process deadline %d, partition %d", term.Deadline, term.Partition,
		)
	}
	err := toProcess.Check(AddressedPartitionsMax, AddressedSectorsMax)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "cannot process requested parameters")

	store := adt.AsStore(rt)
	var st State
	feeToBurn := abi.NewTokenAmount(0)
	rt.StateTransaction(&st, func() {
		// Verify unlocked funds cover both InitialPledgeRequirement and FeeDebt
		// and repay fee debt now.
		feeToBurn = RepayDebtsOrAbort(rt, &st)

		info := getMinerInfo(rt, &st)
		rt.ValidateImmediateCallerIs(append(info.ControlAddresses, info.Owner, info.Worker)...)
		if ConsensusFaultActive(info, rt.CurrEpoch()) {
			rt.Abortf(exitcode.ErrForbidden, "recovery not allowed during active consensus fault")
		}

		deadlines, err := st.LoadDeadlines(adt.AsStore(rt))
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadlines")

		sectors, err := LoadSectors(store, st.Sectors)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load sectors array")

		currEpoch := rt.CurrEpoch()
		err = toProcess.ForEach(func(dlIdx uint64, pm PartitionSectorMap) error {
			targetDeadline, err := declarationDeadlineInfo(st.CurrentProvingPeriodStart(currEpoch), dlIdx, currEpoch)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "invalid recovery declaration deadline %d", dlIdx)
			err = validateFRDeclarationDeadline(targetDeadline)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "failed recovery declaration at deadline %d", dlIdx)

			deadline, err := deadlines.LoadDeadline(store, dlIdx)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadline %d", dlIdx)

			err = deadline.DeclareFaultsRecovered(store, sectors, info.SectorSize, pm)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to declare recoveries for deadline %d", dlIdx)

			err = deadlines.UpdateDeadline(store, dlIdx, deadline)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to store deadline %d", dlIdx)
			return nil
		})
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to walk sectors")

		err = st.SaveDeadlines(store, deadlines)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save deadlines")
	})

	burnFunds(rt, feeToBurn, BurnMethodDeclareFaultsRecovered)
	rt.StateReadonly(&st)
	err = st.CheckBalanceInvariants(rt.CurrentBalance())
	builtin.RequireNoErr(rt, err, ErrBalanceInvariantBroken, "balance invariants broken")

	// Power is not restored yet, but when the recovered sectors are successfully PoSted.
	return nil
}

/////////////////
// Maintenance //
/////////////////

//type CompactPartitionsParams struct {
//	Deadline   uint64
//	Partitions bitfield.BitField
//}
type CompactPartitionsParams = miner0.CompactPartitionsParams

// Compacts a number of partitions at one deadline by removing terminated sectors, re-ordering the remaining sectors,
// and assigning them to new partitions so as to completely fill all but one partition with live sectors.
// The addressed partitions are removed from the deadline, and new ones appended.
// The final partition in the deadline is always included in the compaction, whether or not explicitly requested.
// Removed sectors are removed from state entirely.
// May not be invoked if the deadline has any un-processed early terminations.
func (a Actor) CompactPartitions(rt Runtime, params *CompactPartitionsParams) *abi.EmptyValue {
	if params.Deadline >= WPoStPeriodDeadlines {
		rt.Abortf(exitcode.ErrIllegalArgument, "invalid deadline %v", params.Deadline)
	}

	partitionCount, err := params.Partitions.Count()
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "failed to parse partitions bitfield")

	store := adt.AsStore(rt)
	var st State
	rt.StateTransaction(&st, func() {
		info := getMinerInfo(rt, &st)
		rt.ValidateImmediateCallerIs(append(info.ControlAddresses, info.Owner, info.Worker)...)

		if !deadlineAvailableForCompaction(st.CurrentProvingPeriodStart(rt.CurrEpoch()), params.Deadline, rt.CurrEpoch()) {
			rt.Abortf(exitcode.ErrForbidden,
				"cannot compact deadline %d during its challenge window, or the prior challenge window, or before %d epochs have passed since its last challenge window ended", params.Deadline, WPoStDisputeWindow)
		}

		submissionPartitionLimit := loadPartitionsSectorsMax(info.WindowPoStPartitionSectors)
		if partitionCount > submissionPartitionLimit {
			rt.Abortf(exitcode.ErrIllegalArgument, "too many partitions %d, limit %d", partitionCount, submissionPartitionLimit)
		}

		quant := st.QuantSpecForDeadline(params.Deadline)

		deadlines, err := st.LoadDeadlines(store)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadlines")

		deadline, err := deadlines.LoadDeadline(store, params.Deadline)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadline %d", params.Deadline)

		live, dead, removedPower, err := deadline.RemovePartitions(store, params.Partitions, quant)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to remove partitions from deadline %d", params.Deadline)

		err = st.DeleteSectors(store, dead)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to delete dead sectors")

		sectors, err := st.LoadSectorInfos(store, live)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load moved sectors")

		proven := true
		addedPower, err := deadline.AddSectors(store, info.WindowPoStPartitionSectors, proven, sectors, info.SectorSize, quant)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to add back moved sectors")

		if !removedPower.Equals(addedPower) {
			rt.Abortf(exitcode.ErrIllegalState, "power changed when compacting partitions: was %v, is now %v", removedPower, addedPower)
		}
		err = deadlines.UpdateDeadline(store, params.Deadline, deadline)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to update deadline %d", params.Deadline)

		err = st.SaveDeadlines(store, deadlines)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save deadlines")
	})
	return nil
}

//type CompactSectorNumbersParams struct {
//	MaskSectorNumbers bitfield.BitField
//}
type CompactSectorNumbersParams = miner0.CompactSectorNumbersParams

// Compacts sector number allocations to reduce the size of the allocated sector
// number bitfield.
//
// When allocating sector numbers sequentially, or in sequential groups, this
// bitfield should remain fairly small. However, if the bitfield grows large
// enough such that PreCommitSector fails (or becomes expensive), this method
// can be called to mask out (throw away) entire ranges of unused sector IDs.
// For example, if sectors 1-99 and 101-200 have been allocated, sector number
// 99 can be masked out to collapse these two ranges into one.
func (a Actor) CompactSectorNumbers(rt Runtime, params *CompactSectorNumbersParams) *abi.EmptyValue {
	lastSectorNo, err := params.MaskSectorNumbers.Last()
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "invalid mask bitfield")
	if lastSectorNo > abi.MaxSectorNumber {
		rt.Abortf(exitcode.ErrIllegalArgument, "masked sector number %d exceeded max sector number", lastSectorNo)
	}

	store := adt.AsStore(rt)
	var st State
	rt.StateTransaction(&st, func() {
		info := getMinerInfo(rt, &st)
		rt.ValidateImmediateCallerIs(append(info.ControlAddresses, info.Owner, info.Worker)...)

		err := st.AllocateSectorNumbers(store, params.MaskSectorNumbers, AllowCollisions)

		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to mask sector numbers")
	})
	return nil
}

///////////////////////
// Pledge Collateral //
///////////////////////

// Locks up some amount of the miner's unlocked balance (including funds received alongside the invoking message).
func (a Actor) ApplyRewards(rt Runtime, params *builtin.ApplyRewardParams) *abi.EmptyValue {
	if params.Reward.Sign() < 0 {
		rt.Abortf(exitcode.ErrIllegalArgument, "cannot lock up a negative amount of funds")
	}
	if params.Penalty.Sign() < 0 {
		rt.Abortf(exitcode.ErrIllegalArgument, "cannot penalize a negative amount of funds")
	}

	var st State
	pledgeDeltaTotal := big.Zero()
	toBurn := big.Zero()
	rt.StateTransaction(&st, func() {
		var err error
		store := adt.AsStore(rt)
		rt.ValidateImmediateCallerIs(builtin.RewardActorAddr)

		rewardToLock, lockedRewardVestingSpec := LockedRewardFromReward(params.Reward)

		// This ensures the miner has sufficient funds to lock up amountToLock.
		// This should always be true if reward actor sends reward funds with the message.
		unlockedBalance, err := st.GetUnlockedBalance(rt.CurrentBalance())
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to calculate unlocked balance")
		if unlockedBalance.LessThan(rewardToLock) {
			rt.Abortf(exitcode.ErrInsufficientFunds, "insufficient funds to lock, available: %v, requested: %v", unlockedBalance, rewardToLock)
		}

		newlyVested, err := st.AddLockedFunds(store, rt.CurrEpoch(), rewardToLock, lockedRewardVestingSpec)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to lock funds in vesting table")
		pledgeDeltaTotal = big.Sub(pledgeDeltaTotal, newlyVested)
		pledgeDeltaTotal = big.Add(pledgeDeltaTotal, rewardToLock)

		// If the miner incurred block mining penalties charge these to miner's fee debt
		err = st.ApplyPenalty(params.Penalty)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to apply penalty")
		// Attempt to repay all fee debt in this call. In most cases the miner will have enough
		// funds in the *reward alone* to cover the penalty. In the rare case a miner incurs more
		// penalty than it can pay for with reward and existing funds, it will go into fee debt.
		penaltyFromVesting, penaltyFromBalance, err := st.RepayPartialDebtInPriorityOrder(store, rt.CurrEpoch(), rt.CurrentBalance())
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to repay penalty")
		pledgeDeltaTotal = big.Sub(pledgeDeltaTotal, penaltyFromVesting)
		toBurn = big.Add(penaltyFromVesting, penaltyFromBalance)
	})

	notifyPledgeChanged(rt, pledgeDeltaTotal)
	burnFunds(rt, toBurn, BurnMethodApplyRewards)
	rt.StateReadonly(&st)
	err := st.CheckBalanceInvariants(rt.CurrentBalance())
	builtin.RequireNoErr(rt, err, ErrBalanceInvariantBroken, "balance invariants broken")

	return nil
}

//type ReportConsensusFaultParams struct {
//	BlockHeader1     []byte
//	BlockHeader2     []byte
//	BlockHeaderExtra []byte
//}
type ReportConsensusFaultParams = miner0.ReportConsensusFaultParams

func (a Actor) ReportConsensusFault(rt Runtime, params *ReportConsensusFaultParams) *abi.EmptyValue {
	// Note: only the first report of any fault is processed because it sets the
	// ConsensusFaultElapsed state variable to an epoch after the fault, and reports prior to
	// that epoch are no longer valid.
	rt.ValidateImmediateCallerType(builtin.CallerTypesSignable...)
	reporter := rt.Caller()

	fault, err := rt.VerifyConsensusFault(params.BlockHeader1, params.BlockHeader2, params.BlockHeaderExtra)
	if err != nil {
		rt.Abortf(exitcode.ErrIllegalArgument, "fault not verified: %s", err)
	}
	if fault.Target != rt.Receiver() {
		rt.Abortf(exitcode.ErrIllegalArgument, "fault by %v reported to miner %v", fault.Target, rt.Receiver())
	}

	// Elapsed since the fault (i.e. since the higher of the two blocks)
	currEpoch := rt.CurrEpoch()
	faultAge := currEpoch - fault.Epoch
	if faultAge <= 0 {
		rt.Abortf(exitcode.ErrIllegalArgument, "invalid fault epoch %v ahead of current %v", fault.Epoch, currEpoch)
	}

	// Penalize miner consensus fault fee
	// Give a portion of this to the reporter as reward
	var st State
	rewardStats := requestCurrentEpochBlockReward(rt)
	// The policy amounts we should burn and send to reporter
	// These may differ from actual funds send when miner goes into fee debt
	thisEpochReward := rewardStats.ThisEpochRewardSmoothed.Estimate()
	faultPenalty := ConsensusFaultPenalty(thisEpochReward)
	slasherReward := RewardForConsensusSlashReport(thisEpochReward)
	pledgeDelta := big.Zero()

	// The amounts actually sent to burnt funds and reporter
	burnAmount := big.Zero()
	rewardAmount := big.Zero()
	rt.StateTransaction(&st, func() {
		info := getMinerInfo(rt, &st)

		// verify miner hasn't already been faulted
		if fault.Epoch < info.ConsensusFaultElapsed {
			rt.Abortf(exitcode.ErrForbidden, "fault epoch %d is too old, last exclusion period ended at %d", fault.Epoch, info.ConsensusFaultElapsed)
		}

		err := st.ApplyPenalty(faultPenalty)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to apply penalty")

		// Pay penalty
		penaltyFromVesting, penaltyFromBalance, err := st.RepayPartialDebtInPriorityOrder(adt.AsStore(rt), currEpoch, rt.CurrentBalance())
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to pay fees")
		// Burn the amount actually payable. Any difference in this and faultPenalty already recorded as FeeDebt
		burnAmount = big.Add(penaltyFromVesting, penaltyFromBalance)
		pledgeDelta = big.Add(pledgeDelta, penaltyFromVesting.Neg())

		// clamp reward at funds burnt
		rewardAmount = big.Min(burnAmount, slasherReward)
		// reduce burnAmount by rewardAmount
		burnAmount = big.Sub(burnAmount, rewardAmount)
		info.ConsensusFaultElapsed = currEpoch + ConsensusFaultIneligibilityDuration
		err = st.SaveInfo(adt.AsStore(rt), info)
		builtin.RequireNoErr(rt, err, exitcode.ErrSerialization, "failed to save miner info")
	})
	code := rt.Send(reporter, builtin.MethodSend, nil, rewardAmount, &builtin.Discard{})
	if !code.IsSuccess() {
		rt.Log(rtt.ERROR, "failed to send reward")
	}
	burnFunds(rt, burnAmount, BurnMethodReportConsensusFault)
	notifyPledgeChanged(rt, pledgeDelta)

	rt.StateReadonly(&st)
	err = st.CheckBalanceInvariants(rt.CurrentBalance())
	builtin.RequireNoErr(rt, err, ErrBalanceInvariantBroken, "balance invariants broken")

	return nil
}

//type WithdrawBalanceParams struct {
//	AmountRequested abi.TokenAmount
//}
type WithdrawBalanceParams = miner0.WithdrawBalanceParams

// Attempt to withdraw the specified amount from the miner's available balance.
// Only owner key has permission to withdraw.
// If less than the specified amount is available, yields the entire available balance.
// Returns the amount withdrawn.
func (a Actor) WithdrawBalance(rt Runtime, params *WithdrawBalanceParams) *abi.TokenAmount {
	var st State
	if params.AmountRequested.LessThan(big.Zero()) {
		rt.Abortf(exitcode.ErrIllegalArgument, "negative fund requested for withdrawal: %s", params.AmountRequested)
	}
	var info *MinerInfo
	newlyVested := big.Zero()
	feeToBurn := big.Zero()
	availableBalance := big.Zero()
	rt.StateTransaction(&st, func() {
		var err error
		info = getMinerInfo(rt, &st)
		// Only the owner is allowed to withdraw the balance as it belongs to/is controlled by the owner
		// and not the worker.
		rt.ValidateImmediateCallerIs(info.Owner)

		// Ensure we don't have any pending terminations.
		if count, err := st.EarlyTerminations.Count(); err != nil {
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to count early terminations")
		} else if count > 0 {
			rt.Abortf(exitcode.ErrForbidden,
				"cannot withdraw funds while %d deadlines have terminated sectors with outstanding fees",
				count,
			)
		}

		// Unlock vested funds so we can spend them.
		newlyVested, err = st.UnlockVestedFunds(adt.AsStore(rt), rt.CurrEpoch())
		if err != nil {
			rt.Abortf(exitcode.ErrIllegalState, "failed to vest fund: %v", err)
		}
		// available balance already accounts for fee debt so it is correct to call
		// this before RepayDebts. We would have to
		// subtract fee debt explicitly if we called this after.
		availableBalance, err = st.GetAvailableBalance(rt.CurrentBalance())
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to calculate available balance")

		// Verify unlocked funds cover both InitialPledgeRequirement and FeeDebt
		// and repay fee debt now.
		feeToBurn = RepayDebtsOrAbort(rt, &st)
	})

	amountWithdrawn := big.Min(availableBalance, params.AmountRequested)
	builtin.RequireState(rt, amountWithdrawn.GreaterThanEqual(big.Zero()), "negative amount to withdraw: %v", amountWithdrawn)
	builtin.RequireState(rt, amountWithdrawn.LessThanEqual(availableBalance), "amount to withdraw %v < available %v", amountWithdrawn, availableBalance)

	if amountWithdrawn.GreaterThan(abi.NewTokenAmount(0)) {
		code := rt.Send(info.Owner, builtin.MethodSend, nil, amountWithdrawn, &builtin.Discard{})
		builtin.RequireSuccess(rt, code, "failed to withdraw balance")
	}

	burnFunds(rt, feeToBurn, BurnMethodWithdrawBalance)

	pledgeDelta := newlyVested.Neg()
	notifyPledgeChanged(rt, pledgeDelta)

	err := st.CheckBalanceInvariants(rt.CurrentBalance())
	builtin.RequireNoErr(rt, err, ErrBalanceInvariantBroken, "balance invariants broken")

	return &amountWithdrawn
}

func (a Actor) RepayDebt(rt Runtime, _ *abi.EmptyValue) *abi.EmptyValue {
	var st State
	var fromVesting, fromBalance abi.TokenAmount
	rt.StateTransaction(&st, func() {
		var err error
		info := getMinerInfo(rt, &st)
		rt.ValidateImmediateCallerIs(append(info.ControlAddresses, info.Owner, info.Worker)...)

		// Repay as much fee debt as possible.
		fromVesting, fromBalance, err = st.RepayPartialDebtInPriorityOrder(adt.AsStore(rt), rt.CurrEpoch(), rt.CurrentBalance())
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to unlock fee debt")
	})

	notifyPledgeChanged(rt, fromVesting.Neg())
	burnFunds(rt, big.Sum(fromVesting, fromBalance), BurnMethodRepayDebt)
	err := st.CheckBalanceInvariants(rt.CurrentBalance())
	builtin.RequireNoErr(rt, err, ErrBalanceInvariantBroken, "balance invariants broken")

	return nil
}

type ReplicaUpdate struct {
	SectorID           abi.SectorNumber
	Deadline           uint64
	Partition          uint64
	NewSealedSectorCID cid.Cid `checked:"true"`
	Deals              []abi.DealID
	UpdateProofType    abi.RegisteredUpdateProof
	ReplicaProof       []byte
}

type ProveReplicaUpdatesParams struct {
	Updates []ReplicaUpdate
}

func (a Actor) ProveReplicaUpdates(rt Runtime, params *ProveReplicaUpdatesParams) bitfield.BitField {
	// Validate inputs

	builtin.RequireParam(rt, len(params.Updates) <= ProveReplicaUpdatesMaxSize, "too many updates (%d > %d)", len(params.Updates), ProveReplicaUpdatesMaxSize)

	store := adt.AsStore(rt)
	var stReadOnly State
	rt.StateReadonly(&stReadOnly)
	info := getMinerInfo(rt, &stReadOnly)

	rt.ValidateImmediateCallerIs(append(info.ControlAddresses, info.Owner, info.Worker)...)

	sectors, err := LoadSectors(store, stReadOnly.Sectors)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load sectors array")

	powerDelta := NewPowerPairZero()
	pledgeDelta := big.Zero()

	type updateAndSectorInfo struct {
		update     *ReplicaUpdate
		sectorInfo *SectorOnChainInfo
	}

	var sectorsDeals []market.SectorDeals
	var sectorsDataSpec []*market.SectorDataSpec
	var validatedUpdates []*updateAndSectorInfo
	sectorNumbers := bitfield.New()
	for i := range params.Updates {
		update := params.Updates[i]
		// Bitfied.IsSet() is fast when there are only locally-set values.
		set, err := sectorNumbers.IsSet(uint64(update.SectorID))
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "error checking sector number")
		if set {
			rt.Log(rtt.INFO, "duplicate sector being updated %d, skipping", update.SectorID)
			continue
		}

		sectorNumbers.Set(uint64(update.SectorID))

		if len(update.ReplicaProof) > 4096 {
			rt.Log(rtt.INFO, "update proof is too large (%d), skipping sector %d", len(update.ReplicaProof), update.SectorID)
			continue
		}

		if len(update.Deals) <= 0 {
			rt.Log(rtt.INFO, "must have deals to update, skipping sector %d", update.SectorID)
			continue
		}

		if uint64(len(update.Deals)) > SectorDealsMax(info.SectorSize) {
			rt.Log(rtt.INFO, "more deals than policy allows, skipping sector %d", update.SectorID)
			continue
		}

		if update.Deadline >= WPoStPeriodDeadlines {
			rt.Log(rtt.INFO, "deadline %d not in range 0..%d, skipping sector %d", update.Deadline, WPoStPeriodDeadlines, update.SectorID)
			continue
		}

		if !update.NewSealedSectorCID.Defined() {
			rt.Log(rtt.INFO, "new sealed CID undefined, skipping sector %d", update.SectorID)
			continue
		}

		if update.NewSealedSectorCID.Prefix() != SealedCIDPrefix {
			rt.Log(rtt.INFO, "new sealed CID had wrong prefix %s, skipping sector %d", update.NewSealedSectorCID, update.SectorID)
			continue
		}

		healthy, err := stReadOnly.CheckSectorHealthExcludeUnproven(store, update.Deadline, update.Partition, update.SectorID)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "error checking sector health")

		if !healthy {
			rt.Log(rtt.INFO, "sector isn't healthy, skipping sector %d", update.SectorID)
			continue
		}

		sectorInfo, err := sectors.MustGet(update.SectorID)
		if err != nil {
			rt.Log(rtt.INFO, "failed to get sector, skipping sector %d", update.SectorID)
			continue
		}

		if len(sectorInfo.DealIDs) != 0 {
			rt.Log(rtt.INFO, "cannot update sector with deals, skipping sector %d", update.SectorID)
			continue
		}

		code := rt.Send(
			builtin.StorageMarketActorAddr,
			builtin.MethodsMarket.ActivateDeals,
			&market.ActivateDealsParams{
				DealIDs:      update.Deals,
				SectorExpiry: sectorInfo.Expiration,
			},
			abi.NewTokenAmount(0),
			&builtin.Discard{},
		)

		if code != exitcode.Ok {
			rt.Log(rtt.INFO, "failed to activate deals, skipping sector %d", update.SectorID)
			continue
		}

		validatedUpdates = append(validatedUpdates, &updateAndSectorInfo{
			update:     &update,
			sectorInfo: sectorInfo,
		})

		sectorsDeals = append(sectorsDeals, market.SectorDeals{DealIDs: update.Deals, SectorExpiry: sectorInfo.Expiration})
		sectorsDataSpec = append(sectorsDataSpec, &market.SectorDataSpec{
			SectorType: sectorInfo.SealProof,
			DealIDs:    update.Deals,
		})
	}

	builtin.RequireParam(rt, len(validatedUpdates) > 0, "no valid updates")

	// Errors past this point cause the ProveReplicaUpdates call to fail (no more skipping sectors)

	dealWeights := requestDealWeights(rt, sectorsDeals)
	builtin.RequirePredicate(rt, len(dealWeights.Sectors) == len(validatedUpdates), exitcode.ErrIllegalState,
		"deal weight request returned %d records, expected %d", len(dealWeights.Sectors), len(validatedUpdates))

	unsealedSectorCIDs := requestUnsealedSectorCIDs(rt, sectorsDataSpec...)
	builtin.RequirePredicate(rt, len(unsealedSectorCIDs) == len(validatedUpdates), exitcode.ErrIllegalState,
		"unsealed sector cid request returned %d records, expected %d", len(unsealedSectorCIDs), len(validatedUpdates))

	type updateWithDetails struct {
		update            *ReplicaUpdate
		sectorInfo        *SectorOnChainInfo
		dealWeight        market.SectorWeights
		unsealedSectorCID cid.Cid
	}

	// Group declarations by deadline
	declsByDeadline := map[uint64][]*updateWithDetails{}
	var deadlinesToLoad []uint64
	for i, updateWithSectorInfo := range validatedUpdates {
		if _, ok := declsByDeadline[updateWithSectorInfo.update.Deadline]; !ok {
			deadlinesToLoad = append(deadlinesToLoad, updateWithSectorInfo.update.Deadline)
		}
		declsByDeadline[updateWithSectorInfo.update.Deadline] = append(declsByDeadline[updateWithSectorInfo.update.Deadline], &updateWithDetails{
			update:            updateWithSectorInfo.update,
			sectorInfo:        updateWithSectorInfo.sectorInfo,
			dealWeight:        dealWeights.Sectors[i],
			unsealedSectorCID: unsealedSectorCIDs[i],
		})
	}

	rewRet := requestCurrentEpochBlockReward(rt)
	powRet := requestCurrentTotalPower(rt)

	succeededSectors := bitfield.New()
	var st State
	rt.StateTransaction(&st, func() {
		deadlines, err := st.LoadDeadlines(store)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadlines")

		newSectors := make([]*SectorOnChainInfo, len(validatedUpdates))
		for _, dlIdx := range deadlinesToLoad {
			deadline, err := deadlines.LoadDeadline(store, dlIdx)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadline %d", dlIdx)

			partitions, err := deadline.PartitionsArray(store)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load partitions for deadline %d", dlIdx)

			quant := st.QuantSpecForDeadline(dlIdx)

			for i, updateWithDetails := range declsByDeadline[dlIdx] {
				updateProofType, err := updateWithDetails.sectorInfo.SealProof.RegisteredUpdateProof()
				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "couldn't load update proof type")
				builtin.RequirePredicate(rt, updateWithDetails.update.UpdateProofType == updateProofType, exitcode.ErrIllegalArgument, "unsupported update proof type %d", updateWithDetails.update.UpdateProofType)

				err = rt.VerifyReplicaUpdate(
					proof.ReplicaUpdateInfo{
						UpdateProofType:      updateProofType,
						NewSealedSectorCID:   updateWithDetails.update.NewSealedSectorCID,
						OldSealedSectorCID:   updateWithDetails.sectorInfo.SealedCID,
						NewUnsealedSectorCID: updateWithDetails.unsealedSectorCID,
						Proof:                updateWithDetails.update.ReplicaProof,
					})

				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "failed to verify replica proof for sector %d", updateWithDetails.sectorInfo.SectorNumber)

				newSectorInfo := *updateWithDetails.sectorInfo

				newSectorInfo.SealedCID = updateWithDetails.update.NewSealedSectorCID
				if newSectorInfo.SectorKeyCID == nil {
					newSectorInfo.SectorKeyCID = &updateWithDetails.sectorInfo.SealedCID
				}

				newSectorInfo.DealIDs = updateWithDetails.update.Deals
				newSectorInfo.Activation = rt.CurrEpoch()

				newSectorInfo.DealWeight = updateWithDetails.dealWeight.DealWeight
				newSectorInfo.VerifiedDealWeight = updateWithDetails.dealWeight.VerifiedDealWeight

				// compute initial pledge
				duration := updateWithDetails.sectorInfo.Expiration - rt.CurrEpoch()

				pwr := QAPowerForWeight(info.SectorSize, duration, newSectorInfo.DealWeight, newSectorInfo.VerifiedDealWeight)

				newSectorInfo.ReplacedDayReward = updateWithDetails.sectorInfo.ExpectedDayReward
				newSectorInfo.ExpectedDayReward = ExpectedRewardForPower(rewRet.ThisEpochRewardSmoothed, powRet.QualityAdjPowerSmoothed, pwr, builtin.EpochsInDay)
				newSectorInfo.ExpectedStoragePledge = ExpectedRewardForPower(rewRet.ThisEpochRewardSmoothed, powRet.QualityAdjPowerSmoothed, pwr, InitialPledgeProjectionPeriod)
				newSectorInfo.ReplacedSectorAge = maxEpoch(0, rt.CurrEpoch()-updateWithDetails.sectorInfo.Activation)

				initialPledgeAtUpgrade := InitialPledgeForPower(pwr, rewRet.ThisEpochBaselinePower, rewRet.ThisEpochRewardSmoothed,
					powRet.QualityAdjPowerSmoothed, rt.TotalFilCircSupply())

				if initialPledgeAtUpgrade.GreaterThan(updateWithDetails.sectorInfo.InitialPledge) {
					deficit := big.Sub(initialPledgeAtUpgrade, updateWithDetails.sectorInfo.InitialPledge)

					unlockedBalance, err := st.GetUnlockedBalance(rt.CurrentBalance())
					builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to calculate unlocked balance")
					builtin.RequirePredicate(rt, unlockedBalance.GreaterThanEqual(deficit), exitcode.ErrInsufficientFunds, "insufficient funds for new initial pledge requirement %s, available: %s, skipping sector %d",
						deficit, unlockedBalance, updateWithDetails.sectorInfo.SectorNumber)

					err = st.AddInitialPledge(deficit)
					builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to add initial pledge")

					newSectorInfo.InitialPledge = initialPledgeAtUpgrade
				}

				var partition Partition
				found, err := partitions.Get(updateWithDetails.update.Partition, &partition)

				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load deadline %v partition %v",
					updateWithDetails.update.Deadline, updateWithDetails.update.Partition)

				if !found {
					rt.Abortf(exitcode.ErrNotFound, "no such deadline %v partition %v", dlIdx, updateWithDetails.update.Partition)
				}

				partitionPowerDelta, partitionPledgeDelta, err := partition.ReplaceSectors(store,
					[]*SectorOnChainInfo{updateWithDetails.sectorInfo},
					[]*SectorOnChainInfo{&newSectorInfo},
					info.SectorSize,
					quant)

				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to replace sector at deadline %d partition %d", updateWithDetails.update.Deadline, updateWithDetails.update.Partition)

				powerDelta = powerDelta.Add(partitionPowerDelta)
				pledgeDelta = big.Add(pledgeDelta, partitionPledgeDelta)

				err = partitions.Set(updateWithDetails.update.Partition, &partition)
				builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save deadline %v partition %v",
					updateWithDetails.update.Deadline,
					updateWithDetails.update.Partition)

				newSectors[i] = &newSectorInfo
				succeededSectors.Set(uint64(newSectorInfo.SectorNumber))
			}

			deadline.Partitions, err = partitions.Root()
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save partitions for deadline %d", dlIdx)

			err = deadlines.UpdateDeadline(store, dlIdx, deadline)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save deadline %d", dlIdx)
		}

		successCount, err := succeededSectors.Count()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to count succeededSectors")
		builtin.RequirePredicate(rt, successCount == uint64(len(validatedUpdates)), exitcode.ErrIllegalState, "unexpected successcount %d != %d", successCount, len(validatedUpdates))

		// Overwrite sector infos.
		err = sectors.Store(newSectors...)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to update sector infos")

		st.Sectors, err = sectors.Root()
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save sectors")

		err = st.SaveDeadlines(store, deadlines)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to save deadlines")

	})

	notifyPledgeChanged(rt, pledgeDelta)
	requestUpdatePower(rt, powerDelta)

	return succeededSectors
}

//////////
// Cron //
//////////

//type CronEventPayload struct {
//	EventType CronEventType
//}
type CronEventPayload = miner0.CronEventPayload

type CronEventType = miner0.CronEventType

const (
	CronEventProvingDeadline          = miner0.CronEventProvingDeadline
	CronEventProcessEarlyTerminations = miner0.CronEventProcessEarlyTerminations
)

func (a Actor) OnDeferredCronEvent(rt Runtime, params *builtin.DeferredCronEventParams) *abi.EmptyValue {
	rt.ValidateImmediateCallerIs(builtin.StoragePowerActorAddr)

	var payload miner0.CronEventPayload
	err := payload.UnmarshalCBOR(bytes.NewBuffer(params.EventPayload))
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to unmarshal miner cron payload into expected structure")

	switch payload.EventType {
	case CronEventProvingDeadline:
		handleProvingDeadline(rt, params.RewardSmoothed, params.QualityAdjPowerSmoothed)
	case CronEventProcessEarlyTerminations:
		if processEarlyTerminations(rt, params.RewardSmoothed, params.QualityAdjPowerSmoothed) {
			scheduleEarlyTerminationWork(rt)
		}
	default:
		rt.Log(rtt.ERROR, "onDeferredCronEvent invalid event type: %v", payload.EventType)
	}

	var st State
	rt.StateReadonly(&st)
	err = st.CheckBalanceInvariants(rt.CurrentBalance())
	builtin.RequireNoErr(rt, err, ErrBalanceInvariantBroken, "balance invariants broken")
	return nil
}

////////////////////////////////////////////////////////////////////////////////
// Utility functions & helpers
////////////////////////////////////////////////////////////////////////////////

// TODO: We're using the current power+epoch reward. Technically, we
// should use the power/reward at the time of termination.
// https://github.com/filecoin-project/specs-actors/v7/pull/648
func processEarlyTerminations(rt Runtime, rewardSmoothed smoothing.FilterEstimate, qualityAdjPowerSmoothed smoothing.FilterEstimate) (more bool) {
	store := adt.AsStore(rt)

	var (
		result           TerminationResult
		dealsToTerminate []market.OnMinerSectorsTerminateParams
		penalty          = big.Zero()
		pledgeDelta      = big.Zero()
	)

	var st State
	rt.StateTransaction(&st, func() {
		var err error
		result, more, err = st.PopEarlyTerminations(store, AddressedPartitionsMax, AddressedSectorsMax)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to pop early terminations")

		// Nothing to do, don't waste any time.
		// This can happen if we end up processing early terminations
		// before the cron callback fires.
		if result.IsEmpty() {
			rt.Log(rtt.INFO, "no early terminations (maybe cron callback hasn't happened yet?)")
			return
		}

		info := getMinerInfo(rt, &st)

		sectors, err := LoadSectors(store, st.Sectors)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load sectors array")

		totalInitialPledge := big.Zero()
		dealsToTerminate = make([]market.OnMinerSectorsTerminateParams, 0, len(result.Sectors))
		err = result.ForEach(func(epoch abi.ChainEpoch, sectorNos bitfield.BitField) error {
			sectors, err := sectors.Load(sectorNos)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load sector infos")
			params := market.OnMinerSectorsTerminateParams{
				Epoch:   epoch,
				DealIDs: make([]abi.DealID, 0, len(sectors)), // estimate ~one deal per sector.
			}
			for _, sector := range sectors {
				params.DealIDs = append(params.DealIDs, sector.DealIDs...)
				totalInitialPledge = big.Add(totalInitialPledge, sector.InitialPledge)
			}
			penalty = big.Add(penalty, terminationPenalty(info.SectorSize, epoch,
				rewardSmoothed, qualityAdjPowerSmoothed, sectors))
			dealsToTerminate = append(dealsToTerminate, params)

			return nil
		})
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to process terminations")

		// Pay penalty
		err = st.ApplyPenalty(penalty)
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to apply penalty")

		// Remove pledge requirement.
		err = st.AddInitialPledge(totalInitialPledge.Neg())
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to add initial pledge %v", totalInitialPledge.Neg())
		pledgeDelta = big.Sub(pledgeDelta, totalInitialPledge)

		// Use unlocked pledge to pay down outstanding fee debt
		penaltyFromVesting, penaltyFromBalance, err := st.RepayPartialDebtInPriorityOrder(store, rt.CurrEpoch(), rt.CurrentBalance())
		builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to pay penalty")
		penalty = big.Add(penaltyFromVesting, penaltyFromBalance)
		pledgeDelta = big.Sub(pledgeDelta, penaltyFromVesting)
	})

	// We didn't do anything, abort.
	if result.IsEmpty() {
		rt.Log(rtt.INFO, "no early terminations")
		return more
	}

	// Burn penalty.
	rt.Log(rtt.DEBUG, "storage provider %s penalized %s for sector termination", rt.Receiver(), penalty)
	burnFunds(rt, penalty, BurnMethodProcessEarlyTerminations)

	// Return pledge.
	notifyPledgeChanged(rt, pledgeDelta)

	// Terminate deals.
	for _, params := range dealsToTerminate {
		requestTerminateDeals(rt, params.Epoch, params.DealIDs)
	}

	// reschedule cron worker, if necessary.
	return more
}

// Invoked at the end of the last epoch for each proving deadline.
func handleProvingDeadline(rt Runtime,
	rewardSmoothed smoothing.FilterEstimate,
	qualityAdjPowerSmoothed smoothing.FilterEstimate) {
	currEpoch := rt.CurrEpoch()
	store := adt.AsStore(rt)

	hadEarlyTerminations := false

	powerDeltaTotal := NewPowerPairZero()
	penaltyTotal := abi.NewTokenAmount(0)
	pledgeDeltaTotal := abi.NewTokenAmount(0)

	var continueCron bool
	var st State
	rt.StateTransaction(&st, func() {
		{
			// Vest locked funds.
			// This happens first so that any subsequent penalties are taken
			// from locked vesting funds before funds free this epoch.
			newlyVested, err := st.UnlockVestedFunds(store, rt.CurrEpoch())
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to vest funds")
			pledgeDeltaTotal = big.Add(pledgeDeltaTotal, newlyVested.Neg())
		}

		{
			// Process pending worker change if any
			info := getMinerInfo(rt, &st)
			processPendingWorker(info, rt, &st)
		}

		{
			depositToBurn, err := st.CleanUpExpiredPreCommits(store, currEpoch)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to expire pre-committed sectors")

			err = st.ApplyPenalty(depositToBurn)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to apply penalty")
			rt.Log(rtt.DEBUG, "storage provider %s penalized %s for expired pre commits", rt.Receiver(), depositToBurn)
		}

		// Record whether or not we _had_ early terminations in the queue before this method.
		// That way, don't re-schedule a cron callback if one is already scheduled.
		hadEarlyTerminations = havePendingEarlyTerminations(rt, &st)

		{
			result, err := st.AdvanceDeadline(store, currEpoch)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to advance deadline")

			// Faults detected by this missed PoSt pay no penalty, but sectors that were already faulty
			// and remain faulty through this deadline pay the fault fee.
			penaltyTarget := PledgePenaltyForContinuedFault(
				rewardSmoothed,
				qualityAdjPowerSmoothed,
				result.PreviouslyFaultyPower.QA,
			)

			powerDeltaTotal = powerDeltaTotal.Add(result.PowerDelta)
			pledgeDeltaTotal = big.Add(pledgeDeltaTotal, result.PledgeDelta)

			err = st.ApplyPenalty(penaltyTarget)
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to apply penalty")
			rt.Log(rtt.DEBUG, "storage provider %s penalized %s for continued fault", rt.Receiver(), penaltyTarget)

			penaltyFromVesting, penaltyFromBalance, err := st.RepayPartialDebtInPriorityOrder(store, currEpoch, rt.CurrentBalance())
			builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to unlock penalty")
			penaltyTotal = big.Add(penaltyFromVesting, penaltyFromBalance)
			pledgeDeltaTotal = big.Sub(pledgeDeltaTotal, penaltyFromVesting)
		}

		continueCron = st.ContinueDeadlineCron()
		if !continueCron {
			st.DeadlineCronActive = false
		}
	})
	// Remove power for new faults, and burn penalties.
	requestUpdatePower(rt, powerDeltaTotal)
	burnFunds(rt, penaltyTotal, BurnMethodHandleProvingDeadline)
	notifyPledgeChanged(rt, pledgeDeltaTotal)

	// Schedule cron callback for next deadline's last epoch.
	if continueCron {
		newDlInfo := st.DeadlineInfo(currEpoch + 1)
		enrollCronEvent(rt, newDlInfo.Last(), &CronEventPayload{
			EventType: CronEventProvingDeadline,
		})
	} else {
		rt.Log(rtt.INFO, "miner %s going inactive, deadline cron discontinued", rt.Receiver())
	}

	// Record whether or not we _have_ early terminations now.
	hasEarlyTerminations := havePendingEarlyTerminations(rt, &st)

	// If we didn't have pending early terminations before, but we do now,
	// handle them at the next epoch.
	if !hadEarlyTerminations && hasEarlyTerminations {
		// First, try to process some of these terminations.
		if processEarlyTerminations(rt, rewardSmoothed, qualityAdjPowerSmoothed) {
			// If that doesn't work, just defer till the next epoch.
			scheduleEarlyTerminationWork(rt)
		}
		// Note: _don't_ process early terminations if we had a cron
		// callback already scheduled. In that case, we'll already have
		// processed AddressedSectorsMax terminations this epoch.
	}
}

// Check expiry is exactly *the epoch before* the start of a proving period.
func validateExpiration(rt Runtime, activation, expiration abi.ChainEpoch, sealProof abi.RegisteredSealProof) {
	// Expiration must be after activation. Check this explicitly to avoid an underflow below.
	if expiration <= activation {
		rt.Abortf(exitcode.ErrIllegalArgument, "sector expiration %v must be after activation (%v)", expiration, activation)
	}
	// expiration cannot be less than minimum after activation
	if expiration-activation < MinSectorExpiration {
		rt.Abortf(exitcode.ErrIllegalArgument, "invalid expiration %d, total sector lifetime (%d) must exceed %d after activation %d",
			expiration, expiration-activation, MinSectorExpiration, activation)
	}

	// expiration cannot exceed MaxSectorExpirationExtension from now
	if expiration > rt.CurrEpoch()+MaxSectorExpirationExtension {
		rt.Abortf(exitcode.ErrIllegalArgument, "invalid expiration %d, cannot be more than %d past current epoch %d",
			expiration, MaxSectorExpirationExtension, rt.CurrEpoch())
	}

	// total sector lifetime cannot exceed SectorMaximumLifetime for the sector's seal proof
	maxLifetime, err := builtin.SealProofSectorMaximumLifetime(sealProof)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "unrecognized seal proof type %d", sealProof)
	if expiration-activation > maxLifetime {
		rt.Abortf(exitcode.ErrIllegalArgument, "invalid expiration %d, total sector lifetime (%d) cannot exceed %d after activation %d",
			expiration, expiration-activation, maxLifetime, activation)
	}
}

func validateReplaceSector(rt Runtime, st *State, store adt.Store, params *miner0.SectorPreCommitInfo) {
	replaceSector, found, err := st.GetSector(store, params.ReplaceSectorNumber)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to load sector %v", params.SectorNumber)
	if !found {
		rt.Abortf(exitcode.ErrNotFound, "no such sector %v to replace", params.ReplaceSectorNumber)
	}

	if len(replaceSector.DealIDs) > 0 {
		rt.Abortf(exitcode.ErrIllegalArgument, "cannot replace sector %v which has deals", params.ReplaceSectorNumber)
	}
	// From network version 7, the new sector's seal type must have the same Window PoSt proof type as the one
	// being replaced, rather than be exactly the same seal type.
	// This permits replacing sectors with V1 seal types with V1_1 seal types.
	replaceWPoStProof, err := replaceSector.SealProof.RegisteredWindowPoStProof()
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to lookup Window PoSt proof type for sector seal proof %d", replaceSector.SealProof)
	newWPoStProof, err := params.SealProof.RegisteredWindowPoStProof()
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalArgument, "failed to lookup Window PoSt proof type for new seal proof %d", params.SealProof)
	if newWPoStProof != replaceWPoStProof {
		rt.Abortf(exitcode.ErrIllegalArgument, "new sector window PoSt proof type %d must match replaced proof type %d (seal proof type %d)",
			newWPoStProof, replaceWPoStProof, params.SealProof)
	}
	if params.Expiration < replaceSector.Expiration {
		rt.Abortf(exitcode.ErrIllegalArgument, "cannot replace sector %v expiration %v with sooner expiration %v",
			params.ReplaceSectorNumber, replaceSector.Expiration, params.Expiration)
	}

	err = st.CheckSectorHealth(store, params.ReplaceSectorDeadline, params.ReplaceSectorPartition, params.ReplaceSectorNumber)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to replace sector %v", params.ReplaceSectorNumber)
}

func enrollCronEvent(rt Runtime, eventEpoch abi.ChainEpoch, callbackPayload *CronEventPayload) {
	payload := new(bytes.Buffer)
	err := callbackPayload.MarshalCBOR(payload)
	if err != nil {
		rt.Abortf(exitcode.ErrIllegalArgument, "failed to serialize payload: %v", err)
	}
	code := rt.Send(
		builtin.StoragePowerActorAddr,
		builtin.MethodsPower.EnrollCronEvent,
		&power.EnrollCronEventParams{
			EventEpoch: eventEpoch,
			Payload:    payload.Bytes(),
		},
		abi.NewTokenAmount(0),
		&builtin.Discard{},
	)
	builtin.RequireSuccess(rt, code, "failed to enroll cron event")
}

func requestUpdatePower(rt Runtime, delta PowerPair) {
	if delta.IsZero() {
		return
	}
	code := rt.Send(
		builtin.StoragePowerActorAddr,
		builtin.MethodsPower.UpdateClaimedPower,
		&power.UpdateClaimedPowerParams{
			RawByteDelta:         delta.Raw,
			QualityAdjustedDelta: delta.QA,
		},
		abi.NewTokenAmount(0),
		&builtin.Discard{},
	)
	builtin.RequireSuccess(rt, code, "failed to update power with %v", delta)
}

func requestTerminateDeals(rt Runtime, epoch abi.ChainEpoch, dealIDs []abi.DealID) {
	for len(dealIDs) > 0 {
		size := min64(cbg.MaxLength, uint64(len(dealIDs)))
		code := rt.Send(
			builtin.StorageMarketActorAddr,
			builtin.MethodsMarket.OnMinerSectorsTerminate,
			&market.OnMinerSectorsTerminateParams{
				Epoch:   epoch,
				DealIDs: dealIDs[:size],
			},
			abi.NewTokenAmount(0),
			&builtin.Discard{},
		)
		builtin.RequireSuccess(rt, code, "failed to terminate deals, exit code %v", code)
		dealIDs = dealIDs[size:]
	}
}

func scheduleEarlyTerminationWork(rt Runtime) {
	rt.Log(rtt.INFO, "scheduling early terminations with cron...")

	enrollCronEvent(rt, rt.CurrEpoch()+1, &CronEventPayload{
		EventType: CronEventProcessEarlyTerminations,
	})
}

func havePendingEarlyTerminations(rt Runtime, st *State) bool {
	// Record this up-front
	noEarlyTerminations, err := st.EarlyTerminations.IsEmpty()
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "failed to count early terminations")
	return !noEarlyTerminations
}

func verifyWindowedPost(rt Runtime, challengeEpoch abi.ChainEpoch, sectors []*SectorOnChainInfo, proofs []proof.PoStProof) error {
	minerActorID, err := addr.IDFromAddress(rt.Receiver())
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "runtime provided bad receiver address %v", rt.Receiver())

	// Regenerate challenge randomness, which must match that generated for the proof.
	var addrBuf bytes.Buffer
	receiver := rt.Receiver()
	err = receiver.MarshalCBOR(&addrBuf)
	builtin.RequireNoErr(rt, err, exitcode.ErrSerialization, "failed to marshal address for window post challenge")
	postRandomness := rt.GetRandomnessFromBeacon(crypto.DomainSeparationTag_WindowedPoStChallengeSeed, challengeEpoch, addrBuf.Bytes())

	sectorProofInfo := make([]proof.SectorInfo, len(sectors))
	for i, s := range sectors {
		sectorProofInfo[i] = proof.SectorInfo{
			SealProof:    s.SealProof,
			SectorNumber: s.SectorNumber,
			SealedCID:    s.SealedCID,
		}
	}

	// Get public inputs
	pvInfo := proof.WindowPoStVerifyInfo{
		Randomness:        abi.PoStRandomness(postRandomness),
		Proofs:            proofs,
		ChallengedSectors: sectorProofInfo,
		Prover:            abi.ActorID(minerActorID),
	}

	// Verify the PoSt ReplicaProof
	err = rt.VerifyPoSt(pvInfo)
	if err != nil {
		return fmt.Errorf("invalid PoSt %+v: %w", pvInfo, err)
	}
	return nil
}

// SealVerifyParams is the structure of information that must be sent with a
// message to commit a sector. Most of this information is not needed in the
// state tree but will be verified in sm.CommitSector. See SealCommitment for
// data stored on the state tree for each sector.
type SealVerifyStuff struct {
	SealedCID        cid.Cid        // CommR
	InteractiveEpoch abi.ChainEpoch // Used to derive the interactive PoRep challenge.
	abi.RegisteredSealProof
	Proof   []byte
	DealIDs []abi.DealID
	abi.SectorNumber
	SealRandEpoch abi.ChainEpoch // Used to tie the seal to a chain.
}

func getVerifyInfo(rt Runtime, params *SealVerifyStuff) *proof.SealVerifyInfo {
	if rt.CurrEpoch() <= params.InteractiveEpoch {
		rt.Abortf(exitcode.ErrForbidden, "too early to prove sector")
	}

	commDs := requestUnsealedSectorCIDs(rt, &market.SectorDataSpec{
		SectorType: params.RegisteredSealProof,
		DealIDs:    params.DealIDs,
	})

	minerActorID, err := addr.IDFromAddress(rt.Receiver())
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "runtime provided non-ID receiver address %v", rt.Receiver())

	buf := new(bytes.Buffer)
	receiver := rt.Receiver()
	err = receiver.MarshalCBOR(buf)
	builtin.RequireNoErr(rt, err, exitcode.ErrSerialization, "failed to marshal address for seal verification challenge")

	svInfoRandomness := rt.GetRandomnessFromTickets(crypto.DomainSeparationTag_SealRandomness, params.SealRandEpoch, buf.Bytes())
	svInfoInteractiveRandomness := rt.GetRandomnessFromBeacon(crypto.DomainSeparationTag_InteractiveSealChallengeSeed, params.InteractiveEpoch, buf.Bytes())

	return &proof.SealVerifyInfo{
		SealProof: params.RegisteredSealProof,
		SectorID: abi.SectorID{
			Miner:  abi.ActorID(minerActorID),
			Number: params.SectorNumber,
		},
		DealIDs:               params.DealIDs,
		InteractiveRandomness: abi.InteractiveSealRandomness(svInfoInteractiveRandomness),
		Proof:                 params.Proof,
		Randomness:            abi.SealRandomness(svInfoRandomness),
		SealedCID:             params.SealedCID,
		UnsealedCID:           commDs[0],
	}
}

// Requests the storage market actor compute the unsealed sector CID from a sector's deals.
func requestUnsealedSectorCIDs(rt Runtime, dataCommitmentInputs ...*market.SectorDataSpec) []cid.Cid {
	if len(dataCommitmentInputs) == 0 {
		return nil
	}
	var ret market.ComputeDataCommitmentReturn
	code := rt.Send(
		builtin.StorageMarketActorAddr,
		builtin.MethodsMarket.ComputeDataCommitment,
		&market.ComputeDataCommitmentParams{
			Inputs: dataCommitmentInputs,
		},
		abi.NewTokenAmount(0),
		&ret,
	)
	builtin.RequireSuccess(rt, code, "failed request for unsealed sector CIDs")
	builtin.RequireState(rt, len(dataCommitmentInputs) == len(ret.CommDs), "number of data commitments computed %d does not match number of data commitment inputs %d", len(ret.CommDs), len(dataCommitmentInputs))
	unsealedCIDs := make([]cid.Cid, len(ret.CommDs))
	for i, cbgCid := range ret.CommDs {
		unsealedCIDs[i] = cid.Cid(cbgCid)
	}
	return unsealedCIDs
}

func requestDealWeights(rt Runtime, sectors []market.SectorDeals) *market.VerifyDealsForActivationReturn {
	// Short-circuit if there are no deals in any of the sectors.
	dealCount := 0
	for _, sector := range sectors {
		dealCount += len(sector.DealIDs)
	}
	if dealCount == 0 {
		emptyResult := &market.VerifyDealsForActivationReturn{
			Sectors: make([]market.SectorWeights, len(sectors)),
		}
		for i := 0; i < len(sectors); i++ {
			emptyResult.Sectors[i] = market.SectorWeights{
				DealSpace:          0,
				DealWeight:         big.Zero(),
				VerifiedDealWeight: big.Zero(),
			}
		}
		return emptyResult
	}

	var dealWeights market.VerifyDealsForActivationReturn
	code := rt.Send(
		builtin.StorageMarketActorAddr,
		builtin.MethodsMarket.VerifyDealsForActivation,
		&market.VerifyDealsForActivationParams{
			Sectors: sectors,
		},
		abi.NewTokenAmount(0),
		&dealWeights,
	)
	builtin.RequireSuccess(rt, code, "failed to verify deals and get deal weight")
	return &dealWeights
}

// Requests the current epoch target block reward from the reward actor.
// return value includes reward, smoothed estimate of reward, and baseline power
func requestCurrentEpochBlockReward(rt Runtime) reward.ThisEpochRewardReturn {
	var ret reward.ThisEpochRewardReturn
	code := rt.Send(builtin.RewardActorAddr, builtin.MethodsReward.ThisEpochReward, nil, big.Zero(), &ret)
	builtin.RequireSuccess(rt, code, "failed to check epoch baseline power")
	return ret
}

// Requests the current network total power and pledge from the power actor.
func requestCurrentTotalPower(rt Runtime) *power.CurrentTotalPowerReturn {
	var pwr power.CurrentTotalPowerReturn
	code := rt.Send(builtin.StoragePowerActorAddr, builtin.MethodsPower.CurrentTotalPower, nil, big.Zero(), &pwr)
	builtin.RequireSuccess(rt, code, "failed to check current power")
	return &pwr
}

// Resolves an address to an ID address and verifies that it is address of an account or multisig actor.
func resolveControlAddress(rt Runtime, raw addr.Address) addr.Address {
	resolved, ok := rt.ResolveAddress(raw)
	if !ok {
		rt.Abortf(exitcode.ErrIllegalArgument, "unable to resolve address %v", raw)
	}
	ownerCode, ok := rt.GetActorCodeCID(resolved)
	if !ok {
		rt.Abortf(exitcode.ErrIllegalArgument, "no code for address %v", resolved)
	}
	if !builtin.IsPrincipal(ownerCode) {
		rt.Abortf(exitcode.ErrIllegalArgument, "owner actor type must be a principal, was %v", ownerCode)
	}
	return resolved
}

// Resolves an address to an ID address and verifies that it is address of an account actor with an associated BLS key.
// The worker must be BLS since the worker key will be used alongside a BLS-VRF.
func resolveWorkerAddress(rt Runtime, raw addr.Address) addr.Address {
	resolved, ok := rt.ResolveAddress(raw)
	if !ok {
		rt.Abortf(exitcode.ErrIllegalArgument, "unable to resolve address %v", raw)
	}
	workerCode, ok := rt.GetActorCodeCID(resolved)
	if !ok {
		rt.Abortf(exitcode.ErrIllegalArgument, "no code for address %v", resolved)
	}
	if workerCode != builtin.AccountActorCodeID {
		rt.Abortf(exitcode.ErrIllegalArgument, "worker actor type must be an account, was %v", workerCode)
	}

	if raw.Protocol() != addr.BLS {
		var pubkey addr.Address
		code := rt.Send(resolved, builtin.MethodsAccount.PubkeyAddress, nil, big.Zero(), &pubkey)
		builtin.RequireSuccess(rt, code, "failed to fetch account pubkey from %v", resolved)
		if pubkey.Protocol() != addr.BLS {
			rt.Abortf(exitcode.ErrIllegalArgument, "worker account %v must have BLS pubkey, was %v", resolved, pubkey.Protocol())
		}
	}
	return resolved
}

func burnFunds(rt Runtime, amt abi.TokenAmount, bt BurnMethod) {
	if amt.GreaterThan(big.Zero()) {
		rt.Log(rtt.DEBUG, "storage provder %s burn type %s burning %s", rt.Receiver(), bt, amt)
		code := rt.Send(builtin.BurntFundsActorAddr, builtin.MethodSend, nil, amt, &builtin.Discard{})
		builtin.RequireSuccess(rt, code, "failed to burn funds")
	}
}

func notifyPledgeChanged(rt Runtime, pledgeDelta abi.TokenAmount) {
	if !pledgeDelta.IsZero() {
		code := rt.Send(builtin.StoragePowerActorAddr, builtin.MethodsPower.UpdatePledgeTotal, &pledgeDelta, big.Zero(), &builtin.Discard{})
		builtin.RequireSuccess(rt, code, "failed to update total pledge")
	}
}

// Assigns proving period offset randomly in the range [0, WPoStProvingPeriod) by hashing
// the actor's address and current epoch.
func assignProvingPeriodOffset(myAddr addr.Address, currEpoch abi.ChainEpoch, hash func(data []byte) [32]byte) (abi.ChainEpoch, error) {
	offsetSeed := bytes.Buffer{}
	err := myAddr.MarshalCBOR(&offsetSeed)
	if err != nil {
		return 0, fmt.Errorf("failed to serialize address: %w", err)
	}

	err = binary.Write(&offsetSeed, binary.BigEndian, currEpoch)
	if err != nil {
		return 0, fmt.Errorf("failed to serialize epoch: %w", err)
	}

	digest := hash(offsetSeed.Bytes())
	var offset uint64
	err = binary.Read(bytes.NewBuffer(digest[:]), binary.BigEndian, &offset)
	if err != nil {
		return 0, fmt.Errorf("failed to interpret digest: %w", err)
	}

	offset = offset % uint64(WPoStProvingPeriod)
	return abi.ChainEpoch(offset), nil
}

// Computes the epoch at which a proving period should start such that it is greater than the current epoch, and
// has a defined offset from being an exact multiple of WPoStProvingPeriod.
// A miner is exempt from Winow PoSt until the first full proving period starts.
func currentProvingPeriodStart(currEpoch abi.ChainEpoch, offset abi.ChainEpoch) abi.ChainEpoch {
	currModulus := currEpoch % WPoStProvingPeriod
	var periodProgress abi.ChainEpoch // How far ahead is currEpoch from previous offset boundary.
	if currModulus >= offset {
		periodProgress = currModulus - offset
	} else {
		periodProgress = WPoStProvingPeriod - (offset - currModulus)
	}

	periodStart := currEpoch - periodProgress
	return periodStart
}

// Computes the deadline index for the current epoch for a given period start.
// currEpoch must be within the proving period that starts at provingPeriodStart to produce a valid index.
func currentDeadlineIndex(currEpoch abi.ChainEpoch, periodStart abi.ChainEpoch) uint64 {
	return uint64((currEpoch - periodStart) / WPoStChallengeWindow)
}

func zeroReplacedSectorParameters() (pledge abi.TokenAmount, age abi.ChainEpoch, dayReward big.Int) {
	return big.Zero(), abi.ChainEpoch(0), big.Zero()
}

// Update worker address with pending worker key if exists and delay has passed
func processPendingWorker(info *MinerInfo, rt Runtime, st *State) {
	if info.PendingWorkerKey == nil || rt.CurrEpoch() < info.PendingWorkerKey.EffectiveAt {
		return
	}

	info.Worker = info.PendingWorkerKey.NewWorker
	info.PendingWorkerKey = nil

	err := st.SaveInfo(adt.AsStore(rt), info)
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "could not save miner info")
}

// Computes deadline information for a fault or recovery declaration.
// If the deadline has not yet elapsed, the declaration is taken as being for the current proving period.
// If the deadline has elapsed, it's instead taken as being for the next proving period after the current epoch.
func declarationDeadlineInfo(periodStart abi.ChainEpoch, deadlineIdx uint64, currEpoch abi.ChainEpoch) (*dline.Info, error) {
	if deadlineIdx >= WPoStPeriodDeadlines {
		return nil, fmt.Errorf("invalid deadline %d, must be < %d", deadlineIdx, WPoStPeriodDeadlines)
	}

	deadline := NewDeadlineInfo(periodStart, deadlineIdx, currEpoch).NextNotElapsed()
	return deadline, nil
}

// Checks that a fault or recovery declaration at a specific deadline is outside the exclusion window for the deadline.
func validateFRDeclarationDeadline(deadline *dline.Info) error {
	if deadline.FaultCutoffPassed() {
		return fmt.Errorf("late fault or recovery declaration at %v", deadline)
	}
	return nil
}

// Validates that a partition contains the given sectors.
func validatePartitionContainsSectors(partition *Partition, sectors bitfield.BitField) error {
	// Check that the declared sectors are actually assigned to the partition.
	contains, err := BitFieldContainsAll(partition.Sectors, sectors)
	if err != nil {
		return xerrors.Errorf("failed to check sectors: %w", err)
	}
	if !contains {
		return xerrors.Errorf("not all sectors are assigned to the partition")
	}
	return nil
}

func terminationPenalty(sectorSize abi.SectorSize, currEpoch abi.ChainEpoch,
	rewardEstimate, networkQAPowerEstimate smoothing.FilterEstimate, sectors []*SectorOnChainInfo) abi.TokenAmount {
	totalFee := big.Zero()
	for _, s := range sectors {
		sectorPower := QAPowerForSector(sectorSize, s)
		fee := PledgePenaltyForTermination(s.ExpectedDayReward, currEpoch-s.Activation, s.ExpectedStoragePledge,
			networkQAPowerEstimate, sectorPower, rewardEstimate, s.ReplacedDayReward, s.ReplacedSectorAge)
		totalFee = big.Add(fee, totalFee)
	}
	return totalFee
}

func PowerForSector(sectorSize abi.SectorSize, sector *SectorOnChainInfo) PowerPair {
	return PowerPair{
		Raw: big.NewIntUnsigned(uint64(sectorSize)),
		QA:  QAPowerForSector(sectorSize, sector),
	}
}

// Returns the sum of the raw byte and quality-adjusted power for sectors.
func PowerForSectors(ssize abi.SectorSize, sectors []*SectorOnChainInfo) PowerPair {
	qa := big.Zero()
	for _, s := range sectors {
		qa = big.Add(qa, QAPowerForSector(ssize, s))
	}

	return PowerPair{
		Raw: big.Mul(big.NewIntUnsigned(uint64(ssize)), big.NewIntUnsigned(uint64(len(sectors)))),
		QA:  qa,
	}
}

func ConsensusFaultActive(info *MinerInfo, currEpoch abi.ChainEpoch) bool {
	// For penalization period to last for exactly finality epochs
	// consensus faults are active until currEpoch exceeds ConsensusFaultElapsed
	return currEpoch <= info.ConsensusFaultElapsed
}

func getMinerInfo(rt Runtime, st *State) *MinerInfo {
	info, err := st.GetInfo(adt.AsStore(rt))
	builtin.RequireNoErr(rt, err, exitcode.ErrIllegalState, "could not read miner info")
	return info
}

func min64(a, b uint64) uint64 {
	if a < b {
		return a
	}
	return b
}

func max64(a, b uint64) uint64 {
	if a > b {
		return a
	}
	return b
}

func minEpoch(a, b abi.ChainEpoch) abi.ChainEpoch {
	if a < b {
		return a
	}
	return b
}

func maxEpoch(a, b abi.ChainEpoch) abi.ChainEpoch { //nolint:deadcode,unused
	if a > b {
		return a
	}
	return b
}

func checkControlAddresses(rt Runtime, controlAddrs []addr.Address) {
	if len(controlAddrs) > MaxControlAddresses {
		rt.Abortf(exitcode.ErrIllegalArgument, "control addresses length %d exceeds max control addresses length %d", len(controlAddrs), MaxControlAddresses)
	}
}

func checkPeerInfo(rt Runtime, peerID abi.PeerID, multiaddrs []abi.Multiaddrs) {
	if len(peerID) > MaxPeerIDLength {
		rt.Abortf(exitcode.ErrIllegalArgument, "peer ID size of %d exceeds maximum size of %d", peerID, MaxPeerIDLength)
	}

	totalSize := 0
	for _, ma := range multiaddrs {
		if len(ma) == 0 {
			rt.Abortf(exitcode.ErrIllegalArgument, "invalid empty multiaddr")
		}
		totalSize += len(ma)
	}
	if totalSize > MaxMultiaddrData {
		rt.Abortf(exitcode.ErrIllegalArgument, "multiaddr size of %d exceeds maximum of %d", totalSize, MaxMultiaddrData)
	}
}
```

## [Miner Collaterals](https://spec.filecoin.io/#section-systems.filecoin\_mining.miner\_collaterals) <a href="#section-systems.filecoin_mining.miner_collaterals" id="section-systems.filecoin_mining.miner_collaterals"></a>

Most permissionless blockchain networks require upfront investment in resources in order to participate in the consensus. The more power an entity has on the network, the greater the share of total resources it needs to own, both in terms of physical resources and/or staked tokens (collateral).

Filecoin must achieve security via the dedication of resources. By design, Filecoin mining requires commercial hardware only (as opposed to ASIC hardware) that is cheap in amortized cost and easy to repurpose, which means the protocol cannot solely rely on the hardware as the capital investment at stake for attackers. Filecoin also uses upfront token collaterals, as in proof-of-stake protocols, proportional to the storage hardware committed. This gets the best of both worlds: attacking the network requires both acquiring and running the hardware, but it also requires acquiring large quantities of the token.

To satisfy the multiple needs for collateral in a way that is minimally burdensome to miners, Filecoin includes three different collateral mechanisms: _initial pledge collateral, block reward as collateral, and storage deal provider collateral_. The first is an initial commitment of filecoin that a miner must provide with each sector. The second is a mechanism to reduce the initial token commitment by vesting block rewards over time. The third aligns incentives between miner and client, and can allow miners to differentiate themselves in the market. The remainder of this subsection describes each in more detail.

### [**Initial Pledge Collateral**](https://spec.filecoin.io/#section-systems.filecoin\_mining.miner\_collaterals.initial-pledge-collateral)

Filecoin Miners must commit resources in order to participate in the economy; the protocol can use the minersʼ stake in the network to ensure that rational behavior benefits the network, rewarding the creation of value and penalizing malicious behavior via slashing. The pledge size is meant to adequately incentivize the fulfillment of a sectorʼs promised lifetime and provide sufficient consensus security.

Hence, the initial pledge function consists of two components: a _storage pledge_ and a _consensus pledge_.

SectorInitialPledge=SectorInitialStoragePledge+SectorInitialConsensusPledgeSectorInitialPledge=SectorInitialStoragePledge+SectorInitialConsensusPledge

The storage pledge protects the networkʼs quality-of-service for clients by providing starting collateral for the sector in the event of slashing. The storage pledge must be small enough to be feasible for miners joining the network, and large enough to collateralize storage against early faults, penalties, and fees. The vesting of block rewards and the use of unvested rewards as additional collateral reduces initial storage pledge without compromising the incentive alignment of the network. This is discussed in more depth in the following subsection. A balance is achieved by using an initial storage pledge amount approximately sufficient to cover 7 days worth of Sector fault fee and 1 Sector fault detection fee. This is denominated in the number of days of future rewards that a sector is expected to earn.

SectorInitialStoragePledge=Estimated20DaysSectorBlockRewardSectorInitialStoragePledge=Estimated20DaysSectorBlockReward

Since the storage pledge per sector is based on the expected block reward that sector will win, the storage pledge is independent of the networkʼs total storage. As a result, the total network storage pledge depends solely on future block reward. Thus, while the storage pledge provides a clean way to reason about the rationality of adding a sector, it does not provide sufficient long-term security guarantees to the network, making consensus takeovers less costly as the block reward decreases. As such, the second half of the initial pledge function, the consensus pledge, depends on both the amount of quality-adjusted power (QAP) added by the sector and the network circulating supply. The network targets approximately 30% of the network’s circulating supply locked up in initial consensus pledge when it is at or above the baseline. This is achieved with a small pledge share allocated to sectors based on their share of the networkʼs quality-adjusted power. Given an exponentially growing baseline, initial pledge per unit QAP should decrease over time, as should other mining costs.

SectorInitialConsensusPledge=30%×FILCirculatingSupply×SectorQAPmax(NetworkBaseline,NetworkQAP)SectorInitialConsensusPledge=30%×FILCirculatingSupply×max(NetworkBaseline,NetworkQAP)SectorQAP​

### [**Block Reward Collateral**](https://spec.filecoin.io/#section-systems.filecoin\_mining.miner\_collaterals.block-reward-collateral)

Clients need reliable storage. Under certain circumstances, miners might agree to a storage deal, then want to abandon it later as a result of increased costs or other market dynamics. A system where storage miners can freely or cheaply abandon files would drive clients away from Filecoin as a result of serious data loss and low quality of service. To make sure all the incentives are correctly aligned, Filecoin penalizes miners that fail to store files for the promised duration. As such, high collateral could be used to incentivize good behavior and improve the networkʼs quality of service. On the other hand, however, high collateral creates barriers to miners joining the network. Filecoin’s constructions have been designed such that they hit the right balance.

In order to reduce the upfront collateral that a miner needs to provide, the block reward is used as collateral. This allows the protocol to require a smaller but still meaningful initial pledge. Block rewards earned by a sector are subject to slashing if a sector is terminated before its expiration. However, due to chain state limitations, the protocol is unable to do accounting on a per sector level, which would be the most fair and accurate. Instead, the chain performs a per-miner level approximation. Sublinear vesting provides a strong guarantee that miners will always have the incentive to keep data stored until the deal expires and not earlier. An extreme vesting schedule would release all tokens that a sector earns only when the sector promise is fulfilled.

However, the protocol should provide liquidity for miners to support their mining operations, and releasing rewards all at once creates supply impulses to the network. Moreover, there should not be a disincentive for longer sector lifetime if the vesting duration also depends on the lifetime of the sector. As a result, a fixed duration linear vesting for the rewards that a miner earns after a short delay creates the necessary sub-linearity. This sub-linearity has been introduced by the Initial Pledge.

In general, fault fees are slashed first from the soonest-to-vest unvested block rewards followed by the minerʼs account balance. When a minerʼs balance is insufficient to cover their minimum requirements, their ability to participate in consensus, win block rewards, and grow storage power will be restricted until their balance is restored. Overall, this reduces the initial pledge requirement and creates a sufficient economic deterrent for faults without slashing the miner’s balance for every penalty.

### [**Storage Deal Collateral**](https://spec.filecoin.io/#section-systems.filecoin\_mining.miner\_collaterals.storage-deal-collateral)

The third form of collateral is provided by the storage provider to collateralize deals. See the [Storage Market Actor](https://spec.filecoin.io/#section-systems.filecoin\_markets.onchain\_storage\_market.storage\_market\_actor) for further details on the Storage Deal Collateral.

## [Storage Proving](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_proving) <a href="#section-systems.filecoin_mining.storage_proving" id="section-systems.filecoin_mining.storage_proving"></a>

### [**Filecoin Proving Subsystem**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_proving.filecoin-proving-subsystem)

[Example:](https://spec.filecoin.io/#example-)

```go
import abi "github.com/filecoin-project/specs-actors/actors/abi"
import poster "github.com/filecoin-project/specs/systems/filecoin_mining/storage_proving/poster"
import sealer "github.com/filecoin-project/specs/systems/filecoin_mining/storage_proving/sealer"
import block "github.com/filecoin-project/specs/systems/filecoin_blockchain/struct/block"

type StorageProvingSubsystem struct {
    SectorSealer   sealer.SectorSealer
    PoStGenerator  poster.PoStGenerator

    VerifySeal(sv abi.SealVerifyInfo, pieceInfos [abi.PieceInfo]) union {ok bool, err error}
    ComputeUnsealedSectorCID(sectorSize UInt, pieceInfos [abi.PieceInfo]) union {unsealedSectorCID abi.UnsealedSectorCID, err error}

    ValidateBlock(block block.Block)

    // TODO: remove this?
    // GetPieceInclusionProof(pieceRef CID) union { PieceInclusionProofs, error }

    GenerateElectionPoStCandidates(
        challengeSeed  abi.PoStRandomness
        sectorIDs      [abi.SectorID]
    ) [abi.PoStCandidate]

    GenerateSurprisePoStCandidates(
        challengeSeed  abi.PoStRandomness
        sectorIDs      [abi.SectorID]
    ) [abi.PoStCandidate]

    CreateElectionPoStProof(
        challengeSeed  abi.PoStRandomness
        candidates     [abi.PoStCandidate]
    ) [abi.PoStProof]

    CreateSurprisePoStProof(
        challengeSeed  abi.PoStRandomness
        candidates     [abi.PoStCandidate]
    ) [abi.PoStProof]
}
```

### [**Sector Poster**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_proving.poster)

#### [**PoSt Generator object**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_proving.poster.post-generator-object)

[Example:](https://spec.filecoin.io/#example-)

```go
import abi "github.com/filecoin-project/specs-actors/actors/abi"
import sector_index "github.com/filecoin-project/specs/systems/filecoin_mining/sector_index"

type UInt64 UInt

// TODO: move this to somewhere the blockchain can import
// candidates:
// - filproofs - may have to learn about Sectors (and if we move Seal stuff, Deals)
// - "blockchain/builtins" or something like that - a component in the blockchain that handles storage verification
type PoStSubmission struct {
    PostProof   abi.PoStProof
    ChainEpoch  abi.ChainEpoch
}

type PoStGenerator struct {
    SectorStore sector_index.SectorStore

    GeneratePoStCandidates(
        challengeSeed   abi.PoStRandomness
        candidateCount  UInt
        sectors         [abi.SectorID]
    ) [abi.PoStCandidate]

    CreateElectionPoStProof(
        randomness  abi.PoStRandomness
        witness     [abi.PoStCandidate]
    ) [abi.PoStProof]

    CreateSurprisePoStProof(
        randomness  abi.PoStRandomness
        witness     [abi.PoStCandidate]
    ) [abi.PoStProof]

    // FIXME: Verification shouldn't require a PoStGenerator. Move this.
    VerifyPoStProof(
        Proof          abi.PoStProof
        challengeSeed  abi.PoStRandomness
    ) bool
}
```

[Example:](https://spec.filecoin.io/#example-)

```go
package poster

import (
	abi "github.com/filecoin-project/specs-actors/actors/abi"
	filproofs "github.com/filecoin-project/specs/libraries/filcrypto/filproofs"
	util "github.com/filecoin-project/specs/util"
)

type Serialization = util.Serialization

// See "Proof-of-Spacetime Parameters" Section
// TODO: Unify with orient model.
const POST_CHALLENGE_DEADLINE = uint(480)

func (pg *PoStGenerator_I) GeneratePoStCandidates(challengeSeed abi.PoStRandomness, candidateCount int, sectors []abi.SectorID) []abi.PoStCandidate {
	// Question: Should we pass metadata into FilProofs so it can interact with SectorStore directly?
	// Like this:
	// PoStReponse := SectorStorageSubsystem.GeneratePoSt(sectorSize, challenge, faults, sectorsMetatada);

	// Question: Or should we resolve + manifest trees here and pass them in?
	// Like this:
	// trees := sectorsMetadata.map(func(md) { SectorStorage.GetMerkleTree(md.MerkleTreePath) });
	// Done this way, we redundantly pass the tree paths in the metadata. At first thought, the other way
	// seems cleaner.
	// PoStReponse := SectorStorageSubsystem.GeneratePoSt(sectorSize, challenge, faults, sectorsMetadata, trees);

	// For now, dodge this by passing the whole SectorStore. Once we decide how we want to represent this, we can narrow the call.

	return filproofs.GenerateElectionPoStCandidates(challengeSeed, sectors, candidateCount, pg.SectorStore())
}

func (pg *PoStGenerator_I) CreateElectionPoStProof(randomness abi.PoStRandomness, postCandidates []abi.PoStCandidate) []abi.PoStProof {
	var privateProofs []abi.PrivatePoStCandidateProof

	for _, candidate := range postCandidates {
		privateProofs = append(privateProofs, candidate.PrivateProof)
	}

	return filproofs.CreateElectionPoStProof(privateProofs, randomness)
}

func (pg *PoStGenerator_I) CreateSurprisePoStProof(randomness abi.PoStRandomness, postCandidates []abi.PoStCandidate) []abi.PoStProof {
	var privateProofs []abi.PrivatePoStCandidateProof

	for _, candidate := range postCandidates {
		privateProofs = append(privateProofs, candidate.PrivateProof)
	}

	return filproofs.CreateSurprisePoStProof(privateProofs, randomness)
}
```

### [**Sector Sealer**](https://spec.filecoin.io/#section-systems.filecoin\_mining.storage\_proving.sealer)

[Example:](https://spec.filecoin.io/#example-)

```go
import abi "github.com/filecoin-project/specs-actors/actors/abi"
import sector "github.com/filecoin-project/specs/systems/filecoin_mining/sector"
import file "github.com/filecoin-project/specs/systems/filecoin_files/file"
import addr "github.com/filecoin-project/go-address"

type SealInputs struct {
    SectorSize       abi.SectorSize
    RegisteredProof  abi.RegisteredProof  // FIXME: Ensure this is provided.
    SectorID         abi.SectorID
    MinerID          addr.Address
    RandomSeed       abi.SealRandomness  // This should be derived from SEAL_EPOCH = CURRENT_EPOCH - FINALITY.
    UnsealedPath     file.Path
    SealedPath       file.Path
    DealIDs          [abi.DealID]
}

type CreateSealProofInputs struct {
    SectorID               abi.SectorID
    RegisteredProof        abi.RegisteredProof
    InteractiveRandomSeed  abi.InteractiveSealRandomness
    SealedPaths            [file.Path]
    SealOutputs
}

type SealOutputs struct {
    ProofAuxTmp sector.ProofAuxTmp
}

type CreateSealProofOutputs struct {
    SealInfo  abi.SealVerifyInfo
    ProofAux  sector.PersistentProofAux
}

type SectorSealer struct {
    SealSector() union {so SealOutputs, err error}
    CreateSealProof(si CreateSealProofInputs) union {so CreateSealProofOutputs, err error}

    MaxUnsealedBytesPerSector(SectorSize UInt) UInt
}
```
