# Codex Marketplace Spec

title: CODEX-MARKETPLACE
name: Codex Storage Marketplace
status: raw
tags: codex
editor: Dmitriy <dryajov@status.im>
contributors:
- Mark <mark@codex.storage>
- Adam <adam@codex.storage>
- Eric <eric@codex.storage>
- Jimmy Debe <jimmy@status.im>
---

## Abstract

Codex Marketplace and its interactions are defined by a smart contract deployed on an EVM-compatible blockchain. 
This specification describes these flows for all the different roles in the network. 
The specification is meant for a Codex node implementor.  

## Semantics 

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, 
“SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [2119](https://www.ietf.org/rfc/rfc2119.txt).

### Definitions

| Terminology             | Description                                                                                                             |
|-------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Storage Provider (SP) | Node in the Codex network that provides storage services to the marketplace. |
| Validator | Node that helps in finding missing storage proofs |
| Client | Node that interacts with the other nodes in the Codex network to store, locate, and retrieve data. |
| Storage Request or Request | Created by a client node in order to persist data on the Codex network. |
| Slot or Storage Slot | A space alloted by the Storage Request to store a single chunk of the data pertaining to this Storage Request. |
| Smart Contract | A smart contract associated with the marketplace functionality. |
| Codex Tokens | ERC20-based tokens used by the Codex network. |

## Motivation

The Codex network aims to create a peer-to-peer storage engine with strong data durability, data persistence guarantees, and node storage incentives.

The marketplace is an important component of the Codex network. The marketplace can be seen as a place where all the involved parties interact in order to provide persistence in the Codex network. The marketplace also secures ways to enforce agreements and facilitate repair of the data when one or more Storage Nodes fail to fulfil their duties.

The marketplace is implemented as a Smart Contract deployed to an EVM-compatible blockchain. The marketplace Smart Contract allows realization of a number of scenarios where the nodes in the Codex network take one of more roles in the network and which the participating node takes upon itself. It can be one role or multiple at the same time.
The marketplace is implemented as a Smart Contract deployed to an EVM-compatible blockchain. The marketplace Smart Contract allows realization of a number of flows (or scenarios) where the nodes in the Codex network take one of more roles upon themselves while providing a reliable persistance layer to the users.

This specification describes these flows.

The Marketplace contract handles Storage Requests, maintains the state of the alloted Storage Slots, and orchestrates the Storage Provider rewards, collaterals, and storage proofs.

A node that wants to participate in the Codex persistence layer needs to implements one or more roles described in this document.

### Roles

A node can take one the three main roles in the network - the Client, the Storage Provider (SP) and the Validator role (for the sake of conciseness and where it does not cause ambiguity, referred to as a _Client_, _Storage Provider_, and _Validator_ respectively).

A Client is a potentially short-lived node in the network with the purpose of persisting its data in the Codex persistance layer.

A Storage Provider is a long-lived node providing storage for the Clients in exchange for a profit. In order to provide a reliable, robust service to the Clients, for the data they commit to persist, Storage Providers are required to periodically provide evidence that they persist the data.

A Validator validates that Storage Providers correctly fulfil their duties and that they provide proofs of storage when requested by the Smart Contract.

## Storage Request Lifecycle

The diagram below depicts the lifecycle of Storage Request:

```
                      ┌───────────┐                               
                      │ Cancelled │                               
                      └───────────┘                               
                            ▲                                     
                            │ Not all                             
                            │ Slots filled                        
                            │                                     
    ┌───────────┐    ┌──────┴─────────────┐           ┌─────────┐ 
    │ Submitted ├───►│ Slots Being Filled ├──────────►│ Started │ 
    └───────────┘    └────────────────────┘ All Slots └────┬────┘ 
                                            Filled         │      
                                                           │      
                                   ┌───────────────────────┘      
                           Proving ▼                              
    ┌────────────────────────────────────────────────────────────┐
    │                                                            │
    │                 Proof submitted                            │
    │       ┌─────────────────────────► All good                 │
    │       │                                                    │
    │ Proof required                                             │
    │       │                                                    │
    │       │         Proof missed                               │
    │       └─────────────────────────► After some time slashed  │
    │                                   eventually Slot freed    │
    │                                                            │
    └────────┬─┬─────────────────────────────────────────────────┘
             │ │                                      ▲           
             │ │                                      │           
             │ │ SP kicked out and Slot freed ┌───────┴────────┐  
All good     │ ├─────────────────────────────►│ Repair process │  
Time ran out │ │                              └────────────────┘  
             │ │                                                  
             │ │ Too many Slots freed         ┌────────┐          
             │ └─────────────────────────────►│ Failed │          
             ▼                                └────────┘          
       ┌──────────┐                                               
       │ Finished │                                               
       └──────────┘                                               
```

![image](./images/storeRequest.png)

## Client Role

A node implementing the Client role mediates in persisting data within the Codex network.

A Client has two main responsibilities:

 - Requesting storage from the network by sending a _Storage Request_ to the _Smart Contract_.
 - Withdrawing funds from the Storage Requests previously created by the Client.

### Creating Storage Requests

When the user prompts the Client node to create a Storage Request, the Client node SHOULD receive the input parameters for the Storage Request from the user.

To create a request to persist a dataset on the Codex network,
client nodes MUST split the dataset into data chunks, $(c_1, c_2, c_3, \ldots, c_{n})$.
Using the Erasure Coding method and the provided input parameters, the data chunks are encoded and distributed over a number of slots.
The Erasure Coding method applied MUST use the [Reed-Soloman algorithm](https://hackmd.io/FB58eZQoTNm-dnhu0Y1XnA). 
The final slot's roots and other metadata MUST be placed into a Manifest (**TODO: Manifest RFC**). The manifest's CID MUST then 
be used as the `cid` of the stored dataset.

> Perhaps it would be good to write a somehow more verbose intro about the manifest and the roots.

After the dataset is prepared, a Client node MUST call the Smart Contract function `requestStorage(request)` providing the desired request parameters in the `request` parameter. The `request` prameter is of type `Request`:

```solidity
struct Request {
  // The Codex node requesting storage
  address client;

  // Describes parameters of Request
  Ask ask;
  
  // Describes the dataset that will be hosted with the Request
  Content content;

  // Timeout in seconds during which all the slots have to be filled, otherwise Request will get cancelled
  uint256 expiry;

  // Random value to differentiate from other requests of same parameters
  byte32 nonce;
}
  
struct Ask {
  // Amount of tokens that will be awarded to storage providers for finishing the storage request.
  // Reward is per slot per second.
  uint256 reward;

  // Amount of tokens required for collateral by storage providers
  uint256 collateral;

  // Frequency that storage providers need to submit proofs of storage
  uint256 proofProbability;

  // Total duration of the storage request in seconds
  uint256 duration;

  // The number of requested slots
  uint64 slots;

  // Amount of storage per slot in bytes
  uint256 slotSize;

  // Max slots that can be lost without data considered to be lost
  uint64 maxSlotLoss; 
}

struct Content {
  // Content identifier
  string cid;

  // Merkle root of the dataset, used to verify storage proofs
  byte32 merkleRoot;
}

```

The meaning of the Request parameters is given below:

> Note that not all parameters are defined. I think that in a formal spec we need to introduce all of them, even shortly.

`cid` 

An identifier used to locate the Manifest representing the dataset.
- MUST be a [CIDv1](https://github.com/multiformats/cid#cidv1), sha-256 [multihash](https://github.com/multiformats/multihash).
- The data it represents SHOULD be discoverable in the network, otherwise the Request MUST be canceled.

`reward`

- It MUST be an amount of Codex Tokens offered per slot per second.
- The Ethereum address that submits the `requestStorage()` transaction MUST have [approval](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20-approve-address-uint256-) for transfer of at least the same amount on Codex Tokens.

> The second item could have a better explanation. E.g., it is not clear, who is the `approver`.

`collateral`

- Amount of tokens that the storage providers submits when they fill slots.
- Collateral is then slashed or forfeited if the storage providers fail to provide the service requested by the Request (more information below).

`proofProbability`

Determines the average frequency that a proof is required in a period: $\frac{1}{proofProbability}$

> Does it have a unit?

- Storage Providers are required to provide proofs of storage to the Marketplace Smart Contract when they are prompted to do so by the Smart Contract.
- In order to prevent hosts from only coming online when proofs are required, the frequency at which proofs are requested from the Storage Providers is stochastic and is affected by the `proofProbability` parameter.

`expiry`

- The parameter is specified as a duration in seconds, hence the final deadline timestamp is calculated at the moment when the transaction is mined. 

`nonce`

- It SHOULD be a random byte array.

#### Renewal of Storage Requests

It should be noted that Marketplace does not support extending Requests. It is REQUIRED that if the user wants to extend the duration of the Request, a new Request transaction with the same CID needs to be submitted **before the original Request completes**. In this way, the data will be still persisted in the network at the time when the new (or the existing) Storage Providers need to retrieve the complete dataset in order to fill the slots of the new Request.

### Withdrawing Funds

The Client node SHOULD monitor the status of the requests that it created. When the Storage Request enters state `Cancelled` (this happens when not all the slots has been filled after `expiry` timeout) the Client node SHOULD initiate withdrawal of the remaining funds from the Smart Contract using function `withdrawFunds(requestId)`.

- The Request can be concluded as `Cancelled` if no `RequestFulfilled(requestId)` event is observed during the timeout given by the timeout value returned from the function `requestExpiresAt(requestId)`.
- The Request is concluded as `Failed` when `RequestFailed(requestId)` event is observed.
- The Request is concluded as `Finished` after interval given by the value returned from function `getRequestEnd(requestId)`.

## Storage Provider Role

A Codex node acting as a Storage Provider persists data across the network by hosting slots requested by the Clients in their Storage Requests.

The following tasks needs to be considered when hosting a slot:

 - filling a slot,
 - proving,
 - Repairing a slot,
 - Collecting Request reward and collateral.

### Filling Slots

When a new Request is created, the `StorageRequested(requestId, ask, expiry)` event is emitted with the following properties:

- `requestId` - ID of Request.
- `ask` - Specification of the Request parameters. For details see definition of the Request type in Section _Creating Storage Requests_ above.
- `expiry` - a Unix timestamp that specifies when the Request will be cancelled if all slots are not filled by then.

It is then up to the Storage Provider node to decide, based on the parameters provided by the node operator, if it wants to participate in the Request and try to fill its slot(s) (note that one Storage Provider can fill more than one slot).
If the Storage Provider node decides to ignore the Request, no further action from the Storage Provider node is required. If the Storage Provider decides to fill a slot, then, when succeeded, it MUST follow the remaining steps described below.

The node acting as a Storage Provider MUST decide which Slot, specified by the slot index, it wants to fill. The Storage Provider MAY attempt to fill more than one slot. In order to fill a slot, Storage Provider MUST first download the slot data using the CID of the manifest (**TODO: Manifest RFC**) and the slot index. The CID is specified in `request.content.cid`, which can be retrieved from the smart contract using `getRequest(requestId)`.
Then, the node MUST generate a proof over the downloaded data (**TODO: Proving RFC**).

When the proof is ready, the Storage Provider MUST call `fillSlot()` on the Smart Contract with the following being REQUIRED:
 
- Parameters:
  - `requestId` - ID of the Request.
  - `slotIndex` - Slot index that the node wants to fill.
  - `proof` - `Groth16Proof` proof structure, generated over the slot data.
- The Ethereum address of the node from which the transaction originates MUST have [approval](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20-approve-address-uint256-) for transfer of at least the amount of Codex Tokens required as collateral for the Request.

> Also here, the last point may enjoy a more verbal explanation.

If the proof delivered by the Storage Provider is invalid, or the slot was already filled by another Storage Provider, then the transaction will be reverted, otherwise a `SlotFilled(requestId, slotIndex)` event is emitted. If the transaction is successful, then the Storage Provider SHOULD transition into the __proving__ state where it will need to submit proof of the data possession when prompted by the Smart Contract.

It should be noted that if the Storage Provider node observes a `SlotFilled` event emitted for the slot that the Storage Provider node is downloading the dataset or generating the proof for, it means that the slot has been filled by some other node in the meantime. As a response, the Storage Provider SHOULD stop its current operation and attempt to fill a different, not-yet-filled slot.

### Proving

Once Storage Provider successfully fills a slot, it MUST periodically, yet non-deterministically, provide proofs to the Smart Contract that it 
stores the data it committed to store. A Storage Provider node SHOULD detect that a proof is required using the `isProofRequired(slotId)` Smart Contract function, or that the proof will be required using the `willProofBeRequired(slotId)` in case the node is in [downtime](https://github.com/codex-storage/codex-research/blob/41c4b4409d2092d0a5475aca0f28995034e58d14/design/storage-proof-timing.md).

> Maybe we should include some info about _downtime_ here? 

Once Storage Provider knows it has to provide a proof, it MUST retrieve the proof challenge using `getChallenge(slotId)`, which then NEEDS to be incorporated into the proof generation as described in Proving RFC (**TODO: Proving RFC**).

When the proof is generated it MUST be submitted by calling the `submitProof(slotId, proof)` Smart Contract function.

#### Slashing

There is a slashing scheme in place that is orchestrated by the Smart Contract to incentivize the correct behavior and the proper proof submissions by Storage Providers. This scheme is configured on the Smart Contract level and is the same for all the participants in the network. The configuration of the slashing scheme can be obtained from the `getConfig()` contract call.

The _slashing_ works as follows:

- A Storage Provider node MAY miss at most `config.collateral.slashCriterion` proofs before it is slashed.
- It is then slashed `config.collateral.slashPercentage` percentage **of the originally asked collateral** (hence the slashing amount is always the same for the given request).
- If the number of times the node was slashed exceeds `config.collateral.maxNumberOfSlashes`, then the slot is released, the remaining node's collateral is burned, and the slot is offered to other nodes for repair. The Smart Contract also subsequently emits the `SlotFreed(requestId, slotIndex)` event.

If, at any given moment, the number of released slots exceeds the value given by the `request.ask.maxSlotLoss` parameter, then the dataset is considered to be lost and the Request is considered as _failed_. The collateral of all the Storage Provider nodes that hosted the slots belonging to the Request is burned and event `RequestFailed(requestId)` is emitted.

### Repair

When a slot is released because of too many missed proofs, which SHOULD be detected by listening to the `SlotFreed(requestId, slotIndex)` event, then a Storage Provider node can decide if it wants to participate in the repairing of the slot. The node SHOULD, similar to when filling a slot, consider the node operator configuration when making the decision. The Storage Provider node that originally hosted the slot that was subsequently released (because Storage Provider failed to comply with the proving requirements), MAY also participate in the data repair, but by refilling the slot it lost, the Storage Provider **will not** recover its original collateral and needs to submit a new collateral using the `fillSlot()` call.

The repair process is the same as when filling slots. If the original slot dataset is not present in the network anymore, Storage Provider MAY use the Erasure Coding to reconstruct the original slot dataset. 
Reconstructing the original slot dataset requires retrieving other pieces of the dataset that are stored in other slots belonging to the Request. For this reason, the node that will successfully repair a slot is entitled to an additional reward. (**TODO: Implementation**)

The repair process works then as described below:

1. Storage Provider observes `SlotFreed` event and decides to repair it.
2. Storage Provider MUST download the chunks of data required to reconstruct the data of the released slot. The node MUST use the [Reed-Soloman algorithm](https://hackmd.io/FB58eZQoTNm-dnhu0Y1XnA) to reconstruct the missing data.
3. Storage Provider MUST generate proof over the reconstructed data.
4. Storage Provider MUST call the `fillSlot()` Smart Contract function with the same parameters and collateral allowance as described in the [Filling slot](#filling-slot) section.

### Collecting Funds

A Storage Provider node SHOULD monitor Requests and the associated slots it hosts.

When the Storage Requests enters states `Cancelled`, `Finished`, or `Failed`, it SHOULD call the `freeSlot(slotId)` Smart Contract function.

> From the sentence above it is hard to say who is going to call the `freeSlot(slotId)`. I mean I am not sure who "it" refers to.

The above mentioned Storage Request states (`Cancelled`, `Finished`, and `Failed`) can be detected as follows:

 - A Storage Request is concluded as `Cancelled` if no `RequestFulfilled(requestId)` event was observed within time indicated by the `expiry` Request parameter. Notice that there is also a `RequestCancelled` event emitted, but the node SHOULD NOT use this event for asserting the Request expiration as the `RequestCancelled` event is not guaranteed to be emitted at the time of expiry.
 - A Storage Request is considered `Finished` when the time indicated by the value returned by the function `getRequestEnd(requestId)` has elapsed.
 - A node concludes that a Storage Request has `Failed` upon observing the `RequestFailed(requestId)` event.

For each of the states listed above, different funds are collected:

- For the `Cancelled` state, the collateral is returned along with a proportional payout based on the time the node actually hosted the dataset before the expiry was reached.
- For the `Finished` state, the full reward for hosting the slot, along with the collateral, is collected.
- In the `Failed` state, no funds are collected. The reward is returned to the client, and the collateral is burned. The slot is removed from the list of slots and is no longer included in the list of slots returned by the `mySlots()` function.

## Validator Role

In a blockchain, one is unable to act on events that **do not happen** as basically everything is a result of a transaction. For this reason, our Smart Contract needs an external trigger to periodically check and confirm that a storage proof has been delivered by the given Storage Provider. This is where the Validator role comes into play.
The Validator role is taken by the nodes that help to verify that the Storage Providers fulfil their obligation of submitting storage proofs when required.

Notice that it is Smart Contract that checks if the proof requested from the given storage provider has been delivered. The job of the Validator is to trigger that check on the Smart Contract for Storage Providers "observed" by the Validator. To keep validators "motivated", a Validator receives a reward each time it helps finding out that a proof from the given Storage Provider is missing.

Each time a Validator observes the `SlotFilled` event, it adds the slot reported in the `SlotFilled` event to the Validator's list of watched slots. Then at the end of every period, a Validator has at most `config.proofs.timeout` seconds (part of the configuration that can be retrieved with `getConfig()`) to request the proof validation from the Smart Contract for each slot from the Validator's list of watched slots. In the case that a slot is missing the proof, the Validator SHOULD call the `markProofAsMissing(slotId, period)` function on the Smart Contract. The `markProofAsMissing(slotId, period)` function, after attesting the missing proof for the slot with id `slotId` in `period`, will reward the validator.

> Shall we add a small intro/reasoning behind the periods and when they happen?

If the validation of all the slots watched by the Validator is not feasible within the validation `timeout` mentioned above, the validator MAY decide to validate only a subset of the watched slots.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References 

1. [Reed-Soloman algorithm](https://hackmd.io/FB58eZQoTNm-dnhu0Y1XnA)
2. [CIDv1](https://github.com/multiformats/cid#cidv1)
3. [multihash](https://github.com/multiformats/multihash)
4. [Proof-of-Data-Possession](https://hackmd.io/2uRBltuIT7yX0CyczJevYg?view)
5. [Codex market implementation](https://github.com/codex-storage/nim-codex/blob/master/codex/market.nim)
