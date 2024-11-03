# Signature Subpacket Classes

Signature subpacket types may be roughly classified, depending on their usage:

* General subpackets.
	These may be attached to any signature type, and define properties of the signature itself.
	Some of these subpackets are self-verifying (SV), i.e. they contain hints to locate the issuing key that can be confirmed after the fact.
	It MAY be reasonable to place self-verifying general subpackets in the unhashed area.
	All other general subpackets MUST be placed in the hashed area.
	
	Subpacket types: Signature Creation Time, Signature Expiration Time, Issuer Key ID (SV), Notation Data, Signer's User ID, Issuer Fingerprint (SV).
	
	(Notation subpackets are classified here as general subpackets, however the notations within them may have arbitrary semantics at the application layer)

* Context subpackets.
	These have semantics that are meaningful only when used in particular contexts:

	* Preference subpackets.
		These are normally only meaningful in a direct self-sig (or historically a self-cert over the primary User ID) and define usage preferences for the certificate as a whole.
		They MAY be used in self-certs over other User IDs, in which case they define usage preferences for just that User ID (but this is not always meaningful or universally supported).
		The Replacement Key subpacket MAY also be used as a key revocation subpacket.
		They SHOULD NOT be used elsewhere.
		They MUST be placed in the hashed area.
		
		Subpacket types: (Additional Decryption Key), Preferred Symmetric Ciphers, Revocation Key (deprecated), Preferred Hash Algorithms, Preferred Compression Algorithms,
		Key Server Preferences, Preferred Key Server, Features, (Preferred AEAD Algorithms), Preferred AEAD Ciphersuites, (Replacement Key).

	* Revocation subpackets.
		These are only meaningful in a revocation signature.
		They SHOULD NOT be used elsewhere.
		They MUST be placed in the hashed area.

		Subpacket types: Reason for Revocation.
	
	* Key property subpackets.
		These are only meaningful in a direct or binding self-sig (or historically a self-cert over the primary User ID) and define properties of that particular (sub)key.
		They SHOULD NOT be used elsewhere.
		They MUST be placed in the hashed area.

		Subpacket types: Key Expiration Time, Key Flags.

	* Self-certification subpackets.
		These are only meaningful in a self-certification over a User ID, and define properties of that User ID.
		They SHOULD NOT be used elsewhere.
		They MUST be placed in the hashed area.

		Subpacket types: Primary User ID

	* WoT subpackets.
		These are only meaningful in third-party certification signatures and define properties of the Web of Trust.
		They SHOULD NOT be used elsewhere.
		They MUST be placed in the hashed area.
		
		Subpacket types: Exportable Certification, Trust Signature, Regular Expression, Revocable, Policy URI, (Trust Alias).
	
	* Document subpackets.
		These are only meaningful in document signatures, and define properties of the document or message.
		They SHOULD NOT be used elsewhere.
		Some of these subpackets are self-verifying (SV) and MAY be placed in the unhashed area.
		All other document subpackets MUST be placed in the hashed area.
		(Beware that the usefulness of all of these subpackets has been questioned)
		
		Subpacket types: Intended Recipient Fingerprint, (Key Block (SV)), (Literal Data Meta Hash).
	
* Data type subpackets.
	These are only meaningful in signature types whose specification explicitly requires them.
	They SHOULD NOT be used elsewhere.
	They have no intrinsic semantics; all semantics are defined by the enclosing signature.
	
	Subpacket types: Signature Target, Embedded Signature, (Delegated Revoker (->Embedded Key?)), (Approved Certifications (->Signature Target List?)).

Type| 	Name							| Class	| Crit	| SV| Context		| Notes
----|-----------------------------------|-------|-------|---|---------------|-------------------
0   |	Reserved 	 					|	-	|		|	|				| never used
1  	|	Reserved 	 					|	-	|		|	|				| never used
2  	|	Signature Creation Time 		| Gen	| SHOULD|	|				| MUST always be present in hashed area
3  	|	Signature Expiration Time 		| Gen	| SHOULD|	|				|
4  	|	Exportable Certification 		| WoT   | \*	|	|				| boolean, default true (\* MUST only if false)
5  	|	Trust Signature 				| WoT	|		|	|				|
6  	|	Regular Expression 				| WoT	| SHOULD|	|				|
7  	|	Revocable 						| WoT	|		|	| 			 	| boolean, default false (deprecated in [draft-revocation](https://datatracker.ietf.org/doc/html/draft-dkg-openpgp-revocation))
8  	|	Reserved 						|	-	|		|	|				| never used
9  	|	Key Expiration Time 			| Key   | SHOULD|	|				|
10 	|	(Additional Decryption Key/ARR)	| Pref	|		|	|				| PGP.com proprietary feature
11 	|	Preferred Symmetric Ciphers 	| Pref	|		|	|				|
12 	|	Revocation Key (deprecated) 	| Pref	|		|	|				| deprecated in [RFC9580](https://datatracker.ietf.org/doc/html/rfc9580)
13-15 |	Reserved 	 					|	-	|		|	|				| never used
16 	|	Issuer Key ID 					| Gen	|		| Y	|				| issuer fingerprint is preferred
17-19 |	Reserved 	 					|	-	|		|	|				| never used
20 	|	Notation Data 					| Gen	|		|	|				| notations may be further classified
21 	|	Preferred Hash Algorithms 		| Pref	|		|	|				|
22 	|	Preferred Compression Algorithms| Pref	|		|	|				|
23 	|	Key Server Preferences 			| Pref	|		|	|				|
24 	|	Preferred Key Server 			| Pref	| 		|	|				|
25 	|	Primary User ID 				| Self  |		|	|				| boolean, default false
26 	|	Policy URI 						| WoT	|		|	|				| (should have been a notation)
27 	|	Key Flags 						| Key   | SHOULD|	|				| ([also vaguely allowed in third-party certs?](https://gitlab.com/openpgp-wg/rfc4880bis/-/issues/120))
28 	|	Signer's User ID 				| Gen	|		|	|				|
29 	|	Reason for Revocation 			| Rev	|		|	|				| (free text field should have been a notation)
30 	|	Features 						| Pref	|		|	|				|
31 	|	Signature Target 				| Data 	|		|	| 0x50 3-p conf	| [utility unclear (not a unique identifier)](https://gitlab.com/dkg/openpgp-revocation/-/issues/13)
32 	|	Embedded Signature 				| Data 	|		|\* | 0x18 sbind	| (\* Yes IFF it contains an 0x19 primary key binding signature packet)
33 	|	Issuer Fingerprint 				| Gen	|		| Y	|				|
34 	|	(Preferred AEAD Algorithms)		| Pref	|		|	|				| [4880bis](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis-10)
35 	|	Intended Recipient Fingerprint 	| Doc	| SHOULD|	|				|
36 	|	(Delegated Revoker)				| Data 	| MUST	|	| TBD del rev	| [draft-revocation](https://datatracker.ietf.org/doc/html/draft-dkg-openpgp-revocation)
37 	|	(Approved Certifications)		| Data 	|		|	| 0x16 1pa3pc 	| [draft-1pa3pc](https://datatracker.ietf.org/doc/html/draft-dkg-openpgp-1pa3pc)
38 	|	(Key Block)			 	        | Doc	|		| Y	|				| [4880bis](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis-10)
39 	|	Preferred AEAD Ciphersuites 	| Pref	|		|	|				|
40 	|	(Literal Data Meta Hash)		| Doc	|		|	|				| [librepgp](https://datatracker.ietf.org/doc/html/draft-koch-librepgp) not yet implemented
41 	|	(Trust Alias)					| WoT	|		|	|				| [librepgp](https://datatracker.ietf.org/doc/html/draft-koch-librepgp) not yet implemented
TBD	|	(Replacement Key)				| Pref  | SHDNOT|	|				| [draft-replacementkey](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-replacementkey)


## Further notes

* Three subpacket types are Boolean, with different default values for when they are absent (two true, one false).
	It is RECOMMENDED that these subpackets not be used to convey their default values, only the non-default value.
	The default value SHOULD instead be conveyed by the absence of the subpacket.
* Unless otherwise indicated, subpackets SHOULD NOT be marked critical.
	In particular, a critical subpacket that invalidates a self-signature will leave the previous self-signature (or no self-signature!) as the most recent valid self-signature from the PoV of some receiving implementations.
	A generating implementation MUST be sure that all receiving implementations will behave as intended if a signature containing a critical subpacket is invalidated.
	Otherwise, with the possible exception of document signatures, it is NOT RECOMMENDED to set the critical bit.
* It is RECOMMENDED that a signature's creator places all subpackets in the hashed area, even self-verifying subpackets for which this is not strictly necessary.
	The unhashed area SHOULD be reserved for informational subpackets attached by third parties (which can be safely stripped).
* The signature type ranges are as defined in [OpenPGP Signatures and Signed Messages](https://andrewgdotcom.gitlab.io/openpgp-signatures).

# Guidance for management of the Signature Subpacket Registry

* The registry SHOULD be updated to include the extra columns in the above table (apart from the notes!).
* Future boolean subpackets SHOULD NOT contain an explicit value; a value of TRUE SHOULD be indicated by the presence of the subpacket, and FALSE otherwise.
* Specification of new subpackets SHOULD address classification as outlined above.
* Subpackets SHOULD be implemented in the private/experimental area first, then reassigned a permanent code point (not strictly required).
