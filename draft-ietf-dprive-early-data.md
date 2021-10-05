---
title: Using Early Data in DNS over TLS
docname: draft-ietf-dprive-early-data-latest
abbrev: DNS Early Data
category: std
workgroup: DPRIVE

ipr: trust200902

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping]

author:
 -
    ins: A. Ghedini
    name: Alessandro Ghedini
    org: Cloudflare, Inc.
    email: alessandro@cloudflare.com

--- abstract

This document illustrates the risks of using TLS 1.3 early data with encrypted
DNS, and specifies behaviors that can be adopted by clients and servers to
reduce those risks.

--- middle

# Introduction

TLS 1.3 {{?TLS13=RFC8446}} defines a mechanism, called 0-RTT session resumption
or early data, that allows clients to send data to servers in the first
round-trip of a resumed connection without having to wait for the TLS handshake
to complete.

This can be used to send DNS queries using DNS over TLS {{!DOT=RFC7858}} or
another encrypted transport {{?DOH=RFC8484}}{{?DOQ=I-D.draft-ietf-dprive-dnsoquic}}
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

# Early Data in Encrypted DNS {#body}

Early data forms a single stream of data along with other application data,
meaning that one or more DNS queries can either be partially or fully contained
within early data. Once the TLS handshake has completed, the early data is known
to not be a replayed copy of that data, but this doesn't mean that it can't be
replayed, or that it hasn't already been replayed, in another connection.

A server can signal to clients whether it is willing to accept early data in
future connections by providing the "early_data" TLS extension as part of a TLS
session ticket, as well as limit the amount of early data it is willing to
accept using the "max_early_data_size" field of the "early_data" extension.

The 0-RTT mechanism SHOULD NOT be used to send DNS requests that are
not "replayable" transactions. Our analysis so far shows that
such replayable transactions can only be QUERY requests,
although we may need to also consider NOTIFY requests once
the analysis of NOTIFY services is complete, see {{the-notify-service}}.

Servers MUST NOT execute non replayable transactions received in 0-RTT
data. Servers MUST adopt one of the following behaviors:

* Queue the offending transaction and only execute it after the handshake
has been completed, as discussed in {{Section E.5 of !TLS13}} and
{{Section 4.1.1 of !RFC9001}}.
* Reply to the offending transaction with a response code REFUSED and
an Extended DNS Error Code (EDE) "Too Early", see
{{reservation-of-extended-dns-error-code-too-early}}.
* Report a fatal protocol error in the DNS transport (e.g. DOQ_PROTOCOL_ERROR,
HTTP 425).

For zone transfers, it would be possible to replay an XFR QUERY
that had been sent in 0-RTT data. However the relevant authentication mechanisms
(described in {{Section 9 of ?RFC9103}}) will ensure that the response is not sent by
the primary until the identity of the secondary has been verified, i.e. the first
behavior listed above.

# Privacy Considerations

The 0-RTT data can be replayed by adversaries. That data may trigger queries by
a recursive resolver to authoritative resolvers. Adversaries may be able to
pick a time at which the recursive resolver outgoing traffic is observable, and
thus find out what name was queried for in the 0-RTT data.

This risk is in fact a subset of the general problem of observing the behavior
of the recursive resolver discussed in "DNS Privacy Considerations"
{{?RFC9076}}. The attack is partially mitigated by reducing the observability
of this traffic. However, the risk is amplified for 0-RTT data, because the
attacker might replay it at chosen times, several times.

{{Section 8 of ?TLS13}} requires that the capability to use 0-RTT
data should be turned off by default, and only enabled if the user clearly
understands the associated risks. In our case, allowing 0-RTT data
provides significant performance gains, and we are concerned that a
recommendation to not use it would simply be ignored.

The prevention on allowing replayable transactions in 0-RTT data
blocks the most obvious
risks of replay attacks, as it only allows for transactions that will
not change the long term state of the server.

Attacks trying to assess the state of the cache are more powerful if
the attacker can choose the time at which the 0-RTT data will be replayed.
Such attacks are blocked if the server enforces single-use tickets, or
if the server implements a combination of ClientHello
recording and freshness checks, as specified in
{{Section 8 of ?TLS13}}. These blocking mechanisms
rely on shared state between all server instances in a server system. In
the case of encrypted DNS, the protection against replay attacks on the
DNS cache is achieved if this state is shared between all servers
that share the same DNS cache.

The attacks described above apply to the stub resolver to recursive
resolver scenario, but similar attacks might be envisaged in the
recursive resolver to authoritative resolver scenario, and the
same mitigations apply.

# Security Considerations

Accepting early data exposes a server to potential denial of service through
the replay of queries that might be expensive to handle.

When under load, a server MAY reject TLS early data such that the client is
forced to retry them after the handshake is completed.

# IANA Considerations

## Reservation of Extended DNS Error Code Too Early

IANA is requested to add the following value to
the Extended DNS Error Codes registry {{!RFC8914}}:

* INFO-CODE: TBD
* Purpose: Too Early
* Reference: This document

--- back

# Acknowledgments

Thanks to Martin Thomson, Mark Nottingham and Willy Tarreau for writing
{{?RFC8470}} which heavily inspired this document, and to Daniel Kahn Gillmor
and Colm MacCárthaigh who also provided important ideas and contributions.

# The NOTIFY service

This appendix discusses the issue of allowing NOTIFY to be sent in 0-RTT data.

{{body}} says "The 0-RTT mechanism SHOULD NOT be
used to send DNS requests that are not "replayable" transactions", and suggests
this is limited to OPCODE QUERY. It might also be viable to propose that NOTIFY
should be permitted in 0-RTT data because although it technically changes the
state of the receiving server, the effect of replaying NOTIFYs has negligible
impact in practice.

NOTIFY messages prompt a secondary to either send an SOA query or an XFR
request to the primary on the basis that a newer version of the zone is
available. It has long been recognized that NOTIFYs can be forged and, in
theory, used to cause a secondary to send repeated unnecessary requests to the
primary. For this reason, most implementations have some form of throttling of the
SOA/XFR queries triggered by the receipt of one or more NOTIFYs.

{{?RFC9103}} describes the privacy risks associated with both NOTIFY and SOA queries
and does not include addressing those risks within the scope of encrypting zone
transfers. Given this, the privacy benefit of using encrypted transport for NOTIFY is not clear -
but for the same reason, sending NOTIFY as 0-RTT data has no privacy risk above
that of sending it using cleartext DNS.
