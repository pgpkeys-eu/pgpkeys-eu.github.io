# Code Point Exhaustion

We are motivated to define a variable-length encoding for OpenPGP code points so that we can have more than 255 of each.

## Code Point Registries

The following OpenPGP registries currently define one-byte code points:

* OpenPGP String-to-Key (S2K) Types
* OpenPGP Packet Types
* OpenPGP User Attribute Subpacket Types
* OpenPGP Image Attribute Encoding Format
* OpenPGP Signature Subpacket Types
* OpenPGP Reason for Revocation (Revocation Octet)
* OpenPGP Public Key Algorithms
* OpenPGP Symmetric Key Algorithms (*)
* OpenPGP Hash Algorithms
* OpenPGP Compression Algorithms
* OpenPGP Secret Key Encryption (S2K Usage Octet) (*)
* OpenPGP Signature Types
* OpenPGP Key IDs and Fingerprints
* OpenPGP Image Attribute Versions
* OpenPGP AEAD Algorithms
* OpenPGP Key and Signature Versions

The Secret Key Encryption and Symmetric Key Algorithms registries (*) share code points with each other.
Further, the Secret Key Encryption registry is the only one of the above that currently assigns or reserves code points greater than 110 -- specifically 253, 254 and 255.

## Variable-length Encoding of Registereed Code Points

We use a similar algorithm to the packet length parameter:

* single-byte encodings have a byte value < 192 and are fully backwards compatible
* two-byte encodings have a first byte >= 192 and <= 223, and encode values in the range 192..8383
* if the first byte >= 224, this is a legacy single-byte encoding for backwards compatibility

Only defining one- and two-byte encodings allows for legacy single-byte encodings of Secret Key Encryption code points 253, 254, and 255 (in fact, any code point in the range 224..255).

## Signature Malleability

To prevent signature malleability, we require the following:

* when hashing v4 and v6 trailers, an implementation MUST reverse the byte order of code point encodings
* v3 keys and signatures MUST NOT use code points >=192
