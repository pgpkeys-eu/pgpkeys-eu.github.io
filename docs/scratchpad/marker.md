# Use of the Marker Packet in OpenPGP

The Marker Packet is defined as follows:

> The body of the Marker packet consists of:
> * The three octets 0x50, 0x47, 0x50 (which spell "PGP" in UTF-8).
> Such a packet MUST be ignored when received.

No usage guidance is present in this section of RFC9580.
We therefore wish to update the usage guidance of the Marker Packet to read:

> If a receiving implementation encounters a Marker Packet with any other contents, the entire packet sequence SHOULD be ignored.

Otherwise the Marker Packet effectively reverts to its original use as a comment packet, which is not the intent of the specification.
