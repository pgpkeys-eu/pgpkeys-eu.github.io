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

* A new key capability flag, "Ephemeral Key Agreement" (EKA).
    * It is used on a subkey with a signing-capable algorithm, and permits that subkey to authenticate ephemeral key exchange.
* One or more new public-key algorithm IDs (in a special range) that identify ephemeral cipher suites.
    * Implementations MUST support X25519+HKDF+DR and MAY support X448+HKDF+DR.
* A new signature subpacket type, "Embedded Key", of the Document class (( TBC: iff not already defined in draft-revocation )), which contains one of:
    * An public ephemeral key, as a full Public Subkey packet using the appropriate ephemeral algorithm.
    * A public deniable key, as a Public Subkey packet using a standard signing algorithm.
    * A private deniable key, as a Private Subkey packet using a standard signing algorithm.
* A new signature subpacket type, "Preferred Ephemeral Ciphers", of the Preference class.
    * It contains an ordered list of public ephemeral key algorithms.
    * It is used in (( the subkey binding signature over the key exchange subkey OR the usual self-signature locations )) (?TBC?).
* A new signature subpacket type, "Flow Control", of the Document class.
    * It contains one octet of request type, and one octet of flags (see below).

We use standard OpenPGP encrypted messages, but:

* The initial message in each direction is encrypted to the recipient's standard encryption subkey.
    It is signed by the sender's EKA subkey (not their signing key) and the signature contains:
    * A Flow Control subpacket with the DR Init flag set (see below).
    * An Embedded Key subpacket containing an public ephemeral subkey, to initialise a new double ratchet (once in each direction).
    * An Embedded Key subpacket containing a public deniable subkey, for signing future flow control messages (instead of the EKA).
    * Any (reasonable) number of Embedded Key subpacket(s) each containing a private deniable subkey to be burned.
* Subsequent double-ratchet messages use PKESKs with slightly altered semantics:
    * The key ID (PKESKv3) or versioned fingerprint (PKESKv6) identifies the EKA subkey (not an encryption key).
    * The "public key algorithm" octet identifies an ephemeral cipher suite (not the algorithm of the EKA subkey).
    * The algorithm-specific data indicates the sender's current public ratchet state in addition to the encrypted session information.
    * Session keys are encrypted symmetrically using the key derived from the DR state.
    * DR messages are authenticated by knowledge of the DR state and are not normally signed.

A DR encrypted message body contains at least one of:

* Zero or more Standalone (Flow Control) signatures (see below).
* A Literal Data packet.

The Literal Data packet is not signed over; any signature subpackets that would have been contained in a document signature SHOULD be stored in the PKESK instead (see below).

* Any non-routine metadata is contained in the leading Flow Control signature(s).
* The Literal Data packet contains the actual message (if any).

Note that ephemeral and public deniable keys are always represented by a Public *Subkey* packet, so that they cannot be mistaken for a TPK.
A Subkey Binding Signature MUST NOT be made over an ephemeral or deniable subkey.

### Algorithm-specific PKESK data

An ephemeral-algorithm PKESK contains the sender's public ratchet state for the current message, in addition to the symmetric algorithm used in the following SEIPD packet.
The symmetric algorithm MUST have the same key length as the ECDH algorithm of the current DR.

* Symmetric Algorithm (1 octet)
* Symmetric Chain Sequence (2 octets) (( overkill? TBC ))
* Length of the following two fields
* Recipient's last published ephemeral subkey version (1 octet)
* Recipient's last published ephemeral subkey fingerprint (N octets)
* Symmetrically-encrypted session information

The session information is encrypted with the ratchet key calculated from the ratchet state, using the same symmetric algorithm as the encrypted data.
The decrypted session information contains:

* Symmetric session key
* A signature subpacket area length (possibly zero)
* A signature subpacket area (possibly empty)
* A full public subkey packet containing the sender's most recent public ephemeral key, to progress the DH ratchet

The signature subpacket area contains any signature subpackets that would have been included in a signature over the encrypted data, if it had been signed.
The subpackets are instead validated by implied knowledge of the shared ratchet state.
The Public Subkey packet MUST be of the same ephemeral algorithm type as the enclosing PKESK.

## Error recovery

The parties to a double-ratchet communications channel SHOULD negotiate an alternative encrypted channel for recovery if a DR update fails and messages become undecryptable.
The easiest way of doing this is to set up multiple independent DRs, and alternating messages between the DRs.

Multiple DRs may be initialised by including multiple Embedded Key subpackets in the initial message, and pairing them off in order with the Embedded Key subpackets in the initial response.
If the second party responds with fewer ephemeral subkeys, then the first party MUST destroy the excess keys.
After this step, each DR operates independently of the others.
To ensure that all DRs are working, messages SHOULD be sent using each of them in roughly equal proportions, however a predictable sequence is not required and MUST NOT be assumed.

If a failure is detected in any of the double ratchets, a flow control signature SHOULD be sent using one of the other DRs.

If all else fails, one party may attempt to initialise a new double ratchet by encrypting a "first message" to the other party's long term encryption key, even though a DR session already exists.
The other party's software SHOULD warn when this happens, and send a similar warning to the first party using all current DRs.

## Flow control

A Flow Control signature is a Standalone signature (type 0x02), made by either the EKA subkey or a deniable subkey, which contains a Flow Control subpacket in the hashed area.
(( TBC: we could instead define a Flow Control signature as a distinct document signature type, say 0x03 ))
It is included in a DR message as the first packet of the encrypted data.
If a flow control signature is present then a literal data packet is OPTIONAL.

A typical Flow Control signature will contain one or more of the following subpackets, in addition to any generic subpackets such as signature creation time etc.:

* (mandatory) A Flow Control subpacket.
* An Embedded Key subpacket containing an public ephemeral subkey, to initialise a new double ratchet (once in each direction).
* An Embedded Key subpacket containing a public deniable subkey, for signing future flow control messages (instead of the EKA).
* Any (reasonable) number of Embedded Key subpacket(s) each containing a private deniable subkey to be burned (MUST be located in the unhashed area).

### Flow control subpacket

A Flow Control subpacket contains one octet of request type, and one octet of flags.

The following request types are defined:

* None (0): Does nothing, only used as a container for other subpackets (e.g. to roll a deniable subkey).
* DR Init (1): Initialises a new DR.
    * An Embedded Key subpacket with an initial ephemeral subkey MUST be included in the hashed area.
    * The expected response to a DR Init request is another DR Init request containing the other party's initial ephemeral subkey.
* Fork (2): Forks the current double ratchet into two separate DRs.
    * An Embedded Key subpacket with an ephemeral subkey MUST be included in the hashed area.
    * A new DR is created by copying the state of the current DR and applying the ephemeral subkey from the Embedded Key subpacket.
    * The current DR continues as normal using the trailing ephemeral subkey packet.
* Reap (3): Marks another double ratchet as broken and asks for its recent messages to be resent.
    * An Intended Recipient Fingerprint subpacket MUST be included in the hashed area.
    * The IRF subpacket identifies an ephemeral subkey in another DR chain.
    * The DR containing that ephemeral subkey SHOULD be marked as broken at a point immediately before that subkey was applied.
    * Any messages sent using that DR (or any of its forks) since the break point SHOULD be resent using the current DR.
* Reap and Fork (4): Combines the semantics of both Reap and Fork, in that order.
* Resend (5): Asks for a double ratchet's historical messages to be resent, without marking the DR as broken.
    * An Intended Recipient Fingerprint subpacket MUST be included in the hashed area.
    * The IRF identifies the ephemeral subkey at the head of a symmetric chain.
    * Any messages sent using that symmetric chain only SHOULD be resent using the current DR.

The following flag bits are defined:

* Partial (0x01): The current message represents a partial flow control response message, and one or more continuation messages SHOULD follow.
    * Continuation messages SHOULD be sent in the same symmetric chain (this provides a convenient ordering).
    * Continuation messages MUST contain identical flow control response subpackets (modulo the Partial bit).
* ACK (0x02): Positive acknowledgement of one or more flow control requests.
    * The request type is set to the type of the request being responded to.
    * The subpacket(s) of the flow control request being responded to MUST be included for reference.
* NAK (0x04): Negative acknowledgement of one or more flow control requests.
    * The request type is set to the type of the request being responded to.
    * The subpacket(s) of the flow control request(s) being responded to MUST be included for reference.

All other flag bits MUST be zero.

An application SHOULD restrict the number of double ratchets in use for a given connection.
If a corresponding party requests to Fork beyond that limit, the application SHOULD immediately respond with a NAK.
In the case of a Reap and Fork request, the Reap MUST be processed first.

When a Reap or Resend request is processed, the resent messages MUST be accompanied by the corresponding ACK flow control packet.
If resending requires multiple response messages, the Partial flag MUST be set on all but the last message, and all response messages SHOULD be sent using the same symmetric chain.

## Algorithm upgrades

During a long-running conversation it may eventually become necessary to upgrade the DR algorithm.
One party to a conversation may invoke an algorithm upgrade by sending a DR Init request with an Embedded Key subpacket containing an ephemeral subkey of the new algorithm type.
The message ephemeral subkey continues the current DR, while the Embedded Key subpacket in the flow control signature attempts to initialise a new DR.
The other party can accept the upgrade by replying in the existing double ratchet with a matching DR Init request that uses the new algorithm.
Once both parties have sent DR Init requests, the new double ratchet acts as a direct continuation of the old.

## References

* [Signal's double ratchet specification](https://signal.org/docs/specifications/doubleratchet/)
* [Justus's presentation from 2018](https://sequoia-pgp.org/talks/2018-08-moving-forward/moving-forward.pdf)
* [Earlier PFS draft by Brown, Back, and Laurie (2003)](https://datatracker.ietf.org/doc/html/draft-brown-pgp-pfs-03)
