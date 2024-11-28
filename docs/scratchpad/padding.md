Conformal padding for OpenPGP keys
==================================

This document proposes a conformal padding scheme for OpenPGP keystore lookups.
It buckets keys into the same anonymity classes regardless of whether binary or ASCII padding is used, and further preserves that anonymity class when transformed between line-ending formats.

Binary padding (WKD)
--------------------

WKD serves keys in binary format, and is commonly implemented by serving individual static files.
Padding can therefore be applied either on submission, or using a webserver plugin.
The former technique is easier to implement, at the expense of storage inefficiency.
This is probably an acceptable tradeoff for a small domain but not for a large one or an aggregator such as KOO.

It is difficult to mitigate against information leakage made by WKD requests, particularly for small domains.
Note also that a TLS-encrypted email sent to a domain's MX server leaks a similar amount of information to an eavesdropper as a WKD request.
There is therefore little marginal utility in implementing padding for small (static) WKD servers.

We therefore assume that padding is implemented at request time by the server application.

Binary keys are padded by appending a padding packet of the appropriate length and tag 21.
This is only supported by clients that implement the crypto-refresh draft, and so MUST only be used when all keys in the response are v6 or later.

Alternatively, the entire response can be padded by appending a fake (malformed) subkey packet of the appropriate length and a [GREASEy version number](grease.html).
This is safe IFF the client gracefully ignores malformed subkey packets.

An OpenPGP padding packet has a minimum length of two bytes: one byte for the packet tag, and at least one byte for the body length (which can be zero).
Padding packet lengths between p=2 and p=193 bytes consist of a two-byte header and a body of length `p - 2`.
Padding packet lengths of p=194 bytes and above consist of a six-byte header and a body of length `p - 6`.
Note that padding lengths of 194-8389 bytes require an overlong (five-byte) encoding of 188-8383 byte body lengths.
This allows us to generate packets with total lengths (including headers) that are not representable using one- or two-byte length encodings (194 and 8387-8389).

On OpenPGP public (sub)key packet has a minimum length of three bytes: one byte for the packet tag, at least one byte for the body length, and one byte for the key version.
Aside from the packet tag and key version byte, a fake public (sub)key packet is identical to a padding packet of the same length.

We ensure that padding or fake key packets meet the minimum length requirement by always over-padding to three bytes beyond the target padded length.
If the unpadded length is `m` bytes and the target padded length is `M` bytes, the amount of padding bytes is `M-m+3` and the actual length of the padded payload is `M+3`.
This over-padding also ensures that a binary-padded-then-armored key is always the same length as an armored-then-ASCII-padded key (see next section). 

ASCII padding (HKPS)
--------------------

HKPS serves keys in ASCII-armored format.
In the general case, armored data MAY consist of lines up to 76 printable characters and MAY also contain extra headers and a checksum.
If padding is enabled, extra headers and checksums MUST NOT be added, and the Base64 data MUST be line-broken at exactly 64 characters.
This ensures consistency of message length after newline characters are included.

An HKPS server MAY pad the binary key block before ASCII-armoring, in which case it SHOULD use the same technique as described for WKS above.
Otherwise, armored responses SHOULD be padded by prepending the required quantity of Base64 printable characters and newlines.
Only the length of the encoded content should be considered when calculating the ASCII padding length.

We recall that ASCII armored data always comes in a whole number of padded "chunks", each containing four printable characters and representing up to three bytes of data.

A binary-padded key is then represented as an armored block of `J = M/3 + e = (L-1)*16 + 1` chunks and `L` newlines, where
* `L = M/48 + 1` is the number of lines,
* `e = 1` is the number of excess chunks due to binary over-padding,
* `M` is the target message length in bytes.

An unpadded key is represented as an armored block of `j = ceil(m/3) = (l-1)*16 + i` chunks and `l` newlines, where
* `l = ceil(m/48)` is the number of lines,
* `i = ceil(m/3) - (l-1)*16` is the number of Base64 chunks on the last line (`1 <= i <= 16`),
* `m` is the unpadded key length in bytes.

The shortfall of chunks "missing" from an unpadded armored block due to lack of binary padding is `J - j = M/3 + e - ceil(m/3)`,
and the corresponding shortfall of newlines is `L - l = M/48 + 1 - ceil(m/48)`.

### Padding-string splitting

We wish to ensure that a padding string of length equal to the shortfall of chunks also compensates for the shortfall of newlines.

For a newline-terminated string of length `c` split across lines of length `C`, the total number of newlines required is `ceil(c/C)`.
Let us split a padding string of `M/3 + e - ceil(m/3)` chunks across lines of length 64 characters (=16 chunks), where `e` is the number of chunks in our over-padding.
The number of newlines `k` automatically added to the string is therefore:

    k = ceil( (M/3 + e - ceil(m/3)) / 16 )

IFF M is divisible by 48, we can bring it outside:

    k = M/48 + ceil( (e - ceil(m/3)) / 16 )

Since `ceil(-n) = -floor(n)`:

    k = M/48 - floor( (ceil(m/3) - e) / 16 )

Now IFF `e = 1` we can apply the identity `floor( (n-1) / m ) = ceil(n/m) - 1` to get:

    k = M/48 + 1 - ceil( ceil(m/3) / 16 )
    k = M/48 + 1 - ceil(m/48)

which equals the shortfall of newlines `L - l` idenified above.

This means that IFF both M is divisible by 48, and we over-pad by exactly one excess chunk (corresponding to the three bytes of binary over-padding),
the act of splitting a padding string of the required length into 64-character lines exactly compensates for the shortfall of newlines.

### ASCII padding construction

A padding string consists of:
* the 4-character prefix `PAD=`,
* followed by `(M/3 - ceil(m/3))*4` additional random Base64 characters,
* line-wrapped at the 64th column,
* and newline-terminated.

This is output before the armored key, to force clients to consume all of the padding and thereby prevent early connection drops.
A padding string MUST always be present if padding is enabled.

The prefix and line-wrapping requirements ensure that the combined padding and armor always contains a constant number of printable characters AND a constant number of newlines.
As a consequence, the anonymity cohort survives transformation between newline formats, and between binary-padded and ASCII-padded format.

Padding Contents
----------------

The padding SHOULD be a non-compressible octet stream.
It MAY be generated deterministically, for example using an iterated modular exponentiation over the last N octets of the data.
See [Verifiable Random Functions](vrf.html) for further details.


Other considerations
====================

Backwards compatibility
-----------------------

The original HKP draft prohibited extra response data, such as padding, in machine-readable output.
Therefore, padding MUST NOT be used when responding to legacy machine-readable requests.

Index search padding
--------------------

HKPS index searches will return a similar number of entries as a direct lookup request, however these entries will be summaries of the matching keys rather than the keys themselves.
This summary will normally include UserIDs and may also list subkey and signature packets, and will include a separate entry for each of the matching keys.
The size of an index response will therefore leak significant information about the nature of the query.

A human-readable index is verbose, but can be easily padded by adding an arbitrary amount of HTML commentary to the source of the page.
Since this page can often be tens of kilobytes long, buckets of 2^n SHOULD be used.
Variable-sized embedded content such as UAT images SHOULD NOT be present on a padded, human-readable index page.
Note that human-readable index searches are typically performed manually in a browser rather than automatically, and so the benefits of padding may not be significant.

A machine-readable index has a smaller maximum size, typically 100 keys (hockeypuck), corresponding to a few kilobytes of content.
This SHOULD be padded to the nearest 1024 bytes using `X-Padding:` headers of the desired total size.

Client request padding
----------------------

When making machine readable HKP requests, clients SHOULD pad their requests to a standard length.
Since the length and range of query strings is significantly smaller than that of responses, it is sufficient to pad to a multiple of 64 bytes.
This can be done by adding a single HTTP request header `X-Padding:`, with a value between 1 and 65 bytes long.

---

Examples
========

Let us consider a toy model, where the bucket length is 96 bytes (`48*2`), and the byte values of the "data" and "padding" are faked for visual clarity.

Consider an unpadded key of binary length 86 bytes, armored:

```
-----BEGIN PGP PUBLIC KEY BLOCK-----

decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbad
decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddec=
-----END PGP PUBLIC KEY BLOCK-----
```

When binary-padded to 96+3 bytes before armoring, it is 214 bytes long over 6 lines.
Note that the three bytes of binary over-padding cause the encoded data to wrap onto a third line:

```
-----BEGIN PGP PUBLIC KEY BLOCK-----

decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbad
decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecAAAAAAAAAAAAA
AAAA
-----END PGP PUBLIC KEY BLOCK-----
```

When first armored and then ASCII-padded with a `(96/3 - ceil(86/3) + 1)*4 = 16` character padding string, it is also 214 bytes long over 6 lines:

```
PAD=DECAFBADDECA
-----BEGIN PGP PUBLIC KEY BLOCK-----

decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbad
decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddec=
-----END PGP PUBLIC KEY BLOCK-----
```

Now consider keys of other lengths - for comparison we show both the binary-padded and then ASCII-padded armored keys.

If the key is exactly 96 bytes, we still add the minimal `(96/3 - ceil(96/3) + 1)*4 = 4` character padding string `PAD=` together with its CRLF:

```
-----BEGIN PGP PUBLIC KEY BLOCK-----

decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbad
decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbad
AAAA
-----END PGP PUBLIC KEY BLOCK-----

PAD=
-----BEGIN PGP PUBLIC KEY BLOCK-----

decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbad
decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbad
-----END PGP PUBLIC KEY BLOCK-----
```

If the key is exactly N lines shorter than the bucket, the padding string wraps onto the (N+1)th line, compensating for the "missing" CRLF.
This 48-byte key requires `(96/3 - ceil(48/3) + 1)*4 = 68` characters of padding:

```
-----BEGIN PGP PUBLIC KEY BLOCK-----

decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbad
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAA
-----END PGP PUBLIC KEY BLOCK-----

PAD=DECAFBADDECAFBADDECAFBADDECAFBADDECAFBADDECAFBADDECAFBADDECA
FBAD
-----BEGIN PGP PUBLIC KEY BLOCK-----

decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbad
-----END PGP PUBLIC KEY BLOCK-----
```

If the armored key is one chunk longer than an integer number of lines, the padding string doesn't wrap, avoiding an off-by-one error.
This 50-byte key requires `(96/3 - ceil(50/3) + 1)*4 = 64` characters of padding:

```
-----BEGIN PGP PUBLIC KEY BLOCK-----

decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbad
decAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAA
-----END PGP PUBLIC KEY BLOCK-----

PAD=DECAFBADDECAFBADDECAFBADDECAFBADDECAFBADDECAFBADDECAFBADDECA
-----BEGIN PGP PUBLIC KEY BLOCK-----

decafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbaddecafbad
dec=
-----END PGP PUBLIC KEY BLOCK-----
```

Note that in each padded example above there are always 214 bytes over 6 lines.
This is a conserved property of a 96-byte bucket.


Rationale
=========

Take a sample of 1 million keys from the SKS dataset on pgpkeys.eu and log their lengths (as stored in the JSON doc).
Generate a histogram of how many keys exist with each length, and bucket them by percentile.

```
SQLCMD='docker-compose exec -T postgres psql hkp -U hkp'
$SQLCMD -t -c "select doc from keys limit 1000000;" | jq .length >/tmp/lengths
sort -n /tmp/lengths >/tmp/lengths.sorted
uniq -c /tmp/lengths.sorted |awk '{print $2 "," $1}' > /tmp/lengths.stats.csv
ncum=0; nlim=10000; while read -r line; do len="${line%,*}"; n="${line#*,}"; (( ncum+=n )); while (( ncum >= nlim )); do echo "$nlim,$len"; (( nlim+=10000 )); done; done < /tmp/lengths.stats.csv > /tmp/lengths.buckets.csv
```

(( [lengths.stats.csv](lengths.stats.csv) is included in this repo for reference ))

Note that the histogram is far from smooth - there are significant clusters around certain lengths, together with prominent isolate spikes (most notably at 1232, 1280, 1292 and 1306 bytes), and a sparse tail of oversized keys with unique lengths.
The largest peloton of lengths, between roughly 1110 and 1234 bytes, consists mainly of rsa2048 primary keys each with one rsa2048 encryption subkey and a single email UserID, a common default key configuration.
The isolate spikes correspond to known spam campaigns against prominent UserIDs, which consisted of large numbers of almost-identical keys.

Design considerations
---------------------

When we choose a padding scheme, our primary goal is to maximise each cohort of keys that are indistinguishable by length, and our secondary goal is to minimise the amount of padding to prevent waste.
The primary goal ("effectiveness") is trivially met by padding every key to the maximum possible length, so the cohort will always consist of all known keys.
The secondary goal ("efficiency") is trivially met by using no padding at all.

### Effectiveness

Let us define "bucket capacity" as the number of items in the bucket, and "bucket width" as the range of payload lengths covered by the bucket.

Most algorithmic padding schemes (e.g. power-of-2 or Padme), fail to protect the long tail of large keys.
Padme in particular is significantly worse at this than power-of-2, as its padding-to-payload ratio decreases asymptotically with message length.

We are therefore motivated to ensure that a minimum number of keys is included in any cohort, no matter how much padding may be required for long-tail keys.
This requires us to define a _minimum bucket capacity_, derived from the length histogram of our dataset.

We cannot however rely solely on cohorts defined using bucket capacity.
Such a scheme is vulnerable to spam campaigns and other automated processes that may produce statistical anomalies in the dataset.
A cohort that includes a large number of keys may not sufficiently disguise its members if a significant number of the keys in the cohort are known to be fake or automated.

To take an extreme example, we could choose "all keys of length 1306 bytes" as a cohort - this contains over a hundred thousand keys and so may sound sufficiently anonymized, but if we assume that the number of non-spam keys of length 1306 bytes is similar to the number of keys of length 1305 or 1307 bytes (which they should be if their UserID lengths are randomly distributed), then the real users that fall within the 1306B cohort are no more protected than if there was no anonymisation scheme at all.

We are therefore motivated to ensure that a significant range of key lengths are included in any cohort, to ensure a diverse sample.
This requires us to define a _minimum bucket width_ regardless of how many keys this may span.

### Efficiency

Since HTTPS negotiation and HTTP headers typically take up several hundred bytes, there is not much relative efficiency to be gained by economising bandwidth when the payload is less than 1KiB.
Bucket capacities of over 10% of the dataset add little marginal security, and should be optimised out where this does not conflict with minimum bucket widths.

A toy proposal: double bucketing
--------------------------------

If we use percentiles to define _minimum bucket capacity_, we ensure that passive observers can recover a maximum of 7 bits of information from the message length.
If we use the typical length of HTTP response headers to define _minimum bucket width_, we set a reasonable expectation of the padding length added to the majority of queries.

* first create primary cohorts by bucket capacity:
    * allocate each key length to a primary cohort
    * label each primary cohort with the maximum key length that it contains
* then create secondary cohorts by bucket width:
    * starting at the primary cohort with the longest max key length, and iterating downwards:
    * (A) allocate the current primary cohort to a new secondary cohort
    * label the secondary cohort with its maximum key length
    * consider the next primary cohort
        * if its maximum key length differs from the secondary cohort's maximum key length by less than the minimum bucket width, OR
        * if its maximum key length is less than the minimum bucket width, THEN
            * add it to the current secondary cohort and proceed to the next primary cohort
        * otherwise, repeat from (A)

We therefore produce a list of padding targets:

```
width=200; top=1048576; sort -t, -nr /tmp/lengths.buckets.csv | while read -r line; do len="${line#*,}"; if (( len < width)); then break; fi; if (( len+width < top )) ; then top=$len; echo $line; fi; done
1000000,905434
990000,9817
980000,6366
970000,5072
960000,4578
950000,4092
940000,3737
920000,3326
910000,2894
880000,2547
860000,2268
730000,1864
670000,1654
600000,1419
470000,1209
120000,978
60000,676
20000,371
```

About half of the cohorts are of the minimum bucket capacity and cover the largest 10% of the population, while the other 90% fall into narrow cohorts of <2 minimum bucket widths.
Note that there are very few keys of less than 256B in length, so the minimum padded message length should be at least 512B.

Bucketing strategy
------------------

It may be possible for an eavesdropper to observe a sequence of requests to multiple keystores (WKD and HKPS).
If different keystores apply different packet filter policies, the same key will in general have different lengths even after applying padding.
In addition, if each of these keystores performs its own cohort calculation, this increases the amount of information leaked to an eavesdropper when making multiple queries for the same key.
This incentivises us to maximise the capacity of our buckets and pin them down to algorithmic limits.

We should also ensure that if the same key material is returned from both WKD and HKPS, it will be bucketed into the same cohort.
This will be of particular concern for keystores that serve identical material over both protocols (e.g. keys.openpgp.org).
Since WKD returns binary keys and HKPS uses Base64, this can only be done reliably if the raw bucket lengths are exactly divisible by 48 and ASCII armor is consistently line-broken at 64 characters (see below for details).

This motivates us to apply a (`3*2^n`) bucketing scheme prioritising bucket width at small lengths, with a single large-capacity bucket to (inefficiently) cover the long tail:

    768     1536    3072    1048608
    6.5%    58.5%   25.5%   9.5%

`{1,3}*3*2^n` alternative bucket scheme:

    768     1152    1536    2304    3072    1048608
    6.5%    27.5%   31%     22%     5.5%    9.5%

(N.B. 1048608 is the next multiple of 48 above hockeypuck's default length limit of 1048576, however this can safely be set to any multiple of 48 above a server's configured limit)

If keys are padded individually but concatenated when being served, the length of the response may leak the number of results, and by extension the nature of the query.
We MUST therefore pad the entire response rather than individual keys.

### Response size tuning

The histogram of key sizes is a reasonable indicator of the histogram of fingerprint lookup responses, however other queries are possible.
These include queries for shortIDs, long IDs, and UserIDs that may return either an index or a sequence of keys.

* LongID lookups will typically return the same number of keys as a fingerprint search (zero or one), but this cannot be relied upon.
* ShortID lookups are deprecated and SHOULD NOT be supported.
* UserID lookups MAY return multiple keys.

WKD servers only provide UserID lookups, but may return more than one key in response.
HKPS servers will typically return a larger number of keys than WKD in response to a UserID lookup.
These will both tend to skew the combined histogram of response sizes upwards, compared to the histogram of key sizes.
It may therefore be desirable for a server to add further buckets between 3072 bytes and the maximum.
This could take the form of a tuning parameter, but this will have to be carefully tested to ensure that it doesn't result in unexpectedly low-capacity buckets.
