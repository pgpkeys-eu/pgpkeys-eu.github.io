# OpenPGP Signature Semantics

OpenPGP signatures have a rich vocabulary, however this is often ambiguous or ill-defined.
This document attempts to fix several of the most notable omissions from earlier specifications.

The following topics are addressed:

* Proper specification of Timestamp and Third Party Confirmation signatures.
    * Use of countersignatures to validate keys.
* OPS nesting.
    * Signature Subject Encoding registry.
    * OPS packets as MIME headers.
* Recursive embedding inside Signature Subpackets.
* Signature Type ranges and Key Usage flags.
    * Authentication Signatures.
* Deprecation of Signature and Key Expiration Time subpackets in favour of a new Subject Validity Period subpacket.
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

The OPS packet contains a "nesting" flag that controls whether an inner OPS signature is included in the data signed over by the outer OPS signature.
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

## OPS nesting

Packet nesting semantics are complex:

* A non-OPS signature signs over everything that follows, including any other signatures.
* An OPS signature construction without nesting flags signs over everything within the OPS construction (nesting).
* To add multiple signatures over the same data without nesting, all OPS constructions (except possibly the innermost one) have the nesting flag set, to *suppress* nesting.
* It is not clear what SHOULD happen if the nesting-suppression flag is set but no OPS packet follows.

Counterproposal 1:

* To sign over a complete OpenPGP packet stream, it SHOULD be wrapped in a Literal packet and treated as a document.
* An OpenPGP packet parser SHOULD NOT recursively descend into such a document; it is up to the application to decide whether to process the inner message by calling the parser again.

Counterproposal 2:

* To sign over a complete OpenPGP packet stream, the default OPS behaviour SHOULD be used.

In both cases, we wish to concretely specify the nesting behaviour.

### Signature Subject Encoding registry

The nesting flags octet comes last in the OPS packet, and so is trivially extended to a variable-length field.
Any missing octets MUST be treated as if all flags in them are zero.
This field should then be promoted to a registry and renamed to "OpenPGP Signature Subject Encoding", with the following initial entries:

Flag    | Description
--------|-------------------------------------------
0x01..  | OPS packet follows; ignore (deprecated)
0x02..  | Ignore all inner signatures
0x04..  | Pre-hashed

The signature subject is the sequence of bytes that the signature is made over.
By default, the signature subject is the entire sequence of packets between the OPS and matching Signature packet.

* "OPS packet follows; ignore" means that the signature subject is stripped of the following OPS packet and its corresponding Signature packet.
    If no OPS packet follows, the behaviour is undefined; it is therefore deprecated in favour of "Ignore all inner signatures".
* "Ignore all inner signatures" means that the signature subject is stripped of all OPS and Signature packets.
    If no other signatures exist, this flag is a no-op.
* "Pre-hashed" means that the signature subject is hashed and the signature is made over (subject length || subject digest) instead of the subject itself.
    The pre-hash algorithm MUST be the same one used for the signature.

Document signatures SHOULD set the "Ignore all inner signatures" flag.
Timestamp signatures SHOULD NOT set the "Ignore all inner signatures" flag.

### OPS packets as MIME headers

To simplify the definition of MIME one-pass signatures, we encode an entire OPS packet in the headers of the MIME part to be signed.
The MIME header to be used is "OpenPGP-OPS".

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
* Certification Range (0x10..0x17)
* Binding Range (0x18..0x1f)
* Key Revocation Range (0x20..0x27)
* Binding Revocation Range (0x28..0x2f)
* Certification Revocation Range (0x30..0x37)
* Timestamping range (0x40..0x47)
* Countersignature range (0x50..0x57)

Each usage flag permits signatures in the following ranges:

* 0x01.. Certification: Certification and Certification Revocation ranges, and Direct Sig type (third party)
* 0x02.. Signature: Document Signature range (and Primary Key Binding type)
* 0x0008.. Timestamping: Timestamping range
* (TBC) Revocation?: Key Revocation range
* (TBC): Countersignature range

In addition, primary keys are always permitted to make self-signatures in the Certification, Binding, Certification Revocation, Key Revocation and Binding Revocation ranges.

### Authentication Signatures

OpenPGP defines no authentication signature types, but does have an authentication key usage flag.
Traditionally, authentication is performed by converting the key material into that of another protocol (usually OpenSSH) and performing authentication in that protocol.

It should be noted that cross-protocol usage can be exploited to evade the domain separation protections of key usage flags.
For example, there is no distinction between signature, certification and authentication usage in OpenSSH, and once converted an OpenPGP authentication key may be used as a OpenSSH CA or to sign git commits.

Guidance for the use of authentication keys should be provided.

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
