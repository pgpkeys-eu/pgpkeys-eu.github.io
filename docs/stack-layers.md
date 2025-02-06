# OpenPGP Stack Layering

The OpenPGP application stack can be roughly considered to be divided into layers.
These layers have no official meaning, and are somewhat fluid.
They are however useful as a mental model, particularly when defining extensions to OpenPGP.

[RFC9580](https://datatracker.ietf.org/doc/html/rfc9580) fully specifies only layers 2a, 2b, and 2f.

## Layer 1: Cryptographic primitives

* [Symmetric algorithms](https://datatracker.ietf.org/doc/html/rfc9580#name-symmetric-key-algorithms)
* [Asymmetric algorithms](https://datatracker.ietf.org/doc/html/rfc9580#name-public-key-algorithms)
* [Digest algorithms](https://datatracker.ietf.org/doc/html/rfc9580#name-hash-algorithms)
* [AEAD modes](https://datatracker.ietf.org/doc/html/rfc9580#name-aead-algorithms)
* CSPRNGs
* [PKCS1](https://datatracker.ietf.org/doc/html/rfc9580#name-pkcs1-encoding-in-openpgp)
* [Elliptic Curves](https://datatracker.ietf.org/doc/html/rfc9580#name-ecc-curves-for-openpgp)

## Layer 2: OpenPGP

### Layer 2a: Data representation

* [UTF-8](https://datatracker.ietf.org/doc/html/rfc9580#name-text)
* [MPIs](https://datatracker.ietf.org/doc/html/rfc9580#name-multiprecision-integers)
* [Big-endian numbers](https://datatracker.ietf.org/doc/html/rfc9580#name-scalar-numbers)
* [Timestamps](https://datatracker.ietf.org/doc/html/rfc9580#name-time-fields)
* [S2K](https://datatracker.ietf.org/doc/html/rfc9580#name-string-to-key-s2k-specifier)
* [Algorithm IDs](https://datatracker.ietf.org/doc/html/rfc9580#name-constants)
* [Regular expressions](https://datatracker.ietf.org/doc/html/rfc9580#name-regular-expressions)
* [Base64](https://datatracker.ietf.org/doc/html/rfc9580#name-base64-conversions)

### Layer 2b: Packet structure

* [Packet framing](https://datatracker.ietf.org/doc/html/rfc9580#name-packet-headers)
    * Legacy and OpenPGP packet formats
    * Partial packet lengths
* [Packet types](https://datatracker.ietf.org/doc/html/rfc9580#name-packet-types)
* [Subpacket types](https://datatracker.ietf.org/doc/html/rfc9580#name-signature-subpacket-types)
* [Digest construction](https://datatracker.ietf.org/doc/html/rfc9580#name-computing-signatures)
    * [Salting](https://datatracker.ietf.org/doc/html/rfc9580#name-advantages-of-salted-signat)
    * Subject preprocessing
    * Trailers
    * Algorithm-specific signature data
* [Algorithm-specific ESK data](https://datatracker.ietf.org/doc/html/rfc9580#name-algorithm-specific-parts-of)
* [KDFs]()

### Layer 2c: Packet grammar

* [Messages](https://datatracker.ietf.org/doc/html/rfc9580#name-openpgp-messages)
    * Document signature types (0x00..0x0f)
    * Timestamp signatures (0x40)
    * [Literal data](https://datatracker.ietf.org/doc/html/rfc9580#name-literal-data-packet-type-id)
    * [Compression](https://datatracker.ietf.org/doc/html/rfc9580#name-compressed-data-packet-type)
    * [Sign-then-encrypt](https://datatracker.ietf.org/doc/html/rfc9580#name-symmetrically-encrypted-and)
    * [One-pass signatures](https://datatracker.ietf.org/doc/html/rfc9580#name-one-pass-signature-packet-t)
        * Nesting
    * [Intended recipient](https://datatracker.ietf.org/doc/html/rfc9580#name-intended-recipient-fingerpr)
* [Certificates (TPKs)](https://datatracker.ietf.org/doc/html/rfc9580#name-transferable-public-keys)
    * [Key material](https://datatracker.ietf.org/doc/html/rfc9580#name-key-material-packets)
    * [User IDs](https://datatracker.ietf.org/doc/html/rfc9580#name-user-id-packet-type-id-13) and [User Attributes](https://datatracker.ietf.org/doc/html/rfc9580#name-user-attribute-packet-type-)
    * Binding and certification signature types (0x10..0x1f)
    * Revocation signature types (0x20..0x3f)
    * [Criticality](https://datatracker.ietf.org/doc/html/rfc9580#name-signature-subpacket-specifi)
    * [Exportability](https://datatracker.ietf.org/doc/html/rfc9580#name-exportable-certification)
* Oddities
    * [Detached signatures](https://datatracker.ietf.org/doc/html/rfc9580#name-detached-signatures)
    * Bare revocations
    * Third-party confirmation signatures (0x50)

### Layer 2d: Temporal evolution

* Selfsig precedence
* [Cumulation of signatures](https://datatracker.ietf.org/doc/html/draft-gallagher-openpgp-signatures#name-cumulation-of-signatures)
* [Expiry](https://datatracker.ietf.org/doc/html/draft-gallagher-openpgp-signatures#name-key-and-certification-valid)
* [Revocation](https://datatracker.ietf.org/doc/html/draft-dkg-openpgp-revocation)

### Layer 2e: PGPKI

* [Web of Trust](https://sequoia-pgp.gitlab.io/sequoia-wot/)

### Layer 2f: Packet sequence encoding

* [ASCII armor](https://datatracker.ietf.org/doc/html/rfc9580#name-forming-ascii-armor)
* [Cleartext signature framework](https://datatracker.ietf.org/doc/html/rfc9580#name-cleartext-signature-framewo)

## Layer 3: Application

* [Document signature semantics](https://datatracker.ietf.org/doc/html/draft-gallagher-openpgp-signatures)
* [Notation data](https://datatracker.ietf.org/doc/html/rfc9580#name-notation-data)
* [PGP/MIME](https://datatracker.ietf.org/doc/html/rfc3156)
* [Key distribution and discovery](https://datatracker.ietf.org/doc/html/draft-gallagher-openpgp-hkp)
* [Keyrings](https://datatracker.ietf.org/doc/html/rfc9580#name-keyrings)

# Validity

In OpenPGP, the word "valid" is used liberally - but there are at least five kinds of "validity" that must be distinguished:

1. formal validity (layer 2b)
    * packet is well-formed and parseable
2. cryptographic validity (layer 2a)
    * mathematically incorrect signature
    * incorrect digest ("implausible martian")
3. structural validity (layer 2c)
    * missing required packets ("evaporated key")
    * disordered packets
    * missing self-signatures (unbound signable packet)
    * incorrect signature type ("structural martian")
4. temporal validity (layer 2d)
    * expired
    * revoked
        * hard and soft
    * post-dated
5. issuer validity (layer 2e)
    * uncertified
    * incomplete certification chain
    * insufficient certification weight
    * lack of provenance
    * identity mismatch

In addition, there are other forms of breakage that fall outside the common usage of "validity", such as malformed encodings at the packet level (below) and the sequence encoding level (above).

Andrew Gallagher, 6th February 2025
