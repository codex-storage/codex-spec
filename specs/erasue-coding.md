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

1.  



## Security Considerations

