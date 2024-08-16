# A Summary of Known Security Issues in LibrePGP

The [LibrePGP specification](https://datatracker.ietf.org/doc/html/draft-koch-librepgp) exhibits a number of known theoretical security issues.
It should be stressed however that no practical exploits are yet known.

This post will be kept up to date with any further information.

## v5 Signature Downgrade Attack

In 2022, Demi Marie Obenour [discovered a theoretical downgrade attack](https://gitlab.com/openpgp-wg/rfc4880bis/-/issues/130) against v5 signatures.

This arises because the metadata of an OpenPGP signature is hashed in as a "trailer" at the end of the digest input, rather than at the beginning.
In practical terms, this means that the trailer must be designed as if it were being parsed backwards from the end.
Any ambiguity in this backwards parsing opens the possibility of signature malleability, where an attacker takes a genuine signature over one piece of data and converts it into an apparently genuine signature over a different piece of data.

The Obenour attack takes advantage of the fact that [v5 trailers end with an 8-octet length field](https://datatracker.ietf.org/doc/html/draft-koch-librepgp#name-computing-signatures), whereas all previous signature trailers end with a 4-octet field.
It is theoretically possible for an attacker to trick a victim into creating a signature with a specific value in the length field that simulates the trailer of a different signature version. 
v3 signatures are particularly easy to fake by this method, because their entire trailer is only five octets long, and so an attacker would have complete control over the fake trailer contents.

To mitigate this, RFC9580 specifies:
* [the size of the v6 trailer length field is four octets](https://datatracker.ietf.org/doc/html/rfc9580#name-computing-signatures), matching the v3 and v4 trailer formats and thus removing the parsing ambiguity.
* [signature versions must match the version of the key generating them](https://datatracker.ietf.org/doc/html/rfc9580#name-signature-packet-type-id-2), ensuring that any future cross-version malleability attack will be invalidated by the version mismatch.

## Type 20 "OCB Packet" Malleability Attack

In 2024, Falko Strenzke and Johannes Roth [published a paper outlining theoretical attacks against AEAD constructions in both LibrePGP and CMS](https://eprint.iacr.org/2024/1110.pdf).

This arises because LibrePGP's specification of the OCB encryption mode is sufficiently similar to a deprecated CFB encryption mode that partial decryption of the ciphertext is possible even when the wrong mode is used, so long as the symmetric session key is correct.
Since CFB does not contain any intrinsic error-checking, the victim's software may treat an attacker-modified message as genuine.
The attacker may then use various well-known methods to extract the partially-decrypted data.

The Strenzke/Roth attack takes advantage of the fact that the Type 20 "OCB packet" in LibrePGP can be easily transformed into a Type 9 "SED" packet, which has long since been deprecated but is still supported by some applications for the processing of historical data, such as backups.
Since both OCB and SED packets rely on the same construction for encrypting the symmetric session key, it is possible for a receiving application to mistakenly attempt to decrypt an attacker-modified OCB packet as if it were an SED packet using the CFB encryption mode.
This attempt may partially succeed due to the OCB decryption process only differing significantly from CFB once the first block has been processed.
SED packets have several well-known security issues, therefore OCB-encrypted LibrePGP messages are also potentially vulnerable to the same issues.

To mitigate this, RFC9580 [specifies the use of a Key Derivation Function (KDF)](https://datatracker.ietf.org/doc/html/rfc9580#name-version-2-symmetrically-enc) in the alternative SEIPDv2 packet, to ensure that the session key used to encrypt the data in AEAD modes must be calculated at decryption time using the ID of the AEAD encryption mode.
If the victim attempts to decrypt using the wrong encryption mode, they will also therefore calculate an incorrect session key, which will immediately fail before decrypting any of the message.

It is particularly notable that the ["Critique on the OpenPGP updates" section of the LibrePGP website](https://librepgp.org/#critique) calls out the advanced construction of the SEIPDv2 packet as unnecessary, and states that the LibrePGP protocol designers intentionally rejected it (subsection 1, "Symmetric Mode"), despite the use of KDFs to prevent such cross-protocol attacks being a recommended technique since at least 2010 (see e.g. [RFC5869 section 3.2](https://www.rfc-editor.org/rfc/rfc5869#section-3.2)).

Andrew Gallagher, 16th August 2024
