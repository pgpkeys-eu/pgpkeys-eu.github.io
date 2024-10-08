# GREASE in OpenPGP

[GREASE](https://datatracker.ietf.org/doc/html/rfc8701/) is a specification to ensure forwards compatibility in TLS.
We wish to specify a similar mechanism for OpenPGP.

## Reserved Values

We reserve the following values that fall within the currently-available code point range of all the above OpenPGP registries:

* 44
* 55
* 66
* 77
* 88
* 99
* 111

Note that these code points differ from those allocated in TLS-GREASE.
These code points are reserved for PGP-GREASE code points in the following registries:

* OpenPGP String-to-Key (S2K) Types
* OpenPGP User Attribute Subpacket Types
* OpenPGP Image Attribute Encoding Format
* OpenPGP Signature Subpacket Types (+)
* OpenPGP Reason for Revocation (Revocation Octet)
* OpenPGP Public Key Algorithms
* OpenPGP Symmetric Key Algorithms
* OpenPGP Hash Algorithms
* OpenPGP Compression Algorithms
* OpenPGP Secret Key Encryption (S2K Usage Octet)
* OpenPGP Image Attribute Versions
* OpenPGP AEAD Algorithms
* OpenPGP Key and Signature Versions

(+) When PGP-GREASE codepoints are used in the Signature Subpacket Types registry, the critical bit MUST NOT be set.

PGP-GREASE code points are not allocated in the Signature Types or Packet Types registries.
The use of signature types is defined by the packet grammar and is therefore not flexible.
Packet types 44 and 55 fall in the non-critical range, however implementations pre-dating RFC9580 will not support non-critical packet types and so cannot be expected to cleanly ignore them.

### Variable-Length Values

TLS-GREASE reserves the following two-byte code points:

* 0x0A0A (2570)
* 0x1A1A (6682)
* 0x2A2A (10794)
* 0x3A3A (14906)
* 0x4A4A (19018)
* 0x5A5A (23130)
* 0x6A6A (27242)
* 0x7A7A (31354)
* 0x8A8A (35466)
* 0x9A9A (39578)
* 0xAAAA (43690)
* 0xBABA (47802)
* 0xCACA (51914)
* 0xDADA (56026)
* 0xEAEA (60138)
* 0xFAFA (64250)

If [variable-length code point encoding](code-point-exhaustion.html) is supported, these MAY be used as PGP-GREASE code points in the registries listed above.
