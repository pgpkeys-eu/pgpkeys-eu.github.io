
The State of the Keyservers in 2024
===================================

> Cryptography is rarely, if ever, the solution to a security problem. Cryptography is a translation mechanism, usually converting a communications security problem into a key management problem.
> - Dieter Gollmann

In the two and a half years since the [sks-keyservers.net shutdown](https://lists.nongnu.org/archive/html/sks-devel/2021-06/msg00007.html) in June 2021, the concept of OpenPGP keyservers has been called into question.
However, keyservers still provide a vital service to the OpenPGP ecosystem.
It is not necessary (nor desirable!) to have a single contact point for all keyservers, so long as interoperability between keyservers is maintained.
This interoperability is currently incomplete, but significant progress has been made.

The Pretty Good Public Key Infrastructure (PGPKI)
=================================================

A Public Key Infrastructure (PKI) is a set of services and protocols that allow for the automated distribution and management of public keys.
Functions typically include:

* discovery of new keys
* updates to existing keys
* delegation of trust (validation)
* key expiry and revocation (invalidation)

OpenPGP is one of only two widely-used cryptography standards to include a full Public Key Infrastructure.
The other is X509, used by HTTPS and S/MIME.
In addition, SSH implements a partial PKI that supports trust delegation only.
Other cryptography schemes require either manual management of key material (age, minisign), or a centralised trust database (Signal, WhatsApp).

X509 supports a full PKI in principle, however in practice it is limited to a vendor-managed list of ultimately-trusted certificate authorities (CAs).
The receiver (consumer) of a message (resource) has only two choices: to fully trust the CA chosen by the sender, or discard the message.
SSH is mainly used in a private or organisational capacity, where the root of trust is internal or self-assigned.

By contrast, OpenPGP allows receivers/consumers to choose their own roots of trust.
OpenPGP CAs exist, but they are not required.
This places the ultimate responsibility for security on the end user, who needs to explicitly decide on their trust roots, or build their own trust network from scratch.
This extra overhead is one of the main reasons why OpenPGP has struggled to gain widespread adoption.

Public OpenPGP keystores are the basis of the PGPKI.
Keyservers are the most common form of public keystore.
Keystores provide two main services:

* discovery of new keys
* updates of existing keys

The other main PKI functions (validation and invalidation) are usually performed on the client side based on signatures (certifications) on the public keys.

How to use the PGPKI
--------------------

Your OpenPGP software will come with a preconfigured keyserver for both discovery and updates.
The most common default keyserver is keys.openpgp.org, however this can usually be changed in your software settings.
Some software providers, such as mailvelope.com and proton.me, provide their own default keyservers.
Most software can also use the WKD protocol as an additional key discovery method.

Keyserver Architecture
======================

Most keyservers expose an [HKP](https://datatracker.ietf.org/doc/html/draft-gallagher-openpgp-hkp/) API that can be used to look up keys, by either user ID (UID) or fingerprint, and to manage keys.

Keyservers can be divided into several categories:

* General-purpose keyservers can store keys from any domain or service.
* Domain-restricted keyservers can only store keys tied to a particular internet domain or domains.
* Application-restricted keyservers can only store keys belonging to users of a given service.

In addition, email-verifying keyservers will only serve keys by user ID lookup if it contains a verified email address of the key owner.
They can however serve arbitrary keys by fingerprint lookup.

### Synchronising Keyservers (SKS)

Each SKS service is a general-purpose keyserver, and is operated by an independent team.
These operators include OpenPGP software implementors, media organisations, and private individuals.
Most SKS servers (including our keyserver [pgpkeys.eu](https://pgpkeys.eu)) use the [hockeypuck](https://hockeypuck.io) software stack.

The current SKS keyserver network can be viewed at [spider.pgpkeys.eu](https://spider.pgpkeys.eu).
The network members co-operate in maintaining a distributed dataset, however each operator is functionally and legally independent, e.g. for GDPR purposes.
Note that some well-known SKS keyservers (notably [keyserver.ubuntu.com](https://keyserver.ubuntu.com) and [sks.mit.edu](https://sks.mit.edu)) do not currently synchronise with others.

(Note that "SKS" can sometimes mean either the sks-keyserver software (which is effectively obsolete) or the SKS synchronisation protocol (and network) that it introduced.
We will always use "SKS" to mean the protocol, and "sks-keyserver" to mean the software.)

### Other General-Purpose Keyservers

[keys.openpgp.org](https://keys.openpgp.org) (KOO) is an email-verifying general-purpose keyserver.
Its operation is overseen by a board representing a selection of OpenPGP software implementors and other interested parties.
KOO uses the [hagrid](https://gitlab.com/keys.openpgp.org/hagrid) software stack.
In addition to HKP Hagrid exposes its own, functionally similar, VKS API.

[earth.li](https://the.earth.li/pgp_lookup.html) is a general-purpose keyserver.
It is individually operated, and runs the [Onak](https://github.com/u1f35c/onak) software stack.
It does not support full synchronisation, but can be used to forward keys to other known keyservers via email.

### Domain/Application-restricted Keyservers

[Mailvelope](https://mailvelope.com) is a software company that runs an email-verifying [keyserver](https://keys.mailvelope.com) for their users, using a [custom software stack](https://github.com/mailvelope/keyserver).

[Proton](https://proton.me) is an email service provider that runs an application-restricted keyserver for keys belonging to their users, using a custom software stack.

### Other Keystores

[WKD/WKS](https://datatracker.ietf.org/doc/draft-koch-openpgp-webkey-service/) is a set of domain-restricted keystore protocols that can be deployed by any domain owner to manage keys for their own users.
The WKD lookup protocol may optionally be combined with the WKS key submission/management protocol.
WKD can only be used to look up keys by email user ID, not by fingerprint.
When looking up keys by email user ID, it can also distribute revocations for keys with the same user ID, however this is not supported by all clients.

Organisations may also serve OpenPGP keys from their internal LDAP or Active Directory infrastructure.

Keyserver Compromises
---------------------

Each of the keyserver architectures above makes different compromises between completeness, robustness, and decentralisation.

Property            | Hockeypuck/SKS| Hagrid/KOO| Mailvelope| Proton    | Onak      | WKD/WKS       | LDAP/AD
--------------------|---------------|-----------|-----------|-----------|-----------|---------------|--------------
Decentralisation    | Yes(sync)     | No        | No        | No   | Yes(forwarding)| Yes(delegation)| No
Generality          | Yes           | Yes       | No        | No        | Yes       | No            | No
UID verification    | No            | Yes       | Yes       | Yes       | No        | Yes           | Yes
Non-email UIDs      | Yes           | No        | No        | No        | Yes       | No            | Yes
UID search          | Yes           | Yes       | Yes       | Yes       | Yes       | Yes           | Yes
Fingerprint search  | Yes           | Yes       | Yes       | Yes       | Yes       | No            | Yes
Certifications      | Yes           | Limited   | Yes       | Yes       | Yes       | Yes           | Yes
Self-sovereignty    | In progress   | Limited   | Yes(?)    | Yes       | No        | Yes           | Yes
Key deletion (RTBF) | Yes           | Yes       | Yes       | Yes       | Yes(?)    | Yes           | Yes
HKP API             | Yes           | Yes       | Limited   | Limited   | Yes       | No            | No

* Decentralisation:
    Is it operated by a single organisation, or multiple?
* Generality: 
    Will it store keys belonging to anyone, or only certain people?
* UID verification: 
    Does it check the ownership of user IDs?
    This is necessary to limit the number of results returned for a user ID search.
* Non-email UIDs: 
    Does it restrict the format of user IDs to those containing emails only?
* UID search: 
    Can you search by user ID?
    This is necessary (but not sufficient) for key discovery.
* Fingerprint search: 
    Can you search by fingerprint?
    This is required for timely distribution of key updates, including revocations.
    (Note: WKD distributes revocations by other, nonstandard means)
* Certifications: 
    Does it serve third-party signatures?
    This is required for some features of the OpenPGP PKI, but can be a vehicle for spam.
    (Note: Hagrid/KOO distributes third-party signatures with an attestation signature, however this is not supported by all clients)
* Self-sovereignty:
    Can a key owner control the third-party signatures on their key?
    This is required to prevent the size of keys growing without limit.
* Key deletion: 
    Can you delete your own key (or request deletion)?
    This is a requirement for GDPR compliance.
* HKP API:
    Does the keyserver support the standard HKP API?
    This is necessary for integration of third-party clients.
    (Note: Mailvelope and Proton do not support key submission over HKP)

### GDPR

The EU General Data Protection Regulation (GDPR) has two broad requirements for a service that processes personal data (which includes OpenPGP keys).
Data protection laws in other jurisdictions include some or all of the same principles.

* Data controllers (keyservers) must have a valid basis for processing personal data, and must manage that data responsibly.
* Data owners (users) have the right to control their own personal data.

There are six possible basis for processing, of which two are relevant to keyservers:

* (a) the data subject has given consent to the processing of his or her personal data for one or more specific purposes;
* (e) processing is necessary for the performance of a task carried out in the public interest or in the exercise of official authority vested in the controller;

UID-verifying keyservers rely upon consent (a).
Other keyservers rely upon the public interest (e).
It is important to state that neither of these have yet been legally tested.

In either case, the data subject has the right of erasure under GDPR, also known as the "right to be forgotten" (RTBF).
All modern keyservers uphold this right.
Some also provide automated mechanisms, however these vary from service to service.

Open Concerns
-------------

* Some keyservers, particularly KOO, support email verification but not third-party signatures.
    Many clients now treat keys obtained from these servers as trusted by default (implicit validation).
    This effectively treats those servers as authorities, and increases the risk of abuse by operators (or hackers).
* WKD/WKS require the co-operation of the domain owner to set up.
    Users whose email provider is not willing to do so (e.g. gmail, hotmail etc.) cannot therefore distribute their keys via WKD.
* Many keyservers restrict the format of user IDs to email addresses.
    This hampers the distribution of OpenPGP keys for non-email use cases.
    For example, the Monkeysphere project uses URL user IDs in order to replace the SSH and HTTPS PKIs with the OpenPGP PKI.
* SKS keyservers still do not have effective protections against the creation of large numbers of keys with the same user ID.
    Some key owners therefore have keys that are effectively undiscoverable via these keyservers, although they can still be updated.
* Most client software only supports the use of a single keyserver at a time.
    This is a holdover from the days of the sks-keyservers.net pool, which distributed queries across multiple keyservers using a single keyserver address.

### Zooko's Triangle

[Meaningful, decentralised, secure; pick two](https://en.wikipedia.org/wiki/Zooko%27s_triangle).
OpenPGP users are still forced to choose between one of the following failure modes:

* Manually verifying fingerprints (lack of meaningfulness)
* Trusting an authority (lack of decentralisation)
* Impersonation (lack of security)

Most cryptography systems, and some OpenPGP implementations, have chosen centralised authority as the least-worst of these options.
Some, including OpenPGP and Signal, allow users to manually verify fingerprints as a fallback method.

Work in Progress
================

There is work underway in the following areas:

* Key self-sovereignty in Hockeypuck is currently in alpha testing.
* The HKP and WKD/WKS protocols have been shortlisted for adoption by the OpenPGP IETF working group this year.
* The OpenPGP crypto-refresh RFC is due to be officially published in the next few weeks.

Andrew Gallagher (January 2, 2024)
