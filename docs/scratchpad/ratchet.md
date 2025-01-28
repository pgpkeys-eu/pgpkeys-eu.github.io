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

We prefer the use of a double ratchet mechanism because it automatically allows deniable authentication.
This is because authentication relies on knowledge of a shared secret and not asymmetric cryptography:

1. knowledge of the shared ratchet state is sufficient to for Bob to authenticate the messages sent by Alice.
2. a message from Alice to Bob could be convincingly forged by Bob after the fact.

The privacy effectiveness of the double ratchet is significantly improved if disappearing messages are also implemented.

## Design

We define:

* A new key capability flag, "Ephemeral Key Agreement" (EKA).
    * It is used on a subkey with a signing-capable algorithm, and permits that subkey to authenticate ephemeral key exchange.
* One or more new public-key algorithm IDs (in a special range) that identify ephemeral cipher suites.
    * Implementations MUST support X25519+HKDF+DR and MAY support X448+HKDF+DR.
* A new signature subpacket type, "Ephemeral Key", of the Document class, which contains:
    * A public ephemeral key, as a full Public Key packet using the appropriate ephemeral algorithm.
* A new signature subpacket type, "Preferred Ephemeral Ciphers", of the Direct class.
    * It contains an ordered list of public ephemeral key algorithms.
    * It is used in the same way as any other algorithm preference subpacket.
* A new signature subpacket type, "Flow Control request", of the Literal Data class.
    * It contains one octet of request type, and one octet of flags (see below).

We use standard OpenPGP encrypted messages, but:

* The initial message in each direction is encrypted to the recipient's standard encryption subkey.
    It is signed by the sender's EKA subkey (not their signing key) and the signature contains:
    * A Flow Control request subpacket with the DR Init flag set (see below).
    * An Ephemeral Key subpacket containing a public ephemeral key, to initialise a new double ratchet (once in each direction).
* Subsequent double-ratchet messages use PKESKs with slightly altered semantics:
    * The key ID (PKESKv3) or versioned fingerprint (PKESKv6) identifies the EKA subkey (not an encryption key).
    * The "public key algorithm" octet identifies an ephemeral cipher suite (not the algorithm of the EKA subkey).
    * The algorithm-specific data indicates the sender's current public ratchet state in addition to the DR session information.
    * Session keys are encrypted symmetrically using the key derived from the DR state.
    * DR messages are authenticated by knowledge of the DR state and are not normally signed.

Note that public ephemeral keys are always represented by a Public Key packet, so that they cannot be mistaken for a subkey of the keyholder's long-lived primary key.
The ephemeral key is therefore a primary encryption key, which can not make self-signatures.
Binding and Certification Signatures MUST NOT be made over an ephemeral key.

A single DR message may be encrypted to multiple recipients by prefixing it with multiple PKESKs.
Care must be taken to ensure that flow control request subpackets are located in the correct PKESK(s) according to their intended audience.

### Message format

A DR encrypted message body contains:

* A Literal Data packet.
* (Optional) A Padding packet.

The Literal Data packet SHOULD NOT be signed; any signature subpackets that would have been contained in a document signature SHOULD be stored in the PKESK instead (see below).

### Algorithm-specific PKESK data

An ephemeral-algorithm PKESK contains the sender's public ratchet state for the current message, in addition to the symmetric algorithm used in the following SEIPD packet.
The symmetric algorithm MUST have the same key length as the asymmetric algorithm of the current DR.

* Symmetric Algorithm (1 octet)
* Symmetric Chain Sequence (2 octets)
* Previous Symmetric Chain Sequence (2 octets)
* Length in octets of the following two fields (2 octets)
* Counterpart's last published ephemeral key version (1 octet)
* Counterpart's last published ephemeral key fingerprint (N octets)
* Symmetrically-encrypted DR session information

The Symmetric Chain Sequence and Previous Symmetric Chain Sequence fields are used to calculate the number of missing messages.
The DR session information is encrypted with the ratchet key calculated from the ratchet state, using the same symmetric algorithm as the encrypted data.
The decrypted DR session information contains:

* Symmetric session key
* A signature subpacket area length
* A signature subpacket area
* A full public key packet containing the sender's most recent public ephemeral key, to progress the asymmetric ratchet

The signature subpacket area contains any signature subpackets that would have been included in a signature over the encrypted data, if it had been signed (see below).
These MUST include a Signature Creation Time subpacket.
The subpackets are instead validated by implied knowledge of the shared ratchet state.
The Public Key packet MUST be of the same ephemeral algorithm type as the enclosing PKESK.
Unless otherwise specified, this signature subpacket area is equivalent to a hashed subpacket area for the purposes of BCP 14 [RFC2119] [RFC8174].

If the message's content can be encoded entirely within the PKESK, the Literal Data packet MAY be empty.

## Error recovery

The parties to a double-ratchet communications channel SHOULD negotiate an alternative encrypted channel for recovery if a DR update fails and messages become undecryptable.
The easiest way of doing this is to set up multiple independent DRs, and alternating messages between the DRs.

Multiple DRs may be initialised by including multiple Ephemeral Key subpackets in the initial message, and pairing them off in order with the Ephemeral Key subpackets in the initial response.
If the second party responds with fewer ephemeral keys, then the first party MUST destroy the excess keys.
After this step, each DR operates independently of the others.
If a failure is detected in any of the double ratchets, a flow control request SHOULD be sent using one of the other DRs.

If all else fails, one party may attempt to initialise a new double ratchet by encrypting a "first message" to the other party's long term encryption key, even though a DR session already exists.
The other party's software SHOULD warn when this happens, and send a similar warning to the first party using all current DRs.

## Flow control

Flow Control requests are located in the subpacket area of the DR session information.

A typical Flow Control request consists of one or more of the following subpackets, in addition to any generic subpackets such as signature creation time etc.:

* A Flow Control Request subpacket.
* An Ephemeral Key subpacket containing a public ephemeral key, to initialise a new double ratchet.
* An Intended Recipient subpacket, to identify another DR ratchet

### Flow control request subpacket

A Flow Control Request subpacket contains one octet of request type, and one octet of flags.

The following request types are defined:

* DR Init (0): Initialises a new DR.
    * An Ephemeral Key subpacket with an initial ephemeral key MUST be included in the hashed area.
    * The expected response to a DR Init request is another DR Init request containing the other party's initial ephemeral key.
* Fork (1): Forks the current double ratchet into two separate DRs.
    * An Ephemeral Key subpacket with an ephemeral key MUST be included in the hashed area.
    * A new DR is created by copying the state of the current DR and applying the ephemeral key from the Ephemeral Key subpacket.
    * The current DR continues as normal using the trailing ephemeral key packet.
* Reap (2): Marks another double ratchet as broken and asks for its recent messages to be resent.
    * An Intended Recipient Fingerprint subpacket MUST be included in the hashed area.
    * The IRF subpacket identifies an ephemeral key in another DR chain.
    * The DR containing that ephemeral key SHOULD be marked as broken at a point immediately before that key was applied.
    * Any messages sent using that DR (or any of its forks) since the break point SHOULD be resent using the current DR.
* Reap and Fork (3): Combines the semantics of both Reap and Fork, in that order.
* Resend (4): Asks for a double ratchet's historical messages to be resent, without marking the DR as broken.
    * An Intended Recipient Fingerprint subpacket MUST be included in the hashed area.
    * The IRF identifies the ephemeral key at the head of a symmetric chain.
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

### Algorithm upgrades

During a long-running conversation it may eventually become necessary to upgrade the DR algorithm.
One party to a conversation may invoke an algorithm upgrade by sending a DR Init request with an Ephemeral Key subpacket containing an ephemeral key of the new algorithm type.
The DR session information ephemeral key continues the current DR, while the Ephemeral Key subpacket attempts to initialise a new DR.
The other party can accept the upgrade by replying in the existing double ratchet with a matching DR Init request that uses the new algorithm.
Once both parties have sent DR Init requests, the new double ratchet acts as a direct continuation of the old.

## References

* [Signal's double ratchet specification](https://signal.org/docs/specifications/doubleratchet/)
* [Justus's presentation from 2018](https://sequoia-pgp.org/talks/2018-08-moving-forward/moving-forward.pdf)
* [Earlier PFS draft by Brown, Back, and Laurie (2003)](https://datatracker.ietf.org/doc/html/draft-brown-pgp-pfs-03)
