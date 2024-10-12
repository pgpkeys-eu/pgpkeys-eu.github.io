# OpenPGP Signature Semantics

OpenPGP signatures have a rich vocabulary, however this is often ambiguous or ill-defined.
This document attempts to fix several of the most notable omissions from earlier specifications.

The following topics are addressed:

* Proper specification of Timestamp and Third Party Confirmation signatures.
* Signature Type ranges and usage flags.
* Deprecation of Signature and Key Expiration Time subpackets in favour of a new Subject Validity Period subpacket.
* Cumulation of Signatures.
    * Unhashed Subpacket Deduplication.
* Redundancy of Certification Signature Types.

## Specification of Timestamp Signatures

In [RFC1991](https://datatracker.ietf.org/doc/html/rfc1991), it says:

> <40> - time stamping ("I saw this document") (*)
> ...
> Type <40> is intended to be a signature of a signature, as a notary seal on a signed document.

The second statement implies that a v3 0x40 sig is made by hashing a signature packet as if it were a document.
But the first statement implies a simple signature over an arbitrary document, just with different semantics.

In [RFC2440](https://datatracker.ietf.org/doc/html/rfc2440), this has changed to:

> 0x40: Timestamp signature.
> This signature is only meaningful for the timestamp contained in it.

This avoids the apparent contradiction of RFC1991, by not even attempting to explain the semantics.
And there's still no description of how we should construct one.

Then [RFC4880](https://datatracker.ietf.org/doc/html/rfc4880) defines a new signature type 0x50, which is:

> 0x50: Third-Party Confirmation signature.
> This signature is a signature over some other OpenPGP Signature packet(s).
> It is analogous to a notary seal on the signed data.

Which sounds very similar to the description of type 0x40 sigs in RFC1991.
But unlike 0x40, which remains ambiguous, a concrete construction is given.

Note also that there is a key usage flag for timestamping.
This would appear to indicate that timestamping documents is sufficiently different from signing them that separate keys should be used.
This is consistent with the idea that "I wrote this document" and "I saw this document" are distinct statements with different consequences.
This is crucial in the case of an automated timestamping service that makes no claims about the accuracy of document contents.

We therefore define type 0x40 Timestamp signatures as follows:

* A type 0x40 timestamp signature is made over a document, and constructed the same way as a type 0x00 signature (does this imply we need a type 0x41 signature too?).
* It is only valid if made by a (sub)key with the timestamping usage flag.
* It conveys no opinion about the validity of the document; it only claims that the document existed at the time the signature was made.
* It can be made over an otherwise unsigned document, or it can be one of many signatures over the same document.
* It makes no claims about any other signatures on the document.
* It may be used anywhere that a document signature may be used.

Countersigning a signed document is done using the type 0x50 third party confirmation signature.

## Specification of Third Party Confirmation signatures

The construction of type 0x50 signatures is well defined, however their placement and semantics are not.
We define them as follows:

* A type 0x50 signature notarises another signature.
* By default, it makes no claim about the validity of the signature, just its existence.
* It SHOULD be located in an Embedded Signature packet in the unhashed area of the signature it notarises.
   If it is so located, a Signature Target subpacket is not required.

TODO: do we allow the application layer to make validity claims using notations?
TODO: do we need a new usage flag?

## Signature Type Ranges

Signature Type code points are spaced out into identifiable ranges of types with similar semantics.
These also appear to correspond to various Key Usage flags.
These ranges and the corresponding usage flags are not rigorously defined.
We do so now:

* Document Signature Range (0x00..0x07)
* Certification Range (0x10..0x17)
* Binding Range (0x18..0x1f)
* Key Revocation Range (0x20..0x27)
* Binding Revocation Range (0x28..0x2f)
* Certification Revocation Range (0x30..0x37)
* Timestamping range (0x40..0x47)
* Countersignature range (0x50..0x57)

Each usage flag permits signatures in the following ranges:

* Signature: Document Signature range (and Primary Key Binding type)
* Certification: Certification and Certification Revocation ranges (third party)
* Timestamping: Timestamping range
* (TBC) Revocation: Key Revocation range
* (TBC): Countersignature range

TODO: how do third party direct sigs work?

In addition, primary keys are always permitted to make first-party signatures in the Certification, Binding, Certification Revocation, Key Revocation and Binding Revocation ranges.

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
