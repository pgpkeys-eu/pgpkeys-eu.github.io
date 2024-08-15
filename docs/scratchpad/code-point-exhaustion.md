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
(++) The Secret Key Encryption registry automatically includes the code points from the Symmetric Key Algorithms registry.
The Secret Key Encryption registry is therefore the only one that assigns code points greater than 110 -- specifically the S2K algorithms 253, 254 and 255, to avoid code point clash.

The most heavily populated single-octet registries are currently Signature Subpacket Types (51/128 or 102/256) and Public Key Algorithms (28/256).
Other registries are unlikely to ever exceed their capacity, however we define a generic scheme here for future reference.

## Variable-length Encoding of Code Points

We use a similar algorithm to the packet length parameter:

* single-octet encodings have an octet value < 192 and are fully backwards compatible
* two-octet encodings have a first octet in the range 192..223, and represent code points in the range 192..8383
* if the first octet >= 224, this is a legacy single-octet encoding for backwards compatibility

Only defining one- and two-octet encodings allows for legacy single-octet encodings of code points in the range 224..255.
This ensures backwards compatibility of S2K and critical private use subpacket code points (see below).

### Signature Subpacket Criticality

The user-controlled criticality bit (the most-significant bit) in the subpacket type encoding is a misfeature.
We instead explicitly duplicate the entries of the current registry into the range 128 and above, so that it contains explicit non-critical (0..127) and critical (128..255) ranges, similar to the Packet Type registry.
Future Signature Subpacket Type registrations SHOULD only assign either a critical or a non-critical code point, and it MUST NOT be assumed that toggling the critical bit will produce a subpacket with semantics unchanged aside from criticality.

A legacy implementation that encounters a two-octet encoding of a subpacket type will assume it is critical from the most significant bit of the first octet and invalidate the signature.
We therefore define the bottom half of the extended range 256..4287 as critical and reserve the range 4288..8383.
This means that non-critical subpacket code points must remain in short supply for the foreseeable future.
At some point it should be possible to relax this restriction to allow non-critical subpacket types in the range 4288..8383 (bits 7, 6 and 5 of the first octet are 1, and bit 4 is 0).

Note that the private use range (100..110) with the critical bit set (i.e. 228..238) falls within the legacy single-octet encoding range (224..255).

### Private Use Ranges

The existing private use ranges (usually 100..110) may not be sufficient for all use cases.
If a variable-length encoding is in use, the highest 256 code points of each registry (8128..8383) SHOULD be reserved for private use.
In the case of signature subpackets, the highest 256 code points of the critical range (4032..4288) SHOULD also be reserved.

### Signature Malleability

To prevent signature malleability, we require that when hashing trailers, an implementation MUST reverse the octet order of any two-octet encodings of the following fields:

* Signature Type
* Public Key Algorithm Type
* Hash Algorithm Type

v3 keys and signatures MUST NOT use code points in the range 192..223, or attempt to use two-octet code point encodings.

### Support

Implementations MAY support variable-length encodings of code points from any of the registries listed above.
Implementations MUST gracefully ignore variable-length encodings of unknown code points from the following registries:

* Signature Subpacket Types
* Public Key Algorithms

Variable-length code point encodings SHOULD only be used in modern artifacts, e.g.

* v6 key material, signature, and PKESK packets
* SEIPDv2 encrypted data
* etc. ((TBC))

## Packet Types

It may eventually become necessary to also expand the Packet Type registry, which currently has 24 of 64 possible entries allocated.
This can only be done by specifying a novel packet framing.

The two currently supported packet framing systems are distinguished by bit 6, however in both cases bit 7 of the first octet is 1.
Any new packet framing MUST therefore set bit 7 to 0, but MUST NOT prevent further enhancements to packet framing.
It MUST therefore reserve at least one value of the first octet for future use.

### Extended Packet Framing

We use a similar two-octet encoding scheme, however the extended range starts at 64 instead of 192, due to the smaller existing range of Packet Type code points.
The first octet contains:

* Bit 7 is 0
* If bits 6 and 5 are both 1, then bits 4..0 and all eight bits of the second octet indicate the Packet Type (two-octet encoding).
* Values where bit 7 is 0 and bits 6 and 5 are not both 1 are reserved.
* Subsequent octets contain the packet length as in the OpenPGP packet format.

The "OpenPGP" one-octet encoding (bits 7 and 6 both 1) continues to represent code points 0..63 as usual.
The extended two-octet encoding represents code points 64..8255.

By analogy with signature subpackets, the bottom half of the extended range (64..4159) contains critical packet types, of which the top 256 (3904..4159) are private use.
Similarly, the top half of the extended range (4160..8255) contains non-critical packet types, again of which the top 256 code points (8000..8255) are private use.
