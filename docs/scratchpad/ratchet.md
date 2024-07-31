# Double Ratchet Messages in OpenPGP

We consider the development of PFS ("Perfect" Forward Secrecy) based on the double ratchet mechanism.

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
Compromise of long-lived keys alone, without compromise of endpoint devices containing the actual messages, is a plausible attack scenario.

The effectiveness of the double ratchet is significantly improved in the disappearing-message scenario.

## Design

We define:

* A new feature flag, "Ephemeral Key Exchange".
    * This is used on subkeys with signing-capable algorithms and indicates the use of that subkey to authorise ephemeral key exchange.
* A new signature subpacket type, "Ephemeral Key Sync", of the Document class.
* One or more new "public-key algorithms" that identify ephemeral cipher suites, e.g. X25519+HKDF+DR
* A new signature subpacket type, "Preferred Ephemeral Ciphers", constructed similarly to the "Preferred Symmetric Ciphers" subpacket.
    * This is used in the subkey binding signature over the key exchange subkey.

We use the standard PKESK+SEIPD message construction, but:

* The first message is encrypted to an encryption subkey in the usual manner.
* Subsequent messages use a custom PKESK format:
    * The key ID (PKESKv3) or versioned fingerprint (PKESKv6) identify the key-exchange subkey, not an encryption key.
    * The public key algorithm is an ephemeral cipher suite, not the algorithm of the key-exchange subkey.
    * The algorithm-specific data is not an encrypted session key, but instead identifies the current ratchet state.
    * Session keys are obtained from the ratchet state stored locally.
* The first message is signed by the key exchange subkey and the signature contains a EKS subpacket.
    * Subsequent messages are authenticated by knowledge of the ratchet state and are not normally signed.
* DH ratchet updates are passed in another EKS subpacket:
    * This subpacket MAY be transported in an "unhashed signature" packet inside the encrypted data:
        * An unhashed signature DOES NOT use the OPS construction.
        * It is of type 0x02 (standalone signature) and appears before the literal data packet.
        * It has public key algorithm 0, hash algorithm 0, hashed subpacket area length 0, and salt length 0 (if v6).
        * It is only used to convey unhashed signature subpackets inside an encrypted message.
* Note that the PKESK indicates the full ratchet state for the current message, while the EKS subpacket conveys DH ratchet updates.

### EKS subpacket contents

* Symmetric Algorithm ID (1 octet)
* Ephemeral Public Key Algorithm ID (1 octet)
* ECDH Public Modulus (N octets)

### Algorithm-specific PKESK data

* Hash Algorithm (1 octet)
* Hash of Last Received EKS Subpacket (N? octets)
* Symmetric Chain Sequence (4? octets)

## References

* [https://sequoia-pgp.org/talks/2018-08-moving-forward/moving-forward.pdf](Justus's presentation from 2018).
