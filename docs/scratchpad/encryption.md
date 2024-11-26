# Multiple Encryption Keys in OpenPGP

OpenPGP enables the use of multiple secret keys by a single key holder.
These may be secret keys that correspond to component keys of the same certificate, or of multiple certificates.
While it is relatively straightforward to manage multiple signing keys, encryption keys pose particular problems.

# Introduction

There are scenarios where a key holder will have to manage multiple secret keys for encryption.
Issues arising in such a scenario include:

* A correspondent may not know which key material to encrypt to.
* The key holder may not have access to all the secret key material at any given time.
* The key holder may be using multiple clients that do not support the same versions or algorithms.
* The key holder may own multiple devices.
* The key holder may wish to enable opportunistic encryption algorithm upgrades (say, to PQC) before all correspondents support it.

# Proposed solutions

## AD(S)K mechanisms

These consist of preference flags on particular certificates that indicate that a key holder's correspondents should encrypt to more than one (perhaps all) available encryption subkeys.

Pros:

* Simple wire format.

Cons:

* The correspondent may have to create a large number of (computationally expensive) PKESKs.
* Abuse potential for eavesdropping.
* No opportunistic upgrades.

## Preferred-algorithm mechanisms

OpenPGP has several preferred-algorithm mechanisms used to signal to a correspondent which digest, symmetric-key and AEAD algorithms the key holder prefers to receive.
It may be possible to extend this mechanism to the selection of encryption subkey algorithms, if multiple subkeys are available.

Pros:

* Allows for opportunistic algorithm upgrades.

Cons:

* Cannot discriminate between subkeys of the same algorithm.
* Does not help with multi-device or multi-implementation use cases.

# Key material sharing

A key holder may wish to share their secret key material between multiple implementations or devices.

(TBC)
