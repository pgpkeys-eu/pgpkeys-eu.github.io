
Keyserver Updates and Roadmap, August 2024
==========================================

Hockeypuck 2.2.x
----------------

Hockeypuck v2.2 was released in May, with many improvements to sync stability and ease of deployment.

This was achieved in part by dropping support for sync with the obsolete SKS-keyserver software, which is no longer actively maintained.
SKS-keyserver distributed malformed keys without complaint, so previous versions of Hockeypuck attempted to follow suit in order to remain compatible.
Previous versions of Hockeypuck also had slight inconsistencies in how they dealt with malformed material, leading to duplication of data which eventually overwhelmed the sync process.
This has been addressed in Hockeypuck 2.2.x, which now achieves consistent sync.

A [majority of SKS keyservers](https://spider.pgpkeys.eu/sks-peers) have now been upgraded to the 2.2.x branch.
A small number remain on 2.1 for compatibility reasons.

RFC9580
-------

[RFC9580](https://datatracker.ietf.org/doc/html/rfc9580) has finally been published, after a multi-year design process that has involved an [unfortunate schism in OpenPGP](critique-critique.html).
Amongst other things, it defines new version 6 public key formats -- Hockeypuck will require some updating at a low level (particularly the storage schema) to support these v6 keys properly.
Most of this work should also be applicable to v5 (LibrePGP) keys, since the specifications of v5 and v6 keys diverged at a relatively late stage and contain many of the same changes.
While we will prioritise support of v6 keys, we intend to also support v5 keys if possible.

HKP Draft Protocol
------------------

The [HKP draft protocol](https://datatracker.ietf.org/doc/html/draft-gallagher-openpgp-hkp) has been [shortlisted for potential adoption](https://datatracker.ietf.org/wg/openpgp/about/) by the OpenPGP IETF working group this year.
The original HKP draft by Daphne Shaw has been used as a reference for nearly twenty years, but was never officially made a Proposed Standard.
We hope to rectify this in 2024 by updating the draft to take account of developments in the meantime, and to enable the distribution of v6 (and v5) keys in a backwards-compatible manner.

Proposed improvements include:

* An updated "v1" machine-readable API.
* Separation of fingerprint, key ID, and free-text searches.
* Common HKP queries can be served from a static filesystem.
* A key discovery mechanism similar to WKD, but using HKP to serve the key material.
* Specification of the SKS extensions to HKP.

It is hoped to start work implementing these changes in Hockeypuck in the near future, and to use Hockeypuck as one of the reference implementations.

Self-sovereignty
----------------

Key self-sovereignty (third-party signature spam prevention) in Hockeypuck is [currently in its initial stages](https://github.com/hockeypuck/hockeypuck/issues/270).
There are two approaches to this:

1. Attested signatures ([1pa3pc](https://datatracker.ietf.org/doc/html/draft-dkg-openpgp-1pa3pc)).
    These require the key owner to make a new kind of self-signature over their own key stating which third-party signatures may be attached to it.
    Support for 1pa3pc signatures will also need to be implemented on the client side, and the specification has not yet been finalised.
2. Timestamp-aware merge strategy ([HIP-3](https://github.com/hockeypuck/hockeypuck/wiki/HIP-3:-Timestamp-aware-merge-strategy)).
    This requires no changes to client software or user behaviour, but runs a (small) risk of a race condition and a return to key churn.

The current plan is to implement HIP-3 as an initial measure, and then 1pa3pc as a definitive solution later.
Having both methods available will ensure the broadest availability of third-party signature spam prevention measures both in the short and long term. 

Other news
----------

The [OpenPGP Replacement Key Signalling Mechanism](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-replacementkey) has recently been adpoted by the OpenPGP IETF Working Group.

Andrew Gallagher (1st August, 2024)
