# OpenPGP Reserved Code Points

There are quite a few code points in the IANA OpenPGP registries that are marked “Reserved” with little or no explanation.
They all have references to RFC9580, which is the source of the registry data, but the RFC is not always forthcoming about the reasoning behind their reservation.

Some code points are reserved for future use and MAY presumably be un-reserved once a draft spec is finalised.
Others are reserved to avoid incompatibility with deprecated or experimental features and MUST NOT be reused.
Some have been reserved for speculative features that were never implemented, and so MAY be more appropriately classified as “unassigned".
It is not always clear which case is which, or under what conditions an un-reservation should be performed.

In the below, we classify reserved code points as follows:

* [GOOD] (documented and referenced)
* [->...] (documented somewhere but not correctly referenced in the registry)
* [OBV:...] (not documented but intent obvious from context)
* [...?] (inconsistent, incomplete or no documentation)

## OpenPGP String-to-Key (S2K) Types

* 2 [iterated and unsalted S2K?]

## OpenPGP Packet Types

* 0 (MUST NOT be used) [why?]
* 19 (MDC) [GOOD]
* 20 [AEAD encrypted data (deprecated) -> rfc4880bis 5.16]

## OpenPGP User Attribute Subpacket Types

* 0 [aversion to zero?]

## OpenPGP Image Attribute Encoding Format

* 0 [aversion to zero?]

## OpenPGP Signature Subpacket Types

* 0, 1, 8, 13-15, 17-19 [-> experimental PGP5 features?]
* 10 (placeholder for backward compatibility) [-> ADK/ARR?]
* 34 [preferred AEAD algorithms (deprecated) -> rfc4880bis 5.2.3.8]
* 37 (attested certifications) [-> rfc4880bis 5.2.3.30]
* 38 (key block) [-> rfc4880bis 5.2.3.31]

## OpenPGP Features Flags

* 0x02 [AEAD and v5 SKESK packets (deprecated) -> rfc4880bis 5.2.3.25]
* 0x04 [v5 Public Key packet (deprecated) -> rfc4880bis 5.2.3.25]

## OpenPGP Key Flags

* 0x0004 (ADSK) [-> rfc4880bis 5.2.3.22?]
* 0x0008 (timestamping) [-> rfc4880bis 5.2.3.22?]

## OpenPGP Public Key Algorithms

* 0 [OBV: consistency with Symmetric Key Algorithm 0 "unencrypted"]
* 20 (ElGamal Encrypt or Sign, deprecated) [GOOD]
* 21 (X9.42, S/MIME compatibility) [?]
* 23 (AEDH) [?]
* 24 (AEDSA) [?]

## OpenPGP Symmetric Key Algorithms

* 5 [-> RFC2440 SAFER-SK128]
* 6 [-> RFC2440 DES/SK]
* 253-255 (compatibility with s2k usage octet registry) [GOOD]

## OpenPGP Hash Algorithms

* 0 [OBV: consistency with Symmetric Key Algorithm 0 "unencrypted"]
* 4 [-> RFC2440 Double-width SHA (experimental)]
* 5 [-> RFC2440 MD2]
* 6 [-> RFC2440 TIGER/192]
* 7 [-> RFC2440 HAVAL 5-pass 160-bit]
* 13 [OBV: SHA3-384, but then why isn't 15 reserved for SHA3-224?]

## OpenPGP Signature Types

* 0xFF [GOOD]

## OpenPGP AEAD Algorithms

* 0 [OBV: consistency with Symmetric Key Algorithm 0 "unencrypted"]

# Unassigned gaps in OpenPGP registries

There are also a number of unassigned gaps in the registries that are not specifically marked as reserved, but which were presumably left unassigned intentionally:

* Packet type 15 is the only unassigned code point that can be represented in legacy framing, so was presumably kept available in case of a packet that had to be backwards-compatible with PGP 2.
    It was marked "reserved" in draft-ietf-openpgp-formats-00 section 4.3 but was unassigned in draft-01 and the eventual RFC2440.
* Packet type 16 was a "comment packet" in draft-ietf-openpgp-formats-00 section 5.12 but was unassigned in draft-01 and the eventual RFC2440.
    This may have been intended to stand in for RFC1991's never-implemented "comment packet" (type 14), which was repurposed as a subkey packet.
* Public key algorithms 4-15 (these were already missing in draft-ietf-openpgp-formats-00)
* Reasons for revocation 4-31 (reason 32 was added in draft-ietf-openpgp-rfc4880bis)
* The gaps in the Signature Types registry are obviously for grouping into sub-ranges with similar semantics, however they are not formally defined, and the 0x4N and 0x5N signature type ranges are not well motivated.
