---
title: Transport Layer Security (TLS) Resumption across Server Names
abbrev: TLS Cross-SNI Resumption
docname: draft-ietf-tls-cross-sni-resumption-latest
category: std

ipr: trust200902
area: Security
workgroup: TLS Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: V. Vasiliev
    name: Victor Vasiliev
    organization: Google
    email: vasilvv@google.com

normative:
  RFC2119:
  RFC8446:

informative:
  PERF:
    title: "Enhanced Performance for the encrypted Web through TLS Resumption across Hostnames"
    date: 2019-02-07
    author:
    -
      ins: E. Sy
      name: Erik Sy
    -
      ins: M. Moennich
      name: Moritz Moennich
    -
      ins: T. Mueller
      name: Tobias Mueller
    -
      ins: H. Federrath
      name: Hannes Federrath
    -
      ins: M. Fischer
      name: Mathias Fischer
    target: "https://arxiv.org/pdf/1902.02531.pdf"
  FETCH:
    target: https://fetch.spec.whatwg.org/
    title: "Fetch Standard"
    author:
      org: WHATWG
    date: Living Standard

--- abstract

This document specifies a way for the parties in the Transport Layer Security
(TLS) protocol to indicate that an individual session ticket can be used to
perform resumption even if the Server Name of the new connection does not match
the Server Name of the original.

--- middle

Introduction
============

Transport Layer Security protocol [RFC8446] allows the clients to use
an abbreviated handshake in cases where the client has previously established a
secure session with the same server.  This mechanism is known as "session
resumption", and its positive impact on performance makes it desirable to be
able to use it as frequently as possible.

Modern application-level protocols, HTTP in particular, often require accessing
multiple servers within a single workflow.  Since the identity of the server is
established through its certificate, in the ideal case, the resumption would be
possible to all of the domains for which the certificate is valid (see [PERF]
for a survey of potential practical impact of such approach).  TLS, starting
with version 1.3, defines the SNI value to be a property of an individual
connection that is not retained across sessions ([RFC8446], Section 4.2.11).
However, in the absence of additional signals, it discourages using a session
ticket when the SNI value does not match ([RFC8446], Section 4.6.1), as there
is normally no reason to assume that all servers sharing the same certificate
would also share the same session keys.  The extension defined in this document
allows the server to provide such a signal in-band.

Conventions and Definitions
===========================

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

The Flag
=============

Resumption across server names is negotiated using the TLS flags extension
{{!I-D.draft-ietf-tls-tlsflags}}.  The server MAY send a
resumption_across_names(8) flag in a NewSessionTicket message.  The flag is an
assertion by the server that any server for any identity presented in its
certificate would be capable of accepting that ticket.  A client receiving a
ticket with this flag MAY attempt resumption for any server name listed in the
server certificate even if the new SNI value does not match the one used in the
original session.

Security Considerations
=======================

This document does not alter any of the security requirements of [RFC8446], but
merely lifts a performance-motivated "SHOULD NOT" recommendation from Section
4.6.1.  Notably, it still relies on the client ensuring that the server
certificate is valid for the new SNI at the time of session resumption.

If the origianl server's assertion regarding supporting cross-name resumption
turns out to be incorrect, the server receiving a misdirected ticket
will not be able to decrypt it, and will therefore reject it.  This is secure,
as session resumption may be safely rejected for any reason; however, such
misconfiguration will waste tickets stored in the client's cache, as TLS
tickets may be single-use.

Cross-domain resumption implies that any certificate the client provides for
one host would become available to the other hosts using the same server
certificate.  Because of that, when performing cross-domain resumption, the
client MUST use the same policy on whether to present said certificate to the
server as if it were a new TLS session.  For instance, if the client would show
a certificate choice prompt for every individual domain it connects to, it MUST
show that prompt for the new host when performing cross-domain resumption.

Cross-domain resumption, like other similar mechanisms (e.g. cross-domain HTTP
connection reuse), can incentivize the server deployments to create server
certificates valid for a wider range of domains than they would otherwise.
However, any increase in the scope of a certificate comes at a cost: the wider
is the scope of the certificate, the wider is the impact of the key compromise
for that certificate.  In addition, creating a certificate that is valid for
multiple hostnames can lead to complications if some of those hostnames change
ownership, or otherwise require a different operational domain.

Session tickets can contain arbitrary information, and thus could be
potentially used to re-identify a user from a previous connection.
Cross-domain resumption expands the potential list of servers to which an
individual ticket could be presented.  Client applications should partition the
session cache between connections that are meant to be uncorrelated.  For
example, the Web use case uses network partition keys to separate cache lookups
[FETCH].

IANA Considerations
===================

IANA (will add/has added) the following entry to the "TLS Flags"
table of the "Transport Layer Security (TLS) Extensions" registry:

  Value
  : 0x8

  Flag Name
  : resumption_across_names

  Message
  : NST

  Recommended
  : N

  Reference
  : This document

--- back

Acknowledgments
===============
{:numbered="false"}

Cross-name resumption has been previously implemented in the QUIC Crypto
protocol as a preloaded list of hostnames.

Erik Sy has previously proposed a similar mechanism for TLS,
[draft-sy-tls-resumption-group](https://datatracker.ietf.org/doc/draft-sy-tls-resumption-group/).
This document incorporates ideas from that draft.

This document has benefited from contributions and suggestions from
David Benjamin,
Nick Harper,
David Schinazi,
Ryan Sleevi,
Ian Swett
and many others.
