# A Summary of Known Security Issues in LibrePGP

The [LibrePGP specification](https://datatracker.ietf.org/doc/html/draft-koch-librepgp) exhibits a number of known theoretical security issues that are not present in [the latest OpenPGP specification](https://datatracker.ietf.org/doc/html/rfc9580).
It should be stressed however that no practical exploits are yet known.

This post will be kept up to date with any further information.

## v5 Signature Downgrade Attack

In 2022, Demi Marie Obenour [discovered a theoretical downgrade attack](https://gitlab.com/openpgp-wg/rfc4880bis/-/issues/130) against v5 signatures.

This arises because the metadata of an OpenPGP signature is hashed in as a "trailer" at the end of the digest input, rather than at the beginning.
In practical terms, this means that the trailer must be designed as if it were being parsed backwards from the end.
Any ambiguity in this backwards parsing opens the possibility of signature malleability, where an attacker takes a genuine signature over one piece of data and converts it into an apparently genuine signature over a different piece of data.

The Obenour attack takes advantage of the fact that [v5 trailers end with an 8-octet length field](https://datatracker.ietf.org/doc/html/draft-koch-librepgp#name-computing-signatures), whereas all previous signature trailers end with a 4-octet field.
This 8-octet length field is overkill, since in practice the first three octets will always be zero, and the fourth octet will only ever contain zero or (rarely) one.
It is theoretically possible for an attacker to trick a victim into creating a signature with a specific value in the length field that simulates the trailer of a different signature version. 
v3 signatures are particularly susceptible, because a) they are only five octets long, and b) the fifth octet from the end can take many valid values including zero.
This means that every realistic v5 trailer length value could be mis-parsed as a valid v3 trailer for a signature over a document with three trailing null bytes.

To mitigate this, RFC9580 specifies:
* [the size of the v6 trailer length field is four octets](https://datatracker.ietf.org/doc/html/rfc9580#name-computing-signatures), matching the v3 and v4 trailer formats and thus removing the parsing ambiguity (truncating to four octets reduces the effective security by less than one bit).
* [signature versions must match the version of the key generating them](https://datatracker.ietf.org/doc/html/rfc9580#name-signature-packet-type-id-2), ensuring that any future cross-version malleability attack will be invalidated by the version mismatch.

## Type 20 "OCB Packet" Malleability Attack

In 2024, Falko Strenzke and Johannes Roth [published a paper outlining theoretical attacks against AEAD constructions in both LibrePGP and CMS](https://eprint.iacr.org/2024/1110.pdf).

This arises because LibrePGP's specification of the OCB encryption mode is insufficiently distinct from a deprecated CFB encryption mode.
Partial decryption of the ciphertext is therefore possible even when the wrong mode is used, so long as the symmetric session key is correct.
Since CFB does not contain any intrinsic error-checking, the victim's software may treat an attacker-modified message as genuine.
The attacker may then use various well-known methods to extract the partially-decrypted data.

The Strenzke-Roth attack takes advantage of the fact that the Type 20 "OCB packet" in LibrePGP can be easily transformed into a Type 9 "SED" packet, which has long since been deprecated but is still supported by some applications for the processing of historical data, such as backups.
Since both OCB and SED packets rely on the same method for encrypting the symmetric session key, it is possible for a receiving application to mistakenly attempt to decrypt an attacker-modified OCB packet as if it were an SED packet using the CFB encryption mode.
This attempt may partially succeed due to the OCB decryption process only differing significantly from CFB once the first block has been processed.
SED packets have several well-known security issues, therefore OCB-encrypted LibrePGP messages are also potentially vulnerable to the same issues.

To mitigate this, RFC9580 specifies [the use of a Key Derivation Function (KDF)](https://datatracker.ietf.org/doc/html/rfc9580#name-version-2-symmetrically-enc) in the alternative SEIPDv2 packet, to ensure that the symmetric key used to encrypt the data in AEAD modes must be calculated at decryption time using the ID of the AEAD encryption mode.
If the victim attempts to decrypt using the wrong encryption mode, they will also therefore calculate an incorrect symmetric key, which will immediately fail before decrypting any of the message.

It is particularly notable that the ["Critique on the OpenPGP updates" section of the LibrePGP website](https://librepgp.org/#critique) calls out the advanced construction of the SEIPDv2 packet as unnecessary, and states that the LibrePGP protocol designers intentionally rejected it (subsection 1, "Symmetric Mode"), despite the use of KDFs to prevent such cross-protocol attacks being a recommended technique since at least 2010 (see e.g. [RFC5869 section 3.2](https://www.rfc-editor.org/rfc/rfc5869#section-3.2)).

## Key Overwriting Attacks

In 2022, Lara Bruseghini, Kenneth Patterson and Daniel Huigens [published a paper outlining practical attacks against secret key stores in OpenPGP](https://www.kopenpgp.com/assets/paper.pdf).

These arise because LibrePGP and versions of OpenPGP predating RFC9580 do not specify any integrity checking of secret keys on disk.
An attacker may be able to trick or force a victim into importing a modified secret key.
If the victim subsequently signs a message or encrypts a message to themselves using that key, the signature or encrypted data will contain cryptographic errors.
These errors can then be examined by the attacker to derive the original secret key.
The victim may not notice that the key has been modified because the attacker's changes can be designed so that the key is still capable of decrypting data and/or verifying signatures correctly.

Several implementations have addressed this by implementing explicit tests for whether the secret and public keys on disk form valid keypairs before use.
These tests are computationally expensive and need to be performed every time a secret key is used, so while they are sufficient to mitigate the attack they are far from ideal.

The attack is more efficiently mitigated in RFC9580 by specifying [the use of AEAD encryption for secret keys in local storage](https://datatracker.ietf.org/doc/html/rfc9580#name-secret-key-encryption).
AEAD encryption modes automatically provide integrity validation, eliminating the need for explicit validity checks.

Andrew Gallagher, 19th August 2024
(last updated 11th September 2024)
