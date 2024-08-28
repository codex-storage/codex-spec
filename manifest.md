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

## Background
The manifest is used to encode datasets with the a Codex client. 
The file contains key-value pairs.


## Manifest Parameters

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

`treeCid`

- Root of the merkle root
- Hash based on CIDv1

`datasetSize`

- Total size of all blocks, after data is chunked
- it SHOULD be in bytes

`blockSize`

- Size of each contained block

`codec`

- Dataset codec

`hcodec`

- Multihash codec

`version`

- CID version

`protected`

- Protected datasets have erasure coded info
- Only if `true`

`ecK`

- Number of blocks to encode

`ecM`

- Number of resulting parity blocks

`originalTreeCid`

- The original root of the dataset being erasure coded

`originalDatasetSize`

- The size of the original dataset

`protectedStrategy`

- Indexing strategy used to build the slot roots

`verifiable`

- Verifiable datasets can be used to generate storage proofs

`verifyRoot`

- Root of the top level merkle tree built from slot roots

`slotRoots`

- Individual slot root built from the original dataset blocks

`cellSize`

- Size of each slot cell

`verifiableStrategy`

- Indexing strategy used to build the slot roots

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

