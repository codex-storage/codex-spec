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
| storage provider (SP) nodes | A node that provides storage services to the marketplace.                                                               |
| validator nodes         | A node that flags missing storage proofs for a reward.                                  |
| client nodes            | The most common node that interacts with other nodes to store, locate and retrieve data.                                |
| storage request         | Created by client node when it wants to persist data on the network. Represents the dataset and persistence configuration. |
| slots                   | The storage request dataset is split into several pieces (slots) that are then distributed to and stored by SPs.              |

## Motivation

The Codex network aims to create a peer-to-peer storage engine with strong data durability,
data persistence guarantees, and node storage incentives.

An important component of the Codex network is its Marketplace. It is a place which mediates negotiations of all parties
 to provide persistence in the network. It also provides ways to enforce agreements and facilitate repair when a storage node drops out.

The marketplace is defined by a smart contract deployed to an EVM-compatible blockchain. It has several flows which are 
linked with roles in the network and which the participating node takes upon itself. Each node can take on the responsibilities of one role or multiple at the same time.
This specification describes these flows.

The Marketplace handles storage requests, the storage slot state,
storage provider rewards, storage provider collaterals, and storage proof state.

If a node implementation wants to participate in the persistence layer of Codex, it needs to choose which role(s) it wants
to support and implement proper flows. Otherwise, it won't be compatible with the rest of the Codex network.

### Roles

There are three main roles in the network: client, storage provider (SP) and validator. 

A client is a potentially short-lived node in the network that mainly interacts with the purpose of persisting
its data in the network. 

A Storage Provider is a long-term participant in the network that stores other data for profit. It needs to provide a proof
to the smart contract that it possesses the data from time-to-time.

A validator validates that SPs are correctly fulfilling their duties and that they provide proofs of storage
on time.

## Storage Request Lifecycle

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

## Client role

The Client role represents nodes that introduce data to be persisted inside the Codex network. 

There are two parts of a client role:

 - Requesting storage from the network by creating a storage request.
 - Withdrawing funds from storage requests.

### Creating storage requests

When the user prompts the client node to create a storage request, 
it SHOULD receive the input parameters for the storage request from the user. 

To create a request to persist a dataset on the Codex network,
client nodes MUST split the dataset into data chunks, $(c_1, c_2, c_3, \ldots, c_{n})$.
Using an erasure coding technique and input parameters, the data chunks are encoded and placed into separate slots.
The erasure coding technique MUST be the [Reed-Soloman algorithm](https://hackmd.io/FB58eZQoTNm-dnhu0Y1XnA). 
The final slot's roots and other metadata MUST be placed into a Manifest (**TODO: Manifest RFC**). The manifest's CID MUST then 
be used as the `cid` of the stored dataset.

After the dataset is prepared, a node MUST submit a transaction with the desired request parameters which are represented
as a `Request` object and its children to function `requestStorage(request)`. Below are described its properties:

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

Notes about some parameters:

`cid` 

An identifier used to locate the Manifest representing the dataset.
- MUST be a [CIDv1](https://github.com/multiformats/cid#cidv1) with sha-256 based [multihash](https://github.com/multiformats/multihash).
- Data it represents SHOULD be discoverable in the network, otherwise the Request will get canceled. 

`reward`

- It MUST be an amount of tokens offered per slot per second.
- The Ethereum address that submits the `requestStorage()` transaction MUST have [approval](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20-approve-address-uint256-) for transfer of at least the same amount on the ERC20 based token, that the network uses. 

`collateral`

- Amount of tokens that the storage providers submits when they fill slots.
- Collateral is then slashed or forfeited if the storage providers fail to provide the service requested by the Request (more information below).

`proofProbability`

Determines the average frequency that a proof is required in a period: $\frac{1}{proofProbability}$

- Storage providers are required to provide proofs of storage to the marketplace smart contract when they are challenged by the smart contract.
- The frequency is stochastic in order to prevent hosts from only coming online when proofs are required, but it is affected by this parameter.

`expiry`

- The parameter is specified as a duration in seconds, hence the final deadline timestamp is calculated at the moment when the transaction is mined. 

`nonce`

- It SHOULD be a random byte array.

#### Renewal of Storage Requests

It should be noted that Marketplace does not support extending Requests. It is REQUIRED that if the user wants to 
extend the Request's duration, a new Request with the same CID must be [created](#Creating-storage-requests) **well before the original
Request finishes**. In this way, the data will be still persisted in the network with enough time for new (or the current) storage providers 
need to retrieve the dataset and fill slots of the new Request.

### Withdrawing funds

The client node SHOULD monitor the status of Requests that it created. When a Request reaches the `Cancelled` state (not all slots were filled before the `expiry` timeout),
then the client node SHOULD initiate a withdrawal of the remaining funds from the contract using the function, `withdrawFunds(requestId)`.

- The `Cancelled` state MAY be detected using the timeout specified from function `requestExpiresAt(requestId)` **and** not detecting the emitted `RequestFulfilled(requestId)` event.
 - The `Failed` state MAY be detected using `RequestFailed(requestId)` event emitted from the smart contract.
 - The `Finished` state MAY be detected by setting timeout specified from function `getRequestEnd(requestId)`.

## Storage Provider role

Storage Provider (SP) role represents nodes that persist data across the network by hosting Slots of Requests 
that the Client nodes requested.

There are several parts to hosting a slot:

 - Filling a slot
 - Proving
 - Repairing a slot
 - Collecting Request's reward and collateral

### Filling slot

When a new Request is created, a `StorageRequested(requestId, ask, expiry)` event is emitted with the following properties: 

 - `requestId` - ID of Request.
 - `ask` - Specification of [Request parameters](#Creating-storage-requests).
 - `expiry` - Unix timestamp that specifies when the Request will be cancelled if all slots are not filled by then.

It is then up to the Storage Provider node to decide based on the emitted parameters and node's operator configuration
if it wants to participate in the Request and try to fill its slot(s).
If the node decides to ignore this Request, no action is necessary. If the node wants to try to fill a slot, then 
it MUST follow the remaining steps.

The Node MUST decide which Slot specified by slot's index it wants to try to fill in. The Node MAY try filling multiple
slots. In order to fill a slot, the node MUST first download the slot's data using the CID of the manifest (**TODO: Manifest RFC**) and the index of  the slot. The CID is  specified in `request.content.cid`, which can be retrieved from the smart contract using `getRequest(requestId)`.
Then the node MUST generate a proof over the downloaded data (**TODO: Proving RFC**).

When the proof is ready, it MUST then create a transaction for the smart contract call, `fillSlot()`, with the following REQUIRED parameters:
 
 - Parameters:
   - `requestId` - ID of the Request.
   - `slotIndex` - Index of the slot that the node is trying to fill.
   - `proof` - `Groth16Proof` proof structure, generated over the slot's data.
 - The Ethereum address of the node from which the transaction originates MUST have [approval](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#IERC20-approve-address-uint256-) for transfer of at least the amount required as collateral for the Request on the ERC20-based token, that the network utilizes.

If the proof is invalid, or the slot was already filled by another node, then the transaction
will revert, otherwise a `SlotFilled(requestId, slotIndex)` event is emitted. If the transaction is successful, then the
node SHOULD transition into a __proving__ state as it will need to submit proofs of data possession when challenged by the
contract.

It should be noted that if the node sees the `SlotFilled` emitted for a slot that it is downloading the dataset or 
generating a proof for, then the node SHOULD stop and choose a different non-filled slot to try to fill as the chosen slot
has already been filled by another node.

### Proving
Once a node fills a slot, it MUST submit proofs to the smart contract when a challenge is issued by the contract.  
Nodes SHOULD detect that a proof is required for the current period using the `isProofRequired(slotId)` function, 
or that it will be required using the `willProofBeRequired(slotId)` function in the case that the [pointer is in downtime](https://github.com/codex-storage/codex-research/blob/41c4b4409d2092d0a5475aca0f28995034e58d14/design/storage-proof-timing.md).

Once a node knows it has to provide a proof it MUST get the proof challenge using `getChallenge(slotId)` which then
MUST be incorporated into the proof generation as described in Proving RFC (**TODO: Proving RFC**).

When the proof is generated, it MUST be submitted with a transaction calling the `submitProof(slotId, proof)` function.

#### Slashing

There is a slashing scheme in place that is orchestrated by the smart contract to incentivise correct behaviour
and proper proof submissions by the storage provider nodes. This scheme is configured on the smart contract level and is 
the same for all the participants in the network. The concrete values of this scheme can be obtained by the `getConfig()` contract call.

Slashing works in the following way:

 - Node MAY miss at most `config.collateral.slashCriterion` proofs before it is slashed.
 - It is then slashed `config.collateral.slashPercentage` percentage **of the originally asked collateral** (hence the slashing amount is always the same for the given request).
 - If the number of times the node was slashed reaches above `config.collateral.maxNumberOfSlashes`, then the slot is freed, the remainder of the node's collateral is burned, and the slot is offered to other nodes for repair. The Contract also emits the `SlotFreed(requestId, slotIndex)` event.

If the number of concurrent freed slots reaches above the `request.ask.maxSlotLoss`, then the dataset is assumed to be lost and the Request is failed. 
The collateral of all the nodes that hosted Request's slots is burned and the event `RequestFailed(requestId)` is emitted.

### Repair

When a slot is freed because of too many missed proofs, which SHOULD be detected by listening on the `SlotFreed(requestId, slotIndex)` event, then
storage provider node can decide if it wants to participate in the repairing of the slot. The node SHOULD, similar to filling a slot,
consider the node's operator configuration when making the decision. The storage provider node that originally hosted
the freed slot MAY also participate in the data repair, but by refilling the slot it **won't** recover its original collateral
and needs to submit new collateral with the `fillSlot()` call.

The repair process is the same as with the filling slots. If the original slot's dataset is not present in the network
the node MAY use the erasure coding to reconstruct the original slot's dataset. 
As this requires retrieving more data of the dataset from the network, the node that will successfully repair
the slot by filling the freed slot will be granted an additional reward. (**TODO: Implementation**)

The repair process is then as follows:

1. Node detects a `SlotFreed` event and decides to repair it.
1. Node MUST download the required dataset CIDs and MUST use the [Reed-Soloman algorithm](https://hackmd.io/FB58eZQoTNm-dnhu0Y1XnA) to reconstruct the original slot's data.
1. Node MUST generate a proof over the reconstructed data and challenge from the smart contract.
1. Node MUST submit a transaction with a call to `fillSlot()` with the same parameters and collateral allowance, as described in [Filling slot](#filling-slot).

### Collecting funds

A Storage Provider node SHOULD monitor Requests of the slots it hosts. 

When a Request reaches the `Cancelled`, `Finished` or `Failed` state, it SHOULD call the contract's `freeSlot(slotId)` function.
These states can be detected using:

 - `Cancelled` state MAY be detected by setting a timeout using `expiry` **and** not listening for the `RequestFulfilled(requestId)` event. There is also a `RequestCancelled` event emitted, but the node SHOULD NOT use it for asserting expiry as it is not guaranteed to be emitted at the time of expiry.
 - `Finished` state MAY be detected by setting a timeout specified from function `getRequestEnd(requestId)`.
 - `Failed` state MAY be detected by listening for the `RequestFailed(requestId)` event.

For each of these states, different funds are collected:

- For `Cancelled`, the collateral is returned together with the proportional payout based on the time that the node actually hosted the dataset before expiry was reached.
	- For `Finished`, the full reward for hosting the slot, together with the collateral, is gathered.
	- For `Failed`, no funds are collected as the reward is returned to the client and the collateral is burned, but this call removes the slot from the `mySlots()` tracking.

## Validator role

The Validator role represents nodes that help to verify that SP nodes submitted proofs when they were required.  
The smart contract decides whether or not a proof was missed, while the validator triggers the decision-making
function in the smart contract. This is because in a blockchain, a contract cannot change its state without a transaction and gas initiating the state change. The validator nodes then get rewarded for each time they correctly
mark a proof as missing.

Validator nodes MUST observe the slot's space by listening for the `SlotFilled` event, which SHOULD prompt the validator
to add the slot to the watched slots. Then after the end of every period, a validator has at most `config.proofs.timeout` seconds
(config can be retrieved with `getConfig()`) to validate all the slots. If it finds a slot that missed its proof, then 
it SHOULD submit a transaction calling the function `markProofAsMissing(slotId, period)`. This function validates the correctness of the claim,
and if right, will send a reward to the validator.

A Validator MAY decide to validate only part of the slot's space when it detects that it can't validate all watched slots
before the end of the validation `timeout`.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References 

1. [Reed-Soloman algorithm](https://hackmd.io/FB58eZQoTNm-dnhu0Y1XnA)
2. [CIDv1](https://github.com/multiformats/cid#cidv1)
3. [multihash](https://github.com/multiformats/multihash)
4. [Proof-of-Data-Possession](https://hackmd.io/2uRBltuIT7yX0CyczJevYg?view)
5. [Codex market implementation](https://github.com/codex-storage/nim-codex/blob/master/codex/market.nim)
