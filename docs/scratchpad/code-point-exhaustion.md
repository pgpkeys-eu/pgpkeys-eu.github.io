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

We use a similar algorithm to the packet length parameter:

* single-octet encodings have an octet value < 192 and are fully backwards compatible
* two-octet encodings have a first octet in the range 192..223, and represent code points in the range 192..8383
* if the first octet >= 224, this is a legacy single-octet encoding for backwards compatibility

Only defining one- and two-octet encodings allows us to reserve legacy single-octet encodings for code points in the range 224..255.
This ensures backwards compatibility of secret key encryption and critical private use subpacket code points (see below).

### Secret Key Encryption (S2K)

Since Secret Key Encryption code points greater than 224 are in current use, they MUST be represented by the legacy single-octet encoding.

### Signature Subpacket Criticality

The criticality bit (the most-significant bit) in the subpacket type encoding is a misfeature.
Most subpacket types are only functional when the criticality bit is set to one or other value, depending on their specification (see [Subpacket Classes](subpacket-classes.html) for details).

We instead explicitly duplicate the existing entries of the Signature Subpacket Type registry into the range 128 and above, so that it contains explicit non-critical (0..127) and critical (128..191) ranges, similar to the Packet Type registry, that can be represented by the single-octet encoding.
Future Signature Subpacket Type registrations SHOULD only assign either a critical or a non-critical code point, and it MUST NOT be assumed that toggling the critical bit will produce a subpacket with a compatible specification.

We define the bottom half of the extended range 192..4287 as critical (high nybble of the first octet is 0xC) and reserve the range 4288..8383 for future non-critical subpackets (high nybble of the first octet is 0xD).
A legacy implementation that encounters a two-octet encoding of a subpacket type will assume it is critical from the most significant bit of the first octet and invalidate the signature.
This means that non-critical subpacket code points in the extended range must remain unassigned for the foreseeable future.
At some point it may be possible to relax this restriction to allow allocation of non-critical subpacket types in the extended range.

Note that the private use range (100..110) with the critical bit set (i.e. 228..238) falls within the legacy single-octet encoding range (224..255).
Since these code points are currently in use, they MUST be represented by the legacy single-octet encoding.

### Private Use Ranges

The existing private use ranges (usually 100..110) may not be sufficient for all use cases.
If a variable-length encoding is in use, the highest 256 code points of the extended range (8128..8383) SHOULD be reserved for private use.
In the case of signature subpackets, the highest 256 code points of the extended critical range (4032..4287) SHOULD also be reserved.

### Signature Malleability

To prevent signature malleability, in the (highly unlikely!) event that two-octet signature version encodings are ever defined, the signature version number immediately preceding the 0xFF octet in the trailer (fifth octet from the end) MUST be supplied with its octets in reverse order.

### Support

Implementations MAY support variable-length encodings of code points from any of the registries listed above.
Implementations MUST gracefully ignore variable-length encodings of unknown code points from the following registries:

* OpenPGP Signature Subpacket Types
* OpenPGP Public Key Algorithms
* OpenPGP Symmetric Key Algorithms
* OpenPGP Hash Algorithms
* OpenPGP Compression Algorithms
* OpenPGP Secret Key Encryption (S2K Usage Octet)
* OpenPGP AEAD Algorithms

Special care MUST be taken when parsing the following signature subpackets, where unknown code points MUST be gracefully ignored, and the second octet of a two-octet encoding might therefore be misinterpreted:

* Preferred Symmetric Ciphers subpacket (array of code points)
* Preferred AEAD Ciphersuites subpacket (array of code point tuples)
* Preferred Hash Algorithms (array of code points)
* Preferred Compression Algorithms (array of code points)

Note also that assignment of two-octet Secret Key Encryption code points will affect the construction of String-to-Key Specifiers.

Variable-length code point encodings MUST only appear in a modern OpenPGP packet sequence, i.e.

* a key material, signature, OPS, SKESK, or PKESK packet of version 6 or later
* a User Attribute packet attached to a key of version 6 or later
* an SEIPD packet of version 2 or later
* a compressed data packet signed by a signature of version 6 or later, and/or encrypted in an SEIPD packet of version 2 or later

## Packet Types

It may eventually become necessary to also expand the Packet Type registry, which currently has 24 of 64 possible entries allocated.
This can only be done by specifying a novel packet framing.

The two currently supported packet framing systems are distinguished by bit 6, however in both cases bit 7 of the first octet is 1.
Any new packet framing MUST therefore set bit 7 to 0, but MUST NOT prevent further enhancements to packet framing.
It MUST therefore reserve at least one value of the first octet for future use.

### Extended Packet Framing

We use a similar two-octet encoding scheme to represent 8192 extra code points, however the extended range starts at 64 instead of 192, due to the smaller existing range of Packet Type code points.
The first octet contains:

* Bit 7 is 0
* If bits 6 and 5 are both 1, then bits 4..0 and all eight bits of the second octet indicate the Packet Type (two-octet encoding).
* Values where bit 7 is 0 and bits 6 and 5 are not both 1 are reserved.
* Subsequent octets contain the packet length as in the OpenPGP packet format.

The "OpenPGP" one-octet encoding (bits 7 and 6 both 1) continues to represent code points 0..63 as usual.
The extended two-octet encoding represents code points 64..8255.

By analogy with signature subpackets, the bottom half of the extended range (64..4159, high nybble of the first octet is 0x4) contains critical packet types, of which the top 256 (3904..4159) are private use.
Similarly, the top half of the extended range (4160..8255, high nybble of the first octet is 0x05) contains non-critical packet types, again of which the top 256 code points (8000..8255) are private use.
