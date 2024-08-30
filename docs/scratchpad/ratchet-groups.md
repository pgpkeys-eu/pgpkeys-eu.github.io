# Double Ratchet Group Messages in OpenPGP

We can extend the OpenPGP double ratchet mechanism to support groups.

CAUTION: this does not currently scale well, and it is RECOMMENDED to instead construct groups at the application level and use multi-recipient DR messages as the transport layer.

## Extensions

The following extensions to the double ratchet mechanism are defined:

### Embedded keys

Embedded Key subpackets MAY also contain:

* A public deniable key, as a Public Subkey packet using a standard signing algorithm.
* A private deniable key, as a Private Subkey packet using a standard signing algorithm.

The initial encrypted message MAY also contain:

* An Embedded Key subpacket containing a public deniable subkey, for signing future flow control messages (instead of the EKA).
* Any (reasonable) number of Embedded Key subpacket(s) each containing a private deniable subkey to be burned.

### Message format

A DR encrypted message body contains at least one of:

* Zero or more Flow Control signatures (see below).
* A Literal Data packet.

### Algorithm-specific PKESK data

* Symmetric Algorithm (1 octet)
* Symmetric Chain Sequence (2 octets) (( overkill? TBC ))
* Length of the following (2N) fields
* Recipient's last published ephemeral subkey version (1 octet)
* Recipient's last published ephemeral subkey fingerprint (N octets)
* (optional) The previous two fields repeated N-1 times (for each intermediate subkey)
* Symmetrically-encrypted session information

The symmetrically-encrypted session information contains:

* Symmetric session key
* A signature subpacket area length
* A signature subpacket area
* A full public subkey packet containing the sender's most recent ephemeral public key
* (optional) One or more full public subkey packets containing any intermediate subkeys

### Flow control signatures

Flow Control signatures contain requests that are intended for all recipients of the message.
Their semantics are otherwise identical to Flow Control requests in the DR session information.

A Flow Control signature (0x03) is constructed the same way as a Standalone signature (type 0x02), and is made by either the EKA subkey or a deniable subkey.
It is included in a DR message as the first packet of the encrypted data.
If a flow control signature is present then a literal data packet is OPTIONAL.

In addition to the subpackets allowed in the DR session state, a Flow Control signature may also contain:

* An Embedded Key subpacket containing a public deniable subkey, for signing future flow control messages (instead of the EKA) (MUST be located in the hashed area).
* Any (reasonable) number of Embedded Key subpacket(s) each containing a private deniable subkey to be burned (MUST be located in the unhashed area).

Since a Flow Control signature is located inside the message, and may therefore be addressed to more than one person, and the group ratchet state is shared among all members, it is necessary to sign it with a deniable key to prevent one legitimate recipient of a message impersonating another. (( TBC does this not also apply to the actual message? ))

### Flow control types

* Group Init: Initialises a new group (see below).
    * One or more Intended Recipient Fingerprint subpackets MUST be included in the hashed area to indicate the initial group membership.
* Group Secret Init: Initialises a new group secret (see below).
    * An Embedded Key subpacket with an ephemeral subkey MUST be included in the hashed area.
    * An Intended Recipient Fingerprint subpacket MUST be included in the hashed area to indicate the last person in the ECDH chain.
* Group Invite: Invites a new member to the group.
    * One or more Intended Recipient Fingerprint subpackets MUST be included in the hashed area to indicate the extra group members.
* Group Leave: Asks the other members to remove the sender from the group.

## Round robin groups

In a round-robin group, all members are equal and all members must participate in order for the DH ratchet to be progressed.

### Round robin secret key agreement

A new group can be created by one party (the invitor) sending a Group Init flow control message separately to all invitees over existing secure channel(s).
The invitees are identified by Intended Recipient Fingerprint subpackets containing their EKA fingerprints.
The Literal Data SHOULD contain a keyring containing the TPKs of all invitees.

* The invitees order themselves into a cycle by ascending binary comparison of their EKA fingerprints.
* Each accepting invitee sends an ACK to the invitor and then:
    * They generate their own group private key (a) and an intermediate group public key (a)G.
    * They send (a)G to the next invitee using a Group Secret Init request.
    * This is signed with their EKA subkey, and includes an Intended Recipient Fingerprint subpacket indicating the EKA subkey of the *previous* invitee.
* Each invitee reads the intermediate public key sent by the previous invitee and then:
    * They send an ACK to the previous invitee.
    * They apply their own secret key (b) by the ECDH method to produce a new intermediate group public key (ab)G.
    * They send the new intermediate key to the next invitee using a Group Secret Init request.
    * This is signed with their EKA subkey, and includes a copy of the Intended Recipient Fingerprint subpacket from the received message.
* The process repeats until everyone eventually receives an intermediate key with an IRF identifying them.
* At this point they apply their own secret key to obtain the shared secret key.

In order to facilitate subsequent updates, each accepting invitee MUST retain a copy of all intermediate group public keys received.

As this does not scale well (O(n^2) messages), invitees SHOULD wait for multiple keys to arrive and then send on their corresponding intermediate keys in a single message containing multiple flow control signatures.
To avoid deadlock, the invitor takes on the role of "dealer":

* The dealer MUST send the initial intermediate group public key immediately.
* An invitee (A) MUST NOT wait for the arrival of an intermediate group public key with an IRF pointing to another invitee (B) unless:
    * B is before A in the list and after the dealer, or
    * B is the dealer, or
    * B is the last invitee in the list before the dealer.

An invitee can decline membership by sending a NAK to *all* other invitees.
All invitees MUST remove the declining invitee from their local copy of the membership list, and destroy any intermediate keys with an IRF identifying the declining invitee.
If the invitee immediately before the declining invitee has already sent one or more intermediate keys to the declining invitee:

* it removes the declining invitee from the list
* it destroys and recreates its initial group keypair
* it calculates a replacement intermediate key for each intermediate key already sent
* it sends the new intermediate keys to the new next invitee.

NOTE that a group cannot be used until the initial key agreement round is completed.
It is therefore RECOMMENDED that groups be initialised with a small number of members and expanded later.

### Round robin group secret ratchet

The group secret SHOULD be ratcheted regularly, however this will generally take place at a slower rate than a private ECDH ratchet.
Since group members will not in general send messages in a predictable order, this is done on an opportunistic basis.

The group members agree a method of allocating dealer windows to each member using a deterministic method (( TBC )).
During a member's dealer window, that member MAY initiate a ratchet round by acting as dealer and (( TBC )).

### Leaving a round robin group

To leave a group, the leaving member sends a Group Leave flow control message to all other members and then destroys their copy of the group secret(s).
No response is expected.
A new ratchet round MUST be kicked off immediately, with the leaving member removed from the list.
Remaining members SHOULD include a verbatim copy of the Group Leave flow control message in the subsequent ratchet round's messages, in case any members did not receive the original.

### Round robin group expansion

To invite a new member to the group, an existing member (the invitor) sends a Group Invite flow control message to both the group and the invitee.
The message sent to the group includes an IRF subpacket identifying the new user.
The message sent to the invitee includes IRF subpackets identifying the other group members.
The invitee then sends an ACK to all existing members.

A new group secret MUST then be generated, with the invitee taking on the role of dealer:

* The invitee creates a new initial group keypair and calculates its initial group public key as above
* The invitee sends its initial group public key to all existing group members (including the invitor) including all members in IRFs
* The invitor sends the invitee a copy of the last intermediate group public key addressed to the invitor, but with the invitee's EKA in the IRF
* The invitee applies its group private key to the received intermediate group public key to produce a new group secret

(( TBC ))

## Tree groups

A more robust method of managing a group arranges the members into a binary tree, each tuple of which represents a subtree of the group.

The public ratchet state identifier in the PKESK is postfixed by the subtree's public shared ratchet state identifier, and the trailing ephemeral key sent inside every encrypted message is postfixed by the subtree's shared ephemeral key.
If there are multiple layers of subtrees, each public ratchet state is postfixed by the enclosing subtree's public ratchet state, and each key is postfixed by the enclosing subtree's ephemeral key.
Each group member therefore maintains O(log n) ratchet states and ephemeral keys, one for each subtree in its branch.

### Tree group secret key agreement

A new group can be created by one party (the invitor) sending a Group Init flow control message separately to all invitees over existing secure channel(s).
The invitees are identified by Intended Recipient Fingerprint subpackets containing their EKA fingerprints.
The Literal Data SHOULD contain a keyring containing the TPKs of all invitees.

Each invitee SHOULD respond to all other invitees with an ACK containing a new ephemeral subkey.
An invitee can instead decline membership by sending a NAK to all other invitees.
All invitees MUST remove the declining invitee from their local copy of the membership list.

The accepting invitees are arranged into a tree structure based on their EKA fingerprints.
As each accepting invitee sees its immediate peer publish their ephemeral key, it initialises an intermediate keypair using its private ephemeral key and their public ephemeral key.
It then publishes the new intermediate public key by postfixing it to its own ephemeral public key in an ACK message encrypted to the EKA subkeys of the accepting members.

As each member of each subtree sees that subtree's peer subtree initialise, they can generate a further intermediate keypair for the enclosing subtree and postfix its public key to the intermediate public key chain.
This process continues until the group keypair is initialised.

### Tree group secret ratchet

The group DH ratchet of a tree group is similar in principle to the round-robin ratchet, except that the intermediate keys are calculated depth-first.
Since DH exponentiation is commutative, the order in which each member responds is irrelevant.

The PKESK identifies the sender's current position in the tree, and each recipient can then determine the smallest subtree containing both them and the sender.
Any tree levels below the common subtree can then be ignored, and the DH algorithm applied to the local state of the common subtree and above.

### Leaving a tree group

To leave a tree group, the leaving member sends a Group Leave flow control message to all other members and then destroys their copy of the group secret(s).
No response is expected.
The remaining member of the leaver's subtree gains sole control of that subtree's intermediate key.
Each remaining member SHOULD include a verbatim copy of the Group Leave flow control message in at least one subsequent message, in case any members did not receive the original.

### Tree group expansion

A new member can be added to an existing group by one party (the invitor) sending a Group Invite flow control message to the group, and separately to all invitees over existing secure channel(s).
The invitees and group members are identified by Intended Recipient Fingerprint subpackets containing their EKA fingerprints.
The Literal Data SHOULD contain a keyring containing the TPKs of all group members and invitees.

Each invitee SHOULD respond to all other invitees and group members with an ACK containing a new ephemeral subkey.
An invitee can instead decline membership by sending a NAK to all other invitees and group members.
All invitees and group members MUST remove the declining invitee from their local copy of the membership list.

The invitor shares the current private ratchet state of the group with each accepting invitee so that they can immediately start decrypting group messages.

The accepting invitees are inserted into the existing tree structure based on their EKA fingerprints.
As a group member sees its new peer publish their ephemeral key, it initialises an intermediate keypair using its private ephemeral key and their public ephemeral key.
It then publishes the new intermediate public key by postfixing it to its own ephemeral public key in subsequent messages, and DH agreement proceeds as usual.

### Pseudorandom tree construction

A pseudorandom binary tree allocates each member to a tree location based purely on a pseudorandom property of the member.
A natural choice for the pseudorandom property is the EKA key fingerprint.
We construct a binary tree using postfix matching, where items that share postfix digits also share a subtree, and the length of the matching postfix determines the depth of the subtree that they share.
If at any postfix length there are no matches, the tuple at that depth is omitted.
This constructs a unique pseudorandom binary tree that is stable under the addition and deletion of members and scales as O(log(n)).

Consider a toy model where key fingerprints are only four bits long, but remain unique.
Let us arrange some group members into a binary tree by postfix matching:

            0100      0111      1001      1011      1101      1111

          ((0100      0111)   ((1001      1011)    (1101      1111)))

If we insert a new member 1010, this becomes:

                                     1010                
            0100      0111      1001      1011      1101      1111

          ((0100      0111)  ((1001 (1010 1011))   (1101      1111)))

The new member 1010 has a three-bit longest postfix match "101" with the existing member 1011.
It therefore forms a new tuple with that member at postfix depth 3.

If we then insert a new member 0010, the tree becomes:

      0010                                           
            0100      0111      1001 1010 1011      1101      1111

    ((0010 (0100      0111) )((1001 (1010 1011))   (1101      1111)))

The new member 0010 has a one-bit longest postfix match "0" with the entire subtree (0100 0111).
It therefore forms a new tuple with that subtree at postfix depth 1.

## Notes on groups (shared)

To identify a group, a notation subpacket SHOULD be included in all flow control signatures associated with that group.
The notation subpacket is marked as not human readable, with a key of "groupid" and a random value of at least 128 bits.
A group MAY also have a human readable name, with a key of "groupname".

A group ceases to exist when its membership has been reduced to one.

## Device groups

If a single identity is associated with multiple devices, the devices form a device group with each other using the group mechanism above.
If a device is a member of a device group, then the ECDH private keys used by that identity are generated from the device group double ratchet via a KDF.
If one device sees that another device in its device group has generated a public key with an unknown fingerprint, it progresses the symmetric ratchet until it finds the corresponding secret.

## References

* [Multi-party ECDH exchange](https://crypto.stackexchange.com/questions/72207/ecdh-for-more-than-two-parties)
