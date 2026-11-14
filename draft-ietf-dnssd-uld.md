---
title: "Providing Local Unicast DNS-SD Service on Infrastructure"
abbrev: "Unicast Local Discovery"
category: std
updates: 6762, 8765

docname: draft-ietf-dnssd-uld-latest
submissiontype: IETF
number:
date:
consensus: false
v: 3
area: INT
workgroup: DNSSD
keyword:
 - DNSSD
 - mDNS
 - DNS
 - ULD
venue:
  group: DNSSD
  type: Working Group
  mail: dnssd@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/dnssd/
  github: https://github.com/dnssd-wg/draft-ietf-dnssd-uld

author:
 -
    fullname: Ted Lemon
    organization: Apple Inc
    email: mellon@fugue.com

 -
    fullname: Karsten Sperling
    organization: Apple Inc
    email: ksperling@apple.com

normative:
informative:
  IANA-SRV:
    title: Service Name and Transport Protocol Port Number Registry
    target: https://www.iana.org/assignments/service-names-port-numbers
    author:
      org: Internet Assigned Numbers Authority (IANA)
    date: false
  IANA-ICMP6:
    title: Internet Control Message Protocol version 6 (ICMPv6) Parameters
    target: https://www.iana.org/assignments/icmpv6-parameters
    author:
      org: Internet Assigned Numbers Authority (IANA)
    date: false

...

--- abstract

DNS Service Discovery provides several mechanisms whereby hosts can discover and advertise services on an IP network. Such discovery can be done using Multicast DNS (mDNS) or DNS, and advertising can be done with DNS-SD Service Registration Protocol (SRP) or mDNS. This document defines Unicast Local Discovery (ULD), a service that combines an SRP registrar, a Discovery Proxy, and an Advertising Proxy. Hosts can use a ULD server to advertise and discover services on the local link entirely via unicast SRP and DNS while remaining interoperable with hosts that use mDNS. This document updates RFC 6762 and RFC 8765.

--- middle

# Introduction

DNS Service Discovery (DNS-SD) {{!RFC6763}} is a general mechanism for advertising and discovering services on IP networks. While DNS-SD can operate over either unicast DNS or Multicast DNS (mDNS) {{!RFC6762}}, in practice mDNS is the prevalent method for local service discovery in home networks and other unmanaged environments, because unicast DNS-SD requires infrastructure support (managed DNS zones, service registration mechanisms) that is not typically present on such networks.

However, mDNS relies entirely on multicast, and places the responsibility for answering queries on each device that is publishing a service. This interacts poorly with Wi-Fi in several compounding ways: Multicast frames are not acknowledged or retransmitted at the MAC layer, making them inherently less reliable. Unlike unicast frames, they are also not buffered by the access point when a station is sleeping. This creates a problematic tradeoff especially for battery-powered devices: either wake at every DTIM beacon (usually multiple times per second) at a significant power cost, or extend sleep intervals and miss a large proportion of queries, making mDNS unreliable. Finally, multicast frames are transmitted at the lowest mandatory data rate, consuming many times more airtime than equivalent unicast frames. This means that even moderate amounts of mDNS traffic can consume a disproportionate share of available airtime.

To address this, this document defines a way of combining several existing technologies into a Unicast Local Discovery (ULD) service: an SRP registrar {{!RFC9665}} with its Authoritative DNS Server {{!RFC1034}} {{!RFC1035}} to handle registration and discovery over unicast, and an Advertising Proxy {{!I-D.ietf-dnssd-advertising-proxy}} and Discovery Proxy {{!RFC8766}} to provide interoperability with mDNS.

While each of these components can be deployed today, only when they are integrated in a standardized way into a discoverable service can a client rely entirely on unicast discovery and cease participating in mDNS itself. From a client's perspective, ULD is a drop-in replacement for mDNS: If a ULD server is available on a particular link, the client uses it for all local advertisement and discovery on that link; otherwise it falls back to mDNS. Note that because SRP is used for registration, advertisement via ULD is limited to the record patterns defined by DNS-SD.

A ULD server can be deployed as part of the network infrastructure, for example on a CE router {{!RFC7084}}, or on an ad-hoc basis on devices such as SNAC Routers {{?I-D.ietf-snac-simple}} that already have the required capabilities. It can be implemented in any device that is expected to be continuously operational on a network link and has sufficient resources to provide the service.

## Updated Specifications

Section 3 of the Multicast DNS specification {{!RFC6762}} requires any DNS query for a name ending with ".local." to be sent to the mDNS multicast address. As described in {{uld}}, this document additionally allows such queries to be sent to the ULD server for the link.

Section 5 of the DNS Push Notifications specification {{!RFC8765}} requires clients to use DSO running over TLS over TCP, and Section 7 requires the Strict Privacy profile of {{!RFC8310}}. For DSO sessions with a ULD server, which are confined to the local link, this document replaces both requirements with the rules in {{transports}}. These allow clients to use DSO over TCP without TLS if a TLS connection cannot be established, and specify the Opportunistic Privacy profile for TLS. DNS Push Notifications in other contexts are not affected.

# Conventions and Terminology

{::boilerplate bcp14-tagged}

This document uses the terms "infrastructure" and "ad-hoc" to refer to the two different ways a ULD server can be deployed:

Infrastructure server:
: The ULD server has been intentionally deployed as part of the network infrastructure, and is designated as such by a router on the link (see {{ra-option}}). At most one infrastructure server is expected per link.

Ad-hoc server:
: The ULD server is a device on the link that is not part of the network infrastructure but has the required capabilities, such as a SNAC router. Multiple ad-hoc servers may be present on the same link.

This document uses the terms "shared" and "unique" when referring to resource record sets in the sense of Section 2 of {{!RFC6762}}: A unique record set is conceptually under the sole control of a single owning device, whereas a shared record set may contain records contributed by multiple devices.

# Unicast Local Discovery {#uld}

ULD provides service registration and discovery within the ".local." domain, the same domain used by mDNS {{!RFC6762}}. This ensures that ULD is a transparent replacement for mDNS from the perspective of applications and resolver libraries (see {{local-zone}} for further discussion of this design choice). Unlike mDNS, where the separate IPv4 and IPv6 multicast addresses effectively result in two independent .local namespaces (Section 20 of {{!RFC6762}}), ULD maintains a single unified .local zone per link.

This document updates {{!RFC6762}} to allow .local queries to be directed to a ULD server as an alternative to mDNS multicast: Any DNS query for a name ending with ".local." MUST be sent to the ULD server for the link, or to the mDNS IPv4 link-local multicast address 224.0.0.251 or its IPv6 equivalent FF02::FB.

The following sections describe the architecture and operation of a ULD server on a single link. There are five logical parts to a ULD server:

- The DNS {{!RFC1035}} zone in which DNS-SD information will be stored
- The SRP {{!RFC9665}} service, which is used to add and update services in the DNS zone
- The Authoritative DNS Server {{!RFC1035}} which authoritatively answers unicast DNS queries, drawing on both the zone and the Discovery Proxy
- The Discovery Proxy {{!RFC8766}}, which enables unicast discovery of local services that are advertised via mDNS but have not been registered via SRP
- The Advertising Proxy {{!I-D.ietf-dnssd-advertising-proxy}} service, which advertises the contents of the zone using mDNS, ensuring SRP-registered services are discoverable via mDNS

~~~~ aasvg
+-----------------------------------+       +----+
|         Unicast DNS over          |       |    |
|         UDP, TCP, or TLS          |       |    |
+-----------------------------------+       |    |
       |              ^                     |    |
       v              |                     |    |
+-------------+ +-------------------+       |    |
|     SRP     | |   Authoritative   |       |    |
|  Registrar  | |      Server       |       |    |
+-------------+ +-------------------+       |mDNS|
       |            ^   ^                   |    |
       v           /     \                  |    |
+-------------------+   +-----------+       |    |
|                   |   | Discovery |<------|    |
|      .local       |   |   Proxy   |       |    |
|       zone        |   +-----------+       |    |
|                   |-->|Advertising|------>|    |
|                   |   |   Proxy   |       |    |
+-------------------+   +-----------+       +----+
~~~~
{: #fig-architecture title="ULD Server Architecture"}

A device serving multiple links (e.g. a CE router with multiple VLAN interfaces) conceptually maintains a separate ULD instance per link, each with its own .local zone; the server's link-local address on each interface inherently scopes queries to the correct zone. Selection of the preferred server is also per link: a client operating on multiple links performs discovery separately for each, and may use ULD on some links while falling back to mDNS on others.

## Transports {#transports}

A ULD server MUST support DNS over UDP, DNS over TCP, and DNS over TLS {{!RFC7858}}. DNS over TLS support is required by the SRP specification {{!RFC9665}} and is also the basis for DNS Push Notifications. All services (DNS queries, SRP registration, and DNS Push) operate on the standard ports: port 53 for UDP and TCP, port 853 for TLS. A ULD server MUST support IPv6; dual-stack requirements are addressed in {{dual-stack}}.

On IPv6, clients MUST address ULD traffic to the server's link-local address, and a ULD server MUST refuse requests for ".local." that are not received on a link-local address, responding with RCODE 5 (REFUSED).

Clients SHOULD use TLS, and MAY use TCP or UDP without TLS if a TLS connection cannot be established. TLS in ULD is used with the Opportunistic Privacy profile described in Section 5 of {{!RFC8310}}. Servers MUST NOT require client certificates and MAY use self-signed server certificates. Clients SHOULD NOT reject a server's TLS certificate, as server authentication is not a goal in this context.

For DSO sessions with a ULD server, these rules replace the DNS Push {{!RFC8765}} requirement for clients to use TLS with the Strict Privacy profile (Sections 5 and 7 of {{!RFC8765}}).

## Protocol Operations

### Registering Services

A ULD server MUST accept service registrations via the Service Registration Protocol (SRP) {{!RFC9665}}. Registrations MUST use ".local." as the registration domain, matching the names that would be used if the service were advertised via mDNS. The use of any other registration domain, including default.service.arpa, is out of scope for this specification. The server MUST reject any DNS Update message for ".local." that is not an SRP Update, responding with RCODE 5 (REFUSED).

Because the .local zone is link-scoped, clients MUST only include address records (A, AAAA) in their SRP registrations that are valid on the link, following the same rules as for mDNS responses in Section 6.2 of {{!RFC6762}}.

### Discovering Services

A ULD server MUST answer authoritatively for queries in ".local." and MUST support both standard DNS queries (over UDP, TCP, or TLS) and DNS Push Notifications {{!RFC8765}} (over TLS using DSO {{!RFC8490}}).

The server draws on two sources: its authoritative zone (containing SRP-registered services) and the Discovery Proxy (reflecting services advertised via mDNS). Because ULD uses .local for both its zone and mDNS interactions, the Discovery Proxy operates without name rewriting or text-encoding translation — mDNS records are served to unicast clients with names unchanged. This is in contrast to deployments where a Discovery Proxy rewrites names between a delegated subdomain and .local as described in Section 5.5 of {{!RFC8766}}.

Since records matching a query could be shared, by default the server MUST answer from the union of both sources: for a given query it returns the matching records from the zone together with the matching records from the Discovery Proxy; a DNS Push subscription MUST likewise reflect changes from both sources, even when initial results came from only one.

As an important optimization, when the query matches a unique record set in the .local zone, the server SHOULD treat the zone data as complete and not consult the Discovery Proxy. A record set registered via SRP is considered unique if it was part of a Host Description or Service Description Instruction (Section 3.3.1 of {{!RFC9665}}). Because unique records have a single owner, the zone can be assumed to hold the entire set.

Because the Advertising Proxy publishes zone contents via mDNS on the same link that the Discovery Proxy monitors, it is conceivable for the server to see records from its own .local zone reflected back via the Discovery Proxy; the server MUST ensure that such duplicate records are not returned to clients.

The server SHOULD include additional records as defined in Section 12 of {{!RFC6763}} (e.g., SRV, TXT, and address records alongside PTR answers), except where doing so would cause the response to be excessively large.

### Discovery Proxy

The Discovery Proxy's contributions to ULD queries generally follow the answer-aggregation behavior of Section 5.6 of {{!RFC8766}}, answering from its mDNS cache when populated and otherwise issuing mDNS queries on the link. However, the behavior described there for standard DNS queries assumes that such queries are used only for one-shot operations expected to return unique records, while long-lived subscriptions are used when browsing for shared record sets. This is a reasonable assumption in the context of a generic Discovery Proxy, since off-link clients performing wide-area DNS-SD discovery will generally have the resources needed to implement DNS Push {{!RFC8765}} over TLS. In the context of ULD, DNS Push subscriptions remain RECOMMENDED when querying for shared records, but it is desirable to support constrained clients that cannot use them: Such clients MAY implement time-bounded browse operations using repeated standard DNS queries. The interval between repeated queries MUST be at least 9 seconds (90% of the 10-second TTL cap imposed by the Discovery Proxy, Section 5.5.1 of {{!RFC8766}}).

To support such clients, the ULD server MUST continue to issue mDNS queries for the requested records for at least 20 seconds after the most recent matching standard DNS query, as it would for records in which a local client has an active interest (Section 5.2 of {{!RFC6762}}). Where the requested records are unique (indicated by the cache-flush bit), whether received from the link or already held in the cache, the server need issue no further mDNS queries other than to refresh them as they approach expiry. Where the records may be shared, the server responds to each query immediately from its cache, while the continued querying surfaces any additional records on the link and keeps the cache current, so that the client's successive point-in-time snapshots converge on the complete set. This effectively matches the treatment that Section 5.6 of {{!RFC8766}} describes for subscriptions.

In addition, the server MUST maintain an active interest in the `_uld._tcp` service instances on the link, as it would for a local client browsing for them (Section 4 of {{!RFC6763}}), including their SRV, TXT, and address records. Together with the server's own advertisement ({{admin-records}}), this ensures that a single query to any ULD server returns the complete server list, along with the records clients need to rank the servers.

ULD servers do not provide DNSSEC-signed responses, so the NSEC and NSEC3 synthesis described in Section 5.5.3 of {{!RFC8766}} does not apply, and a query for either type is answered with a NODATA response {{?RFC2308}}.

### Advertising Proxy

A ULD server MUST publish the contents of its ".local." zone into mDNS using an Advertising Proxy, ensuring that services registered via SRP are discoverable via mDNS. As with the Discovery Proxy, the Advertising Proxy in ULD effectively operates without name rewriting: Because the records are already in the .local domain, the rewriting operation mandated by Section 2.1.2 of {{!I-D.ietf-dnssd-advertising-proxy}} is a no-op.

A ULD server MUST implement TSR {{!I-D.ietf-dnssd-tsr}} to correctly resolve conflicts that arise when the same records reach mDNS via different paths, for example when a client transitions from direct mDNS participation to using ULD.

### Administrative Records {#admin-records}

In addition to the records registered by clients via SRP, the ".local." zone MUST contain a number of administrative records, described below. None of these records are published by the Advertising Proxy.

The zone MUST contain a copy of the server's own `_uld._tcp` service advertisement, consisting of the records listed in {{advertisement}}. This makes them available to ULD clients, including as part of the server list.

The zone MUST also contain the following records describing the zone and the services offered by the server. They exist only in the zone and are not advertised on mDNS.

- A SOA record for the zone, and exactly one NS record referencing the ULD server's own hostname, as described in Section 6 of {{!RFC8766}}.
- For compatibility with client libraries that perform standard DNS Push or SRP service endpoint discovery, the following SRV records, each pointing to the server's hostname and the relevant well-known port:
  - `_dns-push-tls._tcp.local.` - port 853 (DNS Push Notifications)
  - `_dnssd-srp-tls._tcp.local.` - port 853 (SRP registration over TLS)
  - `_dnssd-srp._tcp.local.` - port 53 (SRP registration over UDP/TCP)

Apart from the PTR record at `_uld._tcp.local.`, which is shared with the advertisements of other servers, all administrative record sets are unique, so the Discovery Proxy SHOULD NOT be consulted for queries matching them.

## Server Discovery and Monitoring

### Server Advertisement {#advertisement}

All active ULD servers MUST advertise their presence using mDNS, as a DNS-SD service instance of type `_uld._tcp`. The service advertisement consists of:

- A PTR record at `_uld._tcp.local.` pointing to the service instance name
- An SRV record for the service instance, pointing to the server's hostname and port 53
- A TXT record for the service instance, containing
  - a `v` key encoding the version of this specification implemented by the server as a US-ASCII decimal number in the range 1-255 (see {{version}})
  - a `pri` key encoding the server's priority as a US-ASCII decimal number in the range 0-15 (see {{priority}})
  - a `seq` key encoding the server's sequence number as a US-ASCII decimal number in the range 0-4294967295 (see {{sequence}})
- At least one AAAA record for the server's hostname with a link-local IPv6 address
- At least one A record for the server's hostname with an on-link IPv4 address, if the server accepts ULD traffic over IPv4 (see {{dual-stack}})

The server MAY advertise other address records for its hostname (e.g. AAAA records with routable addresses); these are not relevant for ULD clients.

Because a ULD service instance is described by a single SRV record, its priority and weight fields SHOULD be set to zero per Section 5 of {{!RFC6763}}, and MUST be ignored when processing an advertisement. Advertisements lacking a version, priority, sequence number, or an IPv6 link-local address, or with a version, priority, or sequence number outside the specified range, are invalid and MUST be ignored.

The server claims its hostname and service instance name as an ordinary mDNS responder, not through its Advertising Proxy, probing for them and renaming itself in the event of a conflict as specified in Sections 8 and 9 of {{!RFC6762}}.

#### Version {#version}

The `v` key identifies the version of this specification implemented by the server. Servers implementing this document MUST advertise `v=1`. Future versions of this specification may define higher values, along with different priority benchmarks for servers advertising them. Clients compare versions numerically as described in {{client-behavior}}, regardless of whether they implement that version, so that all clients converge on the same server.

#### Server Priority {#priority}

Servers advertise a priority based on their deployment mode and capabilities. An infrastructure server MUST advertise priority 0. Ad-hoc servers MUST advertise a priority between 1 and 15, chosen relative to the following benchmarks:

- 4: Unconstrained ad-hoc server on a wired network link
- 8: Unconstrained ad-hoc server on a Wi-Fi link
- 12: Ad-hoc server with limited resources (e.g. CPU, memory, or network interface quality)

Because priority is compared before rank (see {{client-behavior}}), any difference in priority overrides the preference for the longest-running server, and causes clients to migrate when a higher-priority server appears. An ad-hoc server SHOULD therefore advertise the benchmark value for its category, and only deviate from it to reflect a difference in capability that justifies such a migration, e.g. between classes of an implementation's own devices. Because a change in priority triggers discovery work and may cause clients to migrate, servers SHOULD base their priority on stable properties.

#### Sequence Number {#sequence}

Each server advertises a sequence number that allows clients to detect when it has restarted, and to order servers by age, so that a server that has been available for longer is preferred over one that has just appeared. Sequence numbers are unsigned 32-bit integers forming a ring modulo `D = 2^32`. The *distance* from sequence number A to B is calculated as `(B - A) mod D`.

Given the set of distinct sequence numbers of all servers discovered on a link, the *oldest* and *newest* sequence numbers are determined as follows: If the set contains only one value, it is trivially both oldest and newest. Otherwise, the values are sorted in ascending order and the gap (i.e. the distance) from each value to the next is calculated, with the gap from the last value wrapping around to the first. The value at the start of the largest gap is the *newest* sequence number, and the value at the end of it is the *oldest*. If multiple gaps are equally large, the one starting at the numerically lowest value is used. The *rank* of a server is then calculated as the distance from the oldest sequence number to the server's sequence number: a lower rank indicates an older server.

A server MUST choose its sequence number when it starts up or otherwise loses its registration state, and MUST NOT change it at any other time. Before choosing, it browses for other ULD servers on the link; this browse MUST include at least three queries (spaced as specified in {{Section 5.2 of !RFC6762}}), and the server MUST wait at least one second after the last query before choosing its value. This makes it unlikely that packet loss causes the server to miss the existing servers. If no other servers are found, the server chooses a value uniformly at random from the range 0-W, where `W = 3/4 D`. Otherwise, it chooses `(newest + 1 + R) mod D`, where R is a random value in the range 0-65535, making it the newest server. The random component makes it unlikely that a restarted server ends up with its previous sequence number.

Servers that choose sequence numbers independently (e.g. after a power outage, or when previously separate links are joined) can end up spread around the ring with several gaps of similar size. The departure or addition of a server can then make the oldest designation jump to a different server. To reduce the likelihood of this, if the second-largest gap is larger than half the largest gap, a server MUST instead place itself near the middle of the second-largest gap, choosing `(start + floor(gap / 2) + R) mod D`, where start and gap describe that gap. This *split rule* does not apply if the most preferred discovered server is an ad-hoc server with the same version and priority as the starting server, and the resulting value would rank the starting server ahead of it. In that case, the server chooses `(newest + 1 + R) mod D` as above.

A server SHOULD durably record the sequence number it chooses on a link before advertising it, and use it to avoid randomness when it next chooses a sequence number on that link. If no other servers are found, it SHOULD choose `(previous + 1) mod D`, where previous is its recorded sequence number, instead of a random value. Otherwise, it SHOULD use `R = 0` for the random component R in the above formulas, unless that would reproduce its previous sequence number, in which case it uses `R = 1`. This ensures that a restart is always detectable. A server that knows it is attached to a different link than when it recorded its sequence number SHOULD NOT use the recorded value.

The rationale for these rules, and for the choice of D, W, and the range of R, is discussed in {{sequence-rationale}}.

### ULD Server RA Option {#ra-option}

The ULD Server (ULDS) RA option allows a ULD server to be advertised in IPv6 Router Advertisements (RAs) {{!RFC4861}}, providing IPv6 clients with a faster discovery path that does not rely on mDNS. Its format is shown in {{fig-ra-option}}.

An infrastructure ULD server MUST be advertised via the ULDS option with the I flag set. In addition to faster discovery, this enables the designation of the infrastructure ULD server to be protected by RA Guard {{?RFC6105}}. In order for this protection to be effective, an IPv6-capable client MUST NOT use a server advertising priority 0 ({{priority}}) unless it holds a ULDS option that is valid for it, i.e. an unexpired option with the server's address and the I flag set, regardless of how it initially discovered the server. The DNS-SD advertisement remains necessary to enable clients already using an ad-hoc server to discover a new infrastructure server via their existing ULD connection (avoiding the need to wake for multicast RA reception), and to support IPv4-only clients.

If the infrastructure ULD server is a router, it SHOULD directly advertise itself via the ULDS option. Alternatively, another router on the link, which need not be a default router, MAY designate it by sending a ULDS option with the server's address.

An ad-hoc ULD server that is a router MAY advertise itself via the ULDS option with the I flag not set. This allows clients to bootstrap discovery by obtaining the server list from any such server via ULD, instead of browsing via mDNS (see {{client-behavior}}).

The ULDS option SHOULD be included in unsolicited RAs, and MUST be included in the response to a Router Solicitation. When the server stops providing ULD service, the option SHOULD be sent with a Lifetime of zero in subsequent RAs, and then omitted. The same applies when an infrastructure server is no longer designated as such, unless it continues as an ad-hoc server and advertises itself with the I flag not set. A received option replaces any option previously received from the same router for the same server address.

~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type      |    Length     |I|A|         Reserved          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Lifetime                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+             Server Address (present if A is set)              +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #fig-ra-option title="ULDS Option Format"}

Type:
: TBD (see {{iana-nd-option}}).

Length:
: The length of the option in units of 8 octets: 1 if the A flag is not set, 3 if it is set. Receivers MUST ignore an option that is too short for the fields indicated by its flags, and MUST ignore any octets following those fields, so that future versions of this specification can append fields.

I:
: Infrastructure flag. If set, the server is designated as the infrastructure server for the link.

A:
: Server Address flag. If set, the option contains the Server Address field.

Reserved:
: MUST be set to zero by the sender and ignored by the receiver.

Lifetime:
: 32-bit unsigned integer. The time in seconds, relative to the time the RA is received, after which the option is no longer valid. A value of zero means that the server is no longer advertised. The Lifetime is independent of the Router Lifetime in the RA header. Its value SHOULD by default be at least three times MaxRtrAdvInterval (Section 6.2.1 of {{!RFC4861}}), so that the advertisement survives the loss of individual RAs.

Server Address:
: The IPv6 link-local address of the server. If the A flag is not set, the server's address is the source address of the RA. In either case, the server address MUST be the numerically lowest link-local address the server advertises in its AAAA records, so that clients can use it directly to reach the server and for the tiebreaker comparison defined in {{client-behavior}}.

### Client Behavior {#client-behavior}

A client that wishes to use ULD on a particular link must first discover the preferred ULD server. Discovery follows a series of steps:

1. Attempt to discover an infrastructure server via the ULD Server RA option ({{ra-option}}).
2. Failing that, browse for a list of ad-hoc servers, and determine the preferred one. Since all ULD servers MUST support IPv6, an IPv6 client need only query the IPv6 mDNS multicast address (FF02::FB) for this. Clients MAY also obtain the server list by querying any already-discovered ULD server, including an ad-hoc server advertised via RA.
3. If no server is discovered, or if no discovered server appears to work, fall back to mDNS for .local service registration and discovery.

Clients MUST determine server preference by comparing the following in order, with the first difference deciding:

{: type="a"}
1. Whether the server is an infrastructure server (priority 0): infrastructure servers are preferred over ad-hoc servers.
2. For ad-hoc servers, the server's version, i.e. the numeric value of the `v` TXT key ({{version}}). Higher values are preferred.
3. For ad-hoc servers, the server's priority: the numeric value of the `pri` TXT key ({{priority}}). Lower values are preferred.
4. For ad-hoc servers, the server's rank, calculated from the `seq` TXT key of all discovered servers ({{sequence}}). Lower values are preferred.
5. As a tiebreaker, the server's IPv6 link-local address, compared numerically. Lower values are preferred. If the server's hostname has multiple AAAA records with link-local addresses, the numerically lowest of these MUST be used.

Note that infrastructure servers are compared by address alone, not by version or rank, using only the information available from the ULD Server RA option. A link is expected to have at most one infrastructure server, so this comparison is only a last resort.

Once a client has started to use a ULD server, it SHOULD cease its own mDNS participation on that link, and rely on the ULD server for all .local operations. The client MUST also monitor the availability of the service. If the client detects that the service is no longer available, or that its server's advertised priority has changed, it MUST restart the discovery process.

The client MUST consider its ULD server unavailable when operations directed at the server persistently fail (DNS queries time out, SRP lease refresh fails, or a DNS Push session is lost). An IPv6 client using an infrastructure server MUST also consider it unavailable if it stops holding a valid ULD Server RA option for the server while awake and processing RAs. Clients are not required to wake specifically for RA reception, and an option that expires while a client is asleep does not by itself make the server unavailable.

A server that has restarted may have lost the client's registrations, so a client that has registered services MUST re-register them when it detects that its server has restarted, i.e. that the server's sequence number has changed (see {{sequence}}). How and when a client checks for this is implementation-specific: for example, a client with a DNS Push session can use keepalives and check the server's sequence number when it re-establishes a lost session, while a client using only UDP can periodically query the server's TXT record to verify that it still responds and that the sequence number is unchanged.

As motivated in {{convergence}}, clients of an ad-hoc server MUST additionally keep looking for the appearance of a more-preferred server; this could be an infrastructure server or another ad-hoc server. Clients SHOULD utilize a DNS Push subscription with the current server for this purpose. When a client migrates to a new server (whether due to server failure or the appearance of a more-preferred server), it MUST re-register all its services with the new server.

## IPv4 and Dual-Stack Operation {#dual-stack}

ULD is designed primarily for IPv6 operation: Infrastructure server discovery uses IPv6 Router Advertisements, and clients connect to the server's IPv6 link-local address. However, dual-stack mDNS interoperability is required to ensure that services on IPv4-only devices remain discoverable through the Discovery Proxy, and services on dual-stack ULD clients can be discovered over IPv4.

A ULD server MUST therefore participate in mDNS on both the IPv6 multicast address (FF02::FB) and the IPv4 multicast address (224.0.0.251), unless deployed in an IPv6-only environment.

A ULD server SHOULD also accept ULD traffic over IPv4, and clients MAY address ULD traffic to an on-link IPv4 address of the server. Even when doing so, clients MUST use the server's IPv6 link-local address for the tiebreaker comparison defined in {{client-behavior}}; this ensures all clients converge on the same server regardless of transport. If the preferred server is not reachable over IPv4, an IPv4-only client MUST fall back to mDNS. As the RA-based discovery mechanism is IPv6-only, an IPv4-only client discovers all servers, including the infrastructure server, via mDNS.

On IPv6, ULD restricts access to .local to on-link devices by requiring link-local addresses (see {{transports}}). This does not carry over to IPv4, because IPv4 hosts do not usually have an IPv4 link-local address {{!RFC3927}} when a routable address is configured on the same interface. ULD therefore uses ordinary on-link IPv4 addresses and establishes link-locality explicitly. ULD messages sent over IPv4 UDP, by clients and servers alike, MUST be sent with an IP TTL of 255, following the Generalized TTL Security Mechanism {{?RFC5082}}. This is necessary because an on-link source address can be spoofed by an off-link attacker injecting UDP requests, such as SRP Updates, that require no response. Over TCP (including TLS), the three-way handshake already prevents such spoofing, so IPv4-only clients that cannot control the IP TTL can use TCP instead.

A server that accepts ULD traffic over IPv4:

- MUST silently discard requests for ".local." received over IPv4 UDP with an IP TTL less than 255.
- MUST NOT accept TCP Fast Open {{?RFC7413}} payloads on IPv4 connections that do not use TLS, extending the corresponding requirement for SRP Updates (Section 6.1 of {{!RFC9665}}) to all requests.
- MUST refuse other IPv4 requests for ".local." with RCODE 5 (REFUSED) unless the source address is an IPv4 link-local address {{!RFC3927}} or falls within a directly-connected subnet of the receiving interface.
- MUST send traffic destined for IPv4 link-local addresses directly on the receiving interface (Section 2.6.2 of {{!RFC3927}}), even if it has no IPv4 link-local address of its own.

# Operational Considerations

The ideal deployment state for ULD is a single infrastructure server on each link, providing streamlined discovery for all clients.

A device that implements ULD MAY provide ULD service by default. Unless it qualifies as an infrastructure server (see below), it advertises as an ad-hoc server with a priority as described in {{priority}}. A device SHOULD only provide ad-hoc ULD service if it is expected to remain available on the link for extended periods (e.g. because it is mains-powered and does not normally sleep), and has sufficient resources to serve all clients on the link.

In managed networks, the infrastructure ULD server designation MUST be enabled via explicit configuration by the network operator. Where multiple managed routers are present on a link, the operator MUST ensure that at most one infrastructure server is advertised via the ULD Server RA option ({{ra-option}}). Where the infrastructure server is designated by a router other than the server itself, the operator MUST also configure the server to operate as an infrastructure server, so that it advertises priority 0 ({{priority}}).

In unmanaged networks such as home networks, CE routers {{!RFC7084}} are typically autonomously operating devices that form the basis for the network infrastructure. A CE router that provides ULD SHOULD claim infrastructure status by default, since it is already the de facto infrastructure for the link. Indications that a device is serving in this role include being the default router (sending RAs with nonzero Router Lifetime) and providing services such as DHCPv4 that are inherently singleton on the link. A device that is not clearly the primary gateway for the link MUST NOT claim infrastructure status without explicit configuration.

Note that Homenet {{?RFC7788}} does not define a "primary router" designation — it uses a distributed model with no single designated device. ULD's "one infrastructure server" assumption does not align well with this architecture. In Homenet networks, ULD servers may need to operate in ad-hoc mode, or Homenet could be extended to elect a ULD infrastructure server.

# SNAC Router Considerations

TODO: The AdProxy and DiscProxy components of the SNAC router could use ULD, and the device itself can host a ULD server.

# Security Considerations

TODO

# Domain Name Reservation Considerations

The considerations set out in {{!RFC6762}} for handling of names within the ".local." domain continue to apply.
Name resolution APIs and libraries SHOULD continue to recognize .local names as special
and SHOULD NOT send queries for these names to their configured (unicast) caching DNS server,
unless that server is also the ULD server for the link in question.

# IANA Considerations

## Service Name Registration

IANA is requested to add the following entry to the "Service Name and Transport Protocol Port Number Registry" {{IANA-SRV}}, using the fields required by Section 8.1.1 of {{!RFC6335}}:

| Field Name         | Value                         |
|--------------------|-------------------------------|
| Service Name       | uld                           |
| Transport Protocol | tcp                           |
| Assignee           | IESG <iesg@ietf.org\>         |
| Contact            | IETF Chair <chair@ietf.org\>  |
| Description        | Unicast Local Discovery       |
| Reference          | This document                 |
| Port Number        | None                          |
| Assignment Notes   | Defined TXT keys: v, pri, seq |

## IPv6 Neighbor Discovery Option {#iana-nd-option}

IANA is requested to assign a new IPv6 Neighbor Discovery option type for the ULD Server Option in the "IPv6 Neighbor Discovery Option Formats" registry {{IANA-ICMP6}}:

| Type | Description       | Reference     |
|------|-------------------|---------------|
| TBD  | ULD Server Option | This document |

--- back

# Choice of Local Domain {#local-zone}

To make ULD a drop-in replacement for mDNS, a client querying a ULD server must see the same records it would have seen via mDNS, and a device advertising services via the ULD server must be discoverable as if it was advertising those services via mDNS. In other words, the ULD zone must have the same semantics as the ".local." namespace for that link.

Indeed, when users or applications reference names in .local, their intent is generally semantic: to find or resolve services on the local link, not to trigger the use of the Multicast DNS protocol specifically. Because of this, the intended adoption path for ULD is for resolver libraries to use it transparently as the resolution mechanism for .local when a ULD server is available, requiring no changes to most applications.

So while a new locally-served special-use domain could be defined for ULD on the wire, this would create two namespaces with identical content and semantics, and would require implementations and libraries to map between them. It would also contradict the insight that .local is about semantics rather than implementation, further discouraging the intended adoption path. Instead, ULD directly uses the .local zone defined by mDNS.

# Rationale for Supporting Ad-Hoc Servers {#why-ad-hoc}

From the point of view of a ULD client, the simplest deployment would be one where the network's DNS resolver also provides ULD. The client already sends all DNS queries to this resolver, so queries for names in .local could simply be handled alongside all other queries at the same endpoint. Many real-world networks are in fact structured in a way that would support this: In home networks, the CE Router {{!RFC7084}} typically acts as a DNS forwarder, DHCP server, and IPv6 router for the local link. The same architecture extends to many small and medium enterprise networks, where a single site gateway commonly provides these services across multiple network segments (VLANs), making it a natural deployment point for ULD across the entire site.

However, adding ULD support to existing network infrastructure requires firmware updates to devices such as CE routers and site gateways, which may not happen quickly across the installed base. Meanwhile, SNAC routers {{?I-D.ietf-snac-simple}} and similar devices already implement all the components needed for ULD (SRP registrar, Advertising Proxy, Discovery Proxy) and are typically updated more frequently. To enable ULD deployment in the near term, it is therefore important to support a mode of operation where such devices can offer ULD service on an ad-hoc basis.

Supporting ad-hoc ULD servers means that clients must be able to discover and select among them, directing .local queries to the ULD server while sending other DNS queries to the configured resolver. This adds complexity, but also enables deployment in networks where the DNS resolver is an off-link service that cannot provide ULD; this is a common configuration in more complex enterprise networks.

# Convergence on a Preferred Server {#convergence}

At first glance, multiple ULD servers on a link would seem to provide workable service through their mDNS proxies: Services registered on one server would become visible through others via the Advertising Proxy and Discovery Proxy. However, name conflict resolution breaks down in this configuration. If two clients were to register the same name on different servers, both SRP servers would accept the registration, and the resulting conflict would only manifest at the mDNS layer, where it may persist unresolved or be resolved silently and incorrectly, but in both cases without feedback to either client.

This can be addressed either through server-to-server replication of registrations {{?I-D.ietf-dnssd-srp-replication}}, or by having all clients converge on the same server — that is, all clients independently select the same server using a deterministic priority mechanism. ULD takes the latter approach: since its target deployment state is a stable infrastructure server per link, the added complexity of replication is not warranted.

Convergence also needs to be maintained over time: Ad-hoc servers in particular can appear and disappear at any time, and an infrastructure server may become available after clients have already begun using an ad-hoc server. If discovery were a one-time process, clients performing it at different times could observe different sets of available servers and make different server choices, breaking convergence. Therefore, discovery of ULD servers must be an ongoing process: Clients need to monitor the availability of their chosen server, discover newly available servers, and migrate to a higher-priority server when one appears.

# Rationale for Sequence Number Rules {#sequence-rationale}

Every change of the preferred server causes migration work for clients. The sequence number rules specified in {{sequence}} enable clients to prefer the oldest server of a given version and priority, i.e. the one that has been available the longest, so that a new server appearing does not cause migration. The rules work within three constraints: every client must derive the same order from the DNS-SD advertisements alone; a server's value must stay fixed while it runs, so that a change signals a restart; and servers cannot be assumed to have synchronized clocks or persistent state, which rules out advertising a start time.

The basic scheme is similar to a Lamport clock: a starting server chooses a value one step past the newest value it sees. As long as every server sees the others when it starts, the values form a single "run" of closely spaced values, and the largest gap is the remainder of the ring, between the newest and the oldest server. Because each restart moves the newest value forward, the run moves around the ring of size D over the lifetime of the network. Finding the oldest server by the largest gap rather than by absolute value means that this movement, including wrapping past D, does not affect the result. The random component R in each step serves restart detection: without it, a restarting newest server would choose its predecessor's value + 1, i.e. exactly its own previous value. R reduces this to a probability of 1 in 65536. D = 2^32 leaves ample room for steps of this size while keeping values within a common integer size.

The ranking is not entirely stable: the oldest designation can move to a different server while the server that previously held it is still present. Because ranks are measured from the oldest server, such a *spurious re-ranking* changes every server's rank. It causes an unnecessary migration if it changes the preferred server, but is otherwise harmless. The rules and the choice of parameters aim to make spurious re-rankings rare, and in particular to avoid networks in which they occur persistently.

The oldest server is the one at the end of the largest gap, so a spurious re-ranking means that a different gap has become the largest without the oldest server departing. This can happen in two ways: A departure can merge the two gaps adjacent to the departing server into one that exceeds the largest gap. Alternatively, a starting server can choose a value inside the largest gap that is close enough to its end to make that server itself the oldest. Within a single run, neither is a practical concern: a starting server places itself at the start of the largest gap, and it takes at least tens of thousands of restarts (while a subset of servers stays up) for another gap to grow comparably large. Spurious re-rankings therefore mainly occur after servers have chosen their values without seeing each other, in which case they choose from the range 0-W. This happens in two situations, which W affects in opposite directions:

1. Simultaneous start: All servers start at the same time, e.g. after a power outage, and choose values scattered across the range, which can leave several large gaps of similar size. A smaller W is better here: the gaps within the range add up to at most W, while the gap spanning the rest of the ring is at least D - W. If W is at most D/2, merging gaps through departures cannot overtake the largest gap, and only the departure of the oldest server itself changes the designation. Above D/2 this is no longer guaranteed, and the chance of competing gaps grows gradually with W.

2. Lone start: A single server fails to discover the existing servers when it starts, e.g. because of packet loss during discovery or a temporary network partition. Once the other servers become visible, it is designated oldest if its value fell in the half of the largest gap that precedes the oldest server, which causes a spurious re-ranking. That is roughly half the ring, so averaged over all positions of the run this happens half the time, whatever W is. What W determines is how much of the range 0-W can lie in that half. If W is at most D/2, the whole range can lie inside it, so that in some networks nearly every lone start causes a spurious re-ranking. If W is larger than D/2, the range is longer than half the ring, so it cannot lie entirely inside that half, and the probability is at most D/(2W). Joining previously separate links has a similar effect to a lone start, except that the clients on one of the links have to migrate in any case.

The choice of W = 3/4 D is a compromise between the two cases. It limits the probability of a spurious re-ranking on a lone start to at most 2/3, wherever the run has moved. (A larger D would not help, because a W of at most D/2 would still leave some networks in which nearly every lone start causes a spurious re-ranking.) A simultaneous start leaves a gap of at least D/4 spanning the rest of the ring, which is usually still the largest.

After a simultaneous start, a lone start, or the joining of links, values can end up scattered, with several gaps of similar size. In that state, a departure can easily merge two gaps into one that exceeds the largest. The split rule aims to keep every gap other than the largest below half the size of the largest, so that this cannot happen. It gives up exact age order among the other servers, but does not change which server is designated oldest, and therefore leaves the ranks of existing servers unchanged. The exception to the split rule ensures that a split does not itself cause a migration.

Ranks are computed over all servers on the link rather than separately for each version and priority. A starting server therefore only needs to discover some existing server, of any version or priority, to choose a value within the existing run instead of from the range 0-W. This also keeps servers of the same type that restart together, e.g. after a software update, within the existing run. Ranking per priority would make lone starts more likely, and would require a server to choose a new sequence number if it changes its priority.

Persisting the previous sequence number gives a server information about its position relative to the other servers. A server that restarts alone at its previous value + 1 lands within the existing run rather than at a random position, which avoids most spurious re-rankings from lone starts. Servers that persist their values also keep their relative order across a simultaneous restart, so their values do not become scattered. Their restarts are always detectable as well.

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
