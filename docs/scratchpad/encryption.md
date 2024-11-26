# Multiple Encryption Keys in OpenPGP

OpenPGP enables the use of multiple secret keys by a single key holder.
These may be secret keys that correspond to component keys of the same certificate, or of multiple certificates.
While it is relatively straightforward to manage multiple signing keys, encryption keys pose particular problems.

# Problem Statement

There are scenarios where a key holder will have to manage multiple secret keys for encryption.
Issues arising in such a scenario include:

1. A correspondent may not know which key material to encrypt to.
2. The key holder may not have access to all secret key material at any given time (e.g. multi-device)
3. The key holder may need to use a stale implementation that does not support newer specs.
4. The key holder may wish to enable opportunistic algorithm upgrades.

# Proposed solutions

## AD(S)K mechanisms

These consist of preference flags that indicate that a key holder's correspondents should encrypt to more than one (perhaps all) available encryption subkeys.

Problems addressed: 1, 2, 3.

Pros:

* Simple wire format, addresses several issues.

Cons:

* The correspondent may have to create a large number of (computationally expensive) PKESKs.
* Abuse potential for eavesdropping.
* Does not address opportunistic upgrades.

## Preferred-algorithm mechanisms

OpenPGP has several preferred-algorithm mechanisms used to signal to a correspondent which digest, symmetric-key and AEAD algorithms the key holder prefers to receive.
It may be possible to extend this mechanism to the selection of encryption subkey algorithms, if multiple subkeys are available.

Problems addressed: 1, 4.

Pros:

* Allows for opportunistic algorithm upgrades.

Cons:

* Cannot discriminate between subkeys of the same algorithm.
* Does not help with multi-device or stale-implementation use cases.

## Defined Heuristics

A sending implementation may use heuristics such as subkey creation time, binding signature time, or algorithm strength to select an encryption subkey.

Problems addressed: 4.

Pros:

* Requires no new wire formats.
* Already used by some implementations.

Cons:

* Current implementations are not consistent; alignment may cause breakage.
* Does not not address multi-device or stale-implementation use cases.

# Key material sharing

A key holder may wish to share their secret key material between multiple implementations and/or devices.

## Current mechanisms

* Hardware tokens (smartcards etc).
* Secure enclaves.
* Encrypted vaults (password managers etc).
* Encrypted messages to self.

Note that none of these address the stale-implementation use case.

## Proposals

* 
