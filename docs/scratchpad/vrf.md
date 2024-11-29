# Verifiable Pseudorandom Functions in OpenPGP

There are several places in the OpenPGP specifications that require the inclusion of cleartext random values in messages.
In high-confidentiality environments it may be desirable to demonstrate that these values are truly random, and are not used as a hidden channel by a mole (e.g. malware).

We therefore wish to specify the use of Elliptic-Curve Verifiable Pseudorandom Functions, as per [Section 5 of RFC9381](https://www.rfc-editor.org/rfc/rfc9381.html#section-5).

# Problem statement

While exfiltration of data is always possible via encrypted messages, it is also possible for a verifier to observe how many keys a message has been encrypted to.
We therefore only consider the use of hidden channels in this document.
A non-exhaustive list of such hidden channels includes:

* High-bandwidth (few to no length constraints):
    * Slack packet framing.
    * Non-critical unknown or experimental (sub)packets.
    * Unknown versions.
    * Plausible data:
        * Image attributes.
        * URI fields.
        * Free text fields.
    * Overlong marker packets.
    * Padding.
    * Intentionally weakened encryption.
* Medium bandwidth (constrained number of octets):
    * Salt manipulation.
    * Intended recipient fingerprints.
* Low-bandwidth (one octet or less):
    * Undefined octet values.
    * Undefined flag bits.
    * Modulation of semantically-equivalent representations:
        * Certification types.
        * Packet framing.

Most of these channels can be closed by filtering out messages that match obvious patterns.

This document specifically addresses salt minupulation, intentionally-weakened encryption, and padding.

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

Salts, IVs and nonces (collectively here called "salts") are generally small (32 octets or less), however padding packets may be arbitrary large.
Padding randomness is not however required to be cryptographically secure, just incompressible (padding.html).

Random numbers for use in salts SHOULD be securely generated: https://datatracker.ietf.org/doc/html/rfc9580#section-13.10

> An implementation can provide extra security against this form of attack by using separate CSPRNGs to generate random data with different levels of visibility.

The use of VRFs (or similar) to generate salts, and a CSPRNG to generate secrets, would provide this separation.

## Session keys

Session keys are relatively short and are not sent in cleartext, so a passive eavesdropper cannot derive information directly from them.
A mole MAY however use non-random session keys to permit an eavesdropper to more easily brute force the encrypted data packet.
An unprivileged verifier cannot know whether the (encrypted) session key has been manipulated, however the message recipient can.

To prove that a session key was generated pseudorandomly, it SHOULD be derived from a signature over the plaintext form of the message.
The session key entropy would be derived from the entropy of the message, so brute forcing the session key should be impractical.

This process is not one-pass.

## Encryption salts

Unlike session keys, encryption salts are part of the encrypted-data packet metadata and are transmitted in cleartext.
Manipulated salts could therefore convey secrets to a passive eavesdropper.

We wish to prove to an unprivileged verifier that an encryption salt was generated randomly, by reference to contextual entropy.
The entropy of the message itself is not usable, because the encrypted form of the message depends on the salt, and the unencrypted form of the message is not available to an unprivileged verifier.
The only reliable source of contextual entropy is the metadata of the preceding PKESK, but this is not sufficient to seed a CSPRNG.

(( TODO: better source of entropy ))

This process is not one-pass.

## Signature salts

To prove that a signature salt was generated pseudorandomly, a VRF proof MAY be included as a subpacket.
The proof would need to be calculated over unencrypted context data derived from the message and/or the signature trailer.

This process is not one-pass.

## Padding

Padding randomness is not required to be cryptographically secure.
In general, the only requirement of message padding is that it SHOULD be incompressible.
It is therefore sufficient to seed a fast digest function with a relatively short IV, and then apply the digest function to its own output iteratively until enough pseudorandom data is generated.
Collision- and preimage-resistance are not required properties of this digest function, only that its output does not repeat.
Modular exponentiation would be sufficient for this purpose.

Depending on the context, it may also be sufficient to use an agreed property of the message, such as the preceding N octets of data, as the IV for the digest function.
This process could be implemented in a single pass.

# Specification of VRFs

* Use [ECVRF-EDWARDS25519-SHA512-ELL2](https://www.rfc-editor.org/rfc/rfc9381.html#section-5.5)?
* Signature subkey (prover key) with new key flag ("This key may be used to prove verifiable random functions").

## Wire format of proofs

We define a VRF proof format for inclusion in a subpacket.
This contains:

* The versioned fingerprint of the prover key.
* The beta_string.
* The pi_string.

The beta_string and pi_string are length-prefixed fields in the native wire format (as defined in RFC9381).

For a v6 signature, the alpha_string is the trailer of the signature and the salt is the first N octets of the beta_string.

### Native OpenPGP VRF

It would be simplest to implement a VRF using an OpenPGP signature.
This would not be efficient, but would take advantage of existing algorithms and so would be be quickly deployable.

We define the following terms:

* A pre-signature is an ephemeral signature made for the purposes of calculating or verifying a VRF.
* A post-signature is the usual signature made over a message, which may depend on the pre-signature.

Consider the subject and trailer of the post-signature as the VRF alpha_string, then:

* Make a pre-signature using the same version, hash algorithm and hashed subpackets area as the post-signature.
    * The pre-signature uses a novel type to ensure domain separation.
    * The pre-signature is constructed over either:
        * An empty subject (low entropy, fast, one-pass).
        * The full subject of the post-signature (high entropy, slow, not one-pass).
    * The pre-signature is made using the prover key, even if the issuer fingerprint subpacket identifies a different signing key.
    * The salt is the first N octets of the string "OpenPGP Verifiable Random Function", repeated as necessary.
* The pi_string is then the algorithm-specific pre-signature data, and the beta_string is a digest over the pi_string using the same signature hash algorithm.

For a signature salt, the empty subject is sufficient for the pre-signature - chosen-prefix attacks are already unlikely, and we do not need to defend against brute-forcing the preimage.
The salt is taken from the first N octets of the beta_string.

If we are generating a session key, we want to include as much entropy as possible, therefore we MUST use the full subject in the pre-signature.
The session key is taken from the last M octets of the beta_string.

When generating both a signature salt and a session key for a sign-then-encrypt message, the first N octets of the beta_string are used as the salt, and the last M octets of the same beta_string as the session key.
The full subject MUST be used, and a digest algorithm MUST be chosen that provides at least N+M octets of output.

((TODO: is this wise? Should we use a separate signature for the session key? What if there is more than one provably-salted signature?))

Note that if an encrypted message is forwarded unmodified to a third party, this will include the beta_string and therefore the session key.
Since this does not leak any extra data, it is not in itself dangerous - however if only a portion of the message is forwarded, and the recipient has a copy of the original encrypted message, they may be able to recover the full message.
When forwarding such an edited message, care MUST be taken to ensure that the beta_string is not included.
This can easily be done by deleting the signature packet, which is not useful to the recipient.

((TODO: what about encryption salts?))

## Placement of proofs

To prove that a signature salt or session key was generated deterministically, a VRF proof subpacket should be included in the unhashed subpackets area of the post-signature.

((TODO: where do we place the proof for encryption salts?))

# Limitations

A VRF proof is useful under the following circumstances:

* Sufficient entropy can be gathered from non-malleable context info.
* The verifier has access to the same context info as a potential eavesdropper.

The first of these is the most constraining, particularly in the case of encryption salts.
