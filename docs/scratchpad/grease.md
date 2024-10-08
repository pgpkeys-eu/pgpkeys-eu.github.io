# GREASE in OpenPGP

[GREASE](https://datatracker.ietf.org/doc/html/rfc8701/) is a specification to ensure forwards compatibility in TLS.
We wish to specify a similar mechanism for OpenPGP.

## Usage

An implementation MAY insert dummy packets or subpackets into otherwise valid packet sequences.
It MAY also include dummy code points in preference lists.
The data section of dummy packets and subpackets SHOULD contain only the ten octets "PGP-GREASE", in UTF-8 encoding.
All other fields of dummy packets and subpackets SHOULD correspond to those of a real packet or subpacket.

A generating implementation SHOULD choose code points deterministically, while ensuring that all PGP-GREASE code points are eventually used.
This can be achieved by seeding a counter using contextual data, and incrementing it with each PGP-GREASE code point used.
This ensures that any errors encountered are reproducible, and mitigates against fingerprinting.

A receiving implementation MUST gracefully ignore unknown code points as required by RFC9580, including PGP-GREASE code points.
It MUST NOT treat PGP-GREASE code points any differently from other unknown code points; this includes detecting or indicating the presence of PGP-GREASE code points.

PGP-GREASE codepoints MAY be used in either TPKs or messages.

Note that implementations pre-dating RFC9580 are not required to ignore unknown packet types, so care SHOULD be taken when generating them.
If in doubt, a Marker packet MAY be used instead.

### TPKs

An implementation MAY use PGP-GREASE codepoints in the following TPK contexts:

* Dummy algorithm IDs in algorithm-preference signature subpackets (subpacket types 11, 21, 22, and 39).
* Signature subpacket IDs of dummy signature subpackets, in either the hashed or unhashed area.
* User attribute subpacket types, versions and encoding formats of dummy user attribute subpackets.
* Packet versions and algorithm IDs in dummy subkey packets.
* Signature and hash algorithm IDs and signature version numbers in dummy third-party certification signatures.
* Hash algorithm IDs in dummy self-signatures.
* Packet and signature types in dummy packets.

The counter is initialised with the first octet of the primary key fingerprint.

In protocols that support keyrings containing multiple TPKs, a generating implementation MAY also generate dummy primary keys with PGP-GREASE version numbers or algorithm IDs.
In such a case, the counter from the previous TPK is used.

### Messages

An implementation MAY use PGP-GREASE codepoints in the following message contexts:

* Signature subpacket IDs of dummy signature subpackets in the hashed or unhashed signature subpacket area.
* Signature and hash algorithm IDs and signature version numbers in dummy document signature packets and their corresponding OPS packets.
* Packet and signature types in dummy packets.

The counter is initialised with the first octet of the signer's primary key fingerprint.

Note that when an OPS packet is present, any PGP-GREASE code points it contains will be duplicated in the corresponding signature packet.
These extra copies do not increment the counter.

## Reserved Code Points

We reserve the following one-octet code points for PGP-GREASE.
Note that these differ from the one-octet code points allocated in TLS-GREASE.

Sequence| Code Point
--------|-----------
0       | 15
1       | 44
2       | 55
3       | 66
4       | 77
5       | 88
6       | 99
7       | 111

Code points SHOULD be chosen based on the least-significant three bits in the counter, using the sequence given.
When PGP-GREASE code points from the Signature Subpacket Types registry are used, the critical bit MUST NOT be set.

These code points will be reserved for PGP-GREASE in the following registries:

* OpenPGP String-to-Key (S2K) Types (+)
* OpenPGP User Attribute Subpacket Types
* OpenPGP Image Attribute Encoding Format
* OpenPGP Signature Subpacket Types
* OpenPGP Reason for Revocation (Revocation Octet) (+)
* OpenPGP Public Key Algorithms
* OpenPGP Symmetric Key Algorithms
* OpenPGP Hash Algorithms
* OpenPGP Compression Algorithms
* OpenPGP Secret Key Encryption (S2K Usage Octet) (++)
* OpenPGP Signature Types
* OpenPGP Image Attribute Versions
* OpenPGP AEAD Algorithms
* OpenPGP Key and Signature Versions

(+) The use of PGP-GREASE code points in these registries is not currently specified, however the code points will be reserved for consistency.
(++) The Secret Key Encryption registry automatically contains all entries from the Symmetric Key Algorithms registry.

In addition, code point 55 (only) will be reserved for PGP-GREASE in the OpenPGP Packet Types registry.

Note that the [OpenPGP Interoperability Test Suite](https://tests.sequoia-pgp.org/#Detached_signatures_with_unknown_packets) currently uses signature version 23 as a de-facto GREASE code point.

### Variable-Length Values

TLS-GREASE reserves the following two-octet code points:

Sequence| Code Point
--------|---------------
0       | 0x0A0A (2570)
1       | 0x1A1A (6682)
2       | 0x2A2A (10794)
3       | 0x3A3A (14906)
4       | 0x4A4A (19018)
5       | 0x5A5A (23130)
6       | 0x6A6A (27242)
7       | 0x7A7A (31354)
8       | 0x8A8A (35466)
9       | 0x9A9A (39578)
10      | 0xAAAA (43690)
11      | 0xBABA (47802)
12      | 0xCACA (51914)
13      | 0xDADA (56026)
14      | 0xEAEA (60138)
15      | 0xFAFA (64250)

If [variable-length code point encoding](code-point-exhaustion.html) is supported, these MAY be used as additional PGP-GREASE code points in the registries listed above.
They SHOULD be chosen based on the least-significant four bits of the counter, using the sequence given.
