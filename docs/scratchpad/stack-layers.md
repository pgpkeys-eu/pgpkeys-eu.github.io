# OpenPGP Stack Layering

The OpenPGP application stack can be roughly considered to be divided into layers.
These layers have no official meaning, and are somewhat fluid.
They are however useful as a mental model, particularly when defining extensions to OpenPGP.

[RFC9580](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-crypto-refresh) fully specifies only layers 0 and 1.

## Layer 0: Data representation

* UTF-8
* MPIs
* Big-endian numbers
* Timestamps

## Layer 1: Packet structure

* Packet framing (Legacy vs OpenPGP packet formats)
* Packet versions
* Algorithms and algorithm IDs
* Signature subpackets

## Layer 2: Packet grammar

* Messages
    * Sign-then-encrypt
    * One-pass signatures
    * Intended recipient
    * Nesting
* TPKs
    * Selfsig precedence
    * Cumulation of signatures
    * Revocation
    * Criticality
    * Exportability

## Layer 3: PGPKI

* Certification semantics
* Web of Trust

## Layer 4: Application

* Document signature semantics
* Notation data
* ASCII armor
