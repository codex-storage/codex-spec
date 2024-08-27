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
The manifest is the default configuration for a Codex client. 
The file contains key-value pairs that are configuirable.
The structured format used is JSON.

## Manifest Parameters

```json

"manifest" : {
	"treeCid" : "string"
	"datasetSize" : ,
	"codec" : ,
	"hcodec" : ,
	"version" : ,
	"protected" : "bool",
	"ecK" : "int",
	"ecM" : "int",
	"originalTreeCid" : ,
	"originalDatasetSize" :
	"protectedStrategy" :
	"verifiable" : ,
	"verifyRoot" : ,
	"slotRoots" : ,
	"cellSize" : ,
	"verifiableStrategy" : ,
	
}


```

`treeCid`

- 
