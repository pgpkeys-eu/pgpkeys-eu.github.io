
Keyserver Updates and Roadmap, December 2025
============================================

Hockeypuck 2.3
--------------

[Hockeypuck](https://hockeypuck.io) 2.3 was released on the 1st of December, with several improvements to stability and ease of operation.
There are no breaking changes between the 2.2 and 2.3 branches, and SKS sync is supported between 2.2 and 2.3 peers.

The 2.3 release adds support for online reindexing of the database schema, and offline dump-less reloading of the dataset.
Reindexing is enabled by default, and will ensure that the schema is always updated to the latest version.

Offline dump-less reload is implemented by a standalone utility.
This causes the full dataset to be re-validated as if it had been freshly loaded, but without performing a dump first.
Since a dump does not need to be performed, this will not require as much free disk space, and should complete slightly faster.

2.3 also adds support for PKS push updates over both email and HTTP(S).
PKS failover can be enabled on selected SKS sync partners, to help bridge split-brain scenarios in the keyserver network, such as when performing rolling upgrades.
PKS can also be configured statically to push updates to keyservers that do not support SKS.

Hockeypuck 2.3 development is kindly supported by [NGI Zero Core](https://nlnet.nl/project/Hockeypuck/).

### v2.3 Deployment Status

About half of the [public Hockeypuck keyservers](https://spider.pgpkeys.eu/sks-peers) have been upgraded to the 2.3 branch (as of 2025-12-08), including the pgpkeys.eu servers.
A small number remain on 2.1 for compatibility reasons, but the remaining issues preventing upgrade of these 2.1 servers will be addressed in an upcoming 2.3.x release.

Due to changes in the database schema, it is strongly recommended to upgrade to 2.3 as soon as possible.
Hockeypuck 2.4 will be released in early 2026, and will require the new schema to support RFC9580 and HKPv2 (see below).

The Path to v6 and PQC support
------------------------------

[RFC9580](https://datatracker.ietf.org/doc/html/rfc9580) has been published for over a year now, and the [PQC in OpenPGP](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-pqc) draft is nearing publication.
In addition, the [HKP draft protocol](https://datatracker.ietf.org/doc/html/draft-gallagher-openpgp-hkp) has been [shortlisted for potential adoption](https://datatracker.ietf.org/wg/openpgp/about/) by the OpenPGP IETF working group.

Serving v6 keys will only be supported over HKPv2; this is necessary because some older software does not ignore unknown key versions gracefully.
RFC9580 specifically requires newer client software to ignore unknown versions, as does HKPv2.
It is therefore safe to serve v6 key material (and later) over HKPv2.

Hockeypuck 2.3 makes several changes to the storage schema that will enable future support for modern keys and HKPv2.
In particular, HKPv2 defines type-safe "versioned fingerprint", "key ID" and "identity" search operations to replace HKPv1's unified free-text search.
The new schema therefore adds several new columns and one new table to the schema, in order to support these new search methods.
The online reindexing thread will ensure that these are populated in advance of any future releases that will require them.

Experimental support for HKPv2 will be added in future 2.3.x releases.
Since HKPv2 is still a work in progress, the API may change during this time and should not be relied upon for production use.
Support for RFC9580 and PQC keys will require a breaking change to SKS recon, and will be introduced in the 2.4 release, expected some time in 2026.
It is hoped that by this point the HKPv2 API will also have stabilised, however this is dependent on progress in the OpenPGP Working Group.

2.4 and Beyond
--------------

While HKPv2 and RFC9580 support are the current priorities, further improvements are planned for delivery in 2026 and 2027.
These include:

* Allowing OpenPGP key owners to explicitly restrict the distribution of third-party signatures over their User IDs, to prevent signature flooding.
* Out of band email proofs of User ID validity, to mitigate spam and impersonation.
* A fully-featured management API to better handle deletion and blocklisting of incorrect or spammy keys.
* Native rate limiting and tor exit node abuse detection.
* Detection (and potential removal) of keys with known vulnerabilities or weaknesses.
* Improvements to the dump and restore process to allow a running server to be backed up without a restart.

Some of these improvements may land in later 2.3.x patches, if time permits and no breaking changes are involved.
Otherwise, they should appear in either the 2.4 or 2.5 minor releases.

Other news
----------

The [OpenPGP Key Replacement](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-replacementkey) draft is nearing the end of the drafting process; the wire format has been stable for some time and publication as an RFC is now waiting for two interoperable implementations.

Andrew Gallagher (8th December, 2025)
