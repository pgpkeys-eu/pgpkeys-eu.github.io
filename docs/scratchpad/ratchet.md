# Double Ratchet Messages in OpenPGP

We consider the development of PFS ("Perfect" Forward Secrecy) based on the double ratchet mechanism.

## Motivation

PFS increases security under the following circumstances:

1. the adversary captures an encrypted message in transit, and retains a copy,
2. the user's devices have their copies, and their secret keys,
3. the user's secret key for the message is compromised without the user's copy being compromised, either:
    * through cryptanalysis, or
    * by compromise of a device that has deleted the message, or
    * by compromise of a device which has not yet received the message, or
    * by compromise of a device that will never receive the message but still has a copy of the key

In OpenPGP, we have long-lived keys that may be stored on devices that do not receive communications, or even offline.
Compromise of long-lived keys alone, without compromise of endpoint devices containing the actual messages, is a plausible attack scenario.

The privacy effectiveness of the double ratchet is significantly improved if disappearing messages are also implemented.

## Design

We define:

* A new capability flag, "Ephemeral Key Exchange" (EKE).
    * This is used on a subkey with a signing-capable algorithm, and permits that subkey to authenticate ephemeral key exchange.
* A new signature subpacket type, "Ephemeral Key", of the Document class.
* One or more new "public-key algorithms" that identify ephemeral cipher suites, e.g. X25519+HKDF+DR.
* A new signature subpacket type, "Preferred Ephemeral Ciphers", constructed similarly to the "Preferred Symmetric Ciphers" subpacket, but containing a list of ephemeral public-key algorithms.
    * This is used in the subkey binding signature over the key exchange subkey (?).

We use standard PKESK+SEIPD sign-then-encrypt messages, but:

* The first message (in each direction) is encrypted to an encryption subkey in the usual manner.
   * It is signed by an EKE subkey, not a signing key, and the signature contains an EK subpacket to initialise the ratchet (once in each direction).
* Subsequent messages use PKESKs with slightly altered semantics:
    * The key ID (PKESKv3) or versioned fingerprint (PKESKv6) identifies the EKE subkey, not an encryption key.
    * The "public key algorithm" octet identifies an ephemeral cipher suite, not the algorithm of the EKE subkey.
    * The algorithm-specific data indicates the sender's current ratchet state, not an encrypted session key.
    * Session keys are obtained instead from the ratchet state.
    * Subsequent messages are authenticated by knowledge of the ratchet state and are not normally signed.
* Each subsequent message MUST also include a new ephemeral public key:
    * It is contained in a Public Subkey packet of the appropriate ephemeral algorithm type.
    * The Public Subkey packet is placed inside the encrypted data, following the literal data packet.

### Ephemeral Key subpacket

The EK subpacket contains the sender's initial ephemeral public key, as a full Public Subkey packet of the appropriate ephemeral algorithm type.

### Algorithm-specific PKESK data

A double-ratchet PKESK contains the sender's full ratchet state for the current message, in addition to the symmetric algorithm used in the following SEIPD packet.
The symmetric algorithm MUST have the same key length as the ECDH algorithm of the current ratchet.

* Symmetric Algorithm (1 octet)
* Hash Algorithm (1 octet)
* Hash of Last Received EK Subpacket (N? octets)
* Symmetric Chain Sequence (4? octets)

## Algorithm upgrades

During a long-running conversation it may eventually become necessary to upgrade the ratchet algorithm.
One party to a conversation may invoke an algorithm upgrade by appending two ephemeral keys to a message; one using the old algorithm and one using the new.
The old algorithm ephemeral key continues the current ratchet, while the new one attempts to initialise a new ratchet.
The other party can accept the upgrade by replying using the old algorithm but including only an ephemeral key of the new algorithm.

## Error recovery

(Fallback encrypted channel TBC)

If all else fails, one party may attempt to initialise a new ratchet using the EKE subkey of the other party.
The other party's software SHOULD warn when this happens, and send a message to the first party using the current ratchet.

## References

* [Justus's presentation from 2018](https://sequoia-pgp.org/talks/2018-08-moving-forward/moving-forward.pdf).
