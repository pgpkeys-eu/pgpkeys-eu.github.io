
A Critique on "A Critique on the OpenPGP Updates"
=================================================

A proposed fork of the OpenPGP standard, called "LibrePGP" and initiated by GnuPG's maintainer Werner Koch, has made a series of statements on its own website[^1] in order to justify its existence. Some of these criticisms have merit, but most are misleading at best. We address the points raised individually here.

The Critique
============

> The IETF OpenPGP Working Group (WG) at some point decided to give up on its charter to produce an updated specification and instead started to re-invent that standard.
> Whether this is in line with IETF rules is questionable.

The crypto-refresh draft[^2] and LibrePGP[^3] are both backwards-compatible with RFC4880, RFC5581, and RFC6687.[^4] Neither of them attempt to re-invent any existing standard. Crypto-refresh leaves out many extensions that are included in LibrePGP, in part because the Working Group was *not* willing to overstep the limitations of the current charter.[^5] Items included in LibrePGP that were intentionally left out of crypto-refresh include:[^6]

- Restricted Encryption key flag
- Timestamping usage key flag
- Attested Key Signatures
- Attested Certifications subpacket
- Key Block subpacket

These proposed extensions to OpenPGP are compatible with the crypto-refresh draft, and can be specified at a future date once the WG is re-chartered.[^7]

> Considering that new features and discussions for larger updates of the specification delayed a new RFC for many years and were the reason for closing the WG in 2017 and re-opening in 2020,
> we proposed to go back to the last commonly agreed upon draft or to conclude the WG on the grounds that no rough consensus could be found.

Rough consensus *was* found; LibrePGP dissents from it.[^8] [^9] And one thing that the majority of the WG can agree upon was that the "last commonly agreed upon draft" was not agreed upon at all.

01 Symmetric Mode
-----------------

> It seems that this new scheme was introduced for the benefit of allowing GCM as yet another encryption mode.
> GCM is a counter mode and, as can be seen by the large changes required, is hard to get right.
> Meanwhile we have GCM in CMS (the core of S/MIME) because Microsoft decided to go this way.
> However, OpenPGP has made its decisions based on technical soundness and not based on larger vendor, government, or committee decision.

This is the strongest concern that LibrePGP has raised. GCM is hard to get right, but it is widely implemented and its limitations are well-understood.[^10] Anyone who needs to support GCM in OpenPGP has a large number of well-tested libraries to choose from. Anyone who does not need GCM is not required to implement anything. GCM is OPTIONAL.[^11]

> The WG once decided to go with OCB and EAX. EAX was only added to avoid possible patent problems.
> However, in the 4.5 years since the introduction of EAX, the OCB patent expired.
> Thus, there is no more reason to reject OCB, and it should be declared as a RECOMMENDED mode with the intention to make it a MUST mode in some future OpenPGP.

This paragraph is beside the point, since OCB is already a MUST mode in the OpenPGP crypto-refresh.

> It can also be expected that FIPS-140 will eventually allow OCB.

Perhaps, but this is speculation, and no date is suggested.

> Our suggestion was: Drop all the new AEAD ideas and use what has been deployed and agreed upon in the OpenPGP WG a long time ago.
> Further, turn OCB into MUST WG a long time ago. Further, turn OCB into MUST and EAX into MAY (only for backward compatibility with deployed implementations).

Again, this is *already the case* in crypto-refresh:[^12]

> 9.6. AEAD Algorithms 
> ...
> Implementations MUST implement OCB. Implementations MAY implement EAX, GCM and other algorithms.

LibrePGP's concerns are genuine, but have already been addressed.[^8]

02 Padding Packet
-----------------

> A padding packet is introduced with the idea to mitigate traffic analysis.
> However, it is suggested to use random data for the content of this packet and thus this packet opens a huge covert channel.
> This is especially concerning for institutional users efforts regarding Data Leak Prevention (DLP).
> Suggestions to use padding based on a verifiable seed, were rejected despite that this is the standard method to do padding.

OpenPGP already contains numerous potential covert channels.[^13] Unencrypted padding packets are in many ways *safer* than those other covert channels, because they are obvious and can (if necessary) be detected and/or removed at e.g. a gateway appliance. The choice of whether to allow padding packets in a given context can still be made at the application layer.

The proposal to mandate verifiable-seed padding was considered in the WG, but consensus could not be reached.[^14] Many details of how it would work in practice remain unclear.[^15] Updated guidance for how best to generate padding can still be issued at a later date if consensus is forthcoming.

> This padding idea has come up in discussion every once in a while over the last 25 years and has always been rejected
> because it does not belong into the encryption layer but into the application (plaintext) layer.

This is not always true. For example, when downloading keys from a keystore, OpenPGP *is* the application layer. It is not clear for example how else to implement padding in ^16, which only serves binary TPKs.[^16]

03 Changes to the ECDH Encryption
---------------------------------

> ECDH is the standard way to do encryption with elliptic curves.
> For OpenPGP ECDH has been specified in RFC-6637 from 2012 and been implemented by PGP and GnuPG even a year earlier.
> Instead of keeping this solid specification some details have been changed without a sound reason.

The only change to ECDH is that instead of using OpenPGP-specific OIDs to identify ECDH curves (which were never intended to be permanent OID allocations), the new crypto-refresh will use industry-standard OIDs (in v6 packets *only*).[^17]

04 Proliferation for Algorithms
-------------------------------

> The new draft not only allow the use of GCM as a third encryption mode but adds a couple of other required algorithms:
>
>    ^18
>    Argon2

^18 and Argon2 were added to address several known and potential weaknesses in the cryptographic layer of OpenPGP.[^18] [^19] [^20]
LibrePGP does not acknowledge the existence of these issues.

>    Optional modes (EAX, OCB, GCM, and a way to define even more)

OpenPGP has always been highly extensible, so it's unclear what the specific objection is here (aside from GCM, which we address elsewhere).

> Werner Koch joined the AES conference in 2000 on Phil Zimmermann's wish to talk about algorithm proliferation.
> They agreed on pushing the forthcoming AES along with their MDC extension, get Twofish and so out of the focus, and in general resist to add new algorithms.
> That is for the simple reason that neither PGP nor GnuPG wanted to maintain all new algorithms until eternity.
>
> Later, they had to do a political compromise to allow Camellia for the use in Japan and Brainpool curves for European use.
> We should really stick to this and not support algorithms which are just a substitute for existing crypto building blocks.
> Since added complexity makes a review harder and the larger codebase has to be maintained indefinitely for backwards compatibility.

This appears to say that it was OK to make a political compromise for the benefit of users who have to comply with some national regulations, but for some reason it is not OK to make a political compromise for the sake of users who have to comply with other national regulations. It might be fair to say that Werner regrets his previous decision and has decided not to repeat that mistake, but that is his personal viewpoint. Others obviously disagree.

Supporting extra algorithms comes at a cost. Whether this cost is "justifiable" is to some extent subjective. Some people think the cost of including GCM is too high, which is fair. If so, they are not required to implement GCM, since GCM is OPTIONAL.[^11]

05 Removal of Useful Real-world Features
----------------------------------------

OpenPGP is a long-established standard and has accumulated many features over time. It is good practice to deprecate features that have been found to not work as intended, even if they are useful in theory.

> For example, in 2016 an m flag was introduced to indicate that the plaintext shall be interpreted as MIME data.
> This has been removed along with deprecating the traditional t flag to distinguish between binary and text data.
> Having the ability to easily detect MIME data is for example required to process attachments from web mail clients or in air-gaped environments.

The `t` flag was deprecated because it did not specify which character set was being used. UTF-8 has long been the standard encoding in OpenPGP, and all "text" usage is therefore already covered by `u`.[^21]

> The designated revoker feature has also been deprecated with the rationale that a better method is to achieve this with an "escrowed" revocation, pre-created by the user.
> In fact, GnuPG creates such a revocation certificate since version 2.1 (released in 2014), to mitigate the common problem of a forgotten password.
> But this is not a replacement for corporate needs: the designated revoker is an important feature to manage a large scale deployments of OpenPGP keys and acts as a CRL (certificate revocation list) replacement.

No evidence is given that escrowed revocations are less suited to a corporate environment. Escrowed revocations are simple to reason over, and have been supported for decades. By contrast, designated revokers increase the complexity of validity calculations considerably - you cannot be sure if a key has been revoked by looking at the signatures on the key itself, but instead need to have a copy of the revoker's key, which may not be immediately available. And what happens if the designated revoker's key is itself revoked?

06 Removal of Security Fixes
----------------------------

> Due to an implementation bug in PGP 5, the metadata of a signed file was not covered by the signatures.
> RFC-4880 didn't fix that for backward compatibility.
> However, users were often surprised when they learned that the shown filename and file data could be changed while keeping the signature intact.
> With the introduction of the new v5 signature packet format, the opportunity to fix that was taken.
>
> However, the crypto-refresh group then introduced v6 signatures and removed the fix (see this commit)
> with the flimsy explanation that the way to populate that the field is not clear in a theoretical encrypt-then-sign scenario
> and that signatures could not be detached and reattached (which is obvioulsy wrong).
> A later proposed fix for v4 signature packets (Meta Hash subpacket, see discussion) was not considered.

An even later proposed fix, to store the metadata in a Signature Notation subpacket, *did* find initial support.[^22] Since this merely extends an existing feature of OpenPGP, does not require changes to the wire format, and is therefore currently unchartered, the details were deferred to a future document.

07 Salted Signature Issue
-------------------------

> Salted signature were introduced with the idea that they might mitigate a chosen prefix attack in the same way as they will do for a certain SHA-1 based Web-of-Trust attack.
> No research for that statement has been cited just an assumption and a concern related to fault attacks on EdDSA which is about the development of Wireguard-like protocol.
> However, such fault attacks can be more securely detected by checking the signature after verification in the same way as the mitigation to Lenstra's attack on RSA's CRT.

The existence of practical prefix attacks is an assumption. The question is: is it a *reasonable* assumption? The WG felt it best to err on the side of caution.[^23]

> Anyway, the major concern here is that this adds another 32 octet covert channel to each message (and also blow the signature up by 64 octets)
> In this case it is not an optional feature as with the padding packet.
> This is a clear violation of best current practices in sensitive areas where signed mails are mandatory and encryption is not enforced (or monitored by a gateway).

Covert channels have already been discussed above.

08 Regression from Deployed Formats and Standard Behavior
---------------------------------------------------------

The section heading is overblown. The crypto-refresh draft and the LibrePGP draft are equally backward-compatible with all deployed OpenPGP artifacts other than those which have been rendered insecure by the passage of time, or experimental deployments.

> In general the crypto-refresh draft tends to ignore the requirements of long term storage needs and considers online communication and software deployment pattern as the major OpenPGP usage.
> Data and software life-cycle management has not been adequately taken in consideration and thus the draft regresses heavily from 30 years of PGP history.

No example of "ignoring the requirements of long term storage needs" has been given, unless this refers to the OPTIONAL GCM mode, which being OPTIONAL[^11] can be ignored by such applications.

Conclusion
==========

Of the complaints explicitly raised by LibrePGP, one (GCM) is substantial. The Working Group agreed with those concerns, but balanced them with the requirements of some users for regulatory compliance. OpenPGP has precedent here, having already included Brainpool and Camellia ciphers on similar grounds. All such regulatory-driven extensions to OpenPGP have historically been marked OPTIONAL[^11], and GCM is no different. OpenPGP crypto-refresh has conformed to established precedent here.

The strongest implicit complaint is that several implementations have already spent significant time developing support for v5 (LibrePGP) artifacts, and that support for v6 (crypto-refresh) would be burdensome. But it is worth pointing out that other implementations with working v5 code are also implementing v6 despite the extra workload incurred.[^24]

The other explicitly-stated complaints can be divided into two classes, neither of which holds up to scrutiny. Most of them are minor technical issues that have been blown out of proportion, e.g. the superficial change in the curve specifier format. Some have even been described, on the record, as acceptable compromises by Werner in previous discussions.[^25] [^26] It would appear therefore that these concerns have been included purely to bulk out the list of grievances.

The remaining explicit complaints (touched upon in the preamble) concern IETF procedure, in which case there is a route of appeal via the IETF itself.[^27] It is not apparent that such an appeal has even been submitted, let alone ruled upon.

There is also one implicit complaint that is only hinted upon in the text, and that is the increasing breakdown in personal trust between Werner and the wider OpenPGP community.[^28]

It is obvious that LibrePGP is an attempt to create facts on the ground that prejudice the conclusion of the IETF standardisation process, in order to maintain the pre-eminent position of one OpenPGP implementation (GnuPG) at the expense of others. It is also impossible to avoid concluding that the target of this power play is the rival implementation that is currently led by two ex-employees of Werner, with whom he has a long-standing and well-documented personal conflict.[^29] [^30]

Normally, personal conflicts are best left to be resolved at a personal level, but not when they fester to the point where they threaten to destroy an entire community. The root cause of this is not technical but personal, and the solution is therefore not technical but personal.

A way forward
-------------

At this point in time, the OpenPGP community desperately needs to see some acts of good faith.
We strongly suggest that these should include the following:

* The librepgp.org website shall be shut down. The draft-koch-librepgp I-D[^3] shall remain active as a reference.
* The OpenPGP Working Group shall, in accordance with its draft charter, begin the process of adopting the WKD protocol as described in draft-koch-openpgp-webkey-service[^16] as a WG standards track document, engaging in good faith with its author at all times.
* GnuPG shall merge [https://dev.gnupg.org/T4393], and desist from vetoing constructive contributions for personal reasons.
* The OpenPGP WG shall, as a matter of good faith, endeavour for the duration of its new charter to maintain in its work queue at all times at least one of the unchartered items already implemented in v5,[^6] with reference to existing v5 implementations, to the extent that security and practicality concerns permit.

If these things should happen, in whatever order but preferably the order listed, we can then begin the process of discussing the medium-term future of OpenPGP in a professional manner. This SHOULD include a commitment by all implementations to at minimum consume both v5 and v6 artifacts on a best-effort basis, even if they have a preference for which standard to ahdere to when generating artifacts. This should be sufficient to ensure that end users are not adversely affected by ongoing disagreements between implementations on the long term path forward.

References
==========

[^1]: [https://librepgp.org]
[^2]: [https://openpgp-wg.gitlab.io/rfc4880bis]
[^3]: [https://datatracker.ietf.org/doc/html/draft-koch-librepgp]
[^4]: [https://mailarchive.ietf.org/arch/msg/openpgp/O1Rz2VLuGj2UKiHiJ4U2djDD5oI/]
[^5]: [https://datatracker.ietf.org/doc/charter-ietf-openpgp/03/]
[^6]: [https://mailarchive.ietf.org/arch/msg/openpgp/aqBy97lj2P4DVxTds0eKZDVdmms/]
[^7]: [https://datatracker.ietf.org/doc/charter-ietf-openpgp/]
[^8]: [https://datatracker.ietf.org/doc/html/rfc7282#section-3]
[^9]: [https://mailarchive.ietf.org/arch/msg/openpgp/yz6EnZilyk_90j569KDPu4I3muY/]
[^10]: [https://mailarchive.ietf.org/arch/msg/openpgp/ZTYD5VJsG1k2jJBbn5zIAf5o7d4/]
[^11]: [https://www.rfc-editor.org/rfc/rfc2119]
[^12]: [https://openpgp-wg.gitlab.io/rfc4880bis/#name-aead-algorithms]
[^13]: [https://mailarchive.ietf.org/arch/msg/openpgp/QYD_NH2JHRI5tb6oL-dM6GWyiQQ/]
[^14]: [https://gitlab.com/openpgp-wg/rfc4880bis/-/merge_requests/204]
[^15]: [https://mailarchive.ietf.org/arch/msg/openpgp/iS0EENyd98XAnXWzHnplBAReY9E/]
[^16]: [https://datatracker.ietf.org/doc/html/draft-koch-openpgp-webkey-service]
[^17]: [https://mailarchive.ietf.org/arch/msg/openpgp/1mOQRoQ-2yNCiz3zjCniYKnpOUI/]
[^18]: [https://mailarchive.ietf.org/arch/msg/openpgp/9uAm5Y3eSPApxiN-wPOgsqFw5HU/]
[^19]: [https://www.kopenpgp.com/]
[^20]: [https://www.password-hashing.net/]
[^21]: [https://mailarchive.ietf.org/arch/msg/openpgp/u6ZdubpZubjAlerB7WgNzhnXDko/]
[^22]: [https://mailarchive.ietf.org/arch/msg/openpgp/8OHMKF9p_h6lNj185mqzTgL8VJo/]
[^23]: [https://mailarchive.ietf.org/arch/msg/openpgp/l_DMs2LJz4ziymVOB1piBolmxhY/]
[^24]: [https://github.com/ProtonMail/go-crypto/pull/182]
[^25]: [https://mailarchive.ietf.org/arch/msg/openpgp/VoAlcewlAYdqsK4RQH-Wcx23438/]
[^26]: [https://mailarchive.ietf.org/arch/msg/openpgp/EQC4wCPfwDm-CKLbYLGsOC5hQpE/]
[^27]: [https://www.ietf.org/standards/process/appeals/]
[^28]: [https://mailarchive.ietf.org/arch/msg/openpgp/XxZt89Eh7XUenuVRajbgtcWzWdA/]
[^29]: [https://mailarchive.ietf.org/arch/msg/openpgp/SPsdWpf2Kjtkel5z3jfaGAupROI/]
[^30]: [https://mailarchive.ietf.org/arch/msg/openpgp/yZqJMLJ0cv_jGlp0NIclIYJ1HQY/]
