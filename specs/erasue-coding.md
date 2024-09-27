---
title: CODEX-ERASUE-CODING
name: Codex Erasue Coding
status: raw
tags: codex
editor: 
contributors:
---

## Abstract

This specification describes the erasue coding technique used in the Codex protocol.
Erasue coding is used by the Codex client to encode datasets being presented to the [marketplace]().

## Background

Codex uses storage proofs to determine whether a storage provider is storing a certain dataset.
Storage providers agree to store dataset for a period of time and
store an encoded dataset provded by the requester.
Using erasure coding,
client nodes will be able to restore datasets thatare abandoned by storage providers.
Also validator nodes are able to detect whether data is missing within a slot.

## Semantics

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”,
“SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and
“OPTIONAL” in this document are to be interpreted as described in [2119](https://www.ietf.org/rfc/rfc2119.txt).

The Codex client performerasure coding locally before provding dataset to the marketplace.

### Flow

Before data is provided to storage providers on the marketplace,
clients must do the following:

1.  Prepare dataset
2.  Encode data with Reed Solomon erasue coding, more explained below
3.  Derive an CID from encoded chunks share on the marketplace
4.  Error correction by validator nodes once storage contract begins

### Preparing Data

After the user has choosen the prefered data for storage through the marketplace,
the Codex client will divide this data into chunks, e.g. $(c_1, c_2, c_3, \ldots, c_{n})$.
Including the [manifest](manifest), the data chucks will be encoded based on the following parameters:

```js
struct encodingParms {
  ecK: int
  ecM: int
  rounded: int
  steps: int
  blocksCount: int
  strategy: 
}
```
### Encoding Data

With Reed-Solomon algorithm, extra data chunks need to be created for the dataset.
Parity blocks is added to the chucks of data before encoding.
Once data is encoded, it is prepared to be transmitted.


Below is the content of the dag-pb protobuf message 

```protobuf
   Message VerificationInfo {
     bytes verifyRoot = 1;             # Decimal encoded field-element
     repeated bytes slotRoots = 2;     # Decimal encoded field-elements
   }
   Message ErasureInfo {
     optional uint32 ecK = 1;                            # number of encoded blocks
     optional uint32 ecM = 2;                            # number of parity blocks
     optional bytes originalTreeCid = 3;                 # cid of the original dataset
     optional uint32 originalDatasetSize = 4;            # size of the original dataset
     optional VerificationInformation verification = 5;  # verification information
   }

   Message Header {
     optional bytes treeCid = 1;        # cid (root) of the tree
     optional uint32 blockSize = 2;     # size of a single block
     optional uint64 datasetSize = 3;   # size of the dataset
     optional codec: MultiCodec = 4;    # Dataset codec
     optional hcodec: MultiCodec = 5    # Multihash codec
     optional version: CidVersion = 6;  # Cid version
     optional ErasureInfo erasure = 7;  # erasure coding info
   }
```

## Decode Data

There are two node roles that will need to decode data.
- Client nodes to read data
- Validator nodes to verfiy storage providers are storing data as per the marketplace

To ensure data is being stored by by storage providers, data will need to be decoded when vaildator nodes need download data slots.


## Security Considerations

### Encoding Problem

An adversarial storage provider can remove only the first element from more than half of the block, and the slot data can no longer be recovered from the data that the host stores.
For example, with 1TB of slot data erasure coded into 256 data and parity shards, an adversary could strategically remove 129 bytes, and the data can no longer be fully recovered with the erasure coded data that is present on the host.

### Recommended Solution

we should perform our checks on entire shards to protect against adversarial erasure.
In the Merkle storage proofs, we need to hash the entire shard, and then check that hash with a Merkle proof.
Effectively the block size for Merkle proofs should equal the shard size of the erasure coding interleaving. Hashing large amounts of data will be expensive to perform in a SNARK, which is used to compress proofs in size in Codex.



## Copyright

## References
