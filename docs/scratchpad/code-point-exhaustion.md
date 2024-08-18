# Code Point Exhaustion

We are motivated to define a variable-length encoding for OpenPGP code points so that we can have more than 255 of each.
While there is no immediate prospect of any of the OpenPGP registries exceeding 255 entries, problems have already been encountered with the small size of the private use ranges (typically the code points 100..110).
It is also prudent to define and reserve an extension scheme significantly in advance of it becoming necessary.

## Code Point Registries

The following OpenPGP registries currently define single-octet code points with up to 256 values:

* OpenPGP String-to-Key (S2K) Types
* OpenPGP User Attribute Subpacket Types
* OpenPGP Image Attribute Encoding Format
* OpenPGP Signature Subpacket Types (+)
* OpenPGP Reason for Revocation (Revocation Octet)
* OpenPGP Public Key Algorithms
* OpenPGP Symmetric Key Algorithms (++)
* OpenPGP Hash Algorithms
* OpenPGP Compression Algorithms
* OpenPGP Secret Key Encryption (S2K Usage Octet) (++)
* OpenPGP Signature Types
* OpenPGP Image Attribute Versions
* OpenPGP AEAD Algorithms
* OpenPGP Key and Signature Versions

(+) The Signature Subpacket Types registry has only 128 usable code points in practice, due to the criticality bit (see below).

(++) The Secret Key Encryption registry automatically includes the code points from the Symmetric Key Algorithms registry, although their use is deprecated.
The Secret Key Encryption registry is therefore the only one that assigns code points greater than 110 -- specifically the S2K algorithms 253, 254 and 255, to avoid code point clash.

In addition, the following single-octet identifiers have no associated registry:

* SEIPD packet version
* Literal data packet format (printable ASCII character)

The most heavily populated single-octet registries are currently Signature Subpacket Types (51/128 or 102/256) and Public Key Algorithms (28/256).
Other registries are unlikely to ever exceed their capacity, however we define a generic scheme here for future reference.

## Variable-length Encoding of Code Points

We specify two methods for variable length encoding, one generic and one for signature subpacket types only.

### UTF-8 Encoding

For registries other than signature subpacket types, we use the same encoding as UTF-8, i.e.

```
if the 1st octet <  128, then
    lengthOfEncoding = 1
    codePoint = 1st_octet

if the 1st octet >= 192 and < 224, then
    lengthOfEncoding = 2
    codePoint = ((1st_octet - 192) << 6) + (2nd_octet - 128)

if the 1st octet >= 224 and < 240, then
    lengthOfEncoding = 3
    codePoint = ((1st_octet - 224) << 12) + ((2nd_octet - 128) << 6) + (3rd_octet - 128)

if the 1st octet >= 240, then
    lengthOfEncoding = 1
    codePoint = 1st_octet
```

The second and third octets MUST be >= 128 and < 192.
This ensures that none of the octets in a multi-octet encoding can be incorrectly interpreted as a single-octet encoding of another code point.
This means that strings of multi-octet code points are self-synchronising.

* single-octet encodings have an octet value < 128 and are fully backwards compatible
* two-octet encodings have a first octet in the range 192..223, and represent code points in the range 128..2047
* three-octet encodings have a first octet in the range 224..239, and represent code points in the range 2048..65535
* legacy single-octet encodings have an octet value in the range 240..255
* Overlong encodings MUST NOT be used unless specifically exempted (see below)

Omitting four-octet encodings allows us to reserve legacy single-octet encodings for code points in the range 240..255.
This ensures backwards compatibility of existing secret key encryption (S2K Usage) code points.
Since Secret Key Encryption code points greater than 240 are in current use, they MUST be represented by the legacy single-octet encoding.

Three-octet encodings (code points 2048..65535) are reserved for private use.

Note that since all OpenPGP implementations MUST support UTF-8, it MAY be convenient to hand off calculation of this algorithm to UTF-8 encoding/decoding routines, provided that these do not perform any Unicode validation (e.g. checking for invalid code points or surrogates, mapping to canonical forms, etc.).

### Subpacket Type Encoding

The criticality bit means that the first octet of any multi-octet subpacket type encoding is effectively limited to values less than 128.
We therefore cannot use UTF-8 encoding for subpacket types, and cannot make use of its self-synchronisation properties.
Luckily, subpacket types are only found as the second field of a subpacket, immediately after the packet length, so self-synchronisation is not required.

We define a similar algorithm to the signature subpacket length encoding:

```
if the 1st octet <  64, then
    lengthOfEncoding = 1
    codePoint = 1st_octet

if the 1st octet >= 64 and < 128, then
    lengthOfEncoding = 2
    codePoint = ((1st_octet - 64) << 8) + (2nd_octet) + 64
```

* single-octet encodings have an octet value < 64 and are fully backwards compatible with code points in current use
* two-octet encodings have a first octet in the range 64..127, and represent code points in the range 64..16447.

Note that the criticality bit MUST be zeroed before applying the above algorithm.

Note that a one-octet legacy encoding of the private use range (100..110) falls within the two-octet encoding range of first-octet values.
For backwards compatibility, we assign all two-octet encodings with the first octet in this range as private use, i.e. code points 9280..12095.

### Support

Implementations SHOULD support variable-length encoding of Signature Subpacket Types.

Implementations MUST gracefully ignore variable-length encodings of unknown code points, and MAY support known variable-lengeth code points, in the following registries:

* OpenPGP Public Key Algorithms
* OpenPGP Symmetric Key Algorithms
* OpenPGP Hash Algorithms
* OpenPGP Compression Algorithms
* OpenPGP Secret Key Encryption (S2K Usage Octet)
* OpenPGP AEAD Algorithms

These code points may be found in the following contexts:

* key material packets
* signature packets and subpackets
* PKESK and SKESK packets
* OPS packets
* compressed data packets

Special care should be taken when parsing the following signature subpackets, which consist of arrays/strings of code points in which unknown code points MUST be gracefully ignored.

* Preferred Symmetric Ciphers subpacket (array of Symmetric Key Algorithm code points)
* Preferred AEAD Ciphersuites subpacket (array of Symmetric Key Algorithm, AEAD Algorithm code point tuples)
* Preferred Hash Algorithms (array of Hash Algorithm code points)
* Preferred Compression Algorithms (array of Compression Algorithm code points)

Note that three-octet encodings of private use code points in the Preferred AEAD Ciphersuites subpacket should be backwards compatible with legacy code, because encoded tuples always have an even number of octets and the parser should therefore skip over them correctly.
Beware however that two-octet encodings may result in desynchronisation of the tuple parsing in legacy code.
Therefore, when code points between 128..2047 are used in a Preferred AEAD Ciphersuites subpacket, an overlong three-octet encoding MUST be used.

Implementations MAY support variable-length encodings of code points from the following registries:

* Registries for code points used only in signature or user attribute subpackets:
    * OpenPGP Reason for Revocation (Revocation Octet)
    * OpenPGP Image Attribute Versions
    * OpenPGP User Attribute Subpacket Types
    * OpenPGP Image Attribute Encoding Format
* Other registries:
    * OpenPGP String-to-Key (S2K) Types
    * OpenPGP Signature Types
    * OpenPGP Key and Signature Versions

Variable-length code point encodings MUST only appear in a modern OpenPGP packet sequence, i.e.

* a key material, signature, OPS, SKESK, or PKESK packet of version 6 or later
* a User Attribute packet attached to a key of version 6 or later
* an SEIPD packet of version 2 or later
* a compressed data packet signed by a signature of version 6 or later, and/or encrypted in an SEIPD packet of version 2 or later

Note also that assignment of variable-length Secret Key Encryption code points will affect the construction of String-to-Key Specifiers.

(( TODO: provide examples ))

## Packet Types

It may eventually become necessary to also expand the Packet Type registry, which currently has 24 of 64 possible entries allocated.
This can only be done by specifying a novel packet framing.

The two currently supported packet framing systems are distinguished by bit 6, however in both cases bit 7 of the first octet is 1.
Any new packet framing MUST therefore set bit 7 to 0, but MUST NOT prevent further enhancements to packet framing.
It MUST therefore reserve at least one value of the first octet for future use.

### Extended OpenPGP Packet Framing

The first octet contains:

* Bit 7 is 0
* If bits 6 and 5 are both 1, then bits 4..0 and all eight bits of the second octet indicate the Packet Type (two-octet encoding).
* Values where bit 7 is 0 and bits 6 and 5 are not both 1 are reserved.
* Subsequent octets contain the packet length in OpenPGP packet length format.

The "OpenPGP" one-octet encoding (bits 7 and 6 both 1) continues to represent code points 0..63 as usual.
The extended two-octet encoding represents code points 64..8255.

The bottom half of the extended range (64..4159, high nybble of the first octet is 0x4) contains critical packet types, of which the top 256 (3904..4159) are private use.
Similarly, the top half of the extended range (4160..8255, high nybble of the first octet is 0x05) contains non-critical packet types, again of which the top 256 code points (8000..8255) are private use.
