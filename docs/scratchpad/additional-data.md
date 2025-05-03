# OpenPGP Signatures with Additional Data

It is desirable to allow applications to define and enforce granular domain separation beyond the general-purpose capabilities provided by OpenPGP.

OpenPGP enforces domain separation between signature types and signature versions, however does not currently expose this capability to applications.
All document signatures are included in a single domain for the purposes of OpenPGP signature verification, so an application cannot (for example) require that documents of a certain type are only signed by certain keys.
Since the file type of any given document is in general invisible to the OpenPGP layer, this must be enforced at the application layer.

# Signature over a Binary Document and Additional Data (0x48 (TBC), Additional Data Category)

We define an Additional Data category of signature types (0x48..0x4f), initially containing one signature type (0x48).

A type 0x48 signature is constructed identically to a type 0x00 signature, except:

* The three octets corresponding to "PGP" in UTF-8 encoding are hashed in before the salt (if any).
* The following data is hashed in after the salt (if any) and before the document:
    * Additional Data length (4 octets)
    * Additional Data (N octets)

We define a corresponding Key Flag:

Flag    | Definition                                                                            | Reference
--------|---------------------------------------------------------------------------------------|---------------------
TBC     | This key may be used to make signatures in the Additional Data Category (0x48..0x4f)  | This document

# Use of Additional Data Signatures for Domain Separation

We define a notation in the IETF notation namespace:

Name    | Type              | Definition                                                | Reference
--------|-------------------|-----------------------------------------------------------|---------------------
domsep  | human-readable    | A string containing an identifier for domain separation   | This document

A signature domain is identified by an application-defined string.
It is RECOMMENDED that domain-separation identifiers be either:

* A MIME content type
* A string of the form `[type-identifier]@[domain]`, where the domain part represents an internet domain associated with the application.

At least one `domsep` notation subpacket SHOULD be included in the hashed subpacket area of a direct or subkey binding signature that has the Additional Data key flag set.
Without a `domsep` notation subpacket, the component key in question cannot be used to make any Additional Data signatures, because the form of the Additional Data is undefined.

The `domsep` notation subpacket SHOULD NOT be used elsewhere, and MUST be ignored if found elsewhere.
The `domsep` notation subpacket SHOULD NOT be marked critical.

The `domsep` notation subpacket MUST NOT be included in an Additional Data signature.
This ensures that a naive application cannot verify the signature.

## Type 0x48 Signature Creation and Verification

When constructing or verifying a Type 0x48 signature, the relevant `domsep` notation subpacket from the signing key’s binding signature, including the subpacket header, MUST be passed as the initial octets of Additional Data.
An application MAY specify that further Additional Data is included in the digest calculation.

An application MUST check that the `domsep` notation matches the expected identifier, based on the application context.
If the signing key has more than one `domsep` notation, it is the application's responsibility to ensure that the correct notation is passed as Additional Data.
