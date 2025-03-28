# The Principles of User-Friendly Cryptography

Cryptography has a well-deserved reputation for being unfriendly, even hostile, to non-experts.
The most successful applications of cryptography therefore attempt to hide the gory details from the end user as far as possible.
But there is no such thing as a leak-proof abstraction, and the existence of cryptograpy will inevitably become apparent.
It is the responsibility of the application developer to ensure that the unpleasantness is kept to a minimum.

The following is a (non-exhaustive) list of some general principles that application developers should keep in mind.

## A Broken Signature Is No Signature

Applications must not give users a false, or confusing, sense of security.
If an attacker can modify a signature in transit, they can also delete the signature entirely, and vice versa.
This means that both "broken signature" and "no signature" are possible symptoms of the same attack scenario.
If one case triggered a more serious warning than the other, an MITM could trivially alter their attack method so that the less-serious warning message was displayed.

The first principle is therefore:

> A signature that does not verify, for any reason, must be presented to the user in the same way as a missing signature.

If an advanced user wishes to see debugging output that details the difference between missing and broken signatures, it should be allowed - but only if they explicitly ask for it.
Otherwise, the errors (or lack thereof) shown to the user are attacker-controlled information.

## Failure (to Decrypt) Is Not an Option

FTD, and the fear thereof, is the greatest barrier to widespread adoption of encryption.
Users will not trust their precious data to a mysterious box that might never give it back.
If a user feels that they or their correspondent(s) might lose access to their encrypted data, they would rather not use encryption at all.

The second principle is therefore:

> It must not be possible under any reasonable circumstances for a user to lose access to their decryption key material.

Such reliability can only be achieved by making multiple copies of the decryption key material and storing them securely in diverse locations.
It follows that multi-device support is not an optional extra, but a fundamental requirement.

## All Good Things Must End

Cryptography is often used to construct immutable records, which are crucial to prevent tampering and accidental loss.
No cryptographic system is perfect however, and if errors cannot be corrected they will eventually overwhelm the system.
In particular, cryptographic errors that are associated with a user may cause harm.

The third principle is therefore:

> A user must be able to destroy their cryptographic identity at any time.

## Every Cryptography Problem Can Be Solved by Another Global Authority (Which Must Not Exist)

Decentralisation is A Very Hard Problem, and decentralisation of trust roots is practically impossible.
The only viable alternative method is to implement global trusted authorities, and those never end well.
On the other hand, requiring users to manage their own trust roots never even *starts* well.

The fourth principle is therefore:

> Every user must delegate their own cryptographic trust roots, and must not be aware that they have done so.
