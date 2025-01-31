# Out-of-Band Email Verification

Many services use verification of successful email delivery as a security measure.
This may be used either on its own or in conjunction with other measures, and may be used for various security purposes, ranging from antispam to account recovery.

Email delivery incurs costs on the service, most obviously financial but also potentially reputational.
For example, a service that sends out large numbers of verification emails may have them treated as spam, or find itself on a global blocklist.

It is therefore desirable for a service to allow its users to prove ownership of their email address by other means.

# Protocol Overview

In the below, the service that requires email validation is called the "verifier", and the user or client is called the "prover".

There are a small number of variations on the basic protocol:

* "non-interactive" vs "interactive" indicate how challenges are obtained.
* "direct" vs "indirect" indicate how proofs are submitted.

* The prover constructs a challenge:
    * In the interactive case, the prover calls an API endpoint on the verifier to obtain a challenge.
    * In the non-interactive case, the prover constructs a challenge using a well-known method.
* The prover sends an email containing the challenge:
    * In the direct case, this email is sent to the verifier.
    * In the indirect case, the prover sends the email to itself.
* The prover's email provider signs over the email using its private DKIM key.
* In the indirect case, the prover submits the received email via an API endpoint.
* The verifier looks up the email domain's public DKIM key and verifies the submitted proof.

The "interactive direct" case differs from traditional email verification systems only in that SMTP delivery of the challenge is bypassed.
The "non-interactive indirect" case requires only that the verifier receives pre-constructed proofs via an API.

To prevent abuse stemming from advance calculation of many proofs, non-interactive challenges SHOULD require the use of a public source of randomness, e.g DRAND.
