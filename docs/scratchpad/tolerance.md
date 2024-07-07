# Tolerance

If Someone were to propose a recipe for Open/Libre tolerance, what would it look like?

## LibrePGP Encryption Formats (Type 20 AEAD/OCB)

An implementation wishing to support type-20 encrypted data (Feature 0x02):

*    SHOULD accept type-20 packets
*    MAY generate type-20 packets where all recipients advertise Feature 0x02

Whether it wishes to support type-20 or not, an implementation:
*    SHOULD warn about compatibility when importing a secret key with Feature 0x02
*    SHOULD allow a key owner to remove 0x02 from their key and republish

LibrePGP Signature and Key Formats (v5)

An implementation wishing to support v5 sigs/keys (Feature 0x04):

*    MAY import v5 public and secret keys
*    SHOULD warn about possible incompatibilities when importing a v5 secret key
*    MAY accept v5 sigs if the recipient advertises Feature 0x04
*    MAY generate v5 sigs where all (known) recipients advertise Feature 0x04
*    MUST NOT accept non-v5 sigs made by v5 keys (to prevent downgrade attacks) ((or: "MUST NOT accept v3 sigs made by v5 keys"?))

An implementation wishing to support v5 (encryption?) subkeys only:

*    MAY import v4 public and secret keys with v5 (encryption?) subkeys
*    SHOULD warn about possible incompatibilities when importing a v5 secret subkey

An implementation not wishing to support v5 sigs/keys:

*    SHOULD ignore v5 subkeys of v4 keys, and SHOULD warn about possible incompatibility.

Possible additional v5 subkey text for draft-wussler ((lost cause?))

*    A V5 subkey is constructed identically to a V6 subkey, except the version number is set to 5. An implementation wishing to add PQC encryption support to an existing V4 key MAY add a V5 encryption subkey that uses a PQC algorithm to that V4 key, with a V4 subkey binding signature.
*    V5 subkeys MUST NOT be attached to a V6 primary key.
*    PQC algorithms MUST NOT be used in V4 subkeys. ((no need if v5 keys are defined))

((Note that this will only work IFF gnupg reverts their changes to code points and KEM fixed data. Otherwise the implementation of PQC in v5 keys will not be compliant with draft-wussler))

General guidelines

*    SHOULD keep up to date with revised LibrePGP drafts, but:
*    SHOULD NOT implement codepoints outside those already reserved to LibrePGP:
*    Key and signature version 5
*    Packet type 20
*    Feature flags 0x02 and 0x04
*    MUST prefer RFC specifications where any conflict exists

## Miscellaneous

### V5 Fingerprints:

BEWARE! gnupg 2.4 with the `--with-fingerprint` option displays v5 fingerprints whitespace-split into FIVE-character blocks, without double-spacing, and truncated to 50 hex characters.
See https://lists.gnupg.org/pipermail/gnupg-users/2024-March/067022.html

### V6 Fingerprints:

V6 fingerprints SHOULD NOT be displayed unless specifically required
V6 fingerprints MAY be formatted as 44 Base64URL characters, constructed as follows:

*    Prepend the 32-byte wire-format fingerprint with a version byte 0x06
*    base64url_encode the resulting 33 bytes into 44 characters of Base64URL
*    Example: Bh9G8sbr9rtDA9-dvd82RVD1vA81_t1dn12Sre23vfAI
*    This can optionally be whitespace-split into 11 4-character blocks
*    If so, the double-spacing SHOULD be placed after the third and seventh blocks
*    Example: Bh9G 8sbr 9rtD  A9-d vd82 RVD1 vA81  _t1d n12S re23 vfAI

This format of fingerprint has the following features:

*    It is visually distinct from any form of v5 fingerprint (as printed by GnuPG)
*    It encodes the version, so it is safely generisable
*    It is URL-safe, e.g. for HKP:
*    It is the direct Base64URL encoding of the wire format ( Version, Fingerprint )
*    See https://gitlab.com/andrewgdotcom/draft-gallagher-openpgp-hkp/-/issues/19
*    Note however that it uses a different charset from ASCII armor
*    It is only 4 characters longer than a traditional v4 fingerprint
*    It splits into an integer number of 4-character (3-byte) whitespaced blocks
*    (as would v4 fps if encoded in 1+20 bytes...!)
*    (fingerprints that are not 3n-1 bytes long, e.g. v3, would need padding)
*    First 3 blocks exactly contain the long keyID (1B version + 8B keyID -> 12 chars)
*    To convert to a keyID, base64url_decode and then discard the version byte:

        % base64 -d <<<"Bh9G 8sbr 9rtD"|hexdump -C
        00000000  06 1f 46 f2 c6 eb f6 bb  43                       |..F.....C|
        00000009
