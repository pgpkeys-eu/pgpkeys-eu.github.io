# User Attributes in OpenPGP

User Attributes are a much-maligned and often-abused feature of OpenPGP.
Currently their only specified use is to contain images, however it is known that some implementations use the private and experimental range for various purposes.

In this document, we attempt to make User Attributes useful.

# Subpacket Registry Merging

User Attribute Packets are simple containers for User Attribute Subpackets.
These subpackets are defined identically to Signature Subpackets, and the only defined type is the Image Attribute Subpacket type.
It appears likely that this subpacket was originally intended to be a certification signature subpacket, but was removed to keep the signature size manageable.
Signature Subpacket Type 1 remains reserved as a consequence.

We therefore (re)merge the Signature and Image Attribute subpacket registries:

* The OpenPGP Signature Subpacket Registry should be renamed to "OpenPGP Subpacket Registry".
* It should have an added column indicating whether each subpacket is permitted for use in a User Attribute Packet, with the default value of "no".
* The OpenPGP Image Attribute Subpacket Registry should be deleted.

# Image Attribute Subpacket Deprecation

The Image Attribute Subpacket has some odd features, and is wildly over-specified:

* The image header has a little-endian length field, uniquely for OpenPGP.
* It has a version octet that has only one version specified.
* It has an encoding format octet that represents an entire registry with only one value specified.
* The v1 image header has a further 12 octets of unused fields.

The Image Attribute Subpacket has been abused to store large amounts of data on the OpenPGP keyservers, and as a result modern keyservers refuse to handle any User Attribute packets.
We therefore deprecate the use of Image Attribute subpackets in User Attribute packets, and forbid them in Signature packets:

* The Subpacket Registry should be updated to include the Image Attribute Subpacket at code point 1, and mark it as "Deprecated".
* The OpenPGP Image Attribute Encoding Format registry should be deleted.

# New Subpacket Types

We specify the following new subpacket types, all of which are permitted for use in a User Attribute packet.

## Subject Valid From Subpacket

We define a Subject Valid From subpacket that identifies the date before which the other contents of the User Attribute packet are not valid.
It contains a four-octet timestamp in seconds since midnight, 1st January 1970.
It performs the same function as the "creation time" field in a Public Key or Public Subkey packet.
It MUST have the critical bit set.

## Subject URI Subpacket

We define a Subject URI Subpacket that identifies the resource being claimed.
It performs a similar function to the User ID subpacket, in a machine-readable form.
The Subject URI MUST contain a valid URL-encoded URI, i.e an email address is represented as "mailto:user@example.com".
It MUST NOT contain human-readable commentary.

A generating implementation MAY add both a User ID and a User Attribute containing a Subject URI to a certificate.
If a receiving implementation supports the Subject URI subpacket, it SHOULD be used in preference to the User ID packet.

An implementation that wishes to certify an RFC822-ish User ID, but only wishes to certify the machine-readable email address (and not the real name or comment fields) MAY instead sign over a User Attribute with a Subject URI subpacket containing the mailto: URI of the email address.

# User Attribute Packet Grammar

* A User Attribute packet MUST contain only subpackets that are specified for use in a User Attribute packet.
* A User Attribute packet SHOULD contain a Subject Valid From subpacket.
* If a User Attribute packet contains an unknown subpacket whose type field has the critical bit set, it MUST disregard the entire User Attribute packet.

An implementation SHOULD NOT certify a User Attribute packet that contains subpacket types that it does not implement.
