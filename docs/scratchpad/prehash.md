# Pre-hashed signatures

It may be useful to define a pre-hashing scheme for OpenPGP signatures.
Currently the entire document must be passed to the OpenPGP layer to be signed over, which includes an internal hashing step.
Since many documents already have hashes calculated over them for other purposes, it would be efficient if this hashing stage could be reused.

The ([OPS signature subject type registry](https://andrewgdotcom.gitlab.io/openpgp-signatures)) could be expanded to include a "Pre-hashed" value.
"Pre-hashed" means that the subject is hashed and the signature is made over (subject digest || subject length) instead of the subject itself.
The pre-hash algorithm MUST be the same one used for the signature.
