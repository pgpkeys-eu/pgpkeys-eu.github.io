# Verifiable Pseudorandom Functions in OpenPGP

There are several places in the OpenPGP specifications that require the generation of random values.
In high-confidentiality environments it may be desirable to ensure that these values are truly random, and are not used as a hidden channel.

We therefore wish to specify the use of Elliptic-Curve Verifiable Pseudorandom Functions, as per [Section 5 of RFC9381](https://www.rfc-editor.org/rfc/rfc9381.html#section-5).

# Use of Random Values in OpenPGP

((list all the places where randomness is required))

# Specification of VRFs

((x25519 and x448 only?))
