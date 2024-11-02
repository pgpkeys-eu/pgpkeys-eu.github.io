# OpenPGP Signature Semantics

OpenPGP signatures have a rich vocabulary, however this is often ambiguous or ill-defined.
This document attempts to fix several of the most notable omissions from earlier specifications.

The following topics are addressed:

* Proper specification of Timestamp and Third Party Confirmation signatures.
    * Use of countersignatures to validate keys.
* Signed message grammar.
    * Encrypted message grammar.
* Pre-hashed signatures.
* Recursive embedding inside Signature Subpackets.
* Signature Type ranges and Key Usage flags.
    * Authentication Signatures.
* Disambiguation of Expiration Times.
* Cumulation of Signatures.
    * Unhashed Subpacket Deduplication.
* Redundancy of Certification Signature Types.

## Specification of Timestamp Signatures

In [RFC1991](https://datatracker.ietf.org/doc/html/rfc1991), it says:

> <40> - time stamping ("I saw this document") (*)
>
> ...
>
> Type <40> is intended to be a signature of a signature, as a notary seal on a signed document.

The second statement implies that a v3 0x40 sig is made by hashing a signature packet as if it were a document.
But the first statement implies a simple signature over an arbitrary document, just with different semantics.

In [RFC2440](https://datatracker.ietf.org/doc/html/rfc2440), this has changed to:

> 0x40: Timestamp signature.
> This signature is only meaningful for the timestamp contained in it.

This avoids the apparent contradiction of RFC1991, by not even attempting to explain the semantics.
And there's still no explicit construction given.

Then [RFC4880](https://datatracker.ietf.org/doc/html/rfc4880) defines a new signature type 0x50, which is defined as:

> This signature is a signature over some other OpenPGP Signature packet(s).
> It is analogous to a notary seal on the signed data.

This sounds very similar to the second use case for type 0x40 sigs in RFC1991.
But unlike 0x40, which remains ambiguous, a concrete construction is given.

Note also that there is a key usage flag for timestamping.
This would appear to indicate that timestamping documents is sufficiently different from signing them that separate keys should be used.
This is consistent with the idea that "I wrote this document" and "I saw this document" are distinct statements with different consequences.
This is crucial in the case of an automated timestamping service that makes no claims about the accuracy of document contents.

The OPS packet contains a "nesting" octet that controls whether an inner OPS signature is included in the data signed over by the outer OPS signature.
If a Timestamp signature were constructed and placed the same way as a Document signature, it might be possible to reconcile the above variations in semantics.

We therefore define type 0x40 Timestamp signatures as follows:

* A type 0x40 timestamp signature is made over a document, and constructed the same way as a type 0x00 signature (does this imply we need a type 0x41 signature too?).
* It is only valid if made by a (sub)key with the timestamping usage flag.
* It conveys no opinion about the validity of the document; it only claims that the document existed at the time the signature was made.
* It can be made over an otherwise unsigned document, or it can be one of many signatures over the same document.
* It may be used anywhere that a document signature may be used.

Countersigning a signature packet only (i.e. without the associated document) is done using the type 0x50 third party confirmation signature.

## Specification of Third Party Confirmation signatures

The construction of type 0x50 signatures is well defined, however their placement and semantics are not.
We define them as follows:

* A type 0x50 signature notarises another signature.
* It is only valid if made by a (sub)key with the (TBC) usage flag.
* By default, it makes no claim about the validity of the signature, just its existence.
* It SHOULD be located in an Embedded Signature packet in the unhashed area of the signature it notarises.
   If it is so located, a Signature Target subpacket is not required.

Since a blind countersigning party is explicitly permitted ny the spec, we must infer that the countersignature only relates to the signature packet and not to the subject of the signature.
If we wish to also timestamp the subject, a separate signature MUST be created.
Note however that this is only currently possible for document signatures.

### Use of countersignatures to validate keys

We may wish to allow the application layer to make validity claims using countersignatures.
For example, a keyserver may wish to record that it has verified a User ID by automated means.
The keyserver may not wish to make a certification, to prevent the cumulation of many such automated signatures.

(TBC)

## Signed message grammar

The accepted convention is that a leading signature packet signs over the next literal or compressed packet only, skipping any intervening signatures.
This is not explicitly specified, but SHOULD be.

Further, OPS nesting semantics are complex, and the specification is light.
[RFC9580 section 5.4](https://datatracker.ietf.org/doc/html/rfc9580#section-5.4) defines the nesting octet as:

> A 1-octet number holding a flag showing whether the signature is nested.
> A zero value indicates that the next packet is another One-Pass Signature packet that describes another signature to be applied to the same message data.

The terminology and the specification are obscure, but our interpretation is as follows:

* A zero nesting octet means that the following OPS and its counterpart signature are stripped from the signature's subject.
    * This process is recursive if multiple sequential OPS packets have a nesting octet of zero.
* To add multiple OPS signatures over the same message data, all OPS constructions except the innermost one have the nesting octet zeroed.
    * It is not clear what happens if the innermost nesting octet is zero but no OPS packet follows; this SHOULD be forbidden.
* The above implies that an OPS with a nonzero nesting octet signs over its entire contents verbatim, including any further signatures.
    * If not, the nesting octet would be redundant.

There are no additional constraints on signature nesting in [RFC9580 section 10.3](https://datatracker.ietf.org/doc/html/rfc9580#section-10.3).
We wish therefore to properly constrain signature nesting:

* An OPS with a zero nesting octet is a "Skipping OPS".
* An OPS with a nonzero nesting octet is a "Verbatim OPS".
* A leading Signature packet signs over the next literal or compressed packet, skipping any intervening signature or OPS packets.
* If a skipping OPS packet is not followed by a well-formed OPS packet, then the skipping OPS packet is malformed.
* The subject of a verbatim OPS MAY include one or more leading signature packets or OPS constructions.
* If a leading signature and an OPS both sign over the same subject, the leading signature packet MUST precede the OPS.

It is also not clear which packets are signed over by an encrypted-then-signed message.
This is not idiomatic OpenPGP, and SHOULD be forbidden.

The message grammar is therefore updated to:

* OpenPGP Message:
    Encrypted Message | Unencrypted Message.
* Unencrypted Message:
    Signed Message | Unsigned Message.
* Unsigned Message:
    Compressed Message | Literal Message.
* Compressed Message:
    Compressed Data Packet.
* Literal Message:
    Literal Data Packet.
* ESK:
    Public Key Encrypted Session Key Packet | Symmetric Key Encrypted Session Key Packet.
* ESK Sequence:
    ESK | ESK Sequence, ESK.
* Encrypted Data:
    Symmetrically Encrypted Data Packet | Symmetrically Encrypted and Integrity Protected Data Packet.
* Encrypted Message:
    Encrypted Data | ESK Sequence, Encrypted Data.
* Skipping One-Pass Signed Message:
    Skipping One-Pass Signature Packet, Skipping One-Pass Signed Subject, Corresponding Signature Packet.
* Skipping One-Pass Signed Subject:
    Unencrypted Message | Skipping One-Pass Signed Message.
* Verbatim One-Pass Signed Message:
    Verbatim One-Pass Signature Packet, Unencrypted Message, Corresponding Signature Packet.
* One-Pass Signed Message:
    Skipping One-Pass Signed Message | Verbatim One-Pass Signed Message.
* Signed Message:
    Signature Packet, Unencrypted Message | One-Pass Signed Message.
* Optionally Padded Message:
    OpenPGP Message | OpenPGP Message, Padding Packet.

In addition to these rules, a Marker packet (Section 5.8) can appear anywhere in the sequence.

### Encrypted message grammar

It is not currently specified whether an encrypted message may decrypt to another encrypted message.
This SHOULD be forbidden.

## Pre-hashed signatures

(TBC)

The nesting octet comes last in the OPS packet, and so could be extended to a variable-length field if required.
The only special value currently defined is 0 ("Skipping"); all other values are treated as "Verbatim".
This field could easily be promoted to a registry and renamed to "OpenPGP One-Pass Signature Subject Format", with the following initial entries:

Value   | Description
--------|-----------------------
0       | Skipping
1       | Verbatim
2       | Pre-hashed

The signature subject is the sequence of bytes that the signature is made over.
By default, the signature subject is the entire sequence of octets between the end of the last OPS packet and the beginning of the first Signature packet.
By default, the signature is made over the subject directly.

* If the octet is zero ("Skipping"), this means that the subject is identical to that of the following OPS packet (recursively).
* "Verbatim" means that the subject is the exact sequence of octets between the end of the current OPS packet and the beginning of the matching Signature packet.
* "Pre-hashed" means that the subject is hashed and the signature is made over (subject digest || subject length) instead of the subject itself.
    The pre-hash algorithm MUST be the same one used for the signature.

Note that a sequence of skipping OPS signatures MUST use the same subject format.
If a different subject format is required, then a separate signed message should be made.

Text document signatures can be thought of as a special case of signature subject formatting that can be used with non-OPS signatures.
This seems fragile though; would it be better to define a "pre-hashed" document signature type that can be used on non-OPS messages?
Do we then need "pre-hashed binary" and "pre-hashed text" document sig types?
How do we rein in the combinatorics?

## Recursive embedding inside Signature Subpackets

There are currently two places where embedding of signatures is possible in signature subpackets:

* Embedded Signature subpacket (type 32): contains a signature packet
* Key Block (type 38, experimental): contains an entire TPK

Both of these may contain further signatures, and are therefore recursive.
We wish to prevent infinite recursion via embedded signatures, in order to avoid resource exhaustion.
This can be achieved as follows:

* Signatures contained within Embedded Signature subpackets MUST NOT contain any Embedded Signature subpackets.
    This can be done by altering the specification of Embedded Signature subpacket, and/or the signature types that may be contained within it.
    Belt and braces might be the safest option.
* Key Block is deprecated.

Key Block does perform a function, which is to smuggle a key into the OpenPGP layer without requiring support by the application layer.
We could instead update the message grammar to allow TPKs to be appended to a signed message.
This would have to be allowed inside the encrypted layer.

This would also unify the packet sequence syntax so that there is only one kind of sequence, which would include both messages and keyrings.
See also "mixed keyrings" in draft-gallagher-openpgp-hkp.

## Signature Type Ranges and Key Usage flags

Signature Type code points are spaced out into identifiable ranges of types with similar semantics.
These also appear to correspond to various Key Usage flags.
These ranges and the corresponding usage flags are not rigorously defined.
We do so now:

* Document Signature Range (0x00..0x07)
    * 0x00 Binary document
    * 0x01 Text document
    * 0x02 Standalone (null document)
* Unassigned (0x08..0x0f)
* Certification Range (0x10..0x17)
    * 0x10 Generic cert
    * 0x11 Persona cert
    * 0x12 Casual cert
    * 0x13 Positive cert
    * (0x16 Approved certs)
* Binding Range (0x18..0x1f)
    * 0x18 Subkey bind
    * 0x19 Primary key bind
    * 0x1f Direct key (null bind)
* Key Revocation Range (0x20..0x27)
    * 0x20 Primary key revocation
* Subkey Revocation Range (0x28..0x2f)
    * 0x28 Subkey revocation
* Certification Revocation Range (0x30..0x37)
    * 0x30 Cert revocation
* Unassigned (0x38..0x3f)
* Timestamping range (0x40..0x47)
    * 0x40 Timestamp
* Unassigned (0x48..0x4f)
* Countersignature range (0x50..0x57)
    * 0x50 Third party confirmation
* Unassigned (0x58..0x6f)
* Private/Experimental (0x70..0x77)
* Unassigned (0x78..0xfe)
* RESERVED (0xff)

Each usage flag permits signatures in the following ranges:

* 0x01.. Certification: Certification (except for Approved Certs) and Certification Revocation ranges, and Direct Sig type (third party)
* 0x02.. Signature: Document Signature range (and Primary Key Binding type)
* 0x0008.. Timestamping: Timestamping range
* (TBC) Revocation?: Key Revocation range
* (TBC): Countersignature range
* (TBC): Private/Experimental range

In addition, primary keys are always permitted to make self-signatures in the Certification, Binding, Certification Revocation, Key Revocation and Binding Revocation ranges.

We define a Private/Experimental usage flag and a Private/Experimental signature type range.
The private range is 0x70..0x77 (112..119) for both compatibility with [PGP-GREASE](grease.html) and rough consistency with 100..100 in the decimal registries.

### Authentication Signatures

OpenPGP defines no authentication signature types, but does have an authentication key usage flag.
Traditionally, authentication is performed by converting the key material into that of another protocol (usually OpenSSH) and performing authentication in that protocol.

It should be noted that cross-protocol usage can be exploited to evade the domain separation protections of key usage flags.
For example, there is no distinction between signature, certification and authentication usage in OpenSSH, and once converted an OpenPGP authentication key may be used as a OpenSSH CA or to sign git commits.

Guidance for the use of authentication keys should be provided.

## Disambiguation of Expiration Times

Key Expiration Time subpackets are non-intuitive:

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

The ability to create a new signature with an unchanged valid-from date allows historical signatures to be losslessly cleaned from a TPK, saving space.
It is also more compatible with the historical interpretation favoured by PGP.com and GnuPG.

To clean up the ambiguity, we specify the following:

1. Binding signatures (sbinds, direct key signatures, and self-certifications) SHOULD NOT have Signature Expiration Time subpackets
2. The validity of a (sub)key extends from its creation date until its soft-revocation or expiration date
3. A signature is valid if it was made during the (sub)key's validity period
4. The creation time of the binding signature is used only for ordering, not for calculation of signature validity

We also specify that binding signatures cannot be directly revoked; the corresponding revocation signatures affect the key, not the binding.
This means that certification revocation signatures cannot revoke direct key signatures.

See also [this issue in rfc4880bis](https://gitlab.com/openpgp-wg/rfc4880bis/-/issues/71), [this issue in openpgpgjs](https://github.com/openpgpjs/openpgpjs/issues/1800), and [this issue in draft-dkg-openpgp-revocation](https://gitlab.com/dkg/openpgp-revocation/-/issues/19).

## Cumulation of Signatures

A cryptographically valid certification or document signature automatically and permanently invalidates any earlier signature of the same type range, by the same key pair, over the same data.
If a later such signature expires before an earlier one, the earlier signature does not become valid again.

For the purposes of the above:

* "same type range" does not distinguish between the two document signature types (0x00, 0x01) or between the four certification signature types (0x10-0x13).
* "same key pair" refers to the public key packet as identified by the Issuer KeyID or Issuer Fingerprint subpacket.
* "same data" refers only to the data fed into the signature digest function after the end of the salt (if appropriate) and before the beginning of the trailer.

(See also [this openpgp mailing list post](https://mailarchive.ietf.org/arch/msg/openpgp/C0P4MxwqJBbxS6H0YoXFF3oEJ3A/).)

### Unhashed Subpacket Deduplication

If two signature packets are bitwise identical apart from differences in their unhashed subpacket areas, an implementation MAY merge them into a single signature.
The unhashed subpacket area of the merged signature SHOULD contain the subpackets from both original signatures.
If two unhashed subpackets are bitwise identical, they MUST be deduplicated.
Otherwise, all unhashed subpackets SHOULD be included, even if this results in multiple subpackets of the same type.

## Redundancy of Certification Signature Types

There are four types of certification signature defined in the standards (0x10..0x13).
All may be created by either the key owner or a third party, and may be calculated over either a User ID packet or a User Attribute packet.
In addition, a Certification Revocation signature revokes signatures of any certification type.
Historically, as in [RFC1991](https://datatracker.ietf.org/doc/html/rfc1991), certifications were only made by third parties.
First-party self-certifications only became customary later, and were made mandatory when preference subpackets were introduced.

The semantic distinctions between the certification signature types were left ill-defined and most software treats them as equivalent.
As suggested in [this blog post by DKG](https://dkg.fifthhorseman.net/blog/gpg-ask-cert-level-considered-harmful.html), we wish to tighten up their semantics:

* 0x10 Generic Certification SHOULD only be created by a third party
* 0x11 Persona Certification SHOULD NOT be created
* 0x12 Casual Certification SHOULD NOT be created
* 0x13 Positive Certification SHOULD only be created by the key owner
