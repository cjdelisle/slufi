# SLUFI

v0.1

Currently there exist great federated communication protocols, including SMTP, ActivityPub, and Matrix. There also exist powerful overlay networks such as Tor, I2P, cjdns, Yggdrasil and others. Even the DNS problem has been solved numerous times with Namecoin, ENS, Unstoppable Domains, PKT DNS, and others.

However, almost all federated protocols still depend on IPv4 and ICANN DNS (“the public internet”), because there’s currently no good way to federate across network boundaries. A federated network could choose to adopt another network as its official means of federation, and there do exist [Fediverse nodes on Tor](https://fedilist.com/instance?onion=only), but they are invisible to any node that doesn’t also run Tor, so their usage is marginal. IPv4 + ICANN DNS might not be anybody’s favorite, but it’s still the protocol underpinning most federation, because it’s just something that everybody already has.

Here I propose SLUFI, a **Simple, Lightweight, Unopinionated, Federation Interconnect**. SLUFI is intended to embed in existing federation servers, have minimal configuration and administrative overhead, and enable bridging to any network that is capable of supporting their federation protocol - even those which don’t yet exist.

The rest of this post is going to be a relatively technical description of how federation could be achieved across network boundaries. The target audience know who they are, and everybody else can feel free to stop reading here.

[!slufi](https://raw.githubusercontent.com/cjdelisle/slufi/refs/heads/master/images/slufi.png)

Forgive the slop, I couldn’t resist

---

If you’re still with us, well, here we go. Generally speaking, every *network domain* consists of two components: The *transport layer* and the *name service*. In order to communicate directly, two nodes must be compatible on both accounts. For example, a node with IPv4 + ICANN DNS is not compatible with a node using IPv4 + Namecoin, even though they share the same transport layer. Likewise, a node that is IPv4-only cannot communicate with a node that is IPv6-only, even if both use the same ICANN DNS. For the balance of this document, the term “network domain” refers to this: A name service plus a transport.

In pure P2P protocols such as Bittorrent or Bitcoin, the server abstraction does not exist, and as such they do not require their own unique and memorable names. However, in federated networks such as SMTP, ActivityPub, or Matrix, servers are relevant, and therefore they need their own identities. It’s important to note that while name services do generally allow multiple names to point to one internet address, federated protocols require servers to have a single “canonical” name by which they are identified within the federated network.

SLUFI consists of two components, one is a discovery service and the other, a proxy service.

## SLUFI Discovery Service

The SLUFI Discovery Service uses gossip to make federation nodes aware of each other despite existing in different network domains. To this end, the discovery service exchanges **DiscoveryServiceRecords** - small messages with basic information about nodes.

A **DiscoveryServiceRecord** (“DSR”) is made up of the node’s canonical name(s) (including the name service where they are registered), the network transport which it uses, key fingerprints that can be used to identify it, and proxy servers which anyone is allowed to use for the purpose of connecting to it. A **DiscoveryServiceRecord** is created by the node which it describes (hereinafter called the *originator*), it is then announced to other nodes by them, then it is passed along to nodes who trust *them*.

Discovery service records have roughly the following structure:

```rust
/// The record which identifies a federation instance
struct DiscoveryServiceRecord {
    /// The name service where names are registered (e.g. ICANN, Namecoin, PKT, Tor, I2P)
    name_service: NameService,

    /// The canonical names of federation instances hosted by this server
    names: List<String>,

    /// An OPTIONAL domain for federation traffic
    server: Option<String>,

    /// Network transport used by this instance (e.g. IPv4, cjdns, Yggdrasil, DN42, Tor)
    transport: Transport,

    /// Key fingerprints of this node's TLS certificates
    keys: List<KeyFingerprint>,

    /// List of nodes which are willing to proxy for this node
    proxies: List<Proxy>,

    /// When now - first_recv_time exceeds this number, DSR should be forgotten.
    /// Set by the originator.
    time_until_removal: u32,

    /// When this record was first announced by its originator to another node.
    /// Set by the recipient.
    first_recv_time: u64,

    /// When this record was received by the node currently holding it.
    /// Set by the recipient.
    last_recv_time: u64,

    /// Which node relayed the record to the node currently holding it.
    /// Set by the recipient. NOTE: This must be the node canonical name,
    /// NOT necessarily the same as the configured hostname.
    received_from: String,
};
```

The **NameService** object is an enumeration of name services where a name can be registered - in the case of `substack.com` that’s ICANN, but it could also be BIT (Namecoin), ENS, PKT, ONION (Tor), I2P and others.

The `names` field contains a list of node names which are hosted on the same server. In the case of ActivityPub or Matrix, one node is one server, but in the case of SMTP there can be multiple “mail domains” (effectively, nodes) on the same mail server.

The `server` field allows SLUFI proxied traffic to be handled specially. If I have a server at `mysite.tld`, I might set the server field to `slufi-fed.mysite.tld` and then I can expect all SLUFI proxied traffic to come there rather than to `mysite.tld`. If this ends with a : and a port number, that port will be used rather than the default for the federation protocol. If present, this rule is enforced by the proxies themselves. In the case of SMTP this will match an entry in the MX record.

A **Transport** is the other part of a network domain, such as IPv4, IPv6, Darknet42, cjdns, Yggdrasil, Tor or I2P. Every node specifies one transport only, the reason for this is because a name can typically only resolve to one transport. For example, if `mysite.tld` has an AAAA record with a cjdns IPv6 address, nodes without cjdns would be confused if they tried to connect to it. More explicitly, an ONION domain is inherently linked to Tor so it’s completely useless with any other transport. The exception to this rule is IPv6 which can be resolved from the same name as an IPv4 address. Therefore IPv4AndIPv6 is considered a single **Transport** and is granted a special case of being compatible with both IPv4 and IPv6 **Transports**. A node operator who wants their node to be natively accessible on multiple networks should use different names for the public facing side (e.g. xxxxx.onion, mysite.i2p, etc) and use SLUFI proxies to ensure accessibility from a federation perspective. Every **Transport** that is based on IP addressing must have its own distinctive IP range so that the **Transport** can be determined by DNS lookup.

A **KeyFingerprint** is the SHA256 hash of the DER encoded TLS certificate that is used by the node. This is the hash given by `openssl x509 -in ${cert} -noout -fingerprint -sha256`.

A SLUFI compatible federation node needs a TLS certificate which it can use as a client certificate for authenticating itself to the proxies that it may use. The key for this certificate needs to be available to the node because that’s what it uses to make outgoing connections. In http federation, nodes often run behind reverse proxies and have no direct access to their server certificates. So when crafting the **DiscoveryServiceRecord**, every node must include the fingerprint of its client certificate, but http based nodes should also attempt to get their public-facing server certificate, for example by making an https connection to their own canonical name.

It is **not** required that a TLS certificate be signed by any kind of centralized certificate authority, *except* when it is the server certificate and the **NameService** is ICANN.

Then there is a list of proxies, where each **Proxy** consists of a **NameService**, a name, a **Transport**, a port number, and a *priority* number. Priority is used for choosing a proxy and is handled the same way as as the [preference in a DNS MX reply](https://datatracker.ietf.org/doc/html/rfc5321#section-5.1). Some proxies may be reachable from multiple different network domains, in this case each “view” of the proxy is considered a separate proxy.

The `time_until_removal` field is set by the originator and used for controlling the lifecycle of a DSR. Values are in seconds and valid values are between 60 (one minute) and 604800 (7 days). Out-of-range values are clamped by the recipient.

The `first_recv_time` field is used for replacing a record when it has been updated, and the `last_recv_time` and `received_from` fields are for troubleshooting. They’re not needed by the protocol, but they allow tools to visualize the trust flow and track down the source of problems.

---

**DiscoveryServiceRecords** are not self-certifying, so they can only be accepted either from their originator (whose identity is confirmed by connecting back) or from a node that is trusted. They are not digitally signed because any public key that could be used to verify them faces the same trust problem as the record itself.

**DiscoveryServiceRecords** are small, with an imposed size limitation in the single-digit or low-double-digit kilobyte range (depending on format). The exact on-wire format of the records is defined by the specific integration of SLUFI. For example, SLUFI-ActivityPub would transmit these as JSON-LD, whereas SLUFI-SMTP would most likely transmit them in the form of key/value pairs that are more familiar to the SMTP protocol.

### DSR Filtering

There are certain **DiscoveryServiceRecords** which should not exist. For example, a DSR claiming that `substack.com` is a name in the Namecoin name system is obviously malicious and invalid. Likewise, a DSR claiming that an `.onion` domain exists in the ICANN space should be rejected. If ever ICANN were to (quazi-maliciously) add a `.onion` TLD, domains from that TLD *still* should be rejected because the risk of impersonation is too great.

Therefore, we take the convention that TLDs that are known to be ICANN should be restricted to ICANN, TLDs that are known to be of alternative DNS systems should be restricted to those, and TLDs that are not known to exist anywhere should be allowed, but only from ICANN. This rule allows that the name service of a name can be determined statically.

### DSR Exchange

Every SLUFI node must have a (configured) list of node hostnames that are trusted to provide DSRs (“`slufi_trusted`”). For each hostname, the node must periodically poll for updates since the time of its last request, and also announce their own DSR. Valid DSRs are applied to the node’s local database. Implementations should request DSRs using local communication rather than the SLUFI **Connection process** (below). In particular, implementations should attempt local communication regardless of whether the hostname seems to be of a different network domain, because its presence in the configuration suggests that this network domain is actually accessible from the server on which the node is running. In any case, implementations must not request DSRs via a proxy that they do not trust, unless they have a way to establish authenticity of the node they are connecting to (e.g. a certificate chain).

When a SLUFI node announces a DSR, the recipient of the announcement may accept the DSR only if the announcer’s client certificate matches a fingerprint in a previous version of that DSR **which was also announced to them** (fingerprints in a DSR which has been gossiped to them are not trusted). If there is no trusted fingerprint then the recipient of the announcement should connect back to the node via its canonical name and request the DSR that way.

The announce endpoint must be rate-limited with regard to the announcement of *never before seen* nodes so as to limit resource exhaustion denial of service through the creation of fake nodes. A node should not be considered “never before seen” unless its last DSR has expired more than 1 week ago. Every node must also impose a maximum DSR count, and when this number is reached, no DSRs of nodes *not currently in the database* are accepted at all, either by announcement or gossip. Finally, nodes accepting announcements must offer some method of blacklisting such that invalid nodes can be permanently excluded. Additional limits such as maximum nodes per second level domain are also recommended.

It is important to note that loops in the trust graph are acceptable, and DSR version numbers are not necessary beyond the `first_recv_time` field. Consider the following scenario: A trusts B, who trusts C, who trusts A. B receives a DSR from its originator, B sets the `first_recv_time` because he received it from the originator. A then receives it from B, C receives it from A, and finally B receives it back from C, but since the `first_recv_time` is not newer than what B originally set, B does not update his state, and no further propagation takes place.

The recommended default `time_until_removal` is 1 hour. It is the responsibility of the originating node to re-announce their DSR when half of the time to `time_until_removal` has elapsed. Polling for updated DSRs should be at randomized interval between 30 seconds and one minute.

Implementations may also have a configuration of nodes which are announced to, but are not trusted (“`slufi_announce`”).

#### DSR Polling

When polling a trusted node for updates to DSRs, the requestor sends a timestamp, and the server only provides updates with `first_recv_time` newer than this. Response entries should be ordered by `first_recv_time` ascending, and response should indicate whether there are more entries than provided, in which case requestor should re-send request for entries newer than the newest `first_recv_time` in previous batch.

In addition, on-wire protocol for DSR responses should allow for notifying about the removal of a node. While nodes are only removed by administrative action, propagation of removals will allow for fast recovery from fake node denial of service attacks.

## SLUFI Proxy Service

While the discovery service enables discovery across network domains, the proxy service is what allows for actual communication. Unlike the discovery service nodes, the proxy servers are mostly *untrusted*, the communication that is proxied through them is encrypted.

Unlike the discovery service which is embedded in the federation server, the proxy service is a standalone server implementing HTTPS CONNECT protocol. The service requests a client certificate but does not require that it be signed by a certificate authority. For any given connection, either the initiator or the recipient must be authorized by the proxy operator. When the authorized node is the initiator, he is identified by his client certificate, and when he is the recipient he is identified by the hostname/port of the CONNECT request.

In order to keep track of the client certificates of authorized nodes, the proxy server can be asked to make a request to a node and store the certificate that is provided.

It’s important to note that because this proxy uses standard HTTPS CONNECT, it can be implemented using nginx with `ssl_verify_client` and `tunnel_pass` and a small daemon to automatically update nginx config files and provide the metadata. This can be co-located on an existing https server without the need for an additional port.

While incoming traffic is constrained by destination to authorized nodes, outgoing (authorized) traffic has no such limit. Proxy implementations must offer provision for administrators to limit endpoints such as forbidding connections to the proxy’s local network, or to port 25. Documentation and examples should guide administrators to established best practices.

When a federation node connects to the proxy, whether it be incoming or outgoing, it then connects to the final destination using TLS once again, it always verifies the server certificate of the final destination. If that destination uses the ICANN name service, verification is done using the PKI certificate authorities. Otherwise it is checked against the key fingerprints in the **DiscoveryServiceRecord** which was used to discover it. The client is also always prepared to present a client certificate - so an admin may configure their server to request client certificates and may reject federation from those for whom they have no **DiscoveryServiceRecord**. The server field of the **DiscoveryServiceRecord** allows the admin to differentiate SLUFI proxied traffic so requiring client certificates will not block non-SLUFI federation.

In theory, one could imagine chaining together a sequence of proxies in order to bridge multiple network islands, and an implementation may do this, but SLUFI is not intended to be a routing protocol. There already exist plenty of overlay networks which do this job just fine, SLUFI is only intended to do the minimum necessary to bridge them together.

### Certificate checking

The configuration of a proxy only lists the node names that are authorized to connect to it, it does not specify key fingerprints so these must be discovered.

On startup the proxy connects to each of the nodes authorized in the config and requests their certificate and their **DiscoveryServiceRecord**. The DSR is needed for its server field so that the proxy can restrict incoming traffic to exactly that hostname/port. By default, this information is gotten using a pair of https requests, to `/slufi/certificate` and `/slufi/discovery/me`.

The proxy requires `/slufi/certificate` despite the fact the client certificate key fingerprint is also present in the DSR. This gives more flexibility to the proxy, concretely nginx is able to match on SHA-1 based key fingerprints only, so this additional requirement allows the proxy to be implemented using nginx.

But an alternative configuration exists for SMTP nodes, in this case the proxy makes an `EHLO` handshake and then issues `SLUFI STARTTLS` request, the server certificate that is presented in the handshake is registered as the expected client certificate. Then it performs a request `SLUFI DISCOVERY ME` in order to get the DSR.

In the proxy configuration a node is specified by any name that is reachable, and if it uses SMTP mode then it has the `mode=smtp` flag added.

### Proxy http endpoints

In addition to HTTPS CONNECT, each proxy has the following endpoints:

* `GET /network-domains` - The network domains reachable by this proxy, and the name of the proxy in each of these domains.
* `GET /authorized` - The list of names of nodes authorized to use the proxy
* `POST /recheck` - Takes a hostname/port as shown in /authorized, causes the proxy to connect back to that node to update its client certificate fingerprint. This request does not return until the connect back and state update has completed.

The `/network-domains` endpoint allows the proxy to tell people what network domains can be reached with that proxy, and how the proxy is reached from within each of those network domains. This returns `Content-Type: text/plain` like the following example:

```
ICANN Ipv4OrIpv6 my.proxy.tld
BIT Cjdns cjdns.my-proxy.bit:9999
BIT Yggdrasil ygg.my-proxy.bit other-data=for-future-use
ONION Tor exampleexampleexample.onion
I2P I2P my-proxy.i2p
```

This is NameService `<space>` Transport `<space>` host-and-maybe-port, potentially followed by additional space-deliniated key/value pairs for future use. If the port number is omitted then it is assumed to be 443. This format is chosen because it is easily parsed, including by SMTP servers which do not otherwise have a reason for JSON parsing.

The `/authorized` endpoint similarly provides a list of nodes who are authorized to connect. This is also in format `Content-Type: text/plain` and is a list of canonical name `<space>` server name (to connect to) and then space delineated key/value entries including the following:

* `keyfp=` the SHA-256 client certificate key fingerprint.
* `mode=` the method that the proxy uses to check the certificate and DSR (currently only `smtp` or `https`). If omitted, this is assumed to be `https`. If `mode=smtp` then it will be done via SMTP handshake as described above. This information is not relevant to nodes who request this endpoint, it is only exposed for diagnostic purposes.

```
examplenode.tld examplenode.tld keyfp=F7D3B5F326AD2A56DA48BBCE3B80218AEFD6136133FA994B91A2EFDE07274E23
mynode.bit slufi-fed.mynode.bit:587 mode=smtp keyfp=A0915A0D120D922CF8B8EA916F471A5CECDECBED02C12B2AA05860376DC8D5B5
anotherexample.pkt anotherexample.pkt:8080
```

The `/recheck` endpoint takes a `POST` request with a `text/plain` body containing a single node to be re-checked. This must be in exactly the canonical name as shown in the `/authorized` endpoint, for example `mynode.bit`. On success this will return the content of the `/authorized` endpoint. This endpoint is rate limited by target node requested, with a recommended default limit of one request per node per 30 seconds.

## Connection process

To enable cross-network federation, the process of making an https request or SMTP connection is obviously somewhat different. Because of the limitations placed on name services (see: **DSR Filtering**) the name service associated with a hostname can be determined statically. If the name service is local to the requesting node and the name service is one for which the **Transport** is implied (ONION and I2P which can only work with Tor and I2P respectively), then a local request is made.

If the name service is local to the requesting node and the **Transport** is not implied, then a DNS lookup is done to determine it. If the **Transport** is local to the node then local request is made. Implementations may also support insecure transports (e.g. IPv4, IPv6, DN42) with non-ICANN DNS. To do this, the name service is polled for a TLSA/DANE record and the certificate is checked against that.

Otherwise, check the local database for a **DiscoveryServiceRecord** with the correct canonical name. If this is found then a connection can be made via a proxy. The order of preference for selecting a proxy is:

1. The connecting node’s own proxies
2. Proxies mentioned in the DSR for the destination - order by priority as described above.

When connecting to a proxy, the node’s client certificate is always submitted. Likewise, when connecting through the proxy to the final destination, the client certificate is submitted again. In the case of proxied communication, the destination’s key fingerprint must match one of the fingerprints mentioned in the **DiscoveryServiceRecord**, unless the destination network domain is ICANN, in which case it must have a valid PKI certificate chain. Although the server field of the DSR is what’s actually connected to, the server certificate for an ICANN based node must also be valid for the node’s canonical name.

If trying a proxy fails, proxies are iterated in order of priority until one succeeds.

## Initialization Process

On startup, a node is aware of its `slufi_trusted` (sources of DSRs) and its proxy servers, however it does not know what network domains each proxy can reach. It knows its canonical name(s), but it doesn’t know in what **Transports** are available to it.

The process of starting up is as follows:

1. From each canonical name, determine the **NameService** by matching the TLD against TLDs of known name services. In case of no match assume ICANN.
2. Unless the **Transport** is implied by the canonical name (i.e. .onion or .i2p), perform a DNS lookup to get the IP address and determine the transport from this. If one instance has multiple canonical names with different network domains, an implementation would need to submit different DSRs for each, in this case implementations may instead fail with an error.
3. Read the client certificate and get the fingerprint, implementations may automatically create one if none is present.
4. For each of the configured proxies, request `/authorized` to confirm authorization, matching on canonical name.
  * If a proxy has a mismatching or no key fingerprint, or incorrect server name, make a request to `/recheck` to update, then re-attempt step 4 again. If the node is not fixed after one call to `/recheck`, flag the proxy as non-functional and continue.
  * If a proxy has *no* entry that matches the node, flag the proxy as non-functional and continue.
5. For each working proxy, request `/network-domains` to get the network domains that can be proxied, and the “view” (hostname) of the proxy from each network domain.
6. Get your publicly facing server certificate, for example by making an https request to your own canonical name. Store the key fingerprint.
7. Now construct a DSR from the configured canonical names, the **Transport**, the configured server name (if any), the fingerprints for client certificate and (detected) server certificate, and each “view” of each working proxy.
8. Now *announce* your DSR to each of your `slufi_trusted` and `slufi_announce` nodes.
9. Open the DSR database if existing, get the most recent DSR timestamp for each of the `slufi_trusted` nodes, and begin polling them for DSRs newer than that.

Implementations should periodically re-perform this ritual, for example once per hour. And implementations should also offer administrators a way to trigger it manually, and also offer a way to rebuild the database from nothing.

## Security Analysis

The most impactful attack against SLUFI is that one of the trusted nodes begins falsifying **DiscoveryServiceRecords**. This can allow for impersonation of nodes across network boundaries. In order to conduct such an attack, a trusted node needs to craft a falsified DSR with fake key fingerprint and also fake proxies which route requests to a malicious node that has the matching key. It is not possible to use this attack to impersonate any node under the ICANN name service, because connections to these nodes are checked against the certificate authority system rather than against key fingerprints in the DSR, so in that case it is reduced to a denial-of-service attack.

Impersonation of nodes in the *same* network domain as the victim is also impossible because DSRs are only consulted if the network domain is different.

An *edge case* which would cause same network domain impersonation is as follows:

1. ICANN adds a .bit TLD
2. Instance registered on ICANN’s .bit TLD
3. Impersonating instance registered in Namecoin .bit zone
4. Victim sees cross-network Namecoin .bit node when he should see local ICANN .bit node

This is seen as relatively unrealistic as ICANN has so far been known to respect alternative DNS TLDs, and what’s more, the instance registered on ICANN’s `.bit` TLD would never resolve in SLUFI because `.bit` would always be regarded as Namecoin only.

---

In case of client certificate compromise, the recovery process is to rotate the key and restart, causing new DSRs to be generated and the new key to be loaded by the proxies as part of the startup procedure. The legitimacy of the client certificate depends on the fact that it is currently being served by the server, so there’s no special need for revocation logic.

If the server certificate is compromised, the process is the same, with of course updating any TLSA/DANE record that may exist, and (in the case of ICANN) revoking the old certificate. Here the legitimacy depends on either TLSA/DANE, or on the certificate authorities. In the case of a secure transport protocol (e.g. cjdns, Yggdrasil, Tor, I2P), the server certificate is valid because it is currently being served, in the case of insecure transports, the server certificate is valid because of a TLSA/DANE entry, and in the case of ICANN, it’s valid because the PKI system says it is.

A natural concern would be replay attacks of DSRs, but there’s no entity who can replay them except the trusted nodes who are able to forge them anyway.

---

A type of attack which doesn’t require corruption of a trusted node or certificate is a rogue actor setting up a fake nodes in order to generate a large number of **DiscoveryServiceRecords** *as the originator*. In case of resource exhaustion, the immediate impact is that DSRs stop propagating for new nodes and for those who allowed their DSR to expire. Resource exhaustion is mitigated by the rate limit on previously unseen nodes (at the announce level). This limit can also be abused to prevent new nodes from joining, but since it exists on a per-node basis rather than global, combined with other restrictions such as per-second-level-domain limits, it is most probable that legitimate new nodes will find someone to accept their announcement.

Assuming **DiscoveryServiceRecords** are valid, a proxy server is unable to man-in-the-middle attack the proxied traffic, however it could still deny service by blocking or throttling traffic. This could even be on account of an innocent system administration error. However, in order for a proxy server to appear in the **DiscoveryServiceRecord**, it must be mentioned in the configuration of the node that originates it. So a malicious proxy service cannot trick people into using it simply by giving out authorizations that were not requested. With regard to passive metadata collection, proxies are considered equally trusted to a core internet router passing TLS federation traffic.

The proxy servers might be considered a DoS vector since the HTTPS CONNECT protocol does not send an X-Forwarded-For or equivalent header so the sender is effectively anonymized. In the outgoing direction (authorized node is initiator) the proxy can be used to connect to any host/port (within proxy policy). In the incoming direction (authorized node is recipient), connections are restricted to the hostname given in the server field in the **DiscoveryServiceRecord**, or if that’s missing, the node’s canonical name.

All SLUFI traffic has a client certificate to present. So a node can be configured to require a client certificate for any connection to the hostname given in the server field of the DSR. A node may also restrict incoming SLUFI traffic to only that which has a client certificate *matching a key fingerprint from a known DiscoveryServiceRecord*. On other hostnames, the node can impose a per-IP rate limit which will help mitigate request-flood denial of service from SLUFI proxies, as well as from other threats.

## SLUFI-ActivityPub

In order to add SLUFI to ActivityPub, we must add at least one new HTTP endpoint to each node:

* `GET /slufi/discovery/me` Get the current **DiscoveryServiceRecord** for the given node, required by the proxy and for validating announcements.

In addition, nodes who will configure their own proxies must have the following endpoint as well:

* `GET /slufi/certificate` Get the client certificate that is used by the node in PEM format.

And nodes which will act as trusted relays of DSRs must have the following:

* `GET /slufi/discovery?since=<timestamp>` Get all known **DiscoveryServiceRecords** that are newer than a given timestamp (considering `first_recv_time`).
* `POST /slufi/announce` Notify about an update to one’s **DiscoveryServiceRecord**.

In addition to this, the following configuration field need to be added:

* `slufi_trusted`: Array of node names (required): These are the nodes which will be polled for **DiscoveryServiceRecords**.

In addition, a node which has its own configured proxies should have the following:

* `slufi_proxies`: Array of proxy name/port + priority: A list of proxies that a node should use.

The following additional configuration parameters may also be needed:

* `slufi_domain`: String: If specified, this will be advertised as the server in the DSR and all SLUFI compatible federation will be routed through this domain rather than the canonical name.
* `slufi_announce`: A list of servers to submit DSRs to (in addition to `slufi_trusted`)

---

If a `slufi_domain` is configured, implementations may require the http header `Client-Cert` as described in [RFC 9440](https://www.rfc-editor.org/rfc/rfc9440.html), and for compatibility they may accept as an alternative `X-SSL-Client-Escaped-Cert` which is the URL encoded PEM of the requestor’s client certificate (in nginx this is [$ssl_client_escaped_cert](https://nginx.org/en/docs/http/ngx_http_ssl_module.html)). Those who want to receive the certificate from a reverse proxy should standardize on one or both of these headers and must in any case provide clear documentation and examples to avoid configurations that would allow header-spoofing attacks. Implementations may choose to reject traffic to the `slufi_domain` unless it has a *recognized* key fingerprint.

## SLUFI-SMTP

SMTP poses a unique challenge for cross-network bridging because while it does typically have SSL certificates (for STARTTLS), they are not necessarily signed by any certificate authority. And even when they are, they are almost never the same domain as the mail domain because mail domains use NS records to point to their mail servers.

So in the case of SMTP, the PKI system cannot be used, and **DiscoveryServiceRecords** originating from the ICANN network domain need to be just as trusted as those originating from other network domains.

The server field in the DSR must match one of the servers identified in the MX record. This mail server will be the recipient of all SLUFI proxied traffic. SPF, DKIM, and DMARC are not required because sending server identity is verified using a client certificate.

Newly added SMTP commands include:

* `SLUFI STARTTLS` - This works like normal `STARTTLS` but it requires a client certificate, and it will refuse mail unless the client certificate is found in a DSR in the MTA’s local database.
* `SLUFI DISCOVERY <timestamp>` - This will cause the MTA to respond with DSRs in its local database.
* `SLUFI DISCOVERY ME` - This will provide only the MTA’s own DSR.
* `SLUFI ANNOUNCE` - Announce a DSR.

All SLUFI commands are only available after `SLUFI STARTTLS`, however they do not require that the client certificate was successfully matched to a DSR, that is only needed in order to deliver mail.

Connections to an SMTPd are not over port 25, but rather over port 587. The `SLUFI STARTTLS` handshake causes the recipient to behave as though the connection came from port 25. Furthermore, it skips the reverse lookup, MX lookup, DMARC check, etc. The client certificate authenticates the sender to the DSR in the local database.

SLUFI-SMTP uses the same HTTPS CONNECT proxy server as SLUFI-ActivityPub, for outgoing requests the mail agent speaks a small subset of http. The use of port 587 allows proxy servers to block port 25 which would otherwise be an abuse vector for sending spam to non-SLUFI mail servers.

For building the **DiscoveryServiceRecord**, SLUFI-SMTP must call to the proxy `/authorized`, `/network-domains`, and `/recheck` endpoints. Again for this it uses a minimal http implementation.

In the event that a server receives a connection from a node for which it does not have a matching DSR, it sends a temporary failure. Then it examines the FROM address to see if it’s in the same network domain, if it is, then it connects back to the mail server, thus getting the DSR and certificate so on the retry the message will succeed. If it is from a different network domain then it does nothing - hopefully its trusted DSR sources will get the DSR before the message expires.

When getting a DSR *announcement* from an SMTPd, the names field is not trusted until an MX lookup has been performed on each of the names.

## Conclusion

This is not a *great* proposal, in its current form I would not even claim that it is *good*. It will be good when it has been reviewed, commented, and improved by numerous field experts, and it will not be great until it has been implemented and tested in the real world.

However, I think that the problem which needs addressing, and I don’t think there is any other way to solve it that is better.

Certainly this problem could be solved in a *more secure way*, for example with a blockchain based DSR registry. But blockchains require validators (i.e. proof of work or proof of stake), validators need incentives, and the creation of a new currency is practically unavoidable. This would be a precipitous loss of simplicity which would harm adoption - most likely to the point that it would never become a standard at all. One could also imagine a more simple solution, but at a precipitous loss of security - either by adopting a central authority, or else trusting “everyone on the internet” not to perform impersonation attacks.

Simply put, I don’t think there is any other decentralized way to solve this which is significantly more secure without a precipitous loss of simplicity, nor one that is significantly more simple without precipitous loss of security.

If you’re interested in collaborating on this specification, you can email me at cjd@cjdns.fr
