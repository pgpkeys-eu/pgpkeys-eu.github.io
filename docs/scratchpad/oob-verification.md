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

# Caveats

The verification protocol is intended to be managed by the user's MUA, rather than by hand.
The MUA must therefore have explicit support for the protocol.

To prevent abuse stemming from advance calculation of many proofs, non-interactive challenges SHOULD require the use of a public source of randomness, e.g [DRAND](https://drand.love).

An attacker could try to trick a victim into creating an email that the attacker could then present as "proof" that they controlled the victim's email address.
Domain separation could be achieved by including a protocol identification string in a header that is typically DKIM-signed, but is not attacker-controllable.
For example, the From: field allows arbitrary data in the "real name" part of an RFC822 email address, but cannot normally be altered by e.g. a mailto: link.
We could also require that the date is duplicated into one or more signed headers, since it is unlikely that an attacker could control the creation of a timestamp to 1-second resolution.
In an extreme case, we could require that the challenge is contained in every signed email header; postfix quite happily signed and delivered the following test email, for example:

```
DKIM-Signature: v=1; a=rsa-sha256; c=simple/simple; d=andrewg.com;
	s=andrewg-com; t=1738347605;
	bh=frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=;
	h=From:To:Subject:Date:From;
	b=dkrUtwdFDc5kzSjycuN++1F23EnTG37qIQ244AVj6OF1u9iP7iJmiAAEkPJFtTROx
	 MGEciAZiIGnppRSUp1ncRvrT8hW646DYoyjfx5n4Ar43xZIpI/mrU6qiMiP+qge8hK
	 HzNx+ntBpWICDLZDAsgJup+lkazE5DZ32FNcUHRnWbvnmNiRdFUEcbotRA/6WJkNF6
	 x9y2Eg8Qv0y4XeF9Mkd9GGWYA+BwNklvUYhsgyaGHXVB4tI6y5nZqjciEdNc2Q9FVE
	 17zeBSXeY5TkqrW4siVLAJJDkVndnNr0h3Nr88jDlqUUhBDA0EDBt2bbTtAKc4EdA1
	 hqbTPG7GM5/Iw==
Received: from localhost (localhost [127.0.0.1])
	by fum.andrewg.com (Postfix) with ESMTP id B65FD5DC93
	for <andrewg@andrewg.com>; Fri, 31 Jan 2025 18:19:20 +0000 (UTC)
From: frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY= <andrewg@andrewg.com>
To: frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY= <andrewg@andrewg.com>
Subject: frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=
Date: frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=
Message-Id: frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=

```
