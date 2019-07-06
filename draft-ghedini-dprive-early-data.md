---
title: Using Early Data in DNS over TLS
docname: draft-ghedini-dprive-early-data-latest
abbrev: DNS Early Data
category: std

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping]

author:
 -
    ins: A. Ghedini
    name: Alessandro Ghedini
    org: Cloudflare, Inc.
    email: alessandro@cloudflare.com

--- abstract

This document illustrates the risks of using TLS 1.3 early data with DNS over
TLS, and specifies behaviors that can be adopted by clients and servers to
reduce those risks.

--- middle

# Introduction

TLS 1.3 {{?TLS13=RFC8446}} defines a mechanism, called 0-RTT session resumption
or early data, that allows clients to send data to servers in the first
round-trip of a resumed connection without having to wait for the TLS handshake
to complete.

This can be used to send DNS queries to DNS over TLS {{!DOT=RFC7858}} servers
without incurring in the cost of the additional round-trip required by the TLS
handshake. This can provide significant performance improvements in cases where
new DNS over TLS connections need to be established often such as on mobile
clients where the network might not be stable, or on resolvers where keeping an
open connection to many authoritative servers might not be practical.

However the use of early data allows an attacker to capture and replay the
encrypted DNS queries carried on the TLS connection. This can have unwanted
consequences and help in recovering information about those queries. While
{{!TLS13}} describes tecniques to reduce the likelihood of a replay attack,
they are not perfect and still leave some potential for exploitation.

# Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Early Data in DNS over TLS

Early data forms a single stream of data along with other application data,
meaning that one or more DNS queries can either be partially or fully contained
within early data. Once the TLS handshake has completed, the early data is known
to not be a replayed copy of that data, but this doesn't mean that it can't be
replayed, or that it hasn't already been replayed, in another connection.

A server can signal to clients whether it is willing to accept early data in
future connections by providing the "early_data" TLS extension as part of a TLS
session ticket, as well as limit the amount of early data it is willing to
accept using the "max_early_data_size" field of the "early_data" extension.

In addition to the mitigation mechanisms mandated in {{?TLS13}} that reduce the
ability of an attacker to replay early data, but may not completely eliminate
it, a server that decided to offer early data to clients MAY reject early data
at the TLS layer, or delay the processing of early data after the handshake is
completed.

If the server rejects early data at the TLS layer, a client MUST forget
information it optmisitically assumed about the onnection when sending early
data, such as the negotiated protocol {{?ALPN=RFC7301}}. Any DNS queries sent
in early data will need to be sent again, unless the client decides to abandon
them.

Not all types of DNS queries are safe to be sent as early data. Clients MUST
NOT use early data to send DNS Updates ({{?RFC2136}}) or Zone Transfers
({{?RFC5936}}) messages. Servers receiving any of those messages MUST reply with
a "FormErr" response code.

[[TODO: forbid other types? use a different status code? should we define a
  whitelist instead of a blacklist?]]

# Security Considerations

## Information Exposure

By replaying DNS queries that were captured when transmitted over early data,
an attacker might be able to expose information about those queries, even if
encrypted.

For example, it's a common behavior for DNS servers to statefully rotate the
order of RRs when replying to DNS queries for an RRSet that contains multiple
RRs. If the order of rotation is predictable, replaying a captured early data
DNS query and observing the order of RRs in DNS responses before and after the
replayed query, might allow the attacker to confirm whether the query targeted
a specific name that was suspected of being queried.

Servers SHOULD either use fixed ordering for multiple RRs in the same DNS
response or shuffle the RRs at random, but MUST NOT use stateful and
deterministic ordering across multiple queries.

## Denial of Service

Accepting early data exposes a server to potential denial of service through
the replay of queries that might be expensive to handle.

When under load, a server MAY reject TLS early data such that the client is
forced to retry them after the handshake is completed.

## Privacy

TBD

[[TODO: linkability (e.g. clients changing network, ...) and more?]]

# IANA Considerations

This document has no actions for IANA.

# Acknowledgments

Thanks to Martin Thomson, Mark Nottingham and Willy Tarreau for writing
{{?RFC8470}} which heavily inspired this document, and to Daniel Kahn Gillmor
and Colm MacCárthaigh who also provided important ideas and contributions.
