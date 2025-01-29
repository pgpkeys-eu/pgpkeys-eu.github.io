# OpenPGP Reserved Code Points

There are quite a few code points in the IANA OpenPGP registries that are marked “Reserved” with little or no explanation.
They all have references to RFC9580, which is the source of the registry data, but the RFC is not always forthcoming about the reasoning behind their reservation.

Some code points are reserved for future use and MAY presumably be un-reserved once a draft spec is finalised.
Others are reserved to avoid incompatibility with deprecated or experimental features and MUST NOT be reused.
Some have been reserved for speculative features that were never implemented, and so MAY be more appropriately classified as “unassigned".
It is not always clear which case is which, or under what conditions an un-reservation should be performed.

# Analysis and Classification of Reserved Code Points

This blog attempts to reconstruct the rationale for each of the reserved code points.
The primary sources for this analysis are mainly the various RFCs, but earlier draft versions are also referenced where necessary.
All sources are referenced inline below.

In the below, we classify reserved code points as follows:

* [GOOD] (documented and referenced)
* [->...] (documented somewhere but not correctly referenced in the registry)
* [OBV:...] (not documented but intent obvious from context)
* [...?] (inconsistent, incomplete or no documentation)

## OpenPGP String-to-Key (S2K) Types

[Registry](https://www.iana.org/assignments/openpgp/openpgp.xhtml#openpgp-s2k-types)

* 2 [iterated and unsalted S2K?]

## OpenPGP Packet Types

[Registry](https://www.iana.org/assignments/openpgp/openpgp.xhtml#openpgp-packet-types)

* 0 (MUST NOT be used) [why?]
* 19 (MDC) [GOOD]
* 20 [[AEAD encrypted data (deprecated) -> rfc4880bis 5.16](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis#section-5.16)]

## OpenPGP User Attribute Subpacket Types

[Registry](https://www.iana.org/assignments/openpgp/openpgp.xhtml#openpgp-user-attribute-subpacket-types)

* 0 [aversion to zero?]

## OpenPGP Image Attribute Encoding Format

[Registry](https://www.iana.org/assignments/openpgp/openpgp.xhtml#openpgp-user-attribute-encoding-format)

* 0 [aversion to zero?]

## OpenPGP Signature Subpacket Types

[Registry](https://www.iana.org/assignments/openpgp/openpgp.xhtml#openpgp-signature-subpacket-types)

* 0 [aversion to zero?]
* 1 [-> [Image Attribute subpacket](https://andrewgdotcom.gitlab.io/openpgp-user-attributes)?]
* 8, 13-15, 17-19 [-> experimental PGP5 features?]
* 10 (placeholder for backward compatibility) [-> ADK/ARR?]
* 34 [[preferred AEAD algorithms (deprecated) -> rfc4880bis 5.2.3.8](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis#section-5.2.3.8)]
* 37 (attested certifications) [[-> rfc4880bis 5.2.3.30](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis#section-5.2.3.30)]
* 38 (key block) [[-> rfc4880bis 5.2.3.31](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis#section-5.2.3.31)]

## OpenPGP Features Flags

[Registry](https://www.iana.org/assignments/openpgp/openpgp.xhtml#openpgp-features-flags)

* 0x02 [[AEAD and v5 SKESK packets (deprecated) -> rfc4880bis-10 5.2.3.25](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis#section-5.2.3.25)]
* 0x04 [[v5 Public Key packet (deprecated) -> rfc4880bis-10 5.2.3.25](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis#section-5.2.3.25)]

## OpenPGP Key Flags

[Registry](https://www.iana.org/assignments/openpgp/openpgp.xhtml#openpgp-key-flags)

* 0x0004 (ADSK) [[-> rfc4880bis-10 5.2.3.22?](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis#section-5.2.3.22)]
* 0x0008 (timestamping) [[-> rfc4880bis-10 5.2.3.22?](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis#section-5.2.3.22)]

## OpenPGP Public Key Algorithms

[Registry](https://www.iana.org/assignments/openpgp/openpgp.xhtml#openpgp-public-key-algorithms)

* 0 [OBV: 0 implies "none"]
* 20 (ElGamal Encrypt or Sign, deprecated) [GOOD]
* 21 (X9.42, S/MIME compatibility) [SPECIFICATION INCOMPLETE, probably refers to [RFC2631](https://datatracker.ietf.org/doc/html/rfc2631)]
* 23 (AEDH) [SPECIFICATION MISSING]
* 24 (AEDSA) [SPECIFICATION MISSING: obsoleted by [RFC9021](https://www.rfc-editor.org/rfc/rfc9021.pdf), which contains a paywalled normative reference]

See also [section 12.8 of RFC9580](https://datatracker.ietf.org/doc/html/rfc9580#section-12.8).

## OpenPGP Symmetric Key Algorithms

[Registry](https://www.iana.org/assignments/openpgp/openpgp.xhtml#openpgp-symmetric-key-algorithms)

* 5 [SAFER-SK128 -> RFC2440](https://datatracker.ietf.org/doc/html/rfc2440#section-9.2)
* 6 [DES/SK -> RFC2440](https://datatracker.ietf.org/doc/html/rfc2440#section-9.2)
* 253-255 (compatibility with s2k usage octet registry) [GOOD]

## OpenPGP Hash Algorithms

[Registry](https://www.iana.org/assignments/openpgp/openpgp.xhtml#openpgp-hash-algorithms)

* 0 [OBV: 0 implies "none"]
* 4 [[Double-width SHA (experimental) -> RFC2440 9.4](https://datatracker.ietf.org/doc/html/rfc2440#section-9.4)]
* 5 [[MD2 -> RFC2440 9.4](https://datatracker.ietf.org/doc/html/rfc2440#section-9.4)]
* 6 [[TIGER/192 -> RFC2440 9.4](https://datatracker.ietf.org/doc/html/rfc2440#section-9.4)]
* 7 [[HAVAL 5-pass 160-bit -> RFC2440 9.4](https://datatracker.ietf.org/doc/html/rfc2440#section-9.4)]
* 13 [[SHA3-384 -> rfc4880bis-10 10.3.3](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis#section-10.3.3)] (this also implies that 15 == SHA3-224)

Action: code point 15 SHOULD be reserved for SHA3-224.

## OpenPGP Signature Types

[Registry](https://www.iana.org/assignments/openpgp/openpgp.xhtml#openpgp-signature-types)

* 0xFF [GOOD]

## OpenPGP AEAD Algorithms

[Registry](https://www.iana.org/assignments/openpgp/openpgp.xhtml#openpgp-aead-algorithms)

* 0 [OBV: 0 implies "none"]

# Use of special code points

Several of the registries use zero to indicate a special code point "none", and several others obviously reserve zero to prevent accidental interpretation as "none".
Zero is however used in the Signature Types registry as a non-special code point, and some other registries (Image Attribute Encoding Format, and the other Types registries) reserve zero without explanation, even though there is no obvious way to interpret it as "none" in context.
This probably represents a general aversion to using zero as a non-special code point, and the Signature Types registry may be considered an outlier.

RFC1991 used 255 as an "experimental" code point in several registries, however RFC2440 subsequently moved the experimental ranges to their current locations.
255 is otherwise is not considered special in any subsequent document, and is used as a non-special code point in the S2K Usage Octet registry.

# Unassigned gaps in OpenPGP registries

There are a number of unassigned gaps in the registries that are not specifically marked as reserved, but which were presumably left unassigned intentionally:

* Packet type 15 is the only unassigned code point that can be represented in legacy framing, so was presumably kept available for use by a packet type that had to be backwards-compatible with PGP 2.
    It was marked "reserved" in [draft-ietf-openpgp-formats-00 section 4.3](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-formats-00#autoid-116) but was unassigned in subsequent drafts and the eventual [RFC2440](https://datatracker.ietf.org/doc/html/rfc2440).
* Packet type 16 was a "comment packet" in [draft-ietf-openpgp-formats-00 section 5.12](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-formats-00#autoid-162) but was unassigned in subsequent drafts and RFC2440.
    This may have been intended to replace RFC1991's never-implemented "comment packet" (type 14), which was repurposed as a subkey packet.
    Proposed for use as an extension placeholder in [draft-gallagher-openpgp-code-point-exhaustion](https://andrewgdotcom.gitlab.io/openpgp-code-point-exhaustion#section-4.3).
* Signature subpacket 36 was silently omitted when 35 and 37 were added in [draft-ietf-openpgp-rfc4880bis-08](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis-08), and the omission persisted when [the change was ported to crypto-refresh](https://gitlab.com/openpgp-wg/rfc4880bis/-/commit/badfc9fec92ea6833bfab60cb70c99e1d549a79e#ec9f85ae915d32d2e0d0d0e5258927a7e3559c4d_1007_1006).
    36 is provisionally used in [draft-dkg-openpgp-revocation](https://datatracker.ietf.org/doc/html/draft-dkg-openpgp-revocation#name-the-delegated-revoker).
* Public key algorithms 4-15 (born missing in [draft-ietf-openpgp-formats-00](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-formats-00#autoid-164))
* Reasons for revocation 4-31 (born missing in [draft-ietf-openpgp-formats-04](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-formats-04#section-5.2.3.22))
* The Signature Types registry [emerged from prehistory](https://datatracker.ietf.org/doc/html/draft-atkins-pgpformat-01#section-6.2.1) complete with gaps which were obviously intended to group sub-ranges with similar semantics, however they are not formally defined.

Andrew Gallagher (29 January 2025)
