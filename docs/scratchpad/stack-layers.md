# OpenPGP Stack Layering

The OpenPGP application stack can be roughly considered to be divided into layers.
These layers have no official meaning, and are somewhat fluid.
They are however useful as a mental model, particularly when defining extensions to OpenPGP.

[RFC9580](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-crypto-refresh) fully specifies only layers 2a and 2b.

## Layer 1: Cryptographic primitives

* Symmetric algorithms
* Asymmetric algorithms
* Digest algorithms
* CSPRNGs
* PKCS1

## Layer 2: OpenPGP

### Layer 2a: Data representation

* UTF-8
* MPIs
* Big-endian numbers
* Timestamps
* Algorithm IDs
* Regular expressions

### Layer 2b: Packet structure

* Packet framing (Legacy vs OpenPGP packet formats)
* Packet versions
* Signature subpackets
* Digest construction
* KDFs

### Layer 2c: Packet grammar

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

### Layer 2d: Packet sequence encoding

* ASCII armor
* Cleartext signature framework

### Layer 2e: PGPKI

* User IDs
* Certification semantics
* Web of Trust

## Layer 3: Application

* Document signature semantics
* Notation data
