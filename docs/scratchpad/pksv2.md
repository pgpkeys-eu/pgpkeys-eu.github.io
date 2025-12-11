# Second-Generation PKS Protocol

We propose a PKSv2 protocol for gracefully-degraded sync between keyservers that cannot or do not wish to sync using a more tightly-coupled method such as SKS.

## Transports

PKSv2 can operate over multiple transports:

* Email (traditional PKS)
* HKP (RECOMMENDED)
* VKS (required for KOO)

KOO's HKP interface automatically generates verification emails for newly-submitted keys.
We do not wish PKS to trigger these emails, therefore we define a VKS transport for syncing to KOO.

## Headers

* PKS-Sender: the email contact address of the sending keyserver
* PKS-From: the HKP base address of the sending keyserver
* PKS-Recipients: a comma-separated list of receiving keyservers (HKP base addresses or mailto: urls)

When comparing HKP base addresses, port numbers SHOULD be ignored.
