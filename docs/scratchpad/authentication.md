# Authentication Signatures in OpenPGP

OpenPGP defines authentication signature capability, but does not specify any authentication signature types or mechanisms.
We specify them here.

# Terminology

In this document, we use the term "prover" to mean the application that connects to a service, and the term "verifier" to mean the application that performs verification on behalf of that service.

# Wire format

An authentication signature is constructed identically to a signature over a document, but with the signature type 0x08 (Authentication Signature).

The document signed over MAY be empty or MAY contain a response to a challenge.
The content of the signed-over document is generated separately by both the prover and the verifier using a pre-agreed method, and SHOULD NOT be sent over the wire.

## Subpackets

An authentication signature MUST contain the following subpackets:

* a Signature Creation Time subpacket
* a human-readable Notation Data subpacket with the name `login-hostname` and a value containing the domain name of the service being authenticated to (e.g. `example.com`)
* a human-readable Notation Data subpacket with the name `login-service` and a value containing the name of the service being authenticated to (e.g. `https`)

A verifier SHOULD ensure that the creation time of an authentication signature is in the recent past, however the time limit is application-dependent.

# Placement of the Authentication Signature

An authentication signature MAY be distributed as either a detached signature, or in an Embedded Signature subpacket.
It MUST NOT be made over a Literal Data packet or placed where a Literal Data signature would be expected.

# Key Flags

An Authentication signature MUST NOT be made by a component key unless that component key has the Authentication Key Flag (0x20..) in its current binding signature.

# IANA Actions

IANA is requested to register the following entry in the OpenPGP Signature Types registry:

ID      | Name                      | Embeddable    | Reference
--------|---------------------------|---------------|-------------------
0x08    | Authentication signature  | Yes           | This document

The range 0x08..0x0f should be reserved for other Authentication Signature types.

IANA is requested to register the following entries in the OpenPGP Signature Notation Data Subpacket Types registry:

Notation Name   | Data Type         | Allowed Values    | Reference
----------------|-------------------|-------------------|-------------------
login-hostname  | human-readable    | DNS host name     | This document
login-service   | human-readable    | Service name      | This document

IANA is requested to update the following entry in the OpenPGP Key Flags registry:

Flag    | Definition                                                                                    | Reference
--------|-----------------------------------------------------------------------------------------------|---------------------
0x20... | This key may be used to make signatures in the Authentication Signature Category (0x08..0x0f) | This document

# References

* Justus's PoC: [sequoia-login](https://gitlab.com/sequoia-pgp/sequoia-login)
