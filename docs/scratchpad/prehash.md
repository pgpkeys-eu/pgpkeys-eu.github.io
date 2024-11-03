# Pre-hashed signatures

It may be useful to define a pre-hashing scheme for OpenPGP signatures.
Currently the entire document must be passed to the OpenPGP layer to be signed over, which includes an internal hashing step.
Since many documents already have hashes calculated over them for other purposes, it would be efficient if this hashing stage could be reused.

The nesting octet comes last in the OPS packet, and so could be extended to a variable-length field if required.
The only special value currently defined is 0 ("Skipping"); all other values are treated as "Verbatim".
This field could easily be promoted to a registry and renamed to "OpenPGP One-Pass Signature Subject Format", with the following initial entries:

Value   | Description
--------|-----------------------
0       | Skipping
1       | Verbatim
2       | Pre-hashed

The signature subject is the sequence of bytes that the signature is made over.
By default, the signature subject is the entire sequence of octets between the end of the last OPS packet and the beginning of the first Signature packet.
By default, the signature is made over the subject directly.

* If the octet is zero ("Skipping"), this means that the subject is identical to that of the following OPS packet (recursively).
* "Verbatim" means that the subject is the exact sequence of octets between the end of the current OPS packet and the beginning of the matching Signature packet.
* "Pre-hashed" means that the subject is hashed and the signature is made over (subject digest || subject length) instead of the subject itself.
    The pre-hash algorithm MUST be the same one used for the signature.

Note that a sequence of skipping OPS signatures MUST use the same subject format.
If a different subject format is required, then a separate signed message should be made.

Text document signatures can be thought of as a special case of signature subject formatting that can be used with non-OPS signatures.
This seems fragile though; would it be better to define a "pre-hashed" document signature type that can be used on non-OPS messages?
Do we then need "pre-hashed binary" and "pre-hashed text" document sig types?
How do we rein in the combinatorics?
