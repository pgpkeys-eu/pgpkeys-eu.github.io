# OpenPGP Signature Semantics

OpenPGP signatures have a rich vocabulary, however this is often ambiguous or ill-defined.
This document attempts to fix several of the most notable omissions from earlier specifications.

The following topics are addressed:

* Deprecation of Timestamp signatures.
* Deprecation of Signature and Key Expiration Time subpackets in favour of a new Subject Validity Period subpacket.
* Cumulation of Signatures.
    * Unhashed Subpacket Deduplication.
* Redundancy of Certification Signature Types.

## Deprecation of Timestamp Signatures

In [RFC1991](https://datatracker.ietf.org/doc/html/rfc1991), it says:

> Type <40> is intended to be a signature of a signature, as a notary seal on a signed document.

This implies (but does not explicitly state) that a v3 0x40 sig is made by hashing a signature packet as if it were a document.
Alternatively, it could possibly mean a signature over an entire signed document; it is not clear.

But by [RFC2440](https://datatracker.ietf.org/doc/html/rfc2440), this has changed to:

> 0x40: Timestamp signature.
> This signature is only meaningful for the timestamp contained in it.

What does this mean? How does this differ from a standalone signature?
And there's no description of how we should construct one.

Then [RFC4880](https://datatracker.ietf.org/doc/html/rfc4880) defines a new signature type 0x50, which is:

> 0x50: Third-Party Confirmation signature.
> This signature is a signature over some other OpenPGP Signature packet(s).
> It is analogous to a notary seal on the signed data.

Which is the *exact same thing* as the description of type 0x40 sigs in RFC1991!
But unlike 0x40, which remains ambiguous, a concrete construction is given.

We should therefore deprecate type 0x40 Timestamp signatures, as all possible use cases for them are covered by either document signatures, or Third-Party Confirmation signatures.

## Subject Validity Period Subpacket

[Key Expiration Time subpackets are a misfeature](https://gitlab.com/openpgp-wg/rfc4880bis/-/issues/71):

1. They specify an offset rather than an timestamp, but are not usable without first converting to a timetamp
2. The offset is calculated relative to the creation timestamp of *some other packet*
3. Some implementations interpret them as being inheritable in their raw form, so that the same offset value gets applied to *different creation timestamps*.

Further, their semantics overlaps that of Signature Expiration Time:

1. If the binding signature over a key expires, but the key does not, the key is nevertheless unusable due to lack of signatures.
2. If a key expires, but the signature over it does not, the signature is unusable.

This means there are two expiration dates on a binding signature, the key expiration and the signature expiration, but without distinct semantics.

In addition, the Signature Creation Time subpacket has an overloaded meaning:

1. It is used as the "valid from" timestamp of the object being signed over
2. It is used to order multiple similar signatures to determine which is valid

Together, this means that it is not possible to create a new signature that extends the validity of a key further into the past, nor even one that leaves the starting date unchanged.
Some implementations have worked around this by generating signatures with creation dates backdated to one second after that of the previous signature.
The ability to create a new signature with an unchanged creation date would allow historical signatures to be losslessly cleaned from a TPK, saving space.

We therefore define a "Subject Validity Period" subpacket that contains the following fields:

* Valid From (4 octets): The earliest date the subject of the signature is valid
* Valid Until (4 octets): The latest date the subject of the signature is valid

The following special values are defined:

* 0x00000000: The Infinite Past
* 0xffffffff: The Infinite Future

All other values are interpreted as seconds since midnight, 1st Jan 1970.
If no Subject Validity Period subpacket is included, then the validity period begins at the creation date of the signature.

The Signature Expiration Time and Key Expiration Time subpackets should both be deprecated.
However, for a transitional period, it is RECOMMENDED to include both the old and new validity systems.
A receiving implementation SHOULD ignore the deprecated subpacket types and use the Valid Until field of the Subject Validity Period subpacket instead, if one exists.
In such a scenario, the deprecated subpacket SHOULD be marked critical, and the Subject Validity Period subpacket MUST NOT be critical.
If only a Subject Validity Period subpacket is included, then it SHOULD be marked critical.

## Cumulation of Signatures

A cryptographically valid signature automatically and permanently invalidates any earlier signature of the same type, by the same key pair, over the same data.
If a later such signature expires before an earlier one, the earlier signature does not become valid again.

For the purposes of the above:

* "same type" does not distinguish between the two document signature types (0x00, 0x01) or between the four certification signature types (0x10-0x13).
* "same key pair" refers to the public key packet as identified by the Issuer KeyID or Issuer Fingerprint subpacket.
* "same data" refers only to the data fed into the signature digest function after the end of the salt (if appropriate) and before the beginning of the trailer.

If a Subject Validity Period subpacket is present in the later signature, the earlier signature SHOULD be discarded.
If a Subject Validity Period subpacket is not present in the later signature, an implementation MAY retain the earlier signature for the purposes of calculating subject validity at an earlier point in time, for example to determine if a key pair was valid at the time that a signature was apparently generated by it.

(See also [this openpgp mailing list post](https://mailarchive.ietf.org/arch/msg/openpgp/C0P4MxwqJBbxS6H0YoXFF3oEJ3A/).)

### Unhashed Subpacket Deduplication

If two signature packets are bitwise identical apart from differences in their unhashed subpacket areas, an implementation MAY merge them into a single signature.
The unhashed subpacket area of the merged signature SHOULD contain the subpackets from both original signatures.
If two unhashed subpackets are bitwise identical, they MUST be deduplicated.
Otherwise, all unhashed subpackets SHOULD be included, even if this results in multiple subpackets of the same type.

## Redundancy of Certification Signature Types

There are four types of certification signature defined in the standards (0x10..0x13).
All may be created by either the key owner or a third party, and may be calculated over either a User ID packet or a User Attribute packet.
In addition, a Certification Revocation signature revokes signatures of any certification type (and also direct signatures).
Historically, as in [RFC1991](https://datatracker.ietf.org/doc/html/rfc1991), certifications were only made by third parties.
First-party self-certifications only became customary later, and were made mandatory when preference subpackets were introduced.

The semantic distinctions between the certification signature types were left ill-defined and most software treats them as equivalent.
As suggested in [this blog post by DKG](https://dkg.fifthhorseman.net/blog/gpg-ask-cert-level-considered-harmful.html), we wish to tighten up their semantics:

* 0x10 Generic Certification SHOULD only be created by a third party
* 0x11 Persona Certification SHOULD NOT be created
* 0x12 Casual Certification SHOULD NOT be created
* 0x13 Positive Certification SHOULD only be created by the key owner
