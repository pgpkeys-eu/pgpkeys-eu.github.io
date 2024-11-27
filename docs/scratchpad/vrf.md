# Verifiable Pseudorandom Functions in OpenPGP

There are several places in the OpenPGP specifications that require the inclusion of random values in messages.
In high-confidentiality environments it may be desirable to demonstrate that these values are truly random, and are not used as a hidden channel.

We therefore wish to specify the use of Elliptic-Curve Verifiable Pseudorandom Functions, as per [Section 5 of RFC9381](https://www.rfc-editor.org/rfc/rfc9381.html#section-5).

# Random Values in OpenPGP Messages

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

Salts, nonces and session keys are generally small (32 octets or less), however padding packets may be arbitrary large.

Random numbers SHOULD be securely generated: https://datatracker.ietf.org/doc/html/rfc9580#section-13.10

> An implementation can provide extra security against this form of attack by using separate CSPRNGs to generate random data with different levels of visibility.

The use of a VRF to generate message randomness, and a CSPRNG to directly generate secrets, would provide this separation.

# Specification of VRFs

* Use [ECVRF-EDWARDS25519-SHA512-ELL2](https://www.rfc-editor.org/rfc/rfc9381.html#section-5.5)
* Signature subkey with new key flag ("This key may be used for verifiable random functions")
