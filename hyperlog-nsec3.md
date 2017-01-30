                <meta charset="utf-8" emacsmode="-*- markdown -*-">
                            **On the surprising connection between the magic
                            HyperLoglog and DNSSEC NSEC3**

bert.hubert@powerdns.com /
[@powerdns_bert](https://twitter.com/PowerDNS_Bert)

Last updated: 29th of January 2017

Introduction
===============================================================================
A long time ago I discovered (or likely, rediscovered) a trick to rapidly
determine the number of DNSSEC secured delegations in a zone. I thought this
was rather clever. 

A few years later, I finally had a need to learn about the magic of
HyperLogLog (as championed by my friend & [Open-Xchange coworker Neil
Cook](https://blog.open-xchange.com/author/neil/)).  And behold,
I found that my DNSSEC trick and HyperLogLog were mostly the same thing!

Before we go on, I want to thank [Peter van Dijk](http://7bits.nl/blog/pages/about) for running tests that
showed that it wasn't the .NL zone that was wrong, but my understanding of
HyperLogLog!

HyperLogLog
----------- 
The challenge: determine how many unique entries you have in a list with
billions of entries.  You need this for example if you have user cookies of
site visitors, and you want to know how many unique visitors there were in a
day.  This is a common problem in the advertising industry.

The naive way is to sort all cookies and count how many are unique. This may
require terabytes of memory though, even if we only store hashes of the
cookies for example. It might also take you &gt;24 hours to study the
traffic of one day!

A very special trick called HyperLogLog can give a very good estimate of the
number of unique things in your list.. using 1280 bytes of memory.

So how does it work?
---------------------
A lot of [fine words are written on
HyperLogLog](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf), and
this is justified. It is a magic thing, and some math is involved to prove that it actually
works.  But conceptually it is not that difficult.

Here goes.  Let's say that we have around 20 entries in our list, of which
10 are unique.  Let's also say we have a fast non-cryptographic 32 bit hash
function which emits one of roughly 4.3 billion values for each entry, and
we plot these hashes on a bar.  We'd find the hashes of our 20 entries are
distributed randomly:

 **********************************************************************************
 *   +----------------------------------------------------------------------+     *
 *   |      * *  *       *            *   *             *     *     *     * |     *
 *   +----------------------------------------------------------------------+     *
 *   0                1                2                3                4  4.3   *
 *                                  billion                                       *
 **********************************************************************************

So what is the average distance between the hashes if we have $N$ unique entries? 
Given sufficient entries, we can be sure that this is very close to $2^{32}/N$. 
In other words, if we know the average distance between two hashes, we know
the number of unique entries.

This insight is not particularly helpful though - to know this average
distance, we still have to keep track of where all hashes are, and there
might be billions. This doesn't really solve our problem - we still need
$2^{32}/8$ bytes of memory for a bitmap of all hashes.

This however is where statistics come to our rescue. It turns out we don't
need to know the distance between two actual hashes. We simply need to
pick a point and see how close the closest hash is. Conventionally, this point
is at 0. 

**In other words, the lowest hash value on its own gives us information
about the number of unique values in a list**.

In the bar above, if we find the first hash at around 430 million
($\pm2^{32}/10$), this tells us it is likely there are around 10 unique
entries in our list.

In this way, we only have to keep track of the lowest hash we have ever
seen, and we know the number of unique entries. Neat eh?

So why does this work? Intuitively, you can see that if you have only a few
unique hashes, it is pretty unlikely one of them is close to the chosen
point of 0:


 **********************************************************************************
 *   +----------------------------------------------------------------------+     *
 *   |           *                        *                         *       |     *
 *   +----------------------------------------------------------------------+     *
 *   0                1                2                3                4  4.3   *
 *                                  billion                                       *
 **********************************************************************************

But if we have a ton of points, it is far more likely that a lower hash will
appear:

 **********************************************************************************
 *   +----------------------------------------------------------------------+     *
 *   | * *  * *  *   *   *    *  *   **   **  *  *  *** *  *  *  *  *  * ** |     *
 *   +----------------------------------------------------------------------+     *
 *   0                1                2                3                4  4.3   *
 *                                  billion                                       *
 **********************************************************************************


Now - there is a pretty large element of chance in this estimate, so actual
HyperLogLog doesn't look at just 1 hash value, but at many of them.  If you
expect up to 1 billion unique values, one could split up the $2^{32}$ hash
space in 2048 subspaces, and only keep track of the lowest hash value in
each of them.

It also turns out that for this to work, we don't even need to store the
actual lowest hash we found - it suffices to store the leading number of
zeroes in binary representation. In other words, for the 2048 subspaces
example above, if we have two hashes with binary presentations

* `10000101111 01101001011100010101`
* `11111101001 00010110000100010101`

This is what this would look like, where we actually only store the last
column:


Bin (11 bits) |  Lowest hash (21 bits) | Leading zeroes
--------------|------------------------|--------
....          |   ....                 | ....
`10000101111` | `01101001011100010101` | `1`
`11111101001` | `00010110000100010101` | `3`
....          |   ....                 | ....


To get the number of unique entries, determine the *harmonic mean* of all
those lowest hash values, and out comes a pretty precise estimate ($\pm2\%$)
of how many unique values there are in your list. 

In this configuration, the whole HyperLogLog structure requires 5 bits of
storage for each of the 2048 bins (enough to store the maximum of
$32-log_2(2048)=21$ leading zeroes), for a total of 1280 bytes.  To keep
track of up to a billion unique entries!

Note: This is not quite the entire story.  The HLL paper contains correction
factors that make all this slightly more complicated, but also more
correct. Work by [Google and the ETH
Zurich](https://stefanheule.com/papers/edbt13-hyperloglog.pdf) offers even
more precision.

Relation to DNSSEC &amp; NSEC3
==============================
So what does all this have to do with DNSSEC? DNSSEC signs DNS answers
cryptographically. A problem is that if DNS needs to say 'no such record',
it says this with an empty answer. So the question 'What is the IPv6 address
of www.horselesscarriage.com' gets the answer '' to signify there is no IPv6
address. And sadly, we can't usefully sign an empty string - since that
would give a universally valid 'no such record' answer that could be
replayed to make any domain disappear. Sad!

The initial DNSSEC solution for this problem was to sign the *interval*
between non-existent domain names. So for example, a query for
'www.nosuchdomain123.nl' to the .NL servers might get as answer 'nothing known
between nosuchdomain.nl and nosuchdomain321.nl'. And this could be signed
without the risk of replay attacks being used to silence domains that did
exist. Clever.

However, it was quickly pointed out that such ranges of 'empty space'
inherently also document which domain names actually DO exist. In other
words, the full contents of a DNSSEC signed zone would be available for
enumeration. Many European operators did not want this.

So, of course the DNSSEC community resorted to what always saves us: a hash,
more precisely a salted and iterated hash. This did not really solve the
problem, since it is quite easy to calculate billions of hashes and 'walk a
zone' anyhow, but this is what we ended up with (for now).

So what does all of this have to do with HyperLogLog?  Let us send a query
to my favorite zone, the Dutch .NL Top Level Domain:

```
$ dig +dnssec +norecurs www.nosuchdomain234.nl @ns1.dns.nl
;; ->>HEADER&lt;&lt;- opcode: QUERY, status: NXDOMAIN, id: 58311
;; QUESTION SECTION:
;www.nosuchdomain234.nl.		IN	A


;; AUTHORITY SECTION:
nl. IN SOA ns1.dns.nl. hostmaster.domain-registry.nl. 2017012321 3600 600 2419200 600
qauju5pk2k5krcucblgr1l6mkvqfifbd.nl. IN NSEC3 1 1 5 A1..B1 QAUK0QHQ5P430LN1A171K6VUKGHR1UFB NS SOA TXT RRSIG DNSKEY NSEC3PARAM
a4hm8068md36206d7ga8tjin9oi4bmo8.nl. IN NSEC3 1 1 5 A1..B1 A4HNAJIO9T91U4NVVLK3RG2LHF2HGQQK NS DS RRSIG
bqhhp9r6o9uifoi974teka9hithd9tdo.nl. IN NSEC3 1 1 5 A1..B1 BQHIVE08OU9VP6JM5MM6BR0LF1TDJMT0 NS DS RRSIG
```

This answer tells us that 'nosuchdomain234.nl' doesn't exist, and then goes
on to prove that by providing three 'NSEC3' ranges of hashes that do not have
DNSSEC data. 

If we zoom in to one of these:

```
bqhhp9r6o9uifoi974teka9hithd9tdo.nl. IN NSEC3 1 1 5 A1..B1 BQHIVE08OU9VP6JM5MM6BR0LF1TDJMT0 NS DS RRSIG
```
This tells us the salt of the hash (`A10222AECD6609B1`), some other DNSSEC
parameters, but most importantly: between base-32 encoded values
`bqhhp9r6o9uifoi974teka9hithd9tdo` and `BQHIVE08OU9VP6JM5MM6BR0LF1TDJMT0`,
there is nothing signed. 

Note that these values are hashes, and they are therefore distributed
randomly over the available hash space. And the distance between the hashes
is, as with HyperLogLog, telling for the number of unique names in there.

To perform the HLL algorithm, we calculate the distance between the hashes,
and determine the number of leading zeroes of this number. 

In this specific case, if we look at the first 64 bits in hex, this gets us:

$5ea32fb808c793fc - 5ea31ca766c27d27 = 1310a20516d5$

In binary, this difference is:

$0000000000000000000100110001000010100010000001010001011011010101$

Or, 20 leading zeroes. For reasons explained in the HyperLogLog paper, we
can add 1 to that, leading to an estimate of the .NL zone size of $2^{21}$
which is $2.09$ million. At the time of measurement, the actual number was
$2.57$ million. Not bad!

Experimentally, if thousands of NSEC3 hash values are considered,
their distances quickly provide an estimate of the number of DNSSEC
delegation in the .NL zone that turns out to be accurate to 1%. 

Determining number of signed delegations by eye
-----------------------------------------------
As a party trick, this estimation can be performed by eye - if a zone
typically has 3 to 4 overlapping first characters in NSEC3 hashes, this
corresponds to at least 15 to 20 bits of overlap (because of base32).  Add 1
and you can quickly state a zone likely has 'in the order of a million
signed delegations'.

To further impress your friends, see how often you spot 4 overlapping base32
digits and round up accordingly for more precision. 

<!-- Markdeep: --><style class="fallback">body{visibility:hidden;white-space:pre;font-family:monospace}</style><script src="markdeep.min.js"></script><script src="https://casual-effects.com/markdeep/latest/markdeep.min.js"></script><script>window.alreadyProcessedMarkdeep||(document.body.style.visibility="visible")</script>
