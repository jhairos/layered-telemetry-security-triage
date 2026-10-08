# Layered Network Telemetry for Security Incident Triage: Hop-Level AS/Geo Path Context and Per-Layer Active Probing at a Remote Satellite-Connected Site

October 8, 2026 · MSc. Medardo Jairo Suntaxi Cocanguilla, C|EH

## Abstract

Incidents reported by users of a remote site rarely arrive with a layer attached. "Mail does not go out", "the satellite app shows high latency" or "pages do not open" can be a routing problem, a DNS problem, an application problem or a reputation problem, and each one needs different evidence before anyone acts on it.

This paper describes two tools built by the author to collect that evidence from the site itself, and two cases in which they were used. mtr-geo is an open-source wrapper around `mtr` that adds autonomous system (AS), organization and geolocation to every hop. PiProbe is a probe platform, built on SmokePing, that runs scheduled tests per layer: ICMP with two MTU sizes, DNS, TCP towards SMTP port 25, HTTPS, FTP transfers and multipath route comparisons.

In the first case, a latency alert raised by a satellite provider's application was shown not to correspond to the path towards the public Internet. The second case began with the history of infected clients in the island's main town: the local resolver flagged hundreds of devices with botnet activity, outbound SMTP towards a large mail provider was filtered at the transit, and between May and July 2026 an institution's mail server was rejected by a large webmail provider. The server itself was clean, but the address range it belonged to had been escalated on a public blocklist because of other hosts' activity, and the webmail provider kept its own block after the blocklist cleared.

The paper explains how layered telemetry separated network, application and reputation causes in both cases, what the tools cannot show, and which controls would have shortened the second incident. All names and addresses are generic.

## 1. Context

The site studied here is a remote island location whose Internet access leaves through a low-Earth-orbit satellite transit network. Institutions on the site run their own services, including an on-premises mail server, and depend on that single path for everything that reaches the mainland or the rest of the world.

In this kind of site, troubleshooting from a central office has a basic limitation: the operator sees the network from the wrong side. Latency, packet loss and DNS behavior measured from the mainland say little about what a user on the island experiences, and the satellite provider's own dashboard measures something else again, usually the path to its own gateway.

The practical problem was triage. Complaints arrived as symptoms (slow applications, video cutting out, mail rejected, low speed), and the first useful question was always the same: which layer is failing, and is the cause inside the local network, in the transit, at the destination, or in how the destination judges the site's traffic? Security incidents hide well in that ambiguity. A reputation block looks like a mail outage, and a route change towards an unexpected network looks like ordinary latency.

The two tools described in the next section were built to answer that question from the site itself, with history, instead of from the central office on demand.

## 2. Tools and measurement design

### 2.1 mtr-geo

`mtr` already gives loss and latency per hop. What it does not give is ownership: which network each hop belongs to and where that network is registered. mtr-geo runs `mtr` (IPv4 or IPv6) and adds a summary table with AS, organization, geolocation and average RTT per hop. It normalizes organization names, uses GeoLite2 databases (city, ASN, country) with a fallback lookup method, and keeps any credentials in environment variables rather than in the script. It is published as open source under the MIT license at `github.com/jhairos/mtr-geo`.

The security value is in the ownership column. A path that suddenly crosses an unexpected AS, leaves through another country, or reaches the destination through a network that does not belong to it is the first visible sign of a route leak or a hijack, and plain `mtr` output does not make it obvious.

### 2.2 PiProbe

PiProbe is the author's probe platform, built on SmokePing with custom targets, probes and presentation pages. In November 2025 four probes were in operation: one on the island site, two in the capital and one abroad. Their data was centralized in a single concentrator, so the same destination could be compared from different vantage points, and each probe also ran scheduled mtr-geo reports towards a fixed list of 11 destinations: large content and social platforms, a national government portal, a public DNS resolver and two in-country addresses.

Each probe group targets a specific layer, so that a degradation can be placed before anyone opens a ticket:

| Layer | Probe | What it separates |
| --- | --- | --- |
| Network (L3) | ICMP to global sites, CDNs and in-country sites | General latency and loss towards each destination group |
| Network (L3) | ICMP with 1464-byte and 1472-byte payloads | MTU and fragmentation problems on the path |
| Network (L3) | mtr-geo traces and multipath comparisons | Route changes and the AS/country each hop belongs to |
| Transport (L4) | TCP ping to port 25 of a large webmail provider and of the site's own mail server | SMTP reachability, independent of the mail application |
| Application (L7) | DNS resolution tests | Resolver latency and failures |
| Application (L7) | HTTPS checks and FTP 1 MB GET/PUT transfers | Real transfer time in both directions |

The probes also host a speed test and a web tool that runs DNS, TCP, TLS and HTTP checks in sequence, used when a user reports a specific site.

## 3. Case A: satellite latency alert (December 2025)

The satellite provider's application reported high latency on the site's link, and a ticket was requested. Before escalating, the question was whether users on the site were actually seeing that latency towards the Internet.

The first check was an mtr-geo trace from the island probe towards a public DNS resolver. The output, with addresses removed and networks described by role, was:

| Hop | Network (role) | Registered location | Avg RTT (ms) |
| --- | --- | --- | --- |
| 1 | Local access provider (site edge) | Site country | 0.3 |
| 2–3 | Satellite transit AS | United States | 0.8–1.0 |
| 4–5 | Private addresses inside the transit | — | 27.0–33.7 |
| 6–7 | Satellite transit AS | United States | 27.4–33.7 |
| 8–11 | Destination network | United States | 27.5–45.3 |

The trace showed no loss on any hop. The first satellite-transit hops answered in about 1 ms, so they sit on the ground side of the space segment; the satellite delay appears from hop 4 onwards, inside private addressing that belongs to the transit network.

The PiProbe history gave the longer view. Over the first ten days of December, the median RTT towards a large search platform averaged 58.2 ms (minimum 42.3 ms, maximum 80.4 ms) with 1.42 % average loss. Social and video platforms showed medians between 57 and 59 ms with losses between 1 % and 2 %. FTP transfers of 1 MB took about 2.0 s down and 1.9 s up, and the TCP probe to the site's own mail server showed 20.3 ms and no loss. The 1464-byte ICMP probe averaged 1.71 % loss, with isolated spikes up to about 10 % in a three-hour window, which did not affect the transfer tests.

With these results the alert was not treated as a degradation of the site's Internet access. The most likely explanation is that the application measures latency towards the provider's own point of presence or gateway, which can be reassigned dynamically between regions for load or capacity reasons. That could not be confirmed from the site, so the recommendation was to keep the ticket open and ask the provider two concrete questions: whether a gateway reassignment had occurred, and which exact target its application uses to measure latency, so it could be correlated with the probe data.

The security relevance of this case is not the alert itself but the method. The trace also shows a limit of geolocation: hops physically near the site appear registered in the United States because geolocation databases report where an address block is registered, not where the equipment is. A conclusion such as "traffic is leaving through another country" cannot rest on the geolocation column alone.

## 4. Case B: infected clients, SMTP filtering and mail reputation (November 2025 – July 2026)

### 4.1 How it started: infected clients on the island

The work on network security at the site did not start with the mail case. It started with the history of affected clients in the island's main town. While reviewing several customer addresses, it was observed that a significant number of client devices were generating spam or showing botnet behavior. The local resolver's botnet-detection report made the scale visible:

| Month | Observation | Source |
| --- | --- | --- |
| November 2025 | SMTP on port 25 from the site towards the MTAs of a second large mail provider was being filtered. The satellite transit provider adjusted its ACL filters and the traffic was restored; the provider's MX hosts were added to PiProbe on port 25. | Ticket follow-up, `nc` tests |
| November 2025 | The next night, the TCP probe to port 25 of the institution's mail server got no response for about 50 minutes, with 54 % average loss in the window, and recovered without intervention. The cause was not identified. | PiProbe |
| May 2026 | In a one-hour report window, the number of clients flagged as infected on the island's DNS cache rose from zero to about 385 in roughly 15 minutes and fell back to zero in the next 15. | Resolver botnet-detection report |

The spike in May is the clearest indicator: hundreds of devices behind the same access network flagged by the resolver as infected within minutes is consistent with coordinated malware activity, not with isolated users. It also explains what happened next to the address range, described in 4.2.

As infected clients were identified, the response was applied at the network edge in two places, starting in November 2025. ACL filters were placed on the site's edge router, and the satellite transit provider adjusted the filters on its own edge equipment. The adjustment on the transit side restored SMTP on port 25 from the site towards the MTAs of a second large mail provider, which had been filtered; no change was made on the customer's infrastructure. The result was confirmed from the site with a direct connection test to each of that provider's five MX hosts:

```
$ nc -vz -w5 <mx-1> 25 ... <mx-5> 25
Connection to <mx-1> 25 port [tcp/smtp] succeeded!
...
Connection to <mx-5> 25 port [tcp/smtp] succeeded!
```

After the filter was removed, all five MX hosts were added to PiProbe on port 25, so a new block would show up in the history and could be escalated to the transit provider with evidence instead of being discovered through user complaints. Why the transit applied the filter was not stated by the provider. Outbound port 25 filtering is a common anti-spam control, so a link with the infected clients is plausible, but it was not confirmed.

### 4.2 Outbound mail rejected by reputation

An institution on the site runs its own mail server. From May 2026, messages sent to addresses hosted by a large webmail provider bounced. The bounce returned by the provider's inbound gateways was explicit:

```
550 5.7.1 Unfortunately, messages from [<server-ip>] weren't sent.
Please contact your Internet service provider since part of their
network is on our block list (S3150).
```

The wording matters. It does not say the server sent spam; it says that part of the network the server belongs to is on a block list. The case went through several weeks of ticket updates before the cause was fully separated:

| Month | Observation |
| --- | --- |
| May 2026 | Outbound mail to the webmail provider rejected with `550 5.7.1 (S3150)`. Mail to other destinations unaffected. |
| June 2026 | A public DNS blocklist showed the server's address listed at netblock level (the /22 containing it) and at ASN level. The /22 had one directly listed address with about 130 abuse impacts in seven days. That address was not the mail server. |
| June 2026 | The netblock and ASN listings expired, but the webmail provider kept rejecting with the same code. |
| July 2026 | Full validation of the mail server and its DNS records; delisting requested through the provider's sender portal; provider confirmed conditional mitigation. |

The public blocklist involved works in three levels: individual addresses that sent abuse (level 1), address ranges with too many level-1 listings (level 2), and whole autonomous systems (level 3). The mail server was never listed at level 1. It was caught at levels 2 and 3 because another host in the same range kept generating abuse, which is consistent with the infected clients described in 4.1. The blocklist's own description calls this a "bad neighborhood": the server is penalized for sharing an address range, not for its behavior.

The July validation checked the layers in order. PiProbe showed the TCP path to port 25 of the webmail provider working, with a median around 130 ms and 1.21 % average loss over 30 hours, so the network path was not the problem. The DNS records of the domain were reviewed: A, PTR, MX and SPF were consistent, and outgoing messages carried a DKIM signature. With connectivity and authentication discarded, the rejection was attributed to the provider's own reputation for that address, which had outlived the public blocklist. A delisting request was filed through the provider's sender portal. The provider answered that the address had been mitigated under conditions: reduced daily sending limits until reputation recovers, and 24 to 48 hours to propagate across its systems.

As a preventive measure, the domain's SPF record was found to be more permissive than needed. It authorized the server by address but also through the `a` and `mx` mechanisms and ended in softfail:

```
v=spf1 a mx ip4:<server-ip> ~all
```

Since the server is the only authorized sender, the recommendation was to reduce it to:

```
v=spf1 ip4:<server-ip> -all
```

This does not fix a reputation block, but it closes the door to other hosts sending as the domain, which is exactly the kind of abuse that damages reputation in the first place.

One observation from the same period remains open. The TCP probe to the site's own mail server showed a median of 20.7 ms but 4.45 % average loss, with some samples reaching 100 % loss. The rejections were explicit SMTP responses, so these losses did not cause them, but they were not explained during the case.

## 5. What the layered view contributed

In both cases the decisive step was discarding layers with data from the site, not from the central office.

In case A, the combination of a hop-by-hop trace with ownership and ten days of per-destination history was enough to show that users were not experiencing what the provider's application reported. Without the ownership column, the private hops in the middle of the path would have been unexplained; with it, they were clearly inside the transit network.

In case B, the TCP probe to port 25 and the transfer tests removed the network from the list of suspects early. What remained was an application-level answer, a 550 code, whose text pointed to the network the server belongs to. From there the analysis moved to reputation, and the distinction between the public blocklist and the webmail provider's own reputation explained why the problem survived the blocklist expiry.

The same pattern applies to other security events at sites like this. A reputation block, a resolver being abused, a route that starts crossing an unexpected AS, or a volumetric attack that saturates the satellite link all present first as "something is slow or failing". Per-layer probes with history turn that symptom into a question that can be answered.

## 6. Limitations and open points

The tools have limits that should be stated before relying on them in an investigation.

- **Geolocation is registration, not location.** As case A shows, a hop near the site can appear registered in another country. The geolocation column is useful to detect changes, not to prove where traffic physically goes.
- **Active probes are synthetic.** A TCP ping to port 25 proves reachability, not that a complete SMTP transaction will be accepted. Case B was solved because the rejection text was available; the probe alone would not have found it.
- **Single vantage points.** A probe sees the path from where it is installed. Comparing probes in different locations reduces this limit but does not remove it.
- **Provider-internal metrics stay opaque.** In case A, the provider's measurement target could not be confirmed from the site, so the explanation given remains the most probable one, not a verified one.
- **Unexplained loss on the local SMTP probe** during case B was not investigated further and is still to be validated.
- **Correlation, not host-by-host attribution.** The resolver report counts flagged clients but does not show what each one sent. The link between the infected clients, the transit's port 25 filter and the netblock listing is consistent in time, but it was not proven address by address.

## 7. Conclusions

At a remote site with a single satellite path, the operator's main difficulty during an incident is not fixing the problem but locating it. Two inexpensive additions made the difference in the cases studied: knowing who owns each hop of a path, and having per-layer measurements with history taken from the site itself.

Case A showed that a provider-side alarm and the users' real experience can disagree, and that the disagreement can be demonstrated with data before a ticket escalates. Case B showed that a mail outage can be a reputation problem caused by someone else's behavior in a shared address range, and that a public blocklist clearing does not mean every receiving provider has cleared too. In both cases the layered probes discarded the network early, which kept the analysis on the real cause.

The tools do not replace logs, provider data or protocol responses. Their value is in narrowing an ambiguous symptom to one layer and one party quickly enough for the right evidence to be requested.

## 8. Recommendations

For sites that host their own outbound mail behind a shared address range:

1. **Check reputation at three levels, not one.** Monitor the mail server's address, its netblock and its ASN on public blocklists, and alert when any of them is listed. In case B the address was clean while the range and the ASN were not.
2. **Track receiving-provider reputation separately.** Large webmail providers keep their own reputation systems; clearing a public blocklist does not clear them. Keep their sender portals and delisting procedures documented in advance.
3. **Request a dedicated or cleaner range for servers that send mail.** Sharing a range with residential or unmanaged customer devices exposes the server to collateral listings.
4. **Harden mail authentication:** SPF limited to the real sending address and ending in `-all`, DKIM signing on every outbound message, a published DMARC policy, and a PTR record that matches the HELO name.
5. **Ask the access provider to act on the level-1 sources.** Collateral listings stop only when the abusing hosts in the range are cleaned or isolated. The resolver's botnet-detection reports are a direct way to find them: flagged clients can be notified and, if the activity persists, isolated or limited on port 25.

For telemetry at remote sites:

6. **Keep an mtr-geo baseline per critical destination** and compare new traces against it. A new AS or country in the path is worth reviewing even when latency looks normal.
7. **Probe each layer separately** (ICMP with two MTU sizes, TCP to service ports, DNS, HTTPS and real transfers) so a symptom can be placed in one layer from the history before anyone is called.
8. **Correlate provider dashboards with independent probes** before opening tickets, and ask the provider which target its own metrics measure.

## References

1. J. Klensin, [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321), IETF, 2008.
2. G. Vaudreuil, [RFC 3463: Enhanced Mail System Status Codes](https://www.rfc-editor.org/rfc/rfc3463), IETF, 2003.
3. S. Kitterman, [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208), IETF, 2014.
4. D. Crocker, T. Hansen, M. Kucherawy, [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376), IETF, 2011.
5. M. Kucherawy, E. Zwicky, [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489), IETF, 2015.
6. J. Levine, [RFC 5782: DNS Blacklists and Whitelists](https://www.rfc-editor.org/rfc/rfc5782), IETF, 2010.
7. mtr (My Traceroute) project documentation.
8. SmokePing project documentation.
9. MaxMind GeoLite2 databases documentation.
10. M. J. Suntaxi Cocanguilla, mtr-geo v1.0, open-source software, DOI [10.5281/zenodo.23240327](https://doi.org/10.5281/zenodo.23240327), `github.com/jhairos/mtr-geo`.

## Author note

**Author:** MSc. Medardo Jairo Suntaxi Cocanguilla, Certified Ethical Hacker (C|EH), Ecuador.

**Version:** 1.0 – October 2026.

mtr-geo and PiProbe were designed and built by the author. The observations come from an operational environment the author knows first-hand; the analysis and this paper are the author's own study and do not represent the position of any organization. Names of institutions, providers, addresses and case references have been removed or replaced with generic descriptions.
