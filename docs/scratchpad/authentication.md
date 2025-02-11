# Authentication Signatures in OpenPGP

OpenPGP defines authentication signature capability, but does not specify any authentication signature types or mechanisms.
We specify them here.

# Terminology

In this document, we use the term "prover" to mean the application that connects to a service, and the term "verifier" to mean the application that performs verification on behalf of that service.

# Authentication Signatures

An Authentication signature is constructed identically to a signature over a document, but with the signature type 0x08.

The document signed over MAY be empty or MAY contain a challenge.
A challenge MAY be supplied to the prover by the verifier as part of a multi-step protocol, otherwise it SHOULD be generated separately by both the prover and the verifier using a pre-agreed method.
In either case, the challenge SHOULD NOT be included as part of the response.

## Subpackets

An Authentication signature MUST contain the following subpackets in its hashed area:

* a Signature Creation Time subpacket
* an Issuer Fingerprint subpacket containing the fingerprint of the authentication key
* a human-readable Notation Data subpacket with the name `login-hostname` and a value containing the domain name of the service being authenticated to (e.g. `example.com`)
* a human-readable Notation Data subpacket with the name `login-service` and a value containing the name of the service being authenticated to (e.g. `https`)

A verifier SHOULD ensure that the creation time of an authentication signature is in the recent past, however the time limit is application-dependent.

## Placement of the Authentication Signature

An authentication signature MAY be distributed as either a detached signature, or in an Embedded Signature subpacket.
It MUST NOT be made over a Literal Data packet or placed where a Literal Data signature would be expected.

## Key Flags

The Signature Type range 0x08..0x0f is allocated to the Authentication Category.
A signature in the Authentication Category MUST NOT be made by a component key unless that key has the Authentication Key Flag (0x20..) in its current binding signature.

# Pretty Good Authentication Protocol (PGAP)

A simple protocol for authenticating against a remote web service works as follows:

* The prover generates an Authentication signature over a zero-length challenge.
* The prover submits the ASCII-armored signature in the `s` form field of an HTTP POST request to the well-known endpoint `/.well-known/openpgp/login` on the remote service.
* The verifier checks that the signature was made by a known authentication subkey and that the Signature Creation Time and Notation Data subpackets conform to its local policy.
* The verifier records the successful authentication, and sets a session cookie in its response.

# IANA Actions

IANA is requested to register the following entry in the OpenPGP Signature Types registry:

ID      | Name                      | Embeddable    | Reference
--------|---------------------------|---------------|-------------------
0x08    | Authentication signature  | Yes           | This document

IANA is requested to register the following entries in the OpenPGP Signature Notation Data Subpacket Types registry:

Notation Name   | Data Type         | Allowed Values    | Reference
----------------|-------------------|-------------------|-------------------
login-hostname  | human-readable    | DNS host name     | This document
login-service   | human-readable    | Service name      | This document

IANA is requested to update the following entry in the OpenPGP Key Flags registry:

Flag    | Definition                                                                                    | Reference
--------|-----------------------------------------------------------------------------------------------|---------------------
0x20... | This key may be used to make signatures in the Authentication Signature Category (0x08..0x0f) | This document

IANA is requested to register the well-known path `openpgp/login`, with a reference to this document.

# References

This document is based on [previous work by Justus Winter](https://gitlab.com/sequoia-pgp/sequoia-login)
