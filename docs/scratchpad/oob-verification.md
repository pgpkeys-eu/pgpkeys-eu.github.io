# Out-of-Band Email Verification

Many services use verification of successful email delivery as a security measure.
This may be used either on its own or in conjunction with other measures, and may be used for various security purposes, ranging from antispam to account recovery.

Email delivery incurs costs on the service, most obviously financial but also potentially reputational.
For example, a service that sends out large numbers of verification emails may have them treated as spam, or find itself on a global blocklist.

It is therefore desirable for a service to allow its users to prove ownership of their email address by other means.

# OOB Proof Overview

In the below, the service that requires email validation is called the "verifier", and the user (or her client) is called the "prover".

There are a small number of variations on the basic proof format:

* "challenge-response" (c) vs "pre-agreed" (p) indicate how challenges are obtained.
* "direct" (d) vs "indirect" (i) indicate how proofs are submitted.

The *mode* of the proof is a direct sum of the variations usee, e.g. "challenge-response direct" or "pre-agreed indirect".

* The prover constructs a challenge:
    * In the challenge-response mode, the prover calls an API endpoint on the verifier to obtain a challenge.
    * In the pre-agreed mode, the prover constructs a challenge using a well-known method.
* The prover sends an email containing the challenge:
    * In the direct mode, this email is sent to the verifier.
    * In the indirect mode, the prover sends the email to itself.
* The prover's email provider signs over the email using its private DKIM key.
* In the indirect mode, the prover submits the received email via an API endpoint on the verifier.
* The verifier looks up the email domain's public DKIM key and verifies the submitted proof.

The "challerge-response direct" mode differs substantially from traditional email verification systems only in that SMTP delivery of the challenge is bypassed.
The "pre-agreed indirect" mode requires only that the verifier receives pre-constructed proofs via an API; no email service is required by the verifier.

## Caveats

Generation and submission of proofs is intended to be managed by the user's MUA, rather than by hand.
The MUA must therefore have explicit support for OOB proofs.

To prevent abuse via the advance calculation of large numbers of proofs, pre-agreed challenges SHOULD require the use of a public source of randomness, e.g [DRAND](https://drand.love).

# OOB Proof Wire Format

Domain separation is achieved by embedding a `Content-type:` header in the comment field of the `From:` message header, which is normally not attacker-controllable.
In addition, the `From:` header is the only one that MUST be signed by DKIM, and so we place all of the necessary data as additional fields in the embedded Content-type.

The embedded `Content-type` header MUST indicate the MIME type `application/oob-email-proof`.
The additional fields are:

* `v` : the version of the proof format; MUST be 1.
* `p` : the application protocol of the service that the verifier acts on behalf of, e.g. `https`.
* `r` : the domain of the service that the verifier acts on behalf of.
* `m` : the mode, one of `cd`, `ci`, `pd`, or `pi`.
* `d` : additional data (application-dependent).
* `c` : the challenge, in BASE-64 encoding.

All additional fields MUST be supplied.
Protocol designers should ensure that additional data uses an encoding format that is tolerant of line-breaking and whitespace; BASE-64 is RECOMMENDED.

The body of the email MAY be empty, or MAY contain additional data as required by the application protocol.
If it contains application protocol data, a checksum or signature over it SHOULD be included in the OOB additional data field, since not all DKIM implementations sign over the whole email body.

The proof consists of the received email from the beginning of the `DKIM-Signature` header until the end of the body.

## Sample Proof

```
DKIM-Signature: v=1; a=rsa-sha256; c=simple/simple; d=andrewg.com;
	s=andrewg-com; t=1738426955;
	bh=frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=;
	h=From:To:Subject:Date:From;
	b=od2FneewzZk3Ng/14lAFxxcp+W7Fyeklu8uzoXxcByOajejynQn7H925KgBDDJKO/
	 EMyaFxEXvm1dpkxABkgHc3Iuy39fS/CDFA7G20hJnboLdyuxnNG1WFlVhImyrLjG1X
	 HbQSIUVx4r0d/XZaJIxfRI50+ex3i9BEpy40GEPMcPLaMk5wiI36yNYHS0SbQTj9od
	 vUaEnnum5tLfLmU01+ucD2m8L4pWnfCFogI8p0zU13X0rxyDHEIp9sqa58TAlVHE7k
	 mGqUfXbzyuPpZL/Kw0NuB3quWncZQSEieVe6i2Thht3RpJ3qAwShefiw5ggV0gAnNn
	 HuD06u6ntXR6Q==
Received: from localhost (localhost [127.0.0.1])
	by fum.andrewg.com (Postfix) with ESMTP id 0759D5DC93
	for <andrewg@andrewg.com>; Sat,  1 Feb 2025 16:22:18 +0000 (UTC)
From: andrewg@andrewg.com (
  Content-type: application/oob-email-proof; v=1,
    p=hkp, r=verifier.example.com, m=cd, d=,
    c=yIslAhDENZley39x/EIyAnE8twWapjB654MqywKbMRPM1OSRn1s2p1Nh/q4ibnsKpa )
To: Me <andrewg@andrewg.com>
Subject: OOB Proof of Email Identity for verifier.example.com
Message-Id: <20250201162223.0759D5DC93@fum.andrewg.com>
Date: Sat,  1 Feb 2025 16:22:18 +0000 (UTC)

```
