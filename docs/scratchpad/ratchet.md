# Double Ratchet Messages in OpenPGP

We consider the development of "Perfect" Forward Secrecy (PFS) based on the double ratchet (DR) mechanism.

## Motivation

PFS increases security under the following circumstances:

1. the adversary captures an encrypted message in transit, and retains a copy,
2. the user's devices have their copies, and their secret keys,
3. the user's secret key for the message is compromised without the user's copy being compromised, either:
    * through cryptanalysis, or
    * by compromise of a device that has deleted the message, or
    * by compromise of a device which has not yet received the message, or
    * by compromise of a device that will never receive the message but has a copy of the long-lived secret key material.

In OpenPGP, we have long-lived keys that may be stored on devices that do not receive communications, or even offline.
Compromise of long-lived keys alone, without compromise of endpoint devices containing the actual messages, is a plausible attack scenario.

The privacy effectiveness of the double ratchet is significantly improved if disappearing messages are also implemented.

## Design

We define:

* A new key capability flag, "Ephemeral Key Exchange" (EKE).
    * It is used on a subkey with a signing-capable algorithm, and permits that subkey to authenticate ephemeral key exchange.
* One or more new public-key algorithm IDs (in a special range) that identify ephemeral cipher suites.
    * Implementations MUST support X25519+HKDF+DR and MAY support X448+HKDF+DR.
* A new signature subpacket type, "Ephemeral Key", of the Document class.
    * It contains the sender's initial ephemeral public key, as a full Public Subkey packet using the appropriate ephemeral algorithm.
* A new signature subpacket type, "Preferred Ephemeral Ciphers", of the Preference class.
    * It contains an ordered list of ephemeral public-key algorithms.
    * It is used in (( the subkey binding signature over the key exchange subkey OR the usual self-signature locations )) (?TBC?).
* A new signature subpacket type, "Flow Control", of the Document class.
    * It contains one or more octets of flags (see below).

We use standard PKESK+SEIPD sign-then-encrypt messages, but:

* The first message in each direction is encrypted to the recipient's standard encryption subkey.
   * It is signed by the sender's EKE subkey, not a signing key, and the signature contains an EK subpacket to initialise the double ratchet (once in each direction).
* Subsequent double-ratchet messages use PKESKs with slightly altered semantics:
    * The key ID (PKESKv3) or versioned fingerprint (PKESKv6) identifies the EKE subkey, not an encryption key.
    * The "public key algorithm" octet identifies an ephemeral cipher suite, not the algorithm of the EKE subkey.
    * The algorithm-specific data indicates the sender's current DR state, not an encrypted session key.
    * Session keys are obtained instead from the DR state.
    * DR messages are authenticated by knowledge of the DR state and are not normally signed.
* A DR message MAY include a Flow Control signature packet:
    * The Flow Control signature is placed inside the encrypted data, before the literal data packet (see below).
* Each DR message MUST include the sender's latest ephemeral public key:
    * It is contained in a Public Subkey packet of the appropriate ephemeral algorithm type.
    * The Public Subkey packet is placed inside the encrypted data, following the literal data packet.

Note that an ephemeral key is always represented by a Public *Subkey* packet, so that it cannot be mistaken for a TPK.
A Subkey Binding Signature MUST NOT be made over an ephemeral key.

### Algorithm-specific PKESK data

An ephemeral-algorithm PKESK contains the sender's full DR state for the current message, in addition to the symmetric algorithm used in the following SEIPD packet.
The symmetric algorithm MUST have the same key length as the ECDH algorithm of the current DR.

* Symmetric Algorithm (1 octet)
* Recipient's last published ephemeral key version (1 octet)
* Recipient's last published ephemeral key fingerprint (N octets)
* Symmetric Chain Sequence (4 octets) (?TBC?)

## Algorithm upgrades

During a long-running conversation it may eventually become necessary to upgrade the DR algorithm.
One party to a conversation may invoke an algorithm upgrade by appending two ephemeral keys to a message; one using the old algorithm and one using the new.
The old algorithm ephemeral key continues the current DR, while the new one attempts to initialise a new DR.
The other party can accept the upgrade by replying using the old algorithm but appending an ephemeral key of the new algorithm.

## Error recovery

The parties to a double-ratchet communications channel SHOULD negotiate an alternative encrypted channel for recovery if a DR update fails and messages become undecryptable.
The easiest way of doing this is to set up multiple independent DRs, and alternating messages between the DRs.

Multiple DRs may be initialised by including multiple EK subpackets in the initial message, and pairing them off in order with the EK subpackets in the response.
If the second party responds with a lower number of ephemeral keys, then the first party MUST destroy the excess keys.
After this step, each DR operates independently of the others.
To ensure that all DRs are working, messages SHOULD be sent using each of them in roughly equal proportions, however a predictable sequence is not required and MUST NOT be assumed.

### Flow Control signature

If a failure is detected in any of the double ratchets, a flow control signature SHOULD be sent using one of the other DRs.

A flow control signature is a standalone signature (type 0x02), made by the EKE subkey, which contains a Flow Control subpacket in the hashed area.
It is included in a DR message as the first packet of the encrypted data.
If a flow control signature is present, a literal data packet is OPTIONAL, but a new ephemeral key packet MUST be appended.

A Flow Control subpacket contains one or more octets of flags, each representing a particular flow control request.
The following flags are defined:

* Fork (0x01..): Forks the current double ratchet into two separate DRs.
    * An EK subpacket MUST be included in the hashed area.
    * A new DR is created by copying the state of the current DR and applying the ephemeral key from the EK subpacket.
    * The current DR continues as normal using the trailing ephemeral subkey packet.
* Reap (0x02..): Marks another double ratchet as broken and asks for its recent messages to be resent.
    * An Intended Recipient Fingerprint subpacket MUST be included in the hashed area.
    * The DR that contains the key identified by the IRF subpacket SHOULD be marked as broken at a point immediately before that key was applied.
    * Any messages sent using that DR (or any of its forks) since the break point SHOULD be resent using the current DR.
* Resend (0x04..): Asks for a double ratchet's historical messages to be resent, without marking the DR as broken.
    * An Intended Recipient Fingerprint subpacket MUST be included in the hashed area.
    * The IRF identifies the head of a particular symmetric chain.
    * Any messages sent using that symmetric chain only SHOULD be resent using the current DR.
    * Reap and Resend MUST NOT be set in the same Flow Control subpacket.
* Partial (0x20..): The current message represents a partial flow control response, and one or more continuation messages SHOULD follow.
    * Continuation messages SHOULD be sent in the same symmetric chain (this provides a convenient ordering).
    * Continuation messages MUST contain identical flow control response subpackets (modulo the Partial bit).
* ACK (0x40..): Positive acknowledgement of one or more flow control requests.
    * The flag bit(s) and subpacket(s) of the flow control request(s) being responded to MUST be included for reference.
* NAK (0x80..): Negative acknowledgement of one or more flow control requests.
    * The flag bit(s) and subpacket(s) of the flow control request(s) being responded to MUST be included for reference.

All other bits MUST be zero.

An application SHOULD restrict the number of double ratchets in use for a given connection.
If a corresponding party requests to Fork beyond that limit, the application SHOULD immediately respond with a NAK.
If a Flow Control subpacket indicates both Reap and Fork, the Reap SHOULD be processed first.

When a Reap or Resend request is processed, the resent messages MUST be prefixed by the appropriate ACK flow control packet.
If resending requires multiple response messages, the Partial flag MUST be set on all but the last message, and all response messages SHOULD be sent using the same symmetric chain.

### Last resort

If all else fails, one party may attempt to initialise a new double ratchet using the EKE subkey of the other party.
The other party's software SHOULD warn when this happens, and send a similar warning to the first party using all current DRs.

## Multi-party communications

When more than two parties are communicating, or one or more parties have multiple devices, one or more shared (( secrets OR double ratchets )) MUST also be created in addition to any pairwise ratchets.
These are used only for flow control messages and changes in the group state.

### Multiparty secret key agreement

* The members agree an ordering of themselves (say, by binary ordering of their KEK keys).
* Each member of the group generates their own public key (a)G and sends it to the next member in the list
    * This is signed with their KEK key, and includes an Intended Recipient Fingerprint subpacket indicating the KEK subkey of the *previous* member in the list
* Each member of the group reads the public key they just received and does the following:
    * They apply their own secret key by the ECDH method to produce a new public key (ab)G
    * They attach a note listing the people whose keys have been included in the calculation already
    * They send the new key to the next member in the list
* The process repeats with the new keys until everyone eventually receives a key that has an Intended Recipient of their own KEK subkey
* At this point they apply their own private key to obtain the shared secret key

(( TBC! This does not appear to scale well, and subsequent changes look like a nightmare... ))

## References

* [Signal's double ratchet specification](https://signal.org/docs/specifications/doubleratchet/)
* [Justus's presentation from 2018](https://sequoia-pgp.org/talks/2018-08-moving-forward/moving-forward.pdf)
* [Earlier PFS draft by Brown, Back, and Laurie (2003)](https://datatracker.ietf.org/doc/html/draft-brown-pgp-pfs-03)
* [Multi-party ECDH exchange](https://crypto.stackexchange.com/questions/72207/ecdh-for-more-than-two-parties)
