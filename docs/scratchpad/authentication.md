# Authentication Signatures in OpenPGP

OpenPGP defines authentication signature capability, but does not specify any authentication signature types or mechanisms.
We specify them here.

# Terminology

In this document, we use the term "prover" to mean the application that connects to a service, and the term "verifier" to mean the application that performs verification on behalf of that service.

# Challenge-Response Signature (Type 0x08)

A Challenge-Response signature is constructed identically to a signature over a binary document, but with the signature type 0x08.

The type-specific data passed to the signature digest function contains a challenge, which SHOULD be generated separately by both the prover and the verifier using a pre-agreed method.

A Challenge-Response signature SHOULD be distributed as either a detached signature, or in an Embedded Signature subpacket.
It MUST NOT be made over a Literal Data packet or used directly in a Message or Certificate (TPK) packet sequence.

## Subpackets

A Challenge-Response signature MUST contain the following subpackets in its hashed area:

* a Signature Creation Time subpacket
* an Issuer Fingerprint subpacket containing the fingerprint of the authentication key

The creation timestamp SHOULD be in the immediate past, however the acceptable time range is application-dependent.
An application SHOULD tolerate a reasonable amount of clock drift.

Other subpackets SHOULD NOT be included, and any unexpected subpackets MUST be ignored.

## Key Flags

The Signature Type range 0x08..0x0f is allocated to the Authentication Category.
A signature in the Authentication Category MUST NOT be made by a component key unless that key has the Authentication Key Flag (0x20..) in its current binding signature.

# Pretty Good Authentication Protocol (PGAP)

A simple protocol for authenticating against a remote web service is specified as follows:

## Prover actions

* The prover generates a Challenge-Response signature over a challenge consisting of the following CRLF-terminated records:
    * The HTTP request line in case-normalised form, e.g. `GET / HTTP/1.1`.
    * The `Host` HTTP header of the request.
    * A `WWW-Authenticate` HTTP header with the value `PGAP realm="simple"` or `PGAP realm="session"`.
* The prover encodes the authentication credentials in the form `PGAP <base-64-encoded-signature-packet>`.
* If the PGAP realm is `simple`:
    * The prover submits the encoded credentials in the `Authentication` header of an HTTP request for the desired resource.
* If the PGAP realm is `session`:
    * The prover submits the encoded credentials in the `Authentication` header of an HTTP POST request to the well-known endpoint `/.well-known/openpgp/login` on the remote service.

If session authentication is successful, subsequest requests supply `PGAP ~<base-64-encoded-signature-digest>` in the `Authentication` header.
The digest is calculated over the binary signature packet (including framing), and the hash algorithm used MUST be the same as the one used in the signature packet.
This saves bandwidth and CPU, particularly with post-quantum signature algorithms.

Authentication credentials MUST only be submitted over a secure transport layer, such as HTTPS.

## Verifier actions

On throwing a 401 Unauthorized response, the verifier MAY return a `WWW-Authenticate` header that contains the challenge `PGAP realm="simple"` or `PGAP realm="session"` as appropriate.

On receiving a PGAP authentication request:

* The verifier checks that the signature was made by a known authentication subkey and that the Signature Creation Time subpacket is within the acceptable clock drift limit.
* The verifier regenerates the challenge from the request headers and verifies the signature.
* If the PGAP realm is `session`: 
    * After a successful request to `/.well-known/openpgp/login`, the verifier records the digest of the signature packet in a table of active sessions.
    * Requests to other resources are checked against the active session table.
    * Sessions SHOULD time out after a reasonable period of time, which SHOULD be longer than the clock drift limit of the authentication signature.

# IANA Actions

IANA is requested to register the following entry in the OpenPGP Signature Types registry:

ID      | Name                          | Embeddable    | Reference
--------|-------------------------------|---------------|-------------------
0x08    | Challenge-Response signature  | Yes           | This document

IANA is requested to update the following entry in the OpenPGP Key Flags registry:

Flag    | Definition                                                                                    | Reference
--------|-----------------------------------------------------------------------------------------------|---------------------
0x20... | This key may be used to make signatures in the Authentication Signature Category (0x08..0x0f) | This document

IANA is requested to register the well-known path `openpgp/login`, with a reference to this document.

IANA is requested to register the HTTP authentication scheme `PGAP`, with a reference to this document.

# References

This document is based on [previous work by Justus Winter](https://gitlab.com/sequoia-pgp/sequoia-login)

See also [RFC9110](https://datatracker.ietf.org/doc/html/rfc9110)
