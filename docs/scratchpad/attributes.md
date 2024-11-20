# User Attributes in OpenPGP

User Attributes are a much-maligned and often-abused feature of OpenPGP.
Currently their only specified use is to contain images, however it is known that some implementations use the private and experimental range for various purposes.

In this document, we attempt to make User Attributes useful.

# Subpacket Registry Unification

User Attribute Packets are simple containers for one or more User Attribute Subpackets.
These subpackets are wire-format compatible with Signature Subpackets, but the only currently defined type is the Image Attribute Subpacket type.
It appears likely that this subpacket was originally intended to be a certification signature subpacket, but was removed to keep signature sizes manageable.
As a result, the User Attribute Packet was created to contain large subpackets, but only this one subpacket type was ever added to its registry.

We therefore (re)unify the Signature and Image Attribute subpacket registries:

* The OpenPGP Signature Subpacket Registry should be renamed to "OpenPGP Subpacket Registry".
* It should have an added column indicating whether each subpacket is permitted for use in a User Attribute Packet, with the default value of "no".
* The OpenPGP Image Attribute Subpacket Registry should be deleted.

## Image Attribute Subpacket

The Image Attribute Subpacket has some odd features, and is wildly over-specified:

* The image header has a little-endian length field, uniquely for OpenPGP.
* It has a version octet that has only one version specified.
* It has an encoding format octet that represents an entire registry with only one value specified.
* The v1 image header has a further 12 octets of unused fields.

The Image Attribute Subpacket has been abused to store large amounts of data on the OpenPGP keyservers, and as a result most modern keyservers refuse to handle any User Attribute packets.
We therefore deprecate the use of Image Attribute subpackets in User Attribute packets, and forbid them in Signature packets:

* The Subpacket Registry should be updated to include "Image Attribute Subpacket (deprecated)" at code point 1, and marked as permitted for use in User Attribute packets.
* The OpenPGP Image Attribute Encoding Format registry should be deleted.

# New Subpacket Types

We specify the following new subpacket types, all of which are permitted for use in a User Attribute packet.

## Subject Valid From Subpacket

We define a Subject Valid From subpacket that identifies the time before which the other subpackets of the User Attribute packet are not valid.
It contains a four-octet timestamp in seconds since midnight, 1st January 1970.
It performs the same function as the "creation time" field in a Public Key or Public Subkey packet.
It MUST have the critical bit set.

## Subject URI Subpacket

We define a Subject URI Subpacket that identifies the resource being claimed.
It performs a similar function to the User ID packet, in a machine-readable form.
The Subject URI MUST contain a valid URL-encoded URI, e.g. an email address is represented as "mailto:user@example.com".
It MUST NOT contain human-readable comments or "real names".

A certificate MAY include both a User ID containing an email address, and a User Attribute with a Subject URI subpacket representing the same email address.
If a receiving implementation supports the Subject URI subpacket, it SHOULD be used in preference to the User ID packet.

If an implementation intends to certify the email address component of an RFC822-ish User ID, but not the real name or comment components, it SHOULD instead certify a User Attribute with a Subject URI subpacket containing the `mailto:` URI of the email address.
If no such User Attribute packet exists, the certifying application MAY create one -- if so, it is RECOMMENDED to notify the keyholder so that they can self-certify the same User Attribute packet.

# User Attribute Packet Grammar

* A User Attribute packet MUST contain only subpackets that are explicitly permitted for use in a User Attribute packet.
* A User Attribute packet SHOULD contain a Subject Valid From subpacket.
* If a User Attribute packet contains an unknown subpacket whose type field has the critical bit set, it MUST disregard the entire User Attribute packet.

An implementation SHOULD NOT certify a User Attribute packet that contains subpacket types that it does not implement.
