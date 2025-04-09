---
title: CODEX-MANIFEST
name: Codes Manifest
status: raw
category: Standards Track
tags: codex
editor: 
contributors:
---

## Abstract

This specification describes the parameters used in the manifest for a Codex client application.

## Manifest
The manifest is used to encode datasets with the a Codex client node. 
The file contains key-value pairs.

### Parameters

```json

"manifest" : {
	"treeCid" : "string"
	"datasetSize" : ,
	"blockSize" : ,
	"codec" : ,
	"hcodec" : ,
	"version" : ,
	"protected" : "bool",
	"ecK" : "int",
	"ecM" : "int",
	"originalTreeCid" : ,
	"originalDatasetSize" :
	"protectedStrategy" :
	"verifiable" : "bool" ,
	"verifyRoot" : ,
	"slotRoots" : ,
	"cellSize" : ,
	"verifiableStrategy" : ,
	
}

```

The structured format used is JSON.

| attribute | type | description |
|-----------|------|-------------|
| `treeCid` | string | A hash based on [CIDv1](https://github.com/multiformats/cid#cidv1). The root of the merkle tree |
| `datasetSize` | bytes | Total size of all blocks, after data is chunked. |
| `blockSize` | bytes | Size of each contained block. |
| `codec` | [MultiCodec](https://github.com/multiformats/multicodec) |  A dataset codec. |
| `hcodec` | [MultiCodec](https://github.com/multiformats/multicodec) | A multihash codec. |
| `version` | string | The CID version |
| `protected` | bool | The protected datasets have erasure coded info, If true, `ecK`, `ecM`, `originalTreeCid`, `originalDatasetSize`, and `protectedStrategy` SHOULD be used.|
| `ecK` | int | The number of blocks to encode. |
| `ecM` | int | The number of resulting parity blocks. |
| `originalTreeCid` | string | The original root of the dataset being erasure coded. |
| `originalDatasetSize` | bytes | The size of the original dataset. |
| `protectedStrategy` | enum | An indexing strategy used to build the slot roots. |
| `verifiable` | bool | Verifiable datasets can be used to generate storage proofs. If true, `verifyRoot`, `slotRoots`, `cellSize`, and `verifiableStrategy` SHOULD be used. |
| `verifyRoot` | string | The root of the top level merkle tree built from slot roots. |
| `slotRoots` | array[string] | Individual slot root built from the original dataset blocks. |
| `cellSize` | bytes | The size of each slot cell. |
| `verifiableStrategy` | enum | Indexing strategy used to build the slot roots. |

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

- [MultiCodec](https://github.com/multiformats/multicodec)
- [CIDv1](https://github.com/multiformats/cid#cidv1)

