# Use of the Marker Packet in OpenPGP

The Marker Packet is defined as follows:

> The body of the Marker packet consists of:
> * The three octets 0x50, 0x47, 0x50 (which spell "PGP" in UTF-8).
> Such a packet MUST be ignored when received.

No usage guidance is present in this section of RFC9580, however there is more detail in RFC2440 and RFC4880:

> It may be placed at the beginning of a message that uses features not available in PGP 2.6.x in order to cause that version to report that newer software is necessary to process the message.

We therefore wish to update the usage guidance of the Marker Packet to read:

> If a receiving implementation encounters a Marker Packet that contains text other than "PGP", the entire packet sequence SHOULD be ignored.
> This provides a mechanism for future RFCs to make breaking changes to the OpenPGP packet grammar.
