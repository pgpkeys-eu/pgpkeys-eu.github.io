# Signature Subpacket Classes

Signature subpacket types may be roughly classified, depending on their usage:

* General subpackets.
	These may be attached to any signature type, and define properties of the signature itself.
	Some of these subpackets are self-verifying (SV), i.e. they contain hints to locate the issuing key that can be confirmed after the fact.
	It MAY be reasonable to place self-verifying general subpackets in the unhashed area.
	All other general subpackets MUST be placed in the hashed area.
	
	Subpacket types: Signature Creation Time, Signature Expiration Time, Revocable, Issuer Key ID (SV), Notation Data, Signer's User ID, Issuer Fingerprint (SV).
	
	(Notation subpackets are classified here as general subpackets, however the notations within them may have arbitrary semantics at the application layer)

* Context subpackets.
	These have semantics that are meaningful only when used in particular contexts:

	* Preference subpackets.
		These are normally only meaningful in a direct self-sig (or historically a self-cert over the primary User ID) and define usage preferences for the certificate as a whole.
		They MAY be used in self-certs over other User IDs, in which case they define usage preferences for just that User ID (but this is not always meaningful or universally supported).
		Some of these subpackets MAY be used as both preference and key revocation subpackets (PKR).
		They SHOULD NOT be used elsewhere.
		They MUST be placed in the hashed area.
		
		Subpacket types: (Additional Decryption Key), Preferred Symmetric Ciphers, Revocation Key, Preferred Hash Algorithms, Preferred Compression Algorithms,
		Key Server Preferences, Preferred Key Server (PKR), Features, (Preferred AEAD Algorithms), Preferred AEAD Ciphersuites, Replacement Key (PKR).

	* Revocation subpackets.
		These are normally only meaningful in a revocation signature.
		Some of these subpackets MAY be used as both preference and key revocation subpackets (PKR), but are not used in certification or subkey revocations.
		They SHOULD NOT be used elsewhere.
		They MUST be placed in the hashed area.

		Subpacket types: Reason for Revocation, Preferred Key Server (PKR), Replacement Key (PKR).
	
	* Key property subpackets.
		These are only meaningful in a binding signature (or historically a self-cert over the primary User ID) and define properties of that particular (sub)key.
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
		
		Subpacket types: Exportable Certification, Trust Signature, Regular Expression, Policy URI, (Trust Alias).
	
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
	
	Subpacket types: Signature Target, Embedded Signature, Delegated Revoker (->Embedded Key?), Approved Certifications (->Signature Target List?).

Type| 	Name							| Class		| Critical	| SV| Signature Context		| Notes
----|-----------------------------------|-----------|-----------|---|-----------------------|-------------------
0   |	Reserved 	 					|			|			|	|						|
1  	|	Reserved 	 					|			|			|	|						|
2  	|	Signature Creation Time 		| General	| SHOULD	|	|						| MUST always be present in hashed area
3  	|	Signature Expiration Time 		| General	| SHOULD	|	|						|
4  	|	Exportable Certification 		| WoT     	| MUST(\*)	|	|						| boolean, default true (\* if false)
5  	|	Trust Signature 				| WoT		|			|	|						|
6  	|	Regular Expression 				| WoT		| SHOULD	|	|						|
7  	|	Revocable 						| General	|			|	| cert and direct sigs	| boolean, default false (deprecated in [draft-revocation](https://datatracker.ietf.org/doc/html/draft-dkg-openpgp-revocation))
8  	|	Reserved 						|			|			|	|						|
9  	|	Key Expiration Time 			| Key prop  | SHOULD	|	|						|
10 	|	(Additional Decryption Key/ARR)	| Preference|			|	|						| PGP.com proprietary feature
11 	|	Preferred Symmetric Ciphers 	| Preference|			|	|						|
12 	|	Revocation Key (deprecated) 	| Preference|			|	|						| deprecated in [RFC9580](https://datatracker.ietf.org/doc/html/rfc9580)
13-15 |	Reserved 	 					|			|			|	|						|
16 	|	Issuer Key ID 					| General	|			| Y	|						| issuer fingerprint is preferred
17-19 |	Reserved 	 					|			|			|	|						|
20 	|	Notation Data 					| General	|			|	|						| notations may be further classified
21 	|	Preferred Hash Algorithms 		| Preference|			|	|						|
22 	|	Preferred Compression Algorithms| Preference|			|	|						|
23 	|	Key Server Preferences 			| Preference|			|	|						|
24 	|	Preferred Key Server 			| Pref/Rev  | 			|	|						|
25 	|	Primary User ID 				| Self-cert |			|	|						| boolean, default false
26 	|	Policy URI 						| WoT		|			|	|						| (should have been a notation)
27 	|	Key Flags 						| Key prop  | SHOULD	|	|						|
28 	|	Signer's User ID 				| General	|			|	|						|
29 	|	Reason for Revocation 			| Revocation|			|	|						| (free text field should have been a notation)
30 	|	Features 						| Preference|			|	|						|
31 	|	Signature Target 				| Data type	|			|	| 0x50 third-party conf	| [utility unclear (not a unique identifier)](https://gitlab.com/dkg/openpgp-revocation/-/issues/13)
32 	|	Embedded Signature 				| Data type	|			|\* | 0x18 subkey binding	| (\* IFF it contains an 0x19 primary key binding signature packet)
33 	|	Issuer Fingerprint 				| General	|			| Y	|						| 
34 	|	(Preferred AEAD Algorithms)		| Preference|			|	|						| [4880bis](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis-10)
35 	|	Intended Recipient Fingerprint 	| Document	| SHOULD 	|	|						|
36 	|	Delegated Revoker				| Data type	| (MUST)	|	| TBD delegated revoker	| [draft-revocation](https://datatracker.ietf.org/doc/html/draft-dkg-openpgp-revocation)
37 	|	Approved Certifications			| Data type	|			|	| 0x16 cert approval 	| [4880bis](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis-10), [draft-1pa3pc](https://datatracker.ietf.org/doc/html/draft-dkg-openpgp-1pa3pc)
38 	|	(Key Block)			 	        | Document	|			| Y	|						| [4880bis](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-rfc4880bis-10)
39 	|	Preferred AEAD Ciphersuites 	| Preference|			|	|						|
40 	|	(Literal Data Meta Hash)		| Document	|			|	|						| [librepgp](https://datatracker.ietf.org/doc/html/draft-koch-librepgp), [draft-literal-data-metadata](https://datatracker.ietf.org/doc/html/draft-gallagher-openpgp-literal-metadata)
41 	|	(Trust Alias)					| WoT		|			|	|						| [librepgp](https://datatracker.ietf.org/doc/html/draft-koch-librepgp)
TBD	|	Replacement Key					| Pref/Rev  | SHOULD NOT|	|						| [draft-replacementkey](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-replacementkey)


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
* The signature type ranges are as defined in [OpenPGP Signature Semantics](signatures.html).
