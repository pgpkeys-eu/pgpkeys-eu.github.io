# Verifiable Pseudorandom Functions in OpenPGP

There are several places in the OpenPGP specifications that require the inclusion of cleartext random values in messages.
In high-confidentiality environments it may be desirable to demonstrate that these values are truly random, and are not used as a hidden channel.

We therefore wish to specify the use of Elliptic-Curve Verifiable Pseudorandom Functions, as per [Section 5 of RFC9381](https://www.rfc-editor.org/rfc/rfc9381.html#section-5).

# Cleartext Random Values in OpenPGP Messages

* Symmetric session keys
    * https://datatracker.ietf.org/doc/html/rfc9580#section-2.1
* S2K salts
    * https://datatracker.ietf.org/doc/html/rfc9580#section-3.7
* v6 signature salts
    * Signature packet: https://datatracker.ietf.org/doc/html/rfc9580#section-5.2.3
    * OPS packet: https://datatracker.ietf.org/doc/html/rfc9580#section-5.4
* Encryption IVs, salts and nonces
    * SED(deprecated): https://datatracker.ietf.org/doc/html/rfc9580#section-5.7
    * SEIPDv1: https://datatracker.ietf.org/doc/html/rfc9580#section-5.13.1
    * SEIPDv2: https://datatracker.ietf.org/doc/html/rfc9580#section-5.13.2
* Padding
    * Padding packet: https://datatracker.ietf.org/doc/html/rfc9580#section-5.14
    * PKCS1: https://datatracker.ietf.org/doc/html/rfc9580#section-12.1.1

Salts, IVs and nonces (collectively here called "salts") are generally small (32 octets or less), however padding packets may be arbitrary large.
Padding randomness is not however required to be cryptographically secure, just incompressible (padding.html).

Random numbers for use in salts SHOULD be securely generated: https://datatracker.ietf.org/doc/html/rfc9580#section-13.10

> An implementation can provide extra security against this form of attack by using separate CSPRNGs to generate random data with different levels of visibility.

The use of a VRF to generate salts, and a CSPRNG to generate secrets, would provide this separation.

## Padding randomness

Padding randomness is not required to be cryptographically secure.
In general, the only requirement of message padding is that it SHOULD be incompressible.
It is therefore sufficient to seed a fast digest function with a relatively short IV, and then apply the digest function to its own output iteratively until enough pseudorandom data is generated.

Collision- and preimage-resistance are not required properties of this digest function, only that its output does not repeat.
A modular exponentiation would be sufficient in most cases.

Depending on the context, it may also be sufficient to use an agreed property of the message, such as the preceding N octets of data, as the IV for the digest function.

## Session keys

Session keys are relatively short (compared to padding) and are not sent in cleartext.
They are more akin to message padding, in that they are only visible after decryption.
It is also crucial that they are generated securely.

It is therefore NOT RECOMMENDED to use VRFs for session keys.

# Specification of VRFs

* Use [ECVRF-EDWARDS25519-SHA512-ELL2](https://www.rfc-editor.org/rfc/rfc9381.html#section-5.5)?
* Signature subkey (prover key) with new key flag ("This key may be used to prove verifiable random functions").

## Wire format of proofs

We define a VRF notation for inclusion in a notation signature subpacket.
This is marked as non-human-readable and contains:

* The versioned fingerprint of the prover key.
* The beta_string.
* The pi_string.

The beta_string and pi_string are length-prefixed fields in the native wire format (as defined in RFC9381).

For a v6 signature, the alpha_string is the trailer of the signature and the salt is the first N octets of the beta_string.

### Alternative model (native OpenPGP)

Do we really need the full security properties of VRFs?
It would be less disruptive to use an OpenPGP signature as the verifiable function.
This would use existing algorithms.

* Make a signature using the same version, hash algorithm and hashed subpackets area as the real message.
    * The signature uses type 0x08 to ensure domain separation; it is constructed over a zero-length subject (similar to a standalone signature).
    * The signature is made using the prover key, even if the issuer fingerprint subpacket identifies a different signing key.
    * The salt is the first N octets of the string "OpenPGP Verifiable Random Function", repeated as necessary.
* Use the algorithm-specific signature data as the pi_string, and a digest over the pi_string with the signature hash algorithm as the beta_string.

## Placement of proofs

To prove that a signature salt was generated deterministically, a VRF notation signature subpacket MAY be included in the unhashed subpackets area.

((TODO: how do we prove that encrypted-data salts are deterministic, who are we proving it to, what do we use as the alpha_string, and is it a good idea in the first place?))
