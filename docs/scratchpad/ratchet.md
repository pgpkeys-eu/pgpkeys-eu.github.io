# Double Ratchet Messages in OpenPGP

We consider the development of PFS based on the double ratchet mechanism.

## Motivation

PFS increases security under the following circumstances:

ⓐ the adversary captures an encrypted message in transit, and retains a copy,
ⓑ the user's devices have their copies, and their secret keys,
ⓒ the user's secret key for the message is compromised without the user's copy being compromised, either
    * through cryptanalysis, or
    * by compromise of a device that has deleted the message, or
    * by compromise of a device which has not yet received the message, or
    * by compromise of a device that will never receive the message but still has a copy of the key

In OpenPGP, we have long-lived keys that may be stored on devices that do not receive communications, or even offline.
Compromise of long-lived keys alone, without compromise of endpoint devices, is a plausible attack scenario.

Note that re-encryption of messages to a long-lived key removes many of the advantages of the double ratchet.
Without disappearing messages, the usefulness of the double ratchet is limited.

## Design

We define:

* a new feature flag, "Ephemeral Key Exchange".
    * This is used on subkeys with signing-capable algorithms and indicates the use of that subkey to authorise ephemeral key exchange.
* a new signature subpacket type, "Ephemeral Key Sync".
* one or more new "public-key algorithms" that identify ratchet algorithm suites, e.g. AES256+ECDH+KEM
    * ...and probably need to define corresponding preferences

We use the standard PKESK+SEIPD message construction, but:

* Only the first message is encrypted to an encryption subkey.
    * Subsequent messages are nominally "encrypted to" the key-exchange subkey.
    * Subsequent PKESKs identify a ratchet algorithm and the algorithm-specific data identifies the current ratchet state.
    * Subsequent session keys are obtained from the ratchet state.
* The first message is signed by the key exchange subkey and the signature contains a EKS subpacket.
    * Subsequent messages are authenticated by knowledge of the ratchet state and are not signed.
* When ratchet changes are required, updates are passed in another EKS subpacket:
    * This subpacket may be transported in an "unhashed signature" packet inside the encrypted data:
        * An unhashed signature does not use the OPS construction.
        * It is of type 0x02 (standalone signature) and appears before the literal data packet.
        * It has public key algorithm 0, hash algorithm 0, hashed subpacket area length 0, and salt length 0 (if v6).
        * It is only used to convey unhashed signature subpackets inside an encrypted message.
* Note that the PKESK indicates the ratchet state for the current message, while the EKS subpacket conveys new ratchet information.

### EKS subpacket contents

For an N-bit symmetric algorithm:

* DH Public Modulus (N bits)

### Algorithm-specific PKESK data

For an N-bit symmetric algorithm:

* (hash of?) DH Public Modulus (N bits)
* Symmetric Chain Sequence (N? bits)

## References

* [https://sequoia-pgp.org/talks/2018-08-moving-forward/moving-forward.pdf](Justus's presentation from 2018).
