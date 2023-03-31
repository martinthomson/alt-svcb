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
  SVCB: I-D.ietf-dnsop-svcb-https

informative:
  H2:
    =: RFC9113
    display: "HTTP/2"
  H3:
    =: RFC9114
    display: "HTTP/3"


--- abstract

HTTP servers deployments that include multiple service endpoints can use
alternative services to direct clients to use a different service endpoint.

This document obsoletes RFC 7838.

--- middle

# Introduction

HTTP origins are often comprised of multiple service endpoints.  This can be
driven by multiple requirements, such as a need to scale by adding multiple
physical servers, the need to place endpoints in network locations that are
closer to clients for performance reasons, or the need to support multiple HTTP
versions, like HTTP/2 {{H2}} or HTTP/3 {{H3}}.

For servers that operate multiple service endpoints, it can be advantageous to
have clients make requests to a specific service endpoint.

* Some deployments might seek to direct a client to a service endpoint that is
  better able to serve requests for that client.  This might occur if DNS
  resolution of the server name produces the address of a server instance that
  is further from the client.

* Servers might seek to reduce load, perhaps in anticipation of an imminent
  shutdown or maintenance action. An alternative service declaration
  can reduce either server load or the number of clients that might be affected.

* Many deployments of HTTP/3 {{H3}} use the protocol identifiers in an
  alternative service declaration to make clients aware of support for the newer
  protocol.

HTTP alternative services provide a means of indicating to clients which service
endpoints a server would prefer be used for future requests.  Clients use
alternative service advertisements as prompt to discover and use these more
preferred service endpoints.

Clients that learn about an alternative service can establish a connection to
the identified service endpoint, which - if successfully established and
authenticated - is then used for future requests.  Any existing connections the
client has are retained and used until the new connection is successful.  This
ensures that clients can continue making requests of the server without
interruption.


## Previous Alternative Services Designs

RFC 7838 {{?ALT-SVC=RFC7838}} provided the first alternative service design for
HTTP.  This design turned out to have a number of shortcomings in deployment.
Though these issues were anticipated in the design, the measures that were used
often did not work particularly well.

The RFC 7838 design included caching logic based on setting an "ma" (or max-age)
parameter.  This turned out to be challenging for many server deployments.
Setting too large a max-age meant that clients used the indicated service
endpoint for longer than was desired when operating conditions changed.
Conversely, a short cache period for an advertisement for HTTP/3 resulted in
frequently reverting to previous versions on subsequent connections.

Alternative services turned out to interact poorly with service configuration
information that is published in the DNS.  With the introduction of HTTPS
records {{SVCB}}, more details of service endpoints can be advertised in the
DNS, including the support for HTTP/3.  But this created two independent sources
of this information, each with its own approach to caching.

Alternative services are dependent on networking conditions.  RFC 7838 attempted
to manage this by having clients be responsible for invalidating alternatives
when changes in their network are detected, unless the alternative is explicitly
marked as "persistent".  In practice, detecting the necessary changes is
difficult for many clients, so this requirement is not consistently implemented.

The result being that the alternative services mechanisms defined in RFC 7838
produced suboptimal or even detrimental outcomes in some deployments.

This document obsoletes RFC 7838.


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


## Conventions and Definitions {#terms}

{::boilerplate bcp14-tagged}

The terms "server" and "client" are defined in {{HTTP}}.  The term "origin" is
defined in {{ORIGIN}}.

*[server]: #terms
*[client]: #terms
*[origin]: #terms

The term "alternative name" refers to the name advertised by a server to a
client.  This refers to a domain name that is queried by the client to discover
both service names and service endpoints.

*[alternative name]: #terms
*[alternative names]: #terms (((alternative name)))

A "service name" is the TargetName from an HTTPS ServiceMode record {{SVCB}}.
Service names, their associated parameters (SvcParams), and IP addresses
describe a "service endpoint".  Clients establiish connections to service
endpoints in order to make requests of a server.

*[service name]: #terms
*[service endpoint]: #terms
*[service endpoints]: #terms (((service endpoint)))

A server is identified using its "origin name", which is the domain name from
the target URI of resources the client makes requests toward.  This is the name
that the client authenticates when determining if a service endpoint is
authoritative.  Unlike an alternative name or service name, an origin name can
be an IP address rather than a domain name.

*[origin name]: #terms

There can be different values for origin name, alternative name, and service
name.


# Using Alternative Services {#use}

A server advertises the availability of alternative services by providing the
client with an alternative name.  The server does this using either a field in a
response ({{field}}) or an HTTP/2 or HTTP/3 frame ({{frame}}).

When a client receives a new alternative name from a server, they SHOULD attempt
to discover and use the service endpoints referred to by that name for future
requests to that server.

In order to discover and use the identified service endpoints, the client
attempts to make a request for a resource on the same server using the provided
alternative name as follows:

1. The client makes a DNS query for HTTPS records for the alternative name,
   following the procedures in {{Section 3 of SVCB}}.  Clients make this query
   as a "SVCB-reliant" client, treating missing or unobtainable HTTPS records as
   a failure.  If this process fails to produce service parameters or IP
   addresses, the process is aborted.

2. The client establishes a connection using the service parameters and
   addresses learned from the DNS query.  The client uses the origin name in any
   TLS server name indication {{!SNI=RFC6066}} of the server name from the URL,
   not the alternative name.  This allows the server to produce a certificate
   for the origin name, which the client can validate as applying to the URL it
   is resolving.  If a connection cannot be established, the process is aborted.

3. The client validates that the server is authoritative for the resource using
   the server origin name.  If the server is not authoritative, the
   process is aborted.

4. The client makes a query for the resource.  If the server does not respond or
   responds with a 421 (Misdirected Request) (see {{Section 15.5.20 of HTTP}}),
   the process is aborted.  A client MAY re-attempt a request or request another
   resource if the server responds with a 5xx status code (see {{Section 15.6 of
   HTTP}}).

5. Once a response is received, the connection to the alternative service
   endpoint is complete.  Any other connections can be closed and future
   requests directed to the new connection.  The client SHOULD remember the
   alternative name and the service name that were used; see {{remember}}.

A client MAY send multiple requests using the newly established connection to
the alternative service after it verifies that the server is authoritative.
However, a client MUST NOT remember a service name until at least one request has been
successfully completed with a 2xx or 3xx status code.  The alternative service
is therefore active once the connection is established, but it will not be
reused ({{reuse}}) for future connections until a request completes
successfully.

A client MAY continue sending other requests over any existing connection to the
server until this process completes in order to minimize latency for those
requests.  A client MAY - when presented with an alternative name - proactively
make a request for an arbitrary resource on the server, rather than waiting for
the next time a request is needed.  This might allow the connection to be
available for future requests with less delay.


## Retention of Alternatives {#remember}

Clients SHOULD remember the successful use of an alternative service in order to
support reuse ({{reuse}}).  Two pieces of information are retained:

* the alternative name, which is the name provided by the server in the Alt-SvcB
  field or ALTSVCB frame, and

* the service name, which is the TargetName from the ServiceMode HTTPS record
  that was used to successfully connect to the server.

These two names are saved for the server against the origin of the server
{{ORIGIN}}.  Clients MUST NOT reuse saved information for a server with
a different hostname, port, or scheme.

The alternative name, as carried in an Alt-SvcB field or ALTSVCB frame, is
retained only so that the client can avoid repeated attempts to discover and
connect to alternative services.  A server can send Alt-SvcB fields in multiple
responses or send multiple ALTSVCB frames.  Repeating the discovery process
could be wasteful for a client.

Any time that a server provides a different name in an Alt-SvcB field or ALTSVCB
frame, any existing information MUST be discarded.  A client MAY then initiate a
DNS query and connection attempt using the new alternative name.

Though a server might repeat an alternative name, clients MUST NOT consider the
absence of an Alt-SvcB field in a response as indicative of a retraction of a
previous advertisement.  An alternative name is only removed when replaced with
a different alternative name or when a remembered service name does not appear
in the set of HTTPS query responses (see {{reuse}}).

After a failed attempt to use an alternative service, a failure is remembered by
retaining the alternative name without a service name.  This avoids making
repeated attempts to use an alternative service that is not available, even if
the server repeats the alternative name.  A client MAY periodically attempt to
retry a failed alternative if the information is repeated.

A server can explicitly request that a client remove any remembered service name
by providing an alternative name of "invalid".  The "invalid" domain name
corresponds to a DNS name that will never successfully resolve (see {{Section
6.4 of ?SUDN=RFC6761}}), which guarantees that an attempt to use this name
cannot succeed.  Clients MAY recognize the alternative name "invalid" as special
and avoid any attempt to use this to discover an alternative service.


## Reusing Alternatives {#reuse}

In subsequent connections to the same origin, clients make a DNS query for
HTTPS records for the origin name.  If, after following any CNAME or
AliasMode records, this query returns a ServiceMode resource record
(RR) that includes a TargetName that is identical to the service name that is
remembered for the request origin, the client SHOULD choose that over any
alternatives.  This ignores any SvcPriority attributes that might cause other
records to be chosen and includes any RRs that are marked "alt-only"; see
{{alt-only}}.

Note that when reusing an alternative service, a client does not make a query
for the remembered alternative name.  HTTPS queries are made for the origin
name, which is the domain name from the target URI of the request; see also
{{ip}}.

If a query for HTTPS records does not produce a ServiceMode record with a
TargetName that matches the remembered service name, all remembered information
MUST be removed for that origin.  The client then uses the normal SVCB-optional
resolution logic as defined in {{SVCB}}.

When reusing stored information, if a connection attempt is unsuccessful (see
{{use}}), remembered information for that origin MUST be removed.  Clients clear
retained alternative service information on reuse to prevent stale information
from affecting all future connection attempts.  After removing remembered
information, a client MAY make another attempt to connect using any other
ServiceMode records that the DNS query produced.


### Example of Reuse

A client that is fetching "https://example.com/" might originally perform a DNS
query for "example.com" and receive in response:

~~~ dns
example.com. 7200 IN HTTPS 1 . port=443
example.com. 7200 IN HTTPS 10 alt1.example. port=8443
example.com. 7200 IN HTTPS 10 alt2.example. port=8443
example.com. 7200 IN HTTPS 10 alt2.example. port=8443
~~~

Under normal conditions, the SvcPriority of the "alt?.example" RRs would indicate
that it is not preferred, so the "example.com" record would be used.

If the client received an alternative service advertisement from this server for
"alt.example.net" it would then make a DNS query to that name.  This might
return a different set of records, as follows:

~~~ dns
alt.example.net. 7200 IN HTTPS 1 alt2.example. port=8887 alpn=h3
alt.example.net. 7200 IN HTTPS 1 alt3.example. port=8887 alpn=h3
~~~

If the client selects "alt2.example" and successfully connects to that host, it
remembers both the alternative name ("alt.example.net") and a service name
("alt2.example").

In subsequent connections to "example.com", the client again queries the
"example.com" name.  Importantly, this is the origin name and not any other name
it might have remembered.  The resulting response - after following indirections
through AliasMode, CNAME, or similar mechanisms - produces the same records as
previously (perhaps because these were retained in a cache):

~~~ dns
example.com. 7200 IN HTTPS 1 . port=443
example.com. 7200 IN HTTPS 10 alt1.example. port=8443
example.com. 7200 IN HTTPS 10 alt2.example. port=8443
example.com. 7200 IN HTTPS 10 alt2.example. port=8443
~~~

The ServiceMode HTTPS record for "alt2.example" is used, even though this is a
lower priority than other records.  It is also used despite not using the same
port number or protocol as the previous successful connection.


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
example.com. 7200 IN HTTPS 1 alt1.example. port=443 alt-only mandatory=alt-only
example.com.  7200 IN HTTPS 2 . port=443
~~~


## Servers Identified by IP {#ip}

An alternative name can be provided by a server that is identified by an IP
address or host names that are not domain names.  However, HTTPS queries cannot
be made for servers that are not identified by a domain name.  This makes it
impossible to use such identifiers.  A client MAY disable alternative
services for servers that are not identified by a domain name.


## Port Numbers

An alternative name provided in an Alt-SvcB field or ALTSVCB frame can be any
valid DNS QNAME.  This includes those with underscored labels
{{?ATTRLEAF=RFC8552}} and those that might be used to query for HTTPS records to
a non-default port.

~~~ http-message
200 OK HTTP/1.1
date: Mon, 24 Oct 2022 02:58:31 GMT
alt-svcb: "_8443._https.example.com"
content-length: 0

~~~

This might be used to direct clients to connect to alternative ports using
existing records.  Note that the HTTPS records might direct clients to an
entirely different port number than the name implies.  Clients MUST NOT infer a
port number from the provided name, treating this name no differently than any
other and using the port number derived from the service parameters.


## Interaction with GOAWAY

Servers that advertise alternative services cannot expect clients to switch to
the advertised alternative.  Use of any alternative is entirely at the
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

Clients that provide a proxy with the origin name of a server leave name
resolution to the proxy.  Such a client MUST ignore any alternative service
advertisement it receives.  These clients MAY fallback to using legacy
alternative services; see {{fallback}}.

Clients that make HTTPS queries for any connection attempt via a proxy can use
alternative services.  Such a client can provide the proxy with the IP address
of the server it wishes to contact, rather than providing a name.


## Fallback to Alt-Svc {#fallback}

A client that successfully makes use of HTTPS records in resolving an origin
name or alternative name MUST ignore any Alt-Svc fields or ALTSVC frames
{{?ALT-SVC}} that the server provides.  This document obsoletes the mechanisms
defined in RFC 7838 {{?ALT-SVC}}.

Servers might provide Alt-Svc fields or ALTSVC frames {{?ALT-SVC}} in order to
support clients that cannot use HTTPS records.


## Authority For Service Endpoint Configuration {#no-authority}

This design does not assume that information provided by a server or by the DNS
is authoritative information about the configuration of service endpoints.  This
is despite the information in Alt-SvcB fields or ALTSVCB frames being provided
by a server that is authoritative.

Instead, once a server is determined to be authorative (see {{Section 4.3 of
HTTP}}), that server is treated as the authority on all aspects of its own
configuration.  For example, with protocol selection, {{!ALPN=RFC7301}} and
maybe {{?SNIP=I-D.ietf-tls-snip}} extensions in the TLS handshake
{{!TLS=RFC8446}} determine what protocol is used.

For requests, a server that is determined to be authoritative for an origin can
answer all requests on that origin.  All service endpoints that are
authoritative SHOULD provide equivalent service to any other, though they could
differ in terms of performance, diagnostic information, or other minor details.
Clients will expect service endpoints to provide equivalent - or perhaps
identical - service.


# Protocol Elements

Multiple ways of advertising alternative services are defined.  The Alt-SvcB
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

The "Alt-SvcB" response field is a List of String values (see {{Sections 3.1 and
3.3.3 of !STRUCTURED-FIELDS=RFC8941}}).  This response field MAY appear in a
header or trailer section, though servers need to be aware that some clients
might not process field values.

Each field value includes an alternative name.  Each alternative name is encoded
as an ASCII string, or a series of DNS A-labels, each separated by a single
period character (".", U+2E).  Each value MAY end with a period, though - for
the purposes of the process in {{use}} - the string is treated as an absolute
DNS QNAME whether or not a trailing period is present.

The applicable origin is derived from the origin of the target URI; see
{{Section 7.1 of HTTP}} and {{ORIGIN}}.

If multiple Alt-SvcB fields or field values are present in a response, the
client MAY use any subset of the provided alternative names, including none,
one, or all of the provided names.

Servers SHOULD NOT provide more than one name.  The DNS provides ample
opportunity to present clients with multiple options, including the use of
priority to help manage selection.  A list is tolerated only to allow for the
possibility that multiple field lines might be added to responses without proper
coordination.

Clients MUST ignore unknown parameters that are provided with alternative names.
This document does not define any parameters as the DNS is expected to provide
supplementary information about services; a revision of this document would be
required to enable the use of parameters.


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

Alternative services present servers with a way of influencing how clients
select service endpoints.  This does not change how a service endpoint might be
determined to be authoritative (even more so than its predecessor; see
{{authority7838}}).


## Selecting Service Endpoints {#sec-endpoint}

This design assumes a Dolev-Yao attacker as is typical for Internet protocols
{{?RFC3552}}.  This model assumes that an attacker has complete control of the
network.

This design only supports HTTPS.  Cleartext HTTP, such as might be used for URIs
with a scheme of "http", is not supported.  This means that TLS {{!TLS=RFC8446}}
is always used to establish whether a service endpoint is authoritative,
according to {{Section 4.3.3 of HTTP}}.  TLS protects the configuration of
service endpoints, including the choice of protocol; see {{no-authority}}.
Furthermore, TLS prevents an attacker from inspecting or modifying the content
of connections.

Even with TLS, a client connects to a service endpoint of the attacker's choice.
This is a property of HTTP that the use of alternative services does not change,
as the choice of service endpoint (including IP address and port number) is not
authenticated when establishing a connection.

Certificates used to establish authority for HTTP servers do not include a port
number, which means that all HTTP services that have a certificate for the same
name will be treated by clients as being potentially authoritative.  {{Section
4.3.3 of HTTP}} mandates checks on the target URI to mitigate this attack.
Servers can use a 421 (Misdirected Request) status code (see {{Section 15.5.20
of HTTP}}) to signal any error and avoid the service endpoint being used.

DNS is not assumed to be secure in this threat model.  The use of DNSSEC
{{?DNSSEC=RFC4033}} can ensure that clients do not receive incorrect information
from DNS queries.  However, DNSSEC does not defend against attacks on routing or
forwarding infrastructure that might result in connections being directed toward
a service endpoint chosen by an attacker.  Using DNSSEC therefore does not
change this analysis, though it can make attacks less feasible for some classes
of attacker and so use is encouraged.


## Attacks From Within Servers

In addition to network-based attackers, we also consider the possibility that an
alternative service is advertised by an adversary who is able to generate HTTP
responses.  An adversary might be given the ability to generate responses for a
subset of the resources on a server, where they might provide an Alt-SvcB field
in a response.

This gives such an adversary some ability to direct clients toward a service
endpoint of their choosing; see {{sec-endpoint}}.  It also potentially allows an
adversary to create an unending sequence of alternatives; see {{sec-loop}}.

Servers can mitigate these risks by restricting access to the ability of
advertising an alternative name.


## Tracking Clients

Remembering alternative names and service names might allow a server to connect
activity at different times to the same client.  Clients might be assigned a
unique alternative name and service name in order to make return connections
identifiable.  The need for the service name to appears the set of HTTPS records
at the origin name does limit the ability of servers to track individual clients
at scale, but this still might be used to separate clients into groups for
tracking purposes or to track specific individuals.

Clients that clear origin-specific state in order to manage the risk of tracking
MUST remove any remembered alternative service information when clearing state
for a server (typically, this is associated with clearing cookies {{?RFC6265}}).


## Multiple Alternatives in Sequence {#sec-loop}

A client might receive multiple different alternative names in sequence, causing
it to spend additional resources in discovering and connecting to different
service endpoints.  Repeatedly making connections can adversely affect
performance.

This might be caused by a loop where the alternative name
provided by each service endpoint points to the other or simply an unending
sequence of new alternative names.  This can arise if service endpoints are
poorly configured.

A client can limit the effect of such misconfiguration by ignoring alternative
names that change too frequently.  A client might then continue to use the
service endpoint to which it is connected or disable alternative services
entirely for that origin.


# Internationalization Considerations {#i18n}

An internationalized domain name that appears in either an Alt-SvcB field
({{field}}) or an ALTSVCB frame ({{frame}}) MUST be expressed using A-labels;
see {{Section 2.3.2.1 of !RFC5890}}.


# IANA Considerations {#iana}

TODO register:

* Field
* H2 Frame
* H3 Frame
* alt-only SvcParam


--- back

# Authoritative Information in RFC 7838 {#authority7838}

This design differs from RFC 7838, where alternative services advertisements
were treated as authoritative information.  Clients therefore might have been
less concerned about attacks that compromise the integrity of alternative
services when using RFC 7838.

Though integrity protection might appear to be valuable, it results in
conflicts.  For instance, information about the protocol is ostensibly authentic
when provided in Alt-Svc fields or ALTSVC frames.  However, protocol support is
also authenticated when establishing a connection.  This creates a potential
conflict between two sources of the same information.

Conflicts also arise when alternative service information is retained as any
retained state might disagree with what is currently deployed.  This design
avoids this contention by delegating the service resolution process
almost entirely to the DNS.

This design provides clients with a prompt to discover a new service endpoint.
On subsequent connections, remembered state only affects prioritization of
active DNS records.  Service endpoints are always authoritative for their own
configuration.  Invalid configurations therefore do not persist.


# Contributors
{:numbered="false"}

RFC 7838 {{?ALT-SVC}} was authored by Patrick McManus, Julian Reschke, and Mark
Nottingham.  This draft contains none of that work, but many of those same basic
ideas.

# Acknowledgments
{:numbered="false"}

This work is input to discussions with a design team on HTTP alternative
services, formed after realizing that a simple revision to RFC 7838 would not
fix known problems.  Thanks are due to Ryan Hamilton, Tommy Pauly, and Matthew
Stock for their contributions to these discussions.  David Schinazi also
provided valuable input.
