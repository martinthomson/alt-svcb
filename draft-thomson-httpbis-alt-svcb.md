---
title: "HTTP Alternative Services, Plan B"
abbrev: "Alt-SvcB"
category: std
obsoletes: 7838

docname: draft-thomson-httpbis-alt-svcb-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "HTTP"
keyword:
 - next generation
 - alternative alternative
 - stickiness
venue:
  group: "HTTP"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "martinthomson/alt-svcb"
  latest: "https://martinthomson.github.io/alt-svcb/draft-thomson-httpbis-alt-svcb.html"

author:
 -
    fullname: Martin Thomson
    organization: Mozilla
    email: mt@lowentropy.net
 -
    fullname: Mike Bishop
    org: Akamai Technologies
    email: mbishop@evequefou.be
 -
    fullname: Lucas Pardue
    organization: Cloudflare
    email: lucaspardue.24.7@gmail.com
 -
    fullname: Tommy Jensen
    organization: Microsoft
    email: tojens@microsoft.com

normative:
  HTTP: RFC9110
  ORIGIN: RFC6454

informative:
  H2:
    =: RFC9113
    display: "HTTP/2"
  H3:
    =: RFC9114
    display: "HTTP/3"


--- abstract

HTTP alternative services

This document deprecates RFC 7838.

--- middle

# Introduction

HTTP alternative services provide an HTTP server with a means to direct requests
from clients to an alternative server instance.  Clients that learn about an
alternative service can establish a connection to the identified instance, which
- if successfully established and authenticated - can be used for future
requests.  This design allows for nearly seamless service continuity in some
conditions.

The use cases that might motivate a server to direct future requests to a
different server instance vary.

* Some deployments might seek to direct a client to a server instance that is
  better able to serve requests for that client.  This might occur if DNS
  resolution of the server name produces the address of a server instance that
  is further from the client.

* Servers that might seek to shed load, perhaps in anticipation of an imminent
  shutdown or maintenance action, might use an alternative service declaration
  to reduce either server load or the number of clients that might be affected.

* Many deployments of HTTP/3 {{H3}} use the protocol identifiers in an
  alternative service declaration to make clients aware of support for the newer
  protocol.

These different use cases can sometimes create awkward trade-offs for
deployments of alternative services.


## Caching in Alternative Services

With RFC 7838 {{!ALT-SVC=RFC7838}}, setting the "ma" (or max-age) parameter for
an alternative can be challenging for a server deployment.  Setting too a large
max-age can mean that clients use an alternative for longer than they should.
Conversely, a short cache period for an advertisement for HTTP/3 can mean that
earlier versions might be used more often than is optimal.

Alternative services can interact poorly with service configuration information
that is published in the DNS.  With the introduction of HTTPS records
{{?SVCB=I-D.ietf-dnsop-svcb-https}}, the availability of HTTP/3 can be
advertised in the DNS, creating two independent sources of this information,
with different approaches to caching.

Alternative services can also be highly dependent on networking conditions.  RFC
7838 attempted to manage this by having clients be responsible for invalidating
alternatives when changes in their network are detected, unless the alternative
is explicitly marked as "persistent".  In practice, detecting the necessary
changes is difficult for many clients, so this requirement is not consistently
implemented.

The alternative services mechanisms defined in RFC 7838 can produce suboptimal
or even detrimental outcomes in some deployments.  Consequently, this document
obsoletes RFC 7838.


## A New Alternative

This document describes a different approach to advertising alternative
services.  This approach uses the DNS as the singular source of information
about service reachability.  An alternative service advertisement only acts as a
prompt for clients to seek updated information from the DNS.

To use this new design, a server advertises an alternative name using the
"Alt-SvcB" field.

~~~ http-message
200 OK HTTP/1.1
date: Mon, 24 Oct 2022 02:58:31 GMT
alt-svcb: "instance31.example.com"
content-length: 0

~~~

Clients can then consult the DNS, making HTTPS queries {{SVCB}} starting with
this name. The alternative name is used in place of the name of the authority
and using HTTPS records is mandatory, but the process otherwise follows normal
HTTPS record resolution and connection procedures.  {{use}} defines how this
name is used in detail.

Future connections for requests to resources on the same server use HTTPS record
resolution to the name of the authority, but are reprioritized if a successful
connection was previously made to an alternative service.  {{reuse}} defines how
this process works in more detail.


## Conventions and Definitions

{::boilerplate bcp14-tagged}


# Using Alternative Services {#use}

When a client learns about a new potential alternative from a server, they
SHOULD attempt to use that alternative for future requests to that server.  The
client attempts to make a request for a resource on the same server using the
alternative as follows:

1. The client makes a DNS query for HTTPS records for the alternative name,
   following the procedures in {{Section 3 of !SVCB}}.  Clients make this query
   as a "SVCB-reliant" SVCB client, treating a failure to obtain HTTPS records
   as a failed alternative.  If this process fails to produce service parameters
   or IP addresses, abort this process. \[\[ISSUE: Maybe we can still be
   SVCB-optional here, but that places constraints on the choice of name.]]

2. The client establishes a connection using the service parameters and
   addresses learned from the DNS query.  Important here is that the client uses
   a TLS server name indication {{!SNI=RFC6066}} of the server name from the
   URL, not the alternative name.  This allows the server to produce a
   certificate for that name, which the client can validate as applying to the
   URL it is resolving.  If a connection cannot be established, abort this process.

3. The client validates that the server is authoritative for the resource.  If
   the server is not authoritative, abort this process.

4. The client makes a query for the resource.  If the server does not respond,
   responds with a 421 (Misdirected Request) (see {{Section 15.5.20 of HTTP}}),
   or a 5xx status code (see {{Section 15.6 of HTTP}}), abort this process.

5. Once a response is received, the connection to the alternative is complete.
   Any other connections can be closed and future requests directed to the
   alternative.  The client SHOULD remember the alternative name and the service
   name (the TargetName from the HTTPS ServiceMode record that was used) that
   were used; see {{remember}}.

A client MUST NOT remember a service name for an alternative service until a
request has been successfully completed with a 2xx or 3xx status code.  A client
MAY send additional requests using the newly established connection to the
alternative service after it verifies that the server is authoritative.  The
alternative service is therefore active once the connection is established, but
it will not be reused ({{reuse}}) for future connections until a request
completes successfully.

A client MAY continue sending other requests over any existing connection to the
server until this process completes in order to minimize latency for those
requests.  A client MAY - when presented with an alternative name - proactively
make a request for an arbitrary resource on the server, rather than waiting for
the next time a request is needed.  This might allow the connection to be
available for future requests with less delay.


## Retention of Alternatives {#remember}

Clients SHOULD remember the successful use of an alternative service.  Two
pieces of information are retained:

* the alternative name, which is the name provided by the server in the Alt-SvcB
  field or ALTSVCB frame, and

* the service name, which is the TargetName from the ServiceMode HTTPS record
  that was used to successfully connect to the server.

These two names are saved for the server against the origin of the server
{{ORIGIN}}.  Clients MUST NOT reuse saved information for a server with
a different hostname, port, or scheme.

The name given in an Alt-SvcB field or ALTSVCB frame is retained only so that
the client can avoid initiating a connection to an alternative if it has already
made an attempt.  Any time that a server provides a different name in an
Alt-SvcB field or ALTSVCB frame, any existing information MUST be discarded.  A
client MAY then initiate a DNS query and connection attempt to the identified
alternative.  A client can subsequently ignore repeated fields or frames.

A server might provide an Alt-SvcB field in all responses it sends.  Only acting
on a value once ensures that this repetition has no effect on clients.  Though a
server might repeat the field, clients MUST NOT consider an absent field as
indicative of a retraction of a previous advertisement.  An alternative name is
only removed when explicitly replaced or when a remembered service name does not
appear in the set of HTTPS query responses.

On the first attempt to use an alternative, a failed alternative SHOULD be
remembered using an alternative name and a null or absent service name.  This
avoids making repeated attempts to use an alternative service that is not
available if the server repeats the information in Alt-SvcB fields or ALTSVCB
frames.  A client MAY periodically attempt to retry a failed alternative if the
information is repeated.

A server can explicitly request that a client remove any remembered service name
by providing an alternative name of "invalid".  The "invalid" domain name
corresponds to a DNS name that will never successfully resolve (see {{Section
6.4 of ?SUDN=RFC6761}}), which guarantees that an attempt to use this name
cannot succeed.  Clients MAY recognize the name "invalid" as special and avoid
any attempt to use this to discover an alternative service.


## Reusing Alternatives {#reuse}

A client remembers the service name, or the TargetName from the ServiceMode
HTTPS record that it successfully used to establish a connection to an
alternative service.  In subsequent connections to the same server, it makes
HTTP queries for the server name.  If this query returns a ServiceMode resource
record (RR) that includes a TargetName that is identical to the remembered
service name, the client SHOULD choose that over any alternatives, including
those RRs that are marked "alt-only"; see {{alt-only}}.

A client does not make a query for the remembered alternative name.  They make a
query for the name of the server, using the QNAME derived from the URI of the
target resource.

The RR that matches the remembered service name is selected, overriding any
SvcPriority that might otherwise result in another ServiceMode record being
chosen.

If a query for HTTPS records does not produce a ServiceMode record with a
matching TargetName, any remembered information MUST be removed for that origin.

When reusing stored information, if a connection is unsuccessful for any reason
(see {{use}}), remembered information for that origin MUST be removed.  Clients
clear retained alternative service information on reuse to prevent stale
information from affecting all future connection attempts.  After removing
remembered information, a client MAY make another attempt to connect using any
other ServiceMode records that the DNS query produced.


### Example of Reuse

A client that is fetching "https://example.com/" might originally perform a DNS
query for "example.com" and receive in response:

~~~ dns
example.com.  7200 IN HTTPS 1 . port=443
alt1.example. 7200 IN HTTPS 10 . port=8443
alt2.example. 7200 IN HTTPS 10 . port=8443
alt3.example. 7200 IN HTTPS 10 . port=8443
~~~

Under normal conditions, the SvcPriority of the "alt?.example." RRs would indicate
that it is not preferred, so the "example.com" record would be used.

If the client received an alternative service advertisement from this server for
"alt.example.net" it would then make a DNS query to that name.  This might
return a different set of records, as follows:

~~~ dns
alt2.example. 7200 IN HTTPS 1 . port=8887
alt3.example. 7200 IN HTTPS 1 . port=8887
~~~

If the client selects "alt2.example." and successfully connects to that host, it
remembers both the name given as an alternative name ("alt.example.net") and a
service name (the TargetName from the ServiceMode HTTPS record,
"alt2.example.").

In subsequent connections to "example.com", the client again queries the
"example.com" name.  Importantly, this is not any other name it might have
learned.  The resulting response - after following indirections through
AliasMode, CNAME, or similar mechanisms - produces the same records as previously:

~~~ dns
example.com.  7200 IN HTTPS 1 . port=443
alt1.example. 7200 IN HTTPS 10 . port=8443
alt2.example. 7200 IN HTTPS 10 . port=8443
alt3.example. 7200 IN HTTPS 10 . port=8443
~~~

The ServiceMode HTTPS record for "alt2.example." is used, even though this is a
lower priority than other records.  It is also used despite not using the same
port number as previously.


### Exclusive Alternative Services {#alt-only}

ServiceMode HTTPS records can be marked as only being available for use as an
alternative.  This allows servers to use alternative services for specific
server instances, without having clients connect to them without being first
invited to do so.

This is achieved with a SvcParam with a key of "alt-only" (codepoint TBD).  The
value of this SvcParamKey MUST be empty.  HTTPS ServiceMode records with this
SvcParamKey MUST NOT be used unless the client is actively seeking an
alternative, either as a result of actively looking up an alternative name or
because the alternative has been remembered.

To prevent clients that do not support this specification from using these
services, the "alt-only" SvcParamKey MUST be listed in the "mandatory" SvcParam.

In the following example, though "alt1.example" is listed at a higher priority
than "example.com", clients will not use this service unless an alternative was
provided by the server:

~~~ dns
alt1.example. 7200 IN HTTPS 1 . port=443,alt-only,mandatory=alt-only
example.com.  7200 IN HTTPS 2 . port=443
~~~

\[\[ISSUE: Do we need this flag in addition to the priority override?  Could one
    of the two be sufficient?]]


## Port Numbers

The name that is provided in the Alt-SvcB field or ALTSVCB frame can be any
valid DNS QNAME.  This includes those with underscored labels
{{?ATTRLEAF=RFC8552}}, including those that might be used to query for HTTPS
records to a non-default port.

~~~ http-message
200 OK HTTP/1.1
date: Mon, 24 Oct 2022 02:58:31 GMT
alt-svcb: "_8443._https.example.com"
content-length: 0

~~~

This might be used to direct clients to connect to alternative ports.  Note that
the HTTPS records might direct clients to an entirely different port number than
the name implies.  Clients MUST NOT infer a port number from the provided name,
instead treating this name no differently than any other.


## Interaction with GOAWAY

Servers that advertise alternative services cannot expect clients to switch to
the advertised alternative.  Use of the alternative is entirely at the
discretion of clients.  If the client is unsuccessful in connecting to an
alternative or does not attempt a connection, they could continue to use the
existing connection for new requests.

A server that seeks to actively encourage clients to disconnect and seek service
elsewhere needs to use graceful shutdown procedures of the HTTP version that is
in use.  HTTP/2 {{H2}} and HTTP/3 {{H3}} each provide a GOAWAY frame that can be
used to initiate the graceful shutdown of a connection.  Alternative services is
not a substitute for these mechanisms.


## Proxies

The procedures in this document apply to clients that connect to gateways or
reverse proxies.  However, clients that connect via a proxy, using HTTP CONNECT
or similar methods, have a choice.

Clients that provide a proxy with the name of a service leave name resolution to
the proxy.  Such a client MUST ignore any alternative service advertisement it
receives and MAY fallback to using legacy alternative services; see {{fallback}}.

Clients that make HTTPS queries for any connection attempt via a proxy can use
alternative services.  Such a client MUST provide the proxy with the IP address
of the server it wishes to contact, rather than providing a name.


## Fallback to Alt-Svc {#fallback}

A client that successfully makes use of HTTPS records in resolving the name of
an HTTP server MUST ignore any Alt-Svc fields or ALTSVC frames {{?ALT-SVC}} that
the server provides.  This document deprecates the mechanisms defined in RFC
7838 {{?ALT-SVC}}.

Servers might provide Alt-Svc fields or ALTSVC frames {{?ALT-SVC}} in order to
support clients that cannot use HTTPS records.


## No Authority

This design does not assume that information that a client learns about
alternatives is authoritative in any way, either by virtue of being provided by
an authoritative server.  Instead, once a server is determined to be authorative
(see {{Section 4.3 of HTTP}}), that server is treated as the authority on all
aspects of service configuration.  For protocol choice, {{!ALPN=RFC7301}} and
maybe {{?SNIP=I-D.ietf-tls-snip}} extensions in the TLS handshake
{{!TLS=RFC8446}} determine what is used.

In contrast RFC 7838, sending alternative services over an HTTP connection
ensures that the information is authoritative.  Clients therefore might have
been less concerned about attacks that compromise the integrity of alternative
services when using RFC 7838.

Though integrity protection might appear to be valuable, it produced conflicts.
For instance, information about the protocol is ostensibly authentic when
provided in Alt-Svc fields or ALTSVC frames.  However, protocol support is also
authenticated when establishing a connection.  This creates a potential conflict
between two equally authoritative sources of the same information.

Conflicts also arise when alternative service information is retained as any
retained state might disagree with what is currently deployed.  This design
avoids this contention by having the entire service resolution process occur
almost entirely to the DNS.

An alternative service advertisement provides only a minimal nudge to perform a
DNS query at the time it is made.  On reconnection, remembered state only
affects prioritization of active DNS records.  Invalid configurations do not
persist.


# Protocol Elements

Multiple ways of encoding an alternative service name is defined.  The Alt-SvcB
field in {{field}} allows servers to indicate a preferred service in responses.
The ALTSVCB frames in {{frame}} allows a server to provide alternative names
outside of the context of a query.

These approaches have different properties.  Alt-SvcB fields are forwarded by
intermediaries and so might reach clients through a gateway or reverse proxy.
Clients that use a proxy without using CONNECT or similar tunnels, might also
receive an alternative name using a field.  In comparison, ALTSVCB frames each
only apply to a single origin within the scope of a single connection.


## Alt-SvcB Field {#field}

*[Alt-SvcB]: #field

The "Alt-SvcB" response field is a String field (see {{Section 3.3.3 of
!STRUCTURED-FIELDS=RFC8941}}).  This response field MAY appear in a header or
trailer section, though servers need to be aware that some clients might not
process field values.

Each field value includes an alternative name.  Each alternative name is encoded
as an ASCII string, or a series of DNS A-labels, each separated by a single
period character (".", U+2E).  Each value MAY end with a period, though - for
the purposes of the process in {{use}} - the string is treated as an absolute
DNS QNAME whether or not a trailing period is present.

The applicable origin {{ORIGIN}} is derived from the origin of the Target
Resource (see {{Section 7.1 of HTTP}}).

If multiple Alt-SvcB fields are present in a response, the client MAY use any or
even all of the provided alternative names.


## ALTSVCB Frame {#frame}

*[ALTSVCB]: #frame

An ALTSVCB frame is defined for both HTTP/2 and HTTP/3.  The frame provides an
alternative name for an identified origin {{ORIGIN}}.

In both protocols, the ALTSVCB frame uses the identifier TBD.  The format for
both protocols is the same; this is shown in {{fig-frame}} using the notation
from {{Section 3 of !QUIC=RFC9000}}.

~~~ ascii-art
ALTSVCB Frame {
  Origin Length (i),
  Origin (..),
  Alternative Name (..),
}
~~~
{: #fig-frame title="ALTSVCB Frame Format"}

The fields in the ALTSVCB frame are defined as follows:

Origin Length:

: An integer, encoded as a QUIC variable-length integer (see {{Section 16 of
  QUIC}}) indicating the length of the Origin field, in bytes.

Origin:

: The ASCII serialization of the affected origin; see {{Section 6.2 of ORIGIN}}.

Alternative Name:

: The remainder of the frame contains a single alternative name, encoded as an
  ASCII string; see the definition in {{field}} for more details on the
  encoding.

If a server sends multiple ALTSVCB frames for the same origin, clients MUST
ignore any frames other than the most recent.


# Security Considerations {#security}

TODO Lots of work to do here.  Review RFC 7838 for relevant information to copy over.


# Internationalization Considerations {#i18n}

An internationalized domain name that appears in either an Alt-SvcB field
({{field}}) or an ALTSVCB frame ({{frame}}) MUST be expressed using A-labels;
see {{Section 2.3.2.1 of !RFC5890}}.


# IANA Considerations {#iana}

TODO register:

* Field
* H2 Frame
* H3 Frame
* alt-only SvcParam


--- back

# Contributors
{:numbered="false"}

RFC 7838 {{?ALT-SVC}} was authored by Patrick McManus, Julian Reschke, and Mark
Nottingham.  This draft contains none of that work, but many of those same basic
ideas.

# Acknowledgments
{:numbered="false"}

This work is input to discussions with a design team on HTTP alternative
services, formed after realizing that a simple revision to RFC 7838.  Though it
is informed by discussions thus far, it is NOT the product of that group.
Thanks are due to those who have participated in those discussions, who the
author is to cowardly to list due to the risk that someone is missed.
