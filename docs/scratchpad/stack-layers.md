# OpenPGP Stack Layering

The OpenPGP application stack can be roughly considered to be divided into layers.
These layers have no official meaning, and are somewhat fluid.
They are however useful as a mental model, particularly when defining extensions to OpenPGP.

[RFC9580](https://datatracker.ietf.org/doc/html/rfc9580) fully specifies only layers 2a, 2b, 2c (perhaps) and 2f.

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
* Packet types
* Signature subpacket types
* Digest construction
* KDFs

### Layer 2c: Packet grammar

* Messages
    * Document signature types (0x00..0x0f)
    * Literal data
    * Sign-then-encrypt
    * One-pass signatures
    * Intended recipient
    * Nesting
* TPKs
    * Key material
    * User IDs and attributes
    * Binding and certification signature types (0x10..0x1f)
    * Revocation signature types (0x20..0x3f)
    * Criticality
    * Exportability
* Oddities
    * Timestamp signatures (0x40)
    * Third-party confirmation signatures (0x50)

### Layer 2d: Temporal evolution

* Selfsig precedence
* Cumulation of signatures
* Expiry
* Revocation

### Layer 2e: PGPKI

* User IDs
* Certification semantics
* Web of Trust

### Layer 2f: Packet sequence encoding

* ASCII armor
* Cleartext signature framework

## Layer 3: Application

* Document signature semantics
* Notation data
* PGP/MIME
* Key distribution and discovery

# Validity

In OpenPGP, the word "valid" is used liberally - but there are at least four kinds of "validity" that must be distinguished:

1. cryptographic validity (layer 2b and below)
    * mathematically incorrect signature
    * incorrect digest (implausible martian)
2. structural validity (layer 2c)
    * missing required packets (evaporated key)
    * disordered packets
    * missing self-signatures (unbound signable)
    * incorrect signature type (structural martian)
3. temporal validity (layer 2d)
    * expired
    * revoked
        * hard and soft
    * post-dated
4. claim validity (layer 2e)
    * uncertified
    * incomplete certification chain
    * insufficient certification weight
    * lack of provenance
    * identity mismatch

In addition, there are other forms of breakage that fall outside the common usage of "validity", such as malformed encodings at the packet level (below) and the sequence encoding level (above).
