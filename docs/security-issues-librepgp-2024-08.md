# A Summary of Known Security Issues in LibrePGP

The [LibrePGP specification](https://datatracker.ietf.org/doc/html/draft-koch-librepgp) has a number of known security issues.
This document will be updated if any further issues come to light.

## v5 Signature Downgrade Attack

In 2022, Demi Marie Obenour [discovered a potential downgrade attack](https://gitlab.com/openpgp-wg/rfc4880bis/-/issues/130) against v5 signatures.

This arises because the metadata of an OpenPGP signature is hashed in as a "trailer" at the end of the digest input, rather than at the beginning.
In practical terms, this means that the trailer must be designed as if it were being parsed backwards from the end.
Any ambiguity in this backwards parsing opens the possibility of signature malleability, where an attacker takes a genuine signature over one piece of data and converts it into an apparently genuine signature over a different piece of data.

The Obeneur attack takes advantage of the fact that [v5 trailers end with an 8-octet length field](https://datatracker.ietf.org/doc/html/draft-koch-librepgp#name-computing-signatures), whereas all previous signature trailers end with a 4-octet field.
It is theoretically possible for an attacker to trick a victim into creating a signature with a specific value in the length field that simulates the trailer of a different signature version. 
v3 signatures are particularly easy to fake by this method, because their entire trailer is only five octets long, and so an attacker would have complete control over the fake trailer contents.

To mitigate this, RFC9580 specifies:
* [the size of the v6 trailer length field is four octets](https://datatracker.ietf.org/doc/html/rfc9580#name-computing-signatures), matching the v3 and v4 trailer formats and thus removing the parsing ambiguity.
* [signature versions must match the version of the key generating them](https://datatracker.ietf.org/doc/html/rfc9580#name-signature-packet-type-id-2), ensuring that any future cross-version malleability attack will be invalidated by the version mismatch.

## Type 20 "OCB Packet" Malleability Attack

In 2024, Falko Strenzke and Johannes Roth [published a paper outlining known attacks against AEAD constructions in both LibrePGP and CMS](https://eprint.iacr.org/2024/1110.pdf).

This arises because LibrePGP's Type 20 OCB encryption mode is similar enough to the historically-supported (but now deprecated) CFB encryption mode that a partial decryption of the initial section of the ciphertext is possible even if the wrong encryption mode is being used, so long as the symmetric session key is correct.
Since CFB does not contain any intrinsic error-checking, a victim might not be warned that the partial decryption was due to interference by an attacker, and may assume that the message was genuine.
The attacker may thus be able to trick the victim into replying with a message including the initial section of the decrypted plaintext.

To mitigate this, RFC9580 [specifies the use of a Key Derivation Function (KDF)](https://datatracker.ietf.org/doc/html/rfc9580#name-version-2-symmetrically-enc) in the alternative SEIPDv2 packet, to ensure that the session key used to encrypt the data in AEAD modes must be calculated at decryption time using (among other parameters) the ID of the AEAD encryption mode.
If the victim attempts to decrypt using the wrong encryption mode, they will therefore automatically use an incorrect session key, which will immediately fail.

It is particularly notable that the ["Critique on the OpenPGP updates" section of the LibrePGP website](https://librepgp.org/#critique) calls out the advanced construction of the SEIPDv2 packet as unnecessary, and states that the LibrePGP protocol designers intentionally rejected it (subsection 1, "Symmetric Mode"), despite the use of KDFs to prevent cross-protocol attacks being a known technique since before 2010 (see e.g. [RFC5869 section 3.2](https://www.rfc-editor.org/rfc/rfc5869#section-3.2)).

Andrew Gallagher, 16th August 2024
