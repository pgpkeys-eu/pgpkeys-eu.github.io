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

The "challenge-response direct" mode differs substantially from traditional email verification systems only in that SMTP delivery of the challenge is bypassed.
The "pre-agreed indirect" mode requires only that the verifier receives pre-constructed proofs via an API; no email service is required by the verifier.

## Caveats

Generation and submission of proofs is intended to be managed by the user's MUA, rather than by hand.
The MUA must therefore have explicit support for OOB proofs.

To prevent abuse via the advance calculation of large numbers of proofs, pre-agreed challenges SHOULD require the use of a public source of randomness, e.g [DRAND](https://drand.love).

# OOB Proof Wire Format

Domain separation is achieved by embedding a `Content-type:` header in the comment field of the `From:` message header, which is normally not attacker-controllable.
In addition, the `From:` header is the only one that MUST be signed by DKIM, and so we place all of the necessary data as additional fields in the embedded Content-type.

The embedded `Content-type` header MUST indicate the MIME type `application/oob-email-proof`, with the following attributes:

* `v` : the version of the proof format; MUST be 1.
* `t` : the time of proof generation, as an integer number of seconds since the UNIX epoch.
* `p` : the application protocol of the service that the verifier acts on behalf of, e.g. `https`.
* `d` : the domain of the service that the verifier acts on behalf of.
* `m` : the mode, one of `cd`, `ci`, `pd`, or `pi`.
* `a` : body hash algorithm; provers SHOULD use sha256.
* `bh`: body hash, in BASE-64 encoding.
* `ch`: the challenge, in BASE-64 encoding.

All attributes MUST be supplied, however the order is not important.

The body of the email MAY be empty, or MAY contain additional data as required by the application protocol.
The complete message body MUST be hashed, even if empty, using DKIM's simple body canonicalisation, and the hash recorded in the `bh` attribute.
This is necessary because not all DKIM deployments sign over the full message body.

The proof consists of the received email from the beginning of the `DKIM-Signature` header until the end of the body.

## Sample Proof

```
DKIM-Signature: v=1; a=rsa-sha256; c=simple/simple; d=andrewg.com;
	s=andrewg-com; t=1738518305;
	bh=frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=;
	h=From:To:Subject:Date:From;
	b=YqoWjdOE2LZhlby0yJ64NiqAR49HDi/HC89V7YzJzg/+8DoES2q31BCVi+C6Fo6CH
	 4g3900BDVjgA1a+Qxb+twiJIQXsdbFVdF+hTtHbf/jgo/u2bksNte1C2ge61x7NX+4
	 7YrZ2CyUFSNxhQb9KLk650Ink49o1/w2sLeu5bDTkEo7WMLjtjtBrQhUVIvjPdXY0q
	 s0nazZmm2vdgd4xf1LGLR/7xKBBKqXvvPKNJo7t5sa97mQcZ+/x4s9ZYln4bLLyCyb
	 nZ44vyk3dnVRbbNslXSwYHwwH19ZNgq6m8Q34ZBJpWKM35pq6mErGZ2bm1AMbrb1oD
	 7Afn+jEkyEgJQ==
Received: from localhost (localhost [127.0.0.1])
	by fum.andrewg.com (Postfix) with ESMTP id 717B05E34C
	for <andrewg@andrewg.com>; Sun,  2 Feb 2025 17:44:44 +0000 (UTC)
From: andrewg@andrewg.com (
  Content-type: application/oob-email-proof; v=1;
    p=hkp; d=verifier.example.com; m=cd; a=sha256; t=1738517505;
    bh=frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=;
    ch=yIslAhDENZley39x/EIyAnE8twWapjB654MqywKbMRP= )
To: Me <andrewg@andrewg.com>
Subject: OOB Proof of Email Identity for verifier.example.com
Message-Id: <20250202174447.717B05E34C@fum.andrewg.com>
Date: Sun,  2 Feb 2025 17:44:44 +0000 (UTC)

```
