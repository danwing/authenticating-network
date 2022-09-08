---
title: "Asserting Wireless Network Connections Using DNS Revolvers' Identities"
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
  scrypt:  RFC7914
  AKA:    I-D.ietf-emu-aka-pfs

--- abstract

This document describes how a host uses the encrypted DNS server identity to reduce
an attacker's capabilities if the attacker is emulating a wireless network. 

--- middle


# Introduction

When a user connects to a wireless network the user
or their device want to be sure the connection is to the expected
network, as different networks provide different services in terms of
performance, security, access to split-horizon DNS servers, and so on.  Although 802.1X provides layer 2
security for both Ethernet and Wi-Fi networks, 802.1X is not widely deployed
and unavailable on LTE and 5G networks -- and often applications are
unaware if the underlying network was protected with 802.1X.

An attacker can operate a rogue WLAN access point
with the same SSID and WPA-PSK as the victim network [Evil-Twin].  Also, 
there are many deployments (for example, coffee shops and bars) that offer free Wi-Fi
as a customer incentive.  Since these businesses are not
Internet service providers, they are often unwilling and/or
unqualified to perform advanced (sometimes, complex) configuration on their network.  In
addition, customers are generally unwilling to do complicated
provisioning on their devices just to obtain free Wi-Fi.  This leads
to a popular deployment technique -- a network protected using a
shared and public Pre-Shared Key (PSK) that is printed on a sandwich
board at the entrance, on a chalkboard on the wall or on a menu.  The
PSK is used in a cryptographic handshake, defined in [IEEE802.11],
called the "4-way handshake" to prove knowledge of the PSK and derive
traffic encryption keys for bulk wireless data. The same deployement
technique is typically used in residential or small office/home office
networks. If the PSK for wireless authentication is
the same for all clients that connect to the same WLAN, the shared key
will be available to all nodes, including attackers, so it is possible
to mount an active on-path attack.

This document describes how a wireless client can utilize
network-advertised encrypted DNS servers to ensure that the attacker has no
more visibility to the client's DNS traffic than the legitimate
network. In cases where the local network provides its own encrypted
DNS server, the client can even ensure it has re-connected to the same
network, offering the client enough information to positively detect a
significant change in the encrypted DNS server configuration -- a
strong indicator of an attacker operating the network.  

The proposed
mechanism is also useful in deployments using Opportunistic Wireless
Encryption [RFC8110] and in LTE/5G mobile networks where the long-term
key in the SIM card on the UE can be compromised (Section 1 of [AKA]).

The theory of operation is described mainly from the perspective of a host that connects to a network. Further interactions may be considered
to seek for specific actions from a user (e.g., consent, validation). Whether and how such interactions are supported is implementation-specific and are, as such,
out of scope. 

The document assumes that the host supports at least one encrypted DNS scheme (e.g., DNS over TLS or DNS over HTTPS).

The current version of the specification focuses on wirless networks. The applicability to other network types may be assessed in future versions. 

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Theory of Operation

A host connects to a network and obtains network-related information via DHCPv4, DHCPv6, or RA.
The network indicates its encrypted DNS server using either [DNR] or [DDR]. If hosts supports an encrypted DNS scheme that is advertised by the network, the host then connects
to at least one of the designated encrypted DNS servers, completes the TLS handshake, and performs public key validation of
the presented certificate following conventional procedures.

The host can associate the network name with the encrypted DNS server's
identity that was learned via [DNR] or [DDR]. The type of the network name is
dependent on the access technology to which the host is attached.
For networks based on IEEE 802.11, the network name will be the SSID of
the network and the PSK. The PSK is used along with SSID to uniquely identify the network to deal with common Wi-Fi names such as
"Airport WiFi" or "Hotel WiFi" or "Guest" but are distinct networks with different PSK. The combination of SSID and PSK is useful
in deployments where the same Wi-Fi name is used in many locations around the world, such as branch offices of a corporation.
It is also useful in Wi-Fi deployments that have multiple Basic Service Set Identifiers (BSSIDs) where 802.11r coordinates session keys amongst access points.
However, in deployments using Opportunistic Wireless Encryption, the network name will be the SSID of the network
and BSSID. For 3GPP access-based networks, it
is the Public Land-based Mobile Network (PLMN) Identifier of the access network, and for 3GPP2 access,
the network name is the Access-Network Identifier (see [RFC7839]).

If DDR is used for discovery, the host would have to perform verified discovery as per Section 4.2 of [DDR]
and the encrypted DNS server identity will be the encrypted DNS server's IP address.

If DNR is used, the encrypted DNS server identity will be the Authentication Domain Name (ADN).

If this is the first time the host connects to that encrypted DNS server, the hosts follows [DNR] or [DDR] validation procedures,
which will authenticate and authorize that encrypted DNS server's identity.

Better authentication can be performed by verifying the encrypted DNS
server's certificate with the fingerprint provided in an extended Wi-Fi
QR code ({{qr}}), consulting a crowd-sourced database, reputation
system, or -- perhaps best -- using a matching SSID and SubjectAltName
described in {{avoid-tofu}}.

After this step, the relationship of SSID, PSK, encrypted resolver
discovery mechanism, and SubjectAltName are stored on the host.

For illustrative purposes, {{example}} provides an example of the data stored for
two Wi-Fi networks, "Example WiFi 1" and "Example WiFi 2" (showing hashed
PSK),

~~~
{
  "networks": [
    {
      "SSID": "Example WiFi 1",
      "PSK": "7786ff815d75063c530608d0aa87e405bfb999dde9d594754358b2d0",
      "Discovery": "DNR",
      "Encrypted DNS": "resolver1.example.com"
    },
    {
      "SSID": "Example WiFi 2",
      "PSK": "75K3eLr+dx6JJFuJ7LwIpEpOFmwGZZkRiB84PURz6U8=",
      "Discovery": "DDR",
      "Encrypted DNS": [
        "192.0.2.1",
        "198.51.100.5"
      ]
    }
  ]
}
~~~
{: #example title="An Example of Data Stored for Two Networks"}

If this is not the first time the host connects to this same SSID, then the Wi-Fi
network name, PSK, encrypted resolver disovery mechanism, and
encrypted DNS server's identity should all match for this
re-connection.  If the encrypted DNS server's identity differs, this
indicates a different network than expected -- either a different
network (that happens to also use the same SSID), change of the
network's encrypted DNS server identity, or an Evil Twin
attack. The host and/or the user can then take appropriate actions.

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
The mechanism specified in this document does not assist the client to
identity if the network-designated resolver is hosted by the local network.
However, it significantly reduces the attacker's capabilities if the attacker
is emulating a network (that is, operating a look-alike network).

More and more content delivery networks, sensitive domains and
endpoints are migrating to TLS 1.3 and ECH.  If the attacker's network
conveys the same encrypted revolver's identity as the legitimate
network, it will not have any visibility into the private and
sensitive information about the target domain. However, the attacker's
network will still have visibility into the traffic metadata like
the destination IP address, sequence of packet lengths, inter-
arrival times, etc.

The network authentication mechanism relies upon an attacker's inability
to obtain an application PKI certificate for the victim's configured encrypted DNS
server.

The plain-text PSK is not necessary for the validation procedure described in this
document; rather, an implementation can use a key identifier or password-based
KDF.

# IANA Considerations

This document has no IANA actions.


--- back

# Extending WiFi QR Code {#qr}

This section is non-normative and merely explains how extending the Wi-Fi QR code could work.  

QR codes come with their
own security risks, most signficant that an attacker can place their own QR code over a legitimate QR code.

Several major smartphone operating systems support a QR code with the following format for the SSID "example" with WPA-PSK "password",

~~~
WIFI:T:WPA;S:example;P:password;;
~~~

This could be extended to add a field containing the fingerprint of the encrypted DNS server's identity.
As several DNS servers can be included in the QR code with "D:", each DNS server with its own identity
using [RFC8792] line folding,

~~~
=============== NOTE: '\' line wrapping per RFC 8792 ================

WIFI:T:WPA;S:example;P:password; \
D:df81dfa6b61eafdffffe1a250240db5d2e6cee25, \
D:28b236db27ff688f919b171e59e2fab81f9e4f2e;;
~~~



# Acknowledgments
{:numbered="false"}

This document was inspired by both Paul Wouters and Tommy Pauly during review of other documents.


