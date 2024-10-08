# GREASE in OpenPGP

[GREASE](https://datatracker.ietf.org/doc/html/rfc8701/) is a specification to ensure forwards compatibility in TLS.
We wish to specify a similar mechanism for OpenPGP.

## Usage

PGP-GREASE codepoints MAY be used in both TPKs and messages.

A generating implementation SHOULD choose code points deterministically, while ensuring that all PGP-GREASE code points are eventually used.
This can be achieved using a standard checksum over the preceding data, which ensures that any errors encountered are reproducible, and prevents fingerprinting of software.

A receiving implementation MUST gracefully ignore unknown code points as required by RFC9580, including PGP-GREASE code points.
It MUST NOT treat PGP-GREASE code points any differently from other unknown code points; this includes detecting or indicating the presence of PGP-GREASE code points.

### TPKs

An implementation MAY use PGP-GREASE codepoints in the following contexts:

* Dummy entries in algorithm-preference signature subpackets.
* Signature and hash algorithm octets in dummy third-party certification signatures.
* Dummy signature subpackets in the hashed or unhashed signature subpacket area.
* Dummy packet and algorithm versions in subkey packets.

In each case, the checksum is the sum mod 256 of the following octets:

* The first octet of the primary key fingerprint.
* The offset in bytes of the first byte of the packet containing the PGP-GREASE codepoint.
* The packet type containing the PGP-GREASE codepoint.
* (Only for code points in signature packets) The signature type containing the PGP-GREASE codepoint.
* (Only for code points in non-GREASE signature subpackets) The signature subpacket type (including critical bit if set) containing the PGP-GREASE codepoint.

Dummy packets and subpackets SHOULD NOT be of excessive size, and SHOULD contain only the string "PGP-GREASE" in UTF-8 encoding.

### Messages

TBC

## Reserved Values

We reserve the following values that fall within the currently-available code point range of all one-octet OpenPGP registries:

* 44
* 55
* 66
* 77
* 88
* 99
* 111

Note that these code points differ from those allocated in TLS-GREASE.
They MUST be chosen based on the least-significant three bits in the standard checksum, in order starting at 001.
(A value of 000 means no code point).

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

(+) When PGP-GREASE codepoints from the Signature Subpacket Types registry are used, the critical bit MUST NOT be set.

PGP-GREASE code points are not allocated in the Signature Types or Packet Types registries.
The use of signature types is constrained by the packet grammar, and although packet types 44 and 55 fall in the non-critical range, implementations pre-dating RFC9580 cannot be expected to cleanly handle them.

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

If [variable-length code point encoding](code-point-exhaustion.html) is supported, these MAY be used as additional PGP-GREASE code points in the registries listed above.
They MUST be chosen based on the least-significant four bits of the checksum, in order starting at 0000.
