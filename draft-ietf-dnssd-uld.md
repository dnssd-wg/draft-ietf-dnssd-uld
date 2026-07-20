---
title: "Providing Local Unicast DNS-SD Service on Infrastructure"
abbrev: "Unicast Local Discovery"
category: std
updates: 6762

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

...

--- abstract

DNS Service Discovery provides several mechanisms whereby hosts can discover and advertise services on an IP network. Such discovery can be done using Multicast DNS (mDNS) or DNS, and advertising can be done with DNS-SD Service Registration Protocol (SRP) or mDNS. This document defines Unicast Local Discovery (ULD), a service that combines an SRP registrar, a Discovery Proxy, and an Advertising Proxy. Hosts can use a ULD server to advertise and discover services on the local link entirely via unicast SRP and DNS while remaining interoperable with hosts that use mDNS.

--- middle

# Introduction

DNS Service Discovery (DNS-SD) {{!RFC6763}} is a general mechanism for advertising and discovering services on IP networks. While DNS-SD can operate over either unicast DNS or Multicast DNS (mDNS) {{!RFC6762}}, in practice mDNS is the prevalent method for local service discovery in home networks and other unmanaged environments, because unicast DNS-SD requires infrastructure support (managed DNS zones, service registration mechanisms) that is not typically present on such networks.

However, mDNS relies entirely on multicast, and places the responsibility for answering queries on each device that is publishing a service. This interacts poorly with Wi-Fi in several compounding ways: Multicast frames are not acknowledged or retransmitted at the MAC layer, making them inherently less reliable. Unlike unicast frames, they are also not buffered by the access point when a station is sleeping. This creates a problematic tradeoff especially for battery-powered devices: either wake at every DTIM beacon (usually multiple times per second) at a significant power cost, or extend sleep intervals and miss a large proportion of queries, making mDNS unreliable. Finally, multicast frames are transmitted at the lowest mandatory data rate, consuming many times more airtime than equivalent unicast frames. This means that even moderate amounts of mDNS traffic can consume a disproportionate share of available airtime.

To address this, this document defines a way of combining several existing technologies into a Unicast Local Discovery (ULD) service: an SRP registrar {{!RFC9665}} with its Authoritative DNS Server {{!RFC1034}} {{!RFC1035}} to handle registration and discovery over unicast, and an Advertising Proxy {{!I-D.ietf-dnssd-advertising-proxy}} and Discovery Proxy {{!RFC8766}} to provide interoperability with mDNS.

While each of these components can be deployed today, only when they are integrated in a standardized way into a discoverable service can a client rely entirely on unicast discovery and cease participating in mDNS itself. From a client's perspective, ULD is a drop-in replacement for mDNS: If a ULD server is available on a particular link, the client uses it for all local advertisement and discovery on that link; otherwise it falls back to mDNS. Note that because SRP is used for registration, advertisement via ULD is limited to the record patterns defined by DNS-SD.

A ULD server can be deployed as part of the network infrastructure, for example on a CE router {{!RFC7084}}, or on an ad-hoc basis on devices such as SNAC Routers {{?I-D.ietf-snac-simple}} that already have the required capabilities. It can be implemented in any device that is expected to be continuously operational on a network link and has sufficient resources to provide the service.

# Conventions and Terminology

{::boilerplate bcp14-tagged}

This document uses the terms "infrastructure" and "ad-hoc" to refer to the two different ways a ULD server can be deployed:

Infrastructure server:
: The ULD server is a router on the link that has been intentionally deployed as part of the network infrastructure. At most one infrastructure server is expected per link.

Ad-hoc server:
: The ULD server is a device on the link that is not part of the network infrastructure but has the required capabilities, such as a SNAC router. Multiple ad-hoc servers may be present on the same link.

This document uses the terms "shared" and "unique" when referring to resource record sets in the sense of Section 2 of {{!RFC6762}}: A unique record set is conceptually under the sole control of a single owning device, whereas a shared record set may contain records contributed by multiple devices.

# Unicast Local Discovery

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

## Transports

A ULD server MUST support DNS over UDP, DNS over TCP, and DNS over TLS {{!RFC7858}}. DNS over TLS support is required by the SRP specification {{!RFC9665}} and is also the basis for DNS Push Notifications. All services (DNS queries, SRP registration, and DNS Push) operate on the standard ports: port 53 for UDP and TCP, port 853 for TLS. A ULD server MUST support IPv6; dual-stack requirements are addressed in {{dual-stack}}.

On IPv6, clients MUST address ULD traffic to the server's link-local address, and a ULD server MUST refuse requests for ".local." that are not received on a link-local address, responding with RCODE 5 (REFUSED).

ULD clients MAY use TLS, however clients that do support TLS SHOULD NOT fall back to plain TCP or UDP. TLS in ULD provides opportunistic encryption as described in Section 4.1 of {{!RFC7858}}. Servers MUST NOT require client certificates and MAY use self-signed server certificates. Clients SHOULD NOT reject a server's TLS certificate, as server authentication is not a goal in this context.

## Protocol Operations

### Registering Services

A ULD server MUST accept service registrations via the Service Registration Protocol (SRP) {{!RFC9665}}. Registrations MUST use ".local." as the registration domain, matching the names that would be used if the service were advertised via mDNS. The use of any other registration domain, including default.service.arpa, is out of scope for this specification.

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

### Advertising Proxy

A ULD server MUST publish the contents of its ".local." zone into mDNS using an Advertising Proxy, ensuring that services registered via SRP are discoverable via mDNS. As with the Discovery Proxy, the Advertising Proxy in ULD effectively operates without name rewriting: Because the records are already in the .local domain, the rewriting operation mandated by Section 2.1.2 of {{!I-D.ietf-dnssd-advertising-proxy}} is a no-op.

A ULD server MUST implement TSR {{!I-D.ietf-dnssd-tsr}} to correctly resolve conflicts that arise when the same records reach mDNS via different paths, for example when a client transitions from direct mDNS participation to using ULD.

### Administrative Records

The ".local." zone MUST contain a number of administrative records. These records have no meaning in the mDNS namespace and MUST NOT be published by the Advertising Proxy.

As described in Section 6 of {{!RFC8766}}, the zone MUST contain:

- A SOA record for the zone
- Exactly one NS record for the zone, referencing the ULD server's own hostname in .local
- AAAA record(s) for that hostname, and A record(s) if applicable

For compatibility with client libraries that perform standard DNS Push or SRP service endpoint discovery, the zone MUST also contain the following SRV records, each pointing to the server's hostname in .local and the relevant well-known port:

- `_dns-push-tls._tcp.local.` — port 853 (DNS Push Notifications)
- `_dnssd-srp-tls._tcp.local.` — port 853 (SRP registration over TLS)
- `_dnssd-srp._tcp.local.` — port 53 (SRP registration over UDP/TCP)

The ULD server MUST ensure its own hostname is unique on the link. This can be achieved either by using a randomly generated name that is statistically guaranteed to be unique, or by claiming the name via mDNS probing as defined in Section 8 of {{!RFC6762}}. In either case, the server's hostname is owned and defended directly by the server as an mDNS participant, not published on behalf of a client via the Advertising Proxy.

## Server Discovery and Monitoring

### Server Advertisement {#advertisement}

All active ULD servers MUST advertise their presence using mDNS, as a DNS-SD service instance of type `<uld-service-name>._tcp`. The service advertisement consists of:

- A PTR record at `<uld-service-name>._tcp.local.` pointing to the service instance name
- An SRV record for the service instance, pointing to the server's hostname and port 53
- A TXT record for the service instance, containing a `pri` key indicating the server's priority as defined below

The SRV priority and weight fields SHOULD be set to zero and MUST be ignored by clients. Per Section 5 of {{!RFC6763}}, these fields are used for selecting among multiple SRV records for a single service instance, which does not apply here; additionally, mDNS APIs do not typically expose them to applications.

Selection among ULD servers is based on the numeric value of the `pri` TXT key (lower priority values are preferred). If there are multiple servers with the same priority, the one with the numerically lowest IPv6 link-local address MUST be preferred. Services MUST advertise a priority based on their deployment mode and capabilities:

- 0: Infrastructure server
- 100: Non-constrained ad-hoc server on a wired network link
- 200: Non-constrained ad-hoc server on a Wi-Fi link
- 1000: Constrained ad-hoc server, but otherwise well able to provide service
- 65535: Ad-hoc server that can provide service if needed, but should not be preferred

### Infrastructure RA Option

An infrastructure ULD server MUST additionally advertise its presence by including the ULD RA option in its IPv6 Router Advertisements. The presence of the RA option signals that the sender of the RA is a ULD server, and the server's link-local source address in the RA is the address clients use to reach it.

This additional advertisement mechanism serves a dual purpose: It provides IPv6 clients with a faster discovery path that does not rely on mDNS, and it enables the designation of the infrastructure ULD server to be protected by RA Guard {{?RFC6105}}. In order for this protection to be effective, IPv6-capable clients MUST use the RA option for infrastructure server discovery, either exclusively or to verify the infrastructure designation of a server discovered via DNS-SD. The DNS-SD advertisement remains necessary to enable clients already using an ad-hoc server to discover a new infrastructure server via their existing ULD connection (avoiding the need to wake for multicast RA reception), and to support IPv4-only clients.

### Client Behavior

A client that wishes to use ULD on a particular link must first discover the preferred ULD server. Discovery follows a series of steps:

1. Attempt to discover an infrastructure server.
2. Failing that, browse for a list of ad-hoc servers, and determine the preferred one using the priority specified in the TXT record (see {{advertisement}}). Since all ULD servers MUST support IPv6, an IPv6 client need only query the IPv6 mDNS multicast address (FF02::FB) for this.
3. If no server is discovered, or if no discovered server appears to work, fall back to mDNS for .local service registration and discovery.

Once a client has started to use a ULD server, it SHOULD cease its own mDNS participation on that link, and rely on the ULD server for all .local operations. The client MUST also monitor the availability of the service. If the client detects that the service is no longer available, it MUST restart the discovery process.

The client MUST consider its ULD server unavailable when operations directed at the server persistently fail (DNS queries time out, SRP lease refresh fails, or a DNS Push session is lost). When a client currently using an infrastructure server is awake and processing Router Advertisements, it MUST check for the continued presence of the ULD RA option. However, clients are not required to wake specifically for RA reception.

As motivated in {{convergence}}, clients of an ad-hoc server MUST additionally keep looking for the appearance of a more-preferred server; this could be an infrastructure server or another ad-hoc server that is preferable to the current server according to the rules defined in {{advertisement}}. Clients SHOULD utilize a DNS Push subscription with the current server for this purpose. When a client migrates to a new server (whether due to server failure or the appearance of a more-preferred server), it MUST re-register all its services with the new server.

## IPv4 and Dual-Stack Operation {#dual-stack}

ULD is designed primarily for IPv6 operation: Infrastructure server discovery uses IPv6 Router Advertisements, and clients connect to the server's IPv6 link-local address, which provides inherent link-scoping. However, dual-stack mDNS interoperability is required to ensure that services on IPv4-only devices remain discoverable through the Discovery Proxy, and services on dual-stack ULD clients can be discovered over IPv4.

A ULD server MUST therefore participate in mDNS on both the IPv6 multicast address (FF02::FB) and the IPv4 multicast address (224.0.0.251), unless deployed in an IPv6-only environment. The server SHOULD also accept ULD client connections over IPv4.

Clients MAY connect to the ULD server over IPv4 using an on-link address. When a server receives ULD traffic over IPv4, it MUST verify that the source address falls within a directly-connected subnet of the receiving interface before processing a ".local." request. Even when connecting over IPv4, clients MUST use the server's IPv6 link-local address for the tiebreaker comparison defined in {{advertisement}}; this ensures all clients converge on the same server regardless of transport. If the preferred server is not reachable over IPv4, an IPv4-only client MUST fall back to mDNS. As the RA-based discovery mechanism is IPv6-only, an IPv4-only client discovers all servers, including the infrastructure server, via mDNS.

# Operational Considerations

The ideal deployment state for ULD is a single infrastructure server on each link, providing streamlined discovery for all clients.

A device that implements ULD MAY provide ULD service by default. Unless it qualifies as an infrastructure server (see below), it MUST advertise as an ad-hoc server with a priority reflecting its capabilities.

In managed networks, the infrastructure ULD server designation MUST be enabled via explicit configuration by the network operator. Where multiple managed routers are present on a link, the operator MUST ensure that at most one advertises the ULD RA option.

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

Allocate `<uld-service-name>`, "_uld" is preferred

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

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
