# Short-Lived Certificate Annotations in OpenPGP

In OpenPGP it is customary to make third-party certification signatures over User IDs to indicate the third-party keyholder's opinion about the UserID.
Third-party certifications are long-lived statements that form the basis of the Web of Trust.
It is often however useful to make short-lived statements independently of the WoT, and it would be further useful to be able to attach these to a certificate using a transport- and implementation-neutral format.

## Issues with third-party certifications as short-lived annotations

* Unlimited cumulation, leading to:
    * Keyholder approval required under 1pa3pc etc
* Keyservers required to hold secret key material

## Data we might wish to annotate:

* Foreign proofs, e.g. DKIM signatures
* Authentication signatures (submission only)
* Provenance logs
* Freshness tests
* Non-certificate user preferences (e.g. expect-signed)

## Prior art

* vCards
* Email and HTTP headers
    * Transport-specific, but could be defined compatibly
* Form data fields
    * Submission only
* Unhashed subpackets
    * In which signature?
    * May also cumulate

## Ideas

* Novel non-critical packet to contain annotations.
* Human-readable annotations could be formatted as email or HTTP headers, for compatibility.
* Binary annotations could be stored in raw format to save base64 encoding costs.
* Could instead be implemented as unhashed notation subpackets, which support both human readable and binary formats.
