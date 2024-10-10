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
* PKESK packet version
* SKESK packet version
* Literal data packet format (printable ASCII character)

The most heavily populated single-octet registries are currently Signature Subpacket Types (51/128 or 102/256) and Public Key Algorithms (28/256).
Other registries are unlikely to ever exceed their capacity, however we define a generic scheme here for future reference.

## Variable-length Encoding of Code Points

We specify two methods for variable length encoding, one generic and one for signature subpacket types only.

### UTF-8ish Encoding

For registries other than signature subpacket types, we use a similar encoding scheme to UTF-8, but with four-octet encodings omitted, i.e.

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
* octet values in the range 240..247 correspond to the first octet of four-octet UTF-8 encodings and MUST NOT be used
* legacy single-octet encodings have an octet value in the range 248..255
* overlong encodings MUST NOT be used

We reserve octet values in the range 248..255, which are not used in UTF-8, for legacy single-octet encodings of code points with the same values.
This ensures backwards compatibility of existing secret key encryption (S2K Usage) code points.
Since Secret Key Encryption code points in this range are in current use, they MUST be represented by legacy single-octet encodings.

Three-octet encodings (code points 2048..65535) are reserved for private use.

Note that since all OpenPGP implementations MUST support UTF-8, it MAY be convenient to hand off calculation of this algorithm to UTF-8 encoding/decoding routines, provided that these do not perform any Unicode validation (e.g. checking for invalid code points or surrogates, mapping to canonical forms, etc.).

### Subpacket Type Encoding

The criticality bit means that the first octet of any multi-octet subpacket type encoding is effectively limited to values less than 128.
We therefore cannot use UTF-8ish encoding for subpacket types, and cannot make use of its self-synchronisation properties.
Luckily, subpacket types are only found as the second field of a subpacket, immediately after the packet length, so self-synchronisation is not required.

We define a similar algorithm to the signature subpacket length encoding:

```
if the 1st octet < 112, then
    lengthOfEncoding = 1
    codePoint = 1st_octet

if the 1st octet >= 112 and < 128, then
    lengthOfEncoding = 2
    codePoint = ((1st_octet - 112) << 8) + (2nd_octet) + 112
```

* single-octet encodings have an octet value < 112 and are fully backwards compatible with code points in current use
* two-octet encodings have a first octet in the range 112..127, and represent code points in the range 112..3951

The criticality bit MUST be zeroed before applying the above algorithm.

Two-octet encodings with a first octet in the range 120..127 (code points 2160..3951) are reserved for private use.

### Packet Type Encoding

It may eventually become necessary to also expand the Packet Type registry, which currently has 24 of 64 possible entries allocated.

For historical reasons packet Tag 16 was never allocated, and has the advantage of being unusable in Legacy packet framing.
We therefore reserve it to indicate that the actual packet type follows the length field, in UTF-8ish encoding.

The "OpenPGP" encoding continues to represent code points 0..63 as usual.
UTF-8ish encoding is used to represent code points 64..65535, and MUST NOT be used to represent code points 0..63.

If UTF-8ish encoding of Packet Types is implemented:

* one-octet encodings represent code points in the range 64..127
* two-octet encodings have a first octet in the range 192..223, and represent code points in the range 128..2047
    * the bottom half of the two-octet UTF-8ish encoding range (63..1023) represents critical packet types
    * the top half (1024..2047) represents non-critical packet types
* three-octet encodings are reserved for private or experimental use
* four-octet encodings MUST NOT be used

## Support and Compatibility

Implementations SHOULD support variable-length encoding of Signature Subpacket Types.

Implementations MUST gracefully ignore variable-length encodings of unknown code points, and MAY support known code points with variable-length encodings, in the following registries:

* OpenPGP Public Key Algorithms
* OpenPGP Symmetric Key Algorithms
* OpenPGP Hash Algorithms
* OpenPGP Compression Algorithms
* OpenPGP AEAD Algorithms

These code points may be found in the following contexts:

* key material packets
* signature packets and subpackets
* OPS packets
* PKESK and SKESK packets
* compressed data packets

See the subsections below for further discussion of each of these contexts.

Implementations MAY support variable-length encodings of code points from the following registries:

* OpenPGP String-to-Key (S2K) Types
* OpenPGP User Attribute Subpacket Types
* OpenPGP Image Attribute Encoding Format
* OpenPGP Reason for Revocation (Revocation Octet)
* OpenPGP Secret Key Encryption (S2K Usage Octet)
* OpenPGP Signature Types
* OpenPGP Image Attribute Versions
* OpenPGP Key and Signature Versions
* OpenPGP Packet Types

UTF-8ish encodings should be legacy-safe in these contexts as parsing is normally halted immediately if the code point in question is not supported.
The same argument applies in principle to the non-registry SEIPD, PKESK and SKESK packet version numbers.
Use of the non-registry literal data packet format identifier is deprecated, however use of UTF-8 in this field would be a natural extension.

Variable-length code point encodings MUST only appear in a modern OpenPGP packet sequence, i.e.

* a key material, signature, OPS, SKESK, or PKESK packet of version 6 or later
* a User Attribute packet attached to a key of version 6 or later
* an SEIPD packet of version 2 or later
* a compressed data packet signed by a signature of version 6 or later, and/or encrypted in an SEIPD packet of version 2 or later

### v6 Key Material Packets

After the packet version identifier, a v6 key material packet contains the following fields:

* Creation time
* Public key algorithm
* Length of public key material

It is therefore theoretically possible for a continuation octet of a UTF-8ish encoding of the public key algorithm to be misinterpreted as the first octet of the following length parameter, resulting in a mis-parsing of the remaning packet data, and a possible out of bounds read.
This introduces no new vulnerabilities however, as an attacker can already create a key material packet with an incorrect "length of public key material" parameter, and so a compliant implementation SHOULD already have protections against out of bounds read of the remaining packet.

A secret key material packet further contains a Secret Key Encryption (S2K Usage) parameter, followed by variable fields that cannot be parsed if the Secret Key Encryption parameter is not supported.
These variable fields may include an S2K specifier, which always begins with an S2K specifier type.
The remainder of the S2K specifier cannot be parsed if the S2K specifier type is not supported.

UTF-8ish encodings are therefore safe in v6 key material packets.

### v6 Signature Packets

After the packet version identifier, a v6 signature packet contains a string of three potentially variable-length code point fields:

* Signature Type ID
* public key algorithm
* hash algorithm

The remainder of the packet cannot be parsed if the public key and hash algorithm code points are unsupported, and a signature digest cannot be constructed if the signature type is unsupported.

The trailer contains the same variable-length fields, however the following features prevent malleability:

* The trailer contains a length field that covers the variable-length fields.
* All octets used in UTF-8ish encoding map to currently unassigned octet values.

UTF-8ish encodings are therefore safe in v6 signature packets.

### Signature Subpackets

Signature subpackets are prefixed by a length and a type parameter.
The remainder of each subpacket cannot be parsed if the type parameter is not supported.

Variable-length encodings are generally safe in signature subpackets, subject to constraints:

#### Revocation Key

The Revocation Key subpacket contains a symmetric algorithm ID field, however it is only defined for v4 targets and is now deprecated.
UTF-8ish encoded code points are therefore not valid in this subpacket and MUST NOT be used.

#### Signature Target

The Signature Target subpacket contains three fields:

* symmetric key algorithm
* hash algorithm
* hashed data

The hashed data cannot be parsed if the hash algorithm is unknown.

#### Reason for Revocation

A Reason for Revocation subpacket contains a (potentially UTF-8ish) reason identifier.
The remainder of the subpacket contains an unparsed human-readable comment in UTF-8.
A UTF-8ish encoding of the reason field might overflow into the human-readable UTF-8 comment, however a UTF-8 compliant client SHOULD convert the continuation bytes to replacement characters and display the rest of the comment unchanged.

#### User Preferences

Care should be taken when handling the following signature subpackets, which consist of arrays/strings of code points in which unknown code points MUST be gracefully ignored.

* Preferred Symmetric Ciphers subpacket (array of Symmetric Key Algorithm code points)
* Preferred AEAD Ciphersuites subpacket (array of {Symmetric Key Algorithm, AEAD Algorithm} code point tuples)
* Preferred Hash Algorithms (array of Hash Algorithm code points)
* Preferred Compression Algorithms (array of Compression Algorithm code points)

Two-octet encodings of code points in the Preferred AEAD Ciphersuites subpacket may result in desynchronisation of the tuple parsing in legacy code that assumes all encoded tuples are two octets wide.
Therefore, the encoding of a code point tuple MUST be padded to an even-octet boundary by appending a trailing 0xFF octet as necessary, i.e. IFF exactly one of the code points in the tuple is encoded in two octets.
This padding octet will appear at an offset that legacy code will assume contains an AEAD Algorithm identifier, and should be safely interpreted as an unassigned code point.
The padding code point 255 (0xFF) falls in the legacy single-octet encoding range, and should be reserved in the AEAD Algorithm registry for this purpose.

By contrast, three-octet encodings of private use code points should be backwards compatible with legacy code, because tuples where both code points are encoded using an odd number of octets will have an even number of octets overall, and a legacy parser should therefore skip over them correctly.

#### Issuer and Intended Recipient Fingerprints

Version-tagged fingerprints (fingerprints prefixed by a version field) are generally safe to use with variable-length encodings of version numbers, as fingerprints are generally understood to have variable lengths and unknown fingerprint versions should be ignored.

### v6 One-Pass Signature Packets

After the packet version identifier, a v6 OPS packet contains a string of three potentially variable-length code point fields:

* Signature Type ID
* hash algorithm
* public key algorithm

The remainder of the packet cannot be parsed if the hash algorithm code point is unsupported.

UTF-8ish encodings are therefore safe in v6 OPS packets.

### v6 PKESK Packets

A v6 PKESK packet contains two potentially variable-length code point fields, the fingerprint version number and the algorithm identifier.
The fingerprint version is covered by a preceding length field that allows unsupported fingerprints to be safely skipped.
The algorithm identifier is immediately followed by a variable-length field that cannot be parsed if the algorithm is unsupported.

UTF-8ish encodings are therefore safe in v6 PKESK packets.

### v6 SKESK Packets

A v6 SKESK packet contains the following fields:

* symmetric algorithm ID
* AEAD algorithm ID
* S2K specifier length
* S2K specifier
* An initialisation value

All five fields are covered by a preceding length field, and the string-to-key specifier is further covered by its own length field.
The S2K specifier always begins with an S2K specifier type, and the remainder of the S2K specifier cannot be parsed if the S2K specifier type is not supported.
The length of the initialisation value depends on the AEAD algorithm, and therefore cannot be parsed if the AEAD algorithm is not supported.

Secret Key Encryption (S2K Usage) parameters are not supplied in a v6 SKESK, as AEAD mode is always used.

UTF-8ish encodings are therefore safe in v6 SKESK packets.

### User Attribute Packets

User attribute packets are composed of subpackets, each of which is prefixed by a subpacket type identifier and cannot be further parsed if the type is not supported.

Image attribute subpackets are prefixed by the following fields:

* Header length (2 octets)
* Image Attribute Version
* Image Attribute Type

The remainder of the subpacket cannot be parsed if the version and type are not supported.

UTF-8ish encodings are therefore safe in user attribute packets.

### v2 SEIPD Packets

After the packet version identifier, a v2 SEIPD packet contains the following fields:

* Symmetric cipher algorithm ID
* AEAD algorithm identifier
* A 1-octet chunk size
* 32 octets of salt
* Encrypted data (variable)

If the values of the symmetric cipher algorithm and the AEAD algorithm were not checked immediately, it might be possible for legacy code to misidentify the field boundaries and pass incorrectly bounded fields to the HKDF function.
Even if this were the case, the decryption process would not be able to proceed any further without the symmetric and AEAD algorithms being supported, so it is highly unlikely that any information could be leaked to an attacker.

UTF-8ish encodings are therefore almost certainly safe in v2 SEIPD packets.

### Compressed Data Packets

A compressed data packet consists of a compression algorithm ID followed by data that cannot be parsed if the compression algorithm is unsupported.

UTF-8ish encodings are therefore safe in compressed data packets.
