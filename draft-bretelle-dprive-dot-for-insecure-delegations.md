---
title: DNS-over-TLS for insecure delegations
abbrev: dot-for-insecure-delegations
docname: draft-bretelle-dprive-dot-for-insecure-delegations-latest
category: std

stand_alone: true

ipr: trust200902
area: Internet
workgroup: Network Working Group
kw: Internet-Draft

pi: [toc, sortrefs, symrefs]

author:
 -
    ins: M. Bretelle
    name: Emmanuel Bretelle
    organization: Facebook
    email: chantra@fb.com

normative:
  RFC2119:
  RFC4033:
  RFC4034:
  RFC4035:
  RFC4956:
  RFC5280:
  RFC6125:
  RFC6698:
  RFC7250:
  RFC7858:
  RFC8174:
  I-D.bortzmeyer-dprive-resolver-to-auth:

--- abstract

This document describes an alternative mechanism to DANE ({{RFC6698}}) in order
to authenticate a DNS-over-TLS (DoT {{RFC7858}}) authoritative server by not
making DNSSEC a hard requirement, making DoT server authentication available for
insecure delegations.

--- middle

# Introduction

This document describes an alternative mechanism to DANE ({{RFC6698}}) as
described in {{I-D.bortzmeyer-dprive-resolver-to-auth}} Section 2 extending the
authentication of DNS over Transport Layer Security (DoT) {{RFC7858}} to
insecure delegations and therefore enabling the onboarding of DoT authoritative
servers without the requirement for the authorities to support DNSSEC
({{RFC4033}}, {{RFC4034}}, and {{RFC4035}}).
To do so, this document introduce the Delegation Subject Public Key Info
(DSPKI) resource record, its purpose, usage and format.

# Terminology

A server that supports DNS-over-TLS is called a "DoT server" to differentiate
it from a "DNS Server" (one that provides DNS service over any other protocol),
likewise, a client that supports this protocol is called a "DoT client"

A secure delegation ({{RFC4956}} Section 2) is a signed name containing a
delegation (NS RRset), and a signed DS RRset, signifying a delegation to a
signed zone.

An insecure delegation ({{RFC4956}} Section 2) is a signed name containing a
delegation (NS RRset), but lacking a DS RRset, signifying a delegation to an
unsigned subzone.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY",
and "OPTIONAL" in this document are to be interpreted as described in
BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they
appear in all capitals, as shown here.

# Authenticating an insecure delegation

To authenticate a DoT server of a secure delegation, it is possible to use the
TLSA resource record {{RFC6698}} of the nameserver as decribed in
{{I-D.bortzmeyer-dprive-resolver-to-auth}} Section 2, while this method
is valid, the absence of support of DNSSEC for such delegations precludes the
onboarding and discovery of nameservers serving those zones as DoT servers.

Without the use of DNSSEC, a delegation is not able to authenticate itself as
the chain of trust cannot be followed, however other mechanisms exist to have a
server authenticate itself, such as Public Key Infrastructure
(PKIX {{RFC6125}}) , SPKI, which have their own pros and cons.

## Public Key Infrastructure (PKIX)

It would be possible to authenticate the name servers of the insecure delegation
using PKIX, relying on an existing trust model and trust anchors.

While simple, a single trusted Certificate Authority (CA) that breaks said trust (voluntarily or involuntarily), can issue certificate for any domains, allowing
an attacker to potentially impersonate both the application and the DoT server.

Another issue that rises is that the DoT servers may use an identity which belong
to the same origin as application servers, which could permit personal
information (such as cookies) to be leaked to the DoT servers.


## Subject Public Key Info (SPKI)

The zone owner generates his own certificate and distribute the SPKI fingerprint
into the DNS.

This is in essence what, amongst other things, TLSA records solve but with
the requirement for DNSSEC to be enabled and functional for the queried zone.
For insecure delegations, simply advertising the SPKI fingerprint would be
trivial to intercept, disable, and modify.

## Authenticating from the parent

While a delegation is not secured, the DNS core infrastructure already support,
for the most part, DNSSEC, meaning that if the owner of an insecure delegation
could set the SPKI fingerprint in a resource record (RR) at the parent, such
fingerprint could be signed and validated by the DoT client.
The DoT client can then establish a TLS connection to the zone name servers and
authenticate the DoT server against the fingerprint acquired earlier from the
parent zone.

### Example

example.com is an insecure delegation from .com which has set the DSPKI RRset.

A DoT client looking for records under example.com will learn from .com that
example.com is delegated to

~~~~
example.com IN 172800 NS ns1.example.com
example.com IN 172800 NS ns2.example.com
# sha256
example.com IN DSPKI (
    1 4e44f900cdeb8c769f4df97e23f8fc81
      4ac4bf45a3d9dc265a2ed925171f0b71 )
# sha512
example.com IN DSPKI (
    2 ab40ed300fd220d8c72a600069f9ceb1
      f9fd7c003117e4ef34b228da1c9d76a0
      500be99e82a0c01e7f80930a46ad28b8
      ed3d5ed2df34d822b5f56c99f45889ef
)
ns1.example.com IN 172800 AAAA 2001:db8:abcd:12:1:2:3:4
ns2.example.com IN 172800 AAAA 2001:db8:abcd:ab:1:2:3:4
~~~~

with the accompanying signature.

The DSPKI RRset signals that the nameservers are able to support DNS-over-TLS.
The DoT client can then establish a TLS connection to the DoT server and authenticate
them by ensuring that the SPKI matches the one learned from the parent zone.

# DSPKI Resource Record

There may be 0 or more DSPKI served by the parent of the delegation. 0 means that DSPKI is not supported, therefore the DoT client could try other
alternatives.
1 or multiple public keys can be distributed to let the DoT client validate
multiple public keys, which can be useful while doing certificate rotation or
when willing to provide different secret keys to different providers that
may serve the delegated zone.

## The DSPKI Resource Record

The DSPKI resource record (RR) is used to associate a DoT server public key
(SPKI) with the zone it is serving.

### DSPKI RDATA Wire Format

The RDATA of the DSPKI RR consists of a one-octet matching type field, and the
DER-encoded binary structure of the SubjectPublicKeyInfo field as defined in
{{RFC5280}}.


~~~~
                     1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2 2 3 3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| matching type | DER-encoded SPKI field                        /
+-+-+-+-+-+-+-+-+                                               /
/                                                               /
/                                                               /
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~


#### The Matching Type Field

A one-octet value, called "matching type", specifies how the SPKI is presented.
The types defined in this document are:

* 0 -- Exact match on SPKI
* 1 -- SHA-256 hash of SPKI
* 2 -- SHA-512 hash of SPKI


Where the SPKI can be extracted as follow:

~~~~
openssl x509 -in cert.pem -pubkey -noout | openssl pkey -pubin -outform der
~~~~

and the SHA-256 as:

~~~~
openssl x509 -in cert.pem -pubkey -noout | openssl pkey -pubin -outform der | \
    openssl dgst -sha256 -binary
~~~~

**FIXME**: consider

* alternate URI to support DoT (host, port, spki), DoH (host, port, URL template), DNS-over-QUIC... would rather be an ALTNS type of record
* CDSPKI a la CDS, CDNSKEY

# Security Considerations

TODO Security


# IANA Considerations

TODO: This document requires IANA actions (new RR type).



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
