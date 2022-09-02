---
title: "Asserting a network connection using revolver's identity"
abbrev: "Network connection using resolver"
category: info

docname: draft-wing-opsawg-authenticating-network-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
area: "ops"
workgroup: "Operations and Management Area Working Group"
keyword:
 - network
 - security
venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
  github: "danwing/authenticating-network"
  latest: "https://danwing.github.io/authenticating-network/draft-wing-authenticating-network.html"

author:
 -
    ins: D. Wing
    fullname: Dan Wing
    organization: Citrix
    email: "dwing-ietf@fuggles.com"
 -
    ins: T. Reddy
    fullname: Tirumaleswar Reddy
    organization: Nokia
    email: "kondtir@gmail.com"



normative:
  DNR:    I-D.ietf-add-dnr
  DDR:    I-D.ietf-add-ddr
  AKA:    I-D.ietf-emu-aka-pfs

informative:
  Evil-Twin:
    title: "Evil twin (wireless networks)"
    date: 2022-06
    target: "https://en.wikipedia.org/wiki/Evil_twin_(wireless_networks)"
    author:
      -
        name: Wikipedia

  IEEE802.11:
     title: "IEEE802.11"
     date: 2022-08
     target: "https://en.wikipedia.org/wiki/IEEE_802.11"
     author:
       -
         name: Wikipedia

  RFC8792: RFC8792
  RFC8110: RFC8110
  RFC7839: RFC7839

--- abstract

This document describes how a a client uses the encrypted DNS server identity to reduce
an attacker's capabilities if the attacker is operating a look-alike
network.

--- middle


# Introduction

When a client connects to a wireless network the user
or their device want to be sure the connection is to the expected
network, as different networks provides different services --
performance, security, access to split-horizon DNS servers, and so on.  Although 802.1X provides layer 2
security for both Ethernet and WiFi networks, 802.1X is not widely deployed
and unavailable on LTE and 5G networks -- and often applications are
unaware if the underlying network was protected with 802.1X.

On WiFi networks a malicious actor can operate a rogue access point
with the same SSID and WPA-PSK as the victim network [Evil-Twin].  In
many deployments (for example, coffee shops and bars) offer free Wi-Fi
as a customer incentive.  Since these businesses are not
Internet service providers, they are often unwilling and/or
unqualified to perform complex configuration on their network.  In
addition, customers are generally unwilling to do complicated
provisioning on their devices just to obtain free Wi-Fi.  This leads
to a popular deployment technique -- a network protected using a
shared and public Pre-Shared Key (PSK) that is printed on a sandwich
board at the entrance, on a chalkboard on the wall, or on a menu.  The
PSK is used in a cryptographic handshake, defined in [IEEE802.11],
called the "4-way handshake" to prove knowledge of the PSK and derive
traffic encryption keys for bulk wireless data. The same deployement
technique is typically used in residential or small office/home office
networks. If the Pre-Shared Key (PSK) for wireless authentication is
the same for all clients that connect to the same WLAN, the shared key
will be available to all nodes, including attackers, so it is possible
to mount an active on-path attack.

This document describes how a wired or wireless client can utilize
network-advertised encrypted DNS servers to ensure the attacker has no
more visibility to the client's DNS traffic than the legitimate
network.  In cases where the local network provides its own encrypted
DNS server, the client can even ensure it has re-connected to the same
network, offering the client enough information to positively detect a
significant change in the encrypted DNS server configuration -- a
strong indicator of an attacker operating the network.  The proposed
mechanism is also useful in deployments using Opportunistic Wireless
Encryption [RFC8110] and in LTE/5G mobile networks where the long-term
key in the SIM card on the UE is compromised (Section 1 of [AKA]).




# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Procedure

Client connects to network and obtains network information via DHCPv4, DHCPv6, or RA.
The network indicates its encrypted DNS server using either [DNR] or [DDR].  The client connects
to that encrypted DNS server, completes the TLS handshake and performs public key validation of
the presented certificate as normal.

The client can associate the network name with the encrypted DNS server's
identity that was learned via [DNR] or [DDR]. The type of the network name is
dependent on the access technology to which the endpoint is attached.
For networks based on IEEE 802.11, the network name will be the SSID of
the network and the Basic Service Set Identifier (BSSID). For 3GPP access-based networks, it
is the Public Land-based Mobile Network (PLMN) Identifier of the access network, and for 3GPP2 access,
the network name is the Access-Network Identifier (see [RFC7839]).
If DDR is used for discovery, the client would have to perform verified discovery (see Section 4.2 of [DDR])
and the encrypted DNS server identity will be the encrypted DNS server's IP address.
If DNR is used, the encrypted DNS server identity will be Authentication Domain Name (ADN).

> todo: Improve the discussion, below, of multiple BSSIDs.  The
  existing text handles joining and re-joining SSID+BSSID, but
  how do we handle re-joining the same network with a not-seen-before
  BSSID.  We should consider pros/cons of not using BSSID versus
  solely using SSID.

Some WiFi deployments have multiple BSSIDs where 802.11r coordinates
session keys amongst access points.  When moving between such
access points, re-authentication ensures those APs are part of
the same network and their BSSIDs can be added to the list of BSSIDs
for this SSID.

If this is the first time connecting to that encrypted DNS server's
identity, normal [DNR] or [DDR] validation procedures are followed, 
which will authenticate and authorize that encrypted DNS server's identity.

Better authentication can be performed by verifying the encrypted DNS
server's certificate with the fingerprint provided in an extended WiFi
QR code ({{qr}}), consulting a crowd-sourced database, reputation
system, or -- perhaps best -- using a matching SSID and SubjectAltName
described in {{avoid-tofu}}.

After this step, the relationship of SSID, BSSID, encrypted resolver
discovery mechanism, and SubjectAltName are stored on the client.


For illustrative purpose, below is an example of the data stored for
two WiFi networks, "Example WiFi" (showing one BSSID) and "Example2 WiFi"
(showing two BSSIDs),

~~~
   { "networks": [{
        "SSID": "Example WiFi",
        "BSSID": ["d8:c7:c8:44:32:40"],
        "Discovery": "DNR",
        "Encrypted DNS": "resolver1.example.com"
   },{
        "SSID": "Example2 WiFi",
        "BSSID": ["d8:c7:c8:44:32:49", "d8:c7:c8:44:32:50"],
        "Discovery": "DDR",
        "Encrypted DNS": ["8.8.8.8","1.1.1.1"]   }]}
~~~

If this is not the first time connecting to this same SSID then the WiFi
network name, BSSID, encrypted resolver disovery mechanism and
encrypted DNS server's identity should all match for this
re-connection.  If the encrypted DNS server's identity differs, this
indicates a different network than expected -- either a different
network (that happens to also use the same SSID), change of the
network's encrypted DNS server identity, or an Evil Twin
attack.  The client can then take appropriate action.

# Avoiding Trust on First Use {#avoid-tofu}

Trust on First Use can be avoided if the SSID name and DNS server's
Subject Alt Name match.  Unfortunately such a constraint disallows
vanity SSID names.  Also, social engineering attacks gain additional
information if the network's physical address
(123-Main-Street.example.net) or name (John-Jones.example.net) is
included as part of the SSID.  Thus the only safe SSID name provides
no information to assist social engineering attacks such as a customer
number (customer-123.example.net), assuming the customer number can
safely be disclosed to neighbors.  Such attacks are not a concern in
deployments where the network name purposefully includes the business
name or address (e.g., Public WiFi hotspots;
123-Main-Street.example.com, coffee-bar.example.com).


# Security Considerations

The network-designated resolver may or may not be local to the network.
DDR is useful in deployments where the local network cannot be upgraded
to host a encrypted resolver and the CPE cannot be upgraded to support DNR.
For example, DDR is typically used to discover the ISP's encrypted resolver
or a public encrypted resolver. The encrypted resolver discovered using DNR may
be a public encrypted resolver or hosted by the local network or by the ISP.
The proposed mechanism in this specification does not assist the client to
identity if the network-designated resolver is hosted by the local network.
However, it significantly reduces the attacker's capabilities if the attacker
is operating a look-alike network.

In near future, content delivery networks, sensitive domains and
endpoints will migrate to TLS 1.3 and ECH.  If the attacker's network
conveys the same encrypted revolver's identity as the legitimate
network, it will not have any visibility into the private and
sensitive information about the target domain. However, the attacker's
network will have visibility into the traffic metadata like
destination IP address, sequence of packet lengths and inter-
arrival times etc.

The network authentication mechanism relies on an attacker's inability
to obtain a Web PKI certificate for the victim's domain name.



# IANA Considerations

This document has no IANA actions.


--- back

# Extending WiFi QR Code {#qr}

This section is non-normative and merely explains how extending the WiFi QR code could work.  QR codes come with their
own security risks, most signficant that an attacker can place their own QR code over a legitimate QR code.

Several major smartphone operating systems support a QR code with the following format for the SSID "example" with WPA-PSK "password",

~~~
WIFI:T:WPA;S:example;P:password;;
~~~

This could be extended to add a field containing the fingerprint of the encrypted DNS server's identity.
As several DNS servers can be included in the QR code with "D:", each DNS server with its own identity
using [RFC8792] line folding,

~~~
WIFI:T:WPA;S:example;P:password; \
D:df81dfa6b61eafdffffe1a250240db5d2e6cee25, \
D:28b236db27ff688f919b171e59e2fab81f9e4f2e;;
~~~



# Acknowledgments
{:numbered="false"}

This document was inspired by both Paul Wouters and Tommy Pauly during review of other documents.


