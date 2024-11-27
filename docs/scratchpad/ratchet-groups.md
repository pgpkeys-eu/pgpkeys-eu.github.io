# Double Ratchet Group Messages in OpenPGP

We can extend the OpenPGP double ratchet mechanism to support groups.

CAUTION: this does not currently scale well, and it is RECOMMENDED to instead construct groups at the application level and use multi-recipient DR messages as the transport layer.

## Extensions

The following extensions to the double ratchet mechanism are defined:

### Ephemeral Keys

The Ephemeral Key subpacket MAY contain an ordered list of ephemeral subkey packets.

### Deniable Keys

We define two new signature subpacket types:

* Deniable Keys
    * These contain one or more public keys,for signing future flow control requests (instead of the EKA).
    * The keys are contained in Public Subkey packets and use a standard signing algorithm.
    * They MUST be located in the hashed area of a signature.
* Deniable Private Keys 
    * These contain one or more private deniable keys containing private keys to be burned.
    * They MUST be located in the unhashed area of a signature.

The initial encrypted message MAY contain zero or more of these subpackets, however they MUST NOT be present in the subpacket area of a PKESK.

### Message format

A DR encrypted message body contains:

* An OPS packet or document signature packet.
* A Literal Data packet.
* (If an OPS packet is present) A document signature packet.
* (Optional) A Padding packet.

Since the group ratchet state is shared among all members, the deniable authentication properties of PFS only authenticate that a message was sent by some member of the group.
It is therefore necessary to explicitly sign group messages to prevent one group member impersonating another.
A group DR message SHOULD be signed by a deniable subkey.

DR message signatures also contain Flow Control requests that are intended for all recipients of the message.
Their semantics are otherwise identical to Flow Control requests in the DR session information.

In addition to the subpackets allowed in the DR session state, a DR message signature may also contain Deniable Keys and Deniable Private Keys subpacket(s).

### Algorithm-specific PKESK data

The DR PKESK is generalised to contain N linked DR states:

* Symmetric Algorithm (1 octet)
* Symmetric Chain Sequence (2 octets)
* Previous Symmetric Chain Sequence (2 octets)
* Length in octets of the following (2N) fields (2 octets)
* Counterpart's last published ephemeral subkey version (1 octet)
* Counterpart's last published ephemeral subkey fingerprint (N octets)
* The previous two fields repeated N-1 times
* Symmetrically-encrypted session information

The symmetrically-encrypted session information contains:

* Symmetric session key
* A signature subpacket area length
* A signature subpacket area
* A full public subkey packet containing the sender's most recent ephemeral public key
* Zero or more full public subkey packets containing any intermediate subkeys

### Flow control types

The following group flow control types are defined:

* Group Init: Initialises a new group (see below).
    * One or more Intended Recipient Fingerprint subpackets MUST be included in the hashed area to indicate the initial group membership.
* Group DR Init: Initialises a new group DR within an existing group (see below).
    * An Ephemeral Key subpacket with an ephemeral subkey and one or more intermediate subkeys MUST be included in the hashed area.
    * An Intended Recipient Fingerprint subpacket MUST be included in the hashed area to indicate the last person in the ECDH chain.
* Group Invite: Invites a new member to the group.
    * One or more Intended Recipient Fingerprint subpackets MUST be included in the hashed area to indicate the extra group members.
* Group Leave: Asks the other members to remove the sender from the group.

## Round robin groups

In a round-robin group, all members are equal and all members must participate in order for the ECDH ratchet to be progressed.

### Round robin secret key agreement

A new group can be created by one party (the invitor) sending a Group Init flow control request separately to all invitees over existing secure channel(s).
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

As this does not scale well (O(n^2) messages), invitees SHOULD wait for multiple keys to arrive and then send on their corresponding intermediate keys in a single message.
The invitee postfixes the extra intermediate keys to its own ephemeral public key, and ECDH agreement proceeds as usual.
Multiple keys are always ordered by increasing distance from the sender, with the sender's own key first.

To avoid deadlock, the invitor takes on the role of "dealer":

* The dealer MUST send the initial intermediate group public key immediately.
* An invitee (A) MUST NOT wait for the arrival of an intermediate group public key with an IRF pointing to another invitee (B) unless:
    * B is before A in the list and after the dealer, or
    * B is the dealer, or
    * B is the last invitee in the list before the dealer.

An invitee can decline membership by sending a NAK to *all* other invitees.
All invitees MUST remove the declining invitee from their local copy of the membership list, and treat any intermediate keys with an IRF identifying the declining invitee as if it identified the previous invitee in the list.
If the invitee immediately before the declining invitee has already sent one or more intermediate keys to the declining invite, it resends the remaining intermediate keys to the new next invitee.

NOTE that a group cannot be used until the initial key agreement round is completed.
It is therefore RECOMMENDED that groups be initialised with a small number of members and expanded later.

### Round robin group secret ratchet

The group secret is ratcheted continuously:

* each member updates their ephemeral keys as soon as the ECDH ratchet round is complete
* each member updates their intermediate keys as the previous member in the list publishes theirs

Due to the requirement for all members to participate in each ratchet round, ratcheting will generally take place at a slower rate than a private ECDH ratchet (measured in messages per ratchet round).

### Leaving a round robin group

To leave a group, the leaving member sends a Group Leave flow control request to all other members and then destroys their copy of the group secret(s).
No response is sent to the leaving member.
A new ratchet round MUST be kicked off immediately, with the leaving member removed from the members list.
Remaining members MUST include a copy of the Group Leave flow control request in the subsequent ratchet round, with the ACK flag set.

### Round robin group expansion

To invite a new member to the group, an existing member (the invitor) sends a Group Invite flow control request to both the group and the invitee.
The message sent to the group includes an IRF subpacket identifying the new user.
The message sent to the invitee includes IRF subpackets identifying the other group members.
The invitee then sends an ACK to all existing members.

The invitor shares the current private ratchet state of the group with each accepting invitee so that they can immediately start decrypting group messages.

The accepting invitee(s) are inserted into the existing list based on their EKA fingerprints.
As a group member sees its previous peer publish their ephemeral key, it initialises an intermediate keypair using its private ephemeral key and their public ephemeral key.
It then publishes the new intermediate public key by postfixing it to its own ephemeral public key in subsequent messages, and ECDH agreement proceeds as usual.

## Tree groups

A more efficient method of managing a large group arranges the members into a binary tree, each tuple of which represents a subtree of the group.

The public ratchet state identifier in the PKESK is postfixed by the subtree's public shared ratchet state identifier, and the trailing ephemeral key sent inside every encrypted message is postfixed by the subtree's shared ephemeral key.
If there are multiple layers of subtrees, each public ratchet state is postfixed by the enclosing subtree's public ratchet state, and each key is postfixed by the enclosing subtree's ephemeral key.
Each group member therefore maintains O(log n) ratchet states and ephemeral keys, one for each subtree in its branch.

### Tree group secret key agreement

A new group can be created by one party (the invitor) sending a Group Init flow control request separately to all invitees over existing secure channel(s).
The invitees are identified by Intended Recipient Fingerprint subpackets containing their EKA fingerprints.
The Literal Data SHOULD contain a keyring containing the TPKs of all invitees.

Each invitee SHOULD respond to all other invitees with an ACK containing a new ephemeral subkey.
An invitee can instead decline membership by sending a NAK to all other invitees.
All invitees MUST remove the declining invitee from their local copy of the membership list.

The accepting invitees are arranged into a tree structure based on their EKA fingerprints.
As each accepting invitee sees its immediate peer publish their ephemeral key, it initialises an intermediate keypair using its private ephemeral key and their public ephemeral key.
It then publishes the new intermediate public key by postfixing it to its own ephemeral public key in an ACK message encrypted to the EKA subkeys of the accepting members.

As each member of each subtree sees that subtree's peer subtree initialise, they can generate a further intermediate keypair for the enclosing subtree and postfix its public key to the intermediate public key chain.
This process continues until the group ECDH ratchet is initialised.

### Tree group secret ratchet

The group ECDH ratchet of a tree group is similar in principle to the round-robin ratchet, except that the intermediate keys are calculated depth-first.
Since ECDH exponentiation is commutative, the order in which each member responds is irrelevant.

The PKESK identifies the sender's current position in the tree, and each recipient can then determine the smallest subtree containing both them and the sender.
Any tree levels below the common subtree can then be ignored, and the ECDH algorithm applied to the local state of the common subtree and above.

In the PKESK algorithm-specific data, the "Counterpart" is the other direct member of the tuple at each enclosing subtree level, whether that is an individual or a subtree.
In the case of a subtree, the "Counterpart Ephemeral Key" refers to the subtree's intermediate key.

### Leaving a tree group

To leave a tree group, the leaving member sends a Group Leave flow control request to all other members and then destroys their copy of the group secret(s).
No response is expected.
The remaining member of the leaver's subtree gains sole control of that subtree's intermediate key.
Each remaining member SHOULD include a verbatim copy of the Group Leave flow control request in at least one subsequent message, in case any members did not receive the original.

### Tree group expansion

A new member can be added to an existing group by one party (the invitor) sending a Group Invite flow control request to the group, and separately to all invitees over existing secure channel(s).
The invitees and group members are identified by Intended Recipient Fingerprint subpackets containing their EKA fingerprints.
The Literal Data SHOULD contain a keyring containing the TPKs of all group members and invitees.

Each invitee SHOULD respond to all other invitees and group members with an ACK containing a new ephemeral subkey.
An invitee can instead decline membership by sending a NAK to all other invitees and group members.
All invitees and group members MUST remove the declining invitee from their local copy of the membership list.

The invitor shares the current private ratchet state of the group with each accepting invitee so that they can immediately start decrypting group messages.

The accepting invitees are inserted into the existing tree structure based on their EKA fingerprints.
As a group member sees its new peer publish their ephemeral key, it initialises an intermediate keypair using its private ephemeral key and their public ephemeral key.
It then publishes the new intermediate public key by postfixing it to its own ephemeral public key in subsequent messages, and ECDH agreement proceeds as usual.

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

To identify a group, a notation subpacket SHOULD be included in all flow control requests associated with that group.
The notation subpacket is marked as not human readable, with a key of "groupid" and a random value of at least 128 bits.
A group MAY also have a human readable name, with a key of "groupname".

A group ceases to exist when its membership has been reduced to one.

## Device groups

If a single identity is associated with multiple devices, the devices form a device group with each other using the group mechanism above.
If a device is a member of a device group, then the ECDH private keys used by that identity are generated from the device group double ratchet via a KDF.
If one device sees that another device in its device group has generated a public key with an unknown fingerprint, it progresses the symmetric ratchet until it finds the corresponding secret.

## References

* [Multi-party ECDH exchange](https://crypto.stackexchange.com/questions/72207/ecdh-for-more-than-two-parties)
* [MLS](https://www.rfc-editor.org/rfc/rfc9420.html)
