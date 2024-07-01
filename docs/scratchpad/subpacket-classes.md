# Signature Subpacket Classes

Signature subpacket types may be roughly classified, depending on their usage:

* General subpackets.
	These may be attached to any signature type, and define properties of the signature itself.
	Some of these subpackets are self-verifying (SV), i.e. they contain hints to locate the issuing key that can be confirmed after the fact.
	It MAY be reasonable to place self-verifying general subpackets in the unhashed area.
	All other general subpackets MUST be placed in the hashed area.
	
	Subpacket types: Creation Time, Expiration Time, Revocable, Issuer Key ID (SV), Notation Data, Signer's User ID, Issuer Fingerprint (SV).
	
	(Notation subpackets are classified here as general subpackets, however the notations within them may have arbitrary semantics at the application layer)

* Context subpackets.
	These have semantics that are meaningful only when used in particular classes of signature:

	* Preference subpackets.
		These may be attached only to a direct self-sig (or historically a self-cert over the primary UserID) and define usage preferences of the TPK.
		They MUST be placed in the hashed area.
		They MAY be used in other self-certs, in which case they only define usage preferences per that UserID (but this is not always well-defined or universally supported).
		Some preference subpackets MAY also be used in revocation signatures.
		
		Subpacket types: Key Expiration Time, Preferred Symmetric Ciphers, Revocation Key, Preferred Hash Algorithms, Preferred Compression Algorithms,
		Key Server Preferences, Preferred Key Server, Primary User ID, Key Flags, Reason for Revocation, Features, Preferred AEAD Ciphersuites, Replacement Key.

	* WoT subpackets.
		These may be attached only to certification signatures and define properties of the Web of Trust.
		They MUST be placed in the hashed area.
		
		Subpacket types: Exportable Certification, Trust Signature, Regular Expression, Policy URI, Trust Alias.
	
	* Document subpackets.
		These may be attached only to document signatures, and define properties of the document or message.
		Some of these subpackets are self-verifying (SV).
		All other document subpackets MUST be placed in the hashed area.
		
		Subpacket types: Intended Recipient Fingerprint, Key Block (SV), Literal Data Meta Hash.
	
* Data type subpackets.
	These may be attached only to specific signature types.
	They have no intrinsic semantics; any semantics are due to the enclosing signature's specification.
	
	Subpacket types: Signature Target, Embedded Signature, Delegated Revoker, Approved Certifications.


Type| 	Name							| Class		| Critical	| Signature Context		| Notes
----|-----------------------------------|-----------|-----------|-----------------------|-------------------
0   |	Reserved 	 					|			|			|						|
1  	|	Reserved 	 					|			|			|						|
2  	|	Signature Creation Time 		| General	| SHOULD	|						| MUST be present in hashed area
3  	|	Signature Expiration Time 		| General	| SHOULD	|						|
4  	|	Exportable Certification 		| WoT     	| MUST*		|						| (*) if non-default value
5  	|	Trust Signature 				| WoT		|			|						|
6  	|	Regular Expression 				| WoT		| SHOULD	|						|
7  	|	Revocable 						| General	|			| revocable signatures	| deprecated in [draft-revocation](https://datatracker.ietf.org/doc/html/draft-gallagher-openpgp-replacementkey)
8  	|	Reserved 						|			|			|						|
9  	|	Key Expiration Time 			| Preference| SHOULD	|						|
10 	|	Reserved						|			|			|						|
11 	|	Preferred Symmetric Ciphers 	| Preference|			|						|
12 	|	Revocation Key (deprecated) 	| Preference|			|						| deprecated in [RFC9580](https://datatracker.ietf.org/doc/html/draft-ietf-openpgp-crypto-refresh)
13-15 |	Reserved 	 					|			|			|						|
16 	|	Issuer Key ID 					| General	|			|						| self-verifying
17-19 |	Reserved 	 					|			|			|						|
20 	|	Notation Data 					| General	|			|						| notations may be further classified
21 	|	Preferred Hash Algorithms 		| Preference|			|						|
22 	|	Preferred Compression Algorithms| Preference|			|						|
23 	|	Key Server Preferences 			| Preference|			|						|
24 	|	Preferred Key Server 			| Preference| 			|						|
25 	|	Primary User ID 				| Preference|			|						|
26 	|	Policy URI 						| WoT		|			|						| (should have been a notation)
27 	|	Key Flags 						| Preference| SHOULD	|						|
28 	|	Signer's User ID 				| General	|			|						|
29 	|	Reason for Revocation 			| Preference|			| (revocations only)	| (free text should have been a notation)
30 	|	Features 						| Preference|			|						|
31 	|	Signature Target 				| Data type	|			| 0x50 third-party conf	| utility unclear (not a unique identifier)
32 	|	Embedded Signature 				| Data type	|			| 0x18 subkey binding	|
33 	|	Issuer Fingerprint 				| General	|			|						| self-verifying
34 	|	Reserved 	 					|			|			|						|
35 	|	Intended Recipient Fingerprint 	| Document	| SHOULD 	|						|
36 	|	Delegated Revoker				| Data type	| (MUST)	| TBD delegated revoker	| [draft-revocation](https://datatracker.ietf.org/doc/html/draft-dkg-openpgp-revocation) (-> "embedded key"?)
37 	|	Approved Certifications			| Data type	|			| 0x16 cert approval 	| [librepgp](https://datatracker.ietf.org/doc/html/draft-koch-librepgp), [draft-1pa3pc](https://datatracker.ietf.org/doc/html/draft-dkg-openpgp-1pa3pc) (-> "signature target list"?)
38 	|	Key Block			 	        | Document	|			|						| [librepgp](https://datatracker.ietf.org/doc/html/draft-koch-librepgp), utility unclear
39 	|	Preferred AEAD Ciphersuites 	| Preference|			|						|
40 	|	Literal Data Meta Hash			| Document	|			|						| [librepgp](https://datatracker.ietf.org/doc/html/draft-koch-librepgp), [draft-literal-data-metadata](https://datatracker.ietf.org/doc/html/draft-gallagher-openpgp-literal-metadata)
41 	|	Trust Alias						| WoT		|			|						| [librepgp](https://datatracker.ietf.org/doc/html/draft-koch-librepgp)
TBD	|	Replacement Key					| Preference| SHOULD NOT|(also 0x20 revocations)| [draft-replacementkey](https://datatracker.ietf.org/doc/html/draft-gallagher-openpgp-replacementkey)
