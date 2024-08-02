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
* A new signature subpacket type, "Embedded Key", of the Document class. (( TBC: iff not already defined in draft-revocation ))
    * It contains the sender's initial ephemeral public key, as a full Public Subkey packet using the appropriate ephemeral algorithm.
* A new signature subpacket type, "Preferred Ephemeral Ciphers", of the Preference class.
    * It contains an ordered list of ephemeral public key algorithms.
    * It is used in (( the subkey binding signature over the key exchange subkey OR the usual self-signature locations )) (?TBC?).
* A new signature subpacket type, "Flow Control", of the Document class.
    * It contains one octet of request type, and one octet of flags (see below).

We use standard PKESK+SEIPD sign-then-encrypt messages, but:

* The initial message in each direction is encrypted to the recipient's standard encryption subkey.
    * It is signed by the sender's EKE subkey (not their signing key) and the signature contains:
        * A Flow Control subpacket with the DR Init flag set (see below).
        * An Embedded Key subpacket containing an ephemeral subkey, to initialise the double ratchet (once in each direction).
* Subsequent double-ratchet messages use PKESKs with slightly altered semantics:
    * The key ID (PKESKv3) or versioned fingerprint (PKESKv6) identifies the EKE subkey (not an encryption key).
    * The "public key algorithm" octet identifies an ephemeral cipher suite (not the algorithm of the EKE subkey).
    * The algorithm-specific data indicates the sender's current DR state (not an encrypted session key).
    * Session keys are obtained instead from the DR state.
    * DR messages are authenticated by knowledge of the DR state and are not normally signed.

A DR encrypted message body contains one or both of a Standalone signature and/or a Literal Data packet, and a mandatory Public Subkey packet.
The Literal Data packet and the Public Subkey packet are not signed.

* Any non-routine metadata is contained in a leading Standalone signature.
* The Literal Data packet contains the message (if any).
* The Public Subkey packet MUST be of the same ephemeral algorithm type as used for the encrypted container.

Note that an ephemeral public key is always represented by a Public *Subkey* packet, so that it cannot be mistaken for a TPK.
A Subkey Binding Signature MUST NOT be made over an ephemeral subkey.

### Algorithm-specific PKESK data

An ephemeral-algorithm PKESK contains the sender's full DR state for the current message, in addition to the symmetric algorithm used in the following SEIPD packet.
The symmetric algorithm MUST have the same key length as the ECDH algorithm of the current DR.

* Symmetric Algorithm (1 octet)
* Symmetric Chain Sequence (4 octets) (( overkill? TBC ))
* Recipient's last published ephemeral subkey version (1 octet)
* Recipient's last published ephemeral subkey fingerprint (N octets)

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

A flow control signature is a Standalone signature (type 0x02), made by the EKE subkey, which contains a Flow Control subpacket in the hashed area.
It is included in a DR message as the first packet of the encrypted data.
If a flow control signature is present then a literal data packet is OPTIONAL.

A Flow Control subpacket contains one octet of request type, and one octet of flags.

The following request types are defined:

* DR Init (0): Initialises a new DR.
    * An Embedded Key subpacket with an initial ephemeral subkey MUST be included in the hashed area.
    * The expected response to a DR Init request is another DR Init request containing the other party's initial ephemeral subkey.
* Fork (1): Forks the current double ratchet into two separate DRs.
    * An Embedded Key subpacket with an ephemeral subkey MUST be included in the hashed area.
    * A new DR is created by copying the state of the current DR and applying the ephemeral subkey from the Embedded Key subpacket.
    * The current DR continues as normal using the trailing ephemeral subkey packet.
* Reap (2): Marks another double ratchet as broken and asks for its recent messages to be resent.
    * An Intended Recipient Fingerprint subpacket MUST be included in the hashed area.
    * The IRF subpacket identifies an ephemeral subkey in another DR chain.
    * The DR containing that ephemeral subkey SHOULD be marked as broken at a point immediately before that subkey was applied.
    * Any messages sent using that DR (or any of its forks) since the break point SHOULD be resent using the current DR.
* Reap and Fork (3): Combines the semantics of both Reap and Fork, in that order.
* Resend (4): Asks for a double ratchet's historical messages to be resent, without marking the DR as broken.
    * An Intended Recipient Fingerprint subpacket MUST be included in the hashed area.
    * The IRF identifies the ephemeral subkey at the head of a symmetric chain.
    * Any messages sent using that symmetric chain only SHOULD be resent using the current DR.
* Group Init (5): Initialises a new group (see below).
    * One or more Intended Recipient Fingerprint subpackets MUST be included in the hashed area to indicate the initial group membership.
* Group Secret Init (6): Initialises a new group secret (see below).
    * An Embedded Key subpacket with an ephemeral subkey MUST be included in the hashed area.
    * An Intended Recipient Fingerprint subpacket MUST be included in the hashed area to indicate the last person in the ECDH chain.
* Group Invite (7): Invites a new member to the group.
    * One or more Intended Recipient Fingerprint subpackets MUST be included in the hashed area to indicate the extra group members.
* Group Leave (8): Asks the other members to remove the sender from the group.
    * A group is destroyed once the last member leaves.

The following flag bits are defined:

* Partial (0x01): The current message represents a partial flow control response message, and one or more continuation messages SHOULD follow.
    * Continuation messages SHOULD be sent in the same symmetric chain (this provides a convenient ordering).
    * Continuation messages MUST contain identical flow control response subpackets (modulo the Partial bit).
* ACK (0x02): Positive acknowledgement of one or more flow control requests.
    * The request type is set to the type of the request being responded to.
    * The subpacket(s) of the flow control request being responded to MUST be included for reference.
* NAK (0x03): Negative acknowledgement of one or more flow control requests.
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

## Multi-party communications ((WIP))

When more than two parties are communicating, or one or more parties have multiple devices, one or more shared (( secrets OR double ratchets )) MUST also be created in addition to any pairwise ratchets.
These are used only for flow control messages and changes in the group state.

(( BEWARE that everything below here is highly flaky and almost certainly doesn't scale. Also, the more people are in a group the less value a DR scheme adds, so there may be no point. Don't @ me ))

### Multiparty secret key agreement

A new group can be created by one party sending a Group Init flow control message separately to all invitees over an existing secure channel, with the invitees' EKE subkeys identified by Intended Recipient Fingerprint subpackets.
The Literal Data SHOULD contain a keyring containing the TPKs of all invitees.

* The invitees order themselves into a cycle by ascending binary comparison of their EKE fingerprints.
* Each accepting invitee sends an ACK to all other invitees and then:
    * They generate their own group private key (a) and public key (a)G.
    * They send (a)G to the next invitee using a Group Secret Init request.
    * This is signed with their EKE subkey, and includes an Intended Recipient Fingerprint subpacket indicating the EKE subkey of the *previous* invitee.
* Each subsequent invitee reads the public key they just received and then:
    * They send an ACK to the sender.
    * They apply their own secret key (b) by the ECDH method to produce a new public key (ab)G.
    * They send the new key to the next invitee using a Group Secret Init request.
    * This is signed with their EKE subkey, and includes the Intended Recipient Fingerprint subpacket from the received message.
* The process repeats with the new keys until everyone eventually receives a key with an Intended Recipient Fingerprint of their own EKE subkey.
* At this point they apply their own secret key to obtain the shared secret key.

Beware that this does not scale well, so it is normally advisable to initialise a small group and then add members individually.

An invitee can decline membership by sending a NAK to all other invitees.
If the previous invitee has already sent one or more Group Secret Init messages to the declining invitee, it destroys and recreates its group keypair, and sends corresponding Group Secret Init messages to the next invitee in the list.

To identify the group, a notation subpacket SHOULD be included in all flow control signatures associated with that group.
The notation subpacket is marked as not human readable, with a key of "groupid" and a random value of at least 128 bits.
A group MAY also have a human readable name, with a key of "groupname".

### Group membership adjustment

To invite a new member to the group, an existing user sends a Group Invite flow control message to both the group and the new user.
The message sent to the group includes an IRF subpacket identifying the new user.
The message sent to the new user includes IRF subpackets identifying the other group members.

A new group secret MUST then be generated (( TBC )).

To leave a group, an existing member sends a Group Leave flow control message and then destroys their copy of the group secret.
No response is expected.

A new group secret MUST then be generated.

## References

* [Signal's double ratchet specification](https://signal.org/docs/specifications/doubleratchet/)
* [Justus's presentation from 2018](https://sequoia-pgp.org/talks/2018-08-moving-forward/moving-forward.pdf)
* [Earlier PFS draft by Brown, Back, and Laurie (2003)](https://datatracker.ietf.org/doc/html/draft-brown-pgp-pfs-03)
* [Multi-party ECDH exchange](https://crypto.stackexchange.com/questions/72207/ecdh-for-more-than-two-parties)
