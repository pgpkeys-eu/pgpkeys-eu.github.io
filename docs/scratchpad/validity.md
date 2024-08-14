Validity
========

In OpenPGP, the word "valid" is used a lot - but there are at least four kinds of "validity" that must be distinguished:

1. cryptographic validity
    * is the signature mathematically sound?
    * has it been made over the preceding signable packet(s)?
2. structural/grammatical validity
    * are all the required packets present?
    * are they in the correct order?
    * do all the signable packets have the expected signatures over them?
    * are the signatures of the expected type?
3. temporal validity
    * is it expired?
    * has it been revoked?
        * hard or soft?
4. claim validity
    * can the identity claim be corroborated?
    * does the identity match the use case?
