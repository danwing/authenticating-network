---
title: "Authenticating a Network Connection"
abbrev: "Authenticating a Network Connection"
category: info

docname: draft-wing-authenticating-network-latest
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
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
network-advertised encrypted DNS servers to ensure the attacker has
no more visibility to the client's DNS traffic than the legitimate
network.  In cases where the local network provides its own
encrypted DNS server, the client can even ensure it has re-connected
to the same network, offering the client enough information to
positively detect a significant change in the encrypted DNS server
configuration -- a strong indicator of an attacker operating the
network.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Procedure

Client connects to network and obtains network information via DHCPv4, DHCPv6, or RA.
The network indicates its encrypted DNS server using either [DNR] or [DDR].  The client connects
to that encrypted DNS server, completes the TLS handshake and performs public key validation of
the presented certificate as normal.

The client can associate the WiFi network name (SSID) and the Basic
Service Set Identifier (BSSID) with the encrypted DNS server's
identity (TLS SubjectAltName) that was learned via [DNR] or [DDR].

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
identity, normal [DNR] or [DDR] procedures are followed, which will
authenticate and authorize that encrypted DNS server's identity.

Better authentication can be performed byverifying the encrypted DNS
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
network (that happens to also use the same SSID) or an Evil Twin
attack.  The client can then take appropriate action.


> todo: if a network advertises 8.8.8.8 via DNR or DDR, we can't
    detect an evil twin.  How do we identify 8.8.8.8 as a public DNS
    server?  Could we say the network-advertise encrypted DNS server
    has to be on the same network (RFC1918 space or same /64) as the
    client obtained??  But that seems somewhat constraining.  Another
    idea is "if you've seen this same certificate via another SSID",
    but that doesn't work well, either:  for example, I have a single
    DNS server in my house for all of my various SSIDs (guest, IoT,
    home network, work network).  Hmm.  Need more ideas.


# Avoiding Trust on First Use {#avoid-tofu}

Trust on First Use can be avoided if the SSID name and DNS server's
Subject Alt Name match.  Unfortunately such a constraint disallows
vanity SSID names.  Also, social engineering attacks gain additional
information if the network's physical address
(123-Main-Street.example.net) or name (John-Jones.example.net) is
included as part of the SSID.  Thus the only safe SSID name provides
no information to assist social engineering attacks such as a customer
number (customer-123.example.net), assuming the customer number can
safely be disclosed to neighbors.  Such attacks are not a concern
in deployments where the network name purposefully includes the
business name or address (e.g., 123-Main-Street.example.com,
coffee-bar.example.com).


# Common WiFi Names

(( probably want to delete this section ))

Some WiFi names are pretty common such as "Airport WiFi" or "Hotel WiFi"
or "Guest" but are distinct networks with different WPA-PSK or are not
using security at all ("open" networks).

In other deployments, the same WiFi name is used in many locations
around the world, such as branch offices of a corporation.




# Security Considerations

The network authentication mechanism relies on an attacker's inability
to obtain a signed certificate for the victim's domain name.



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

This could be extended to add a field containing the fingerprint of the encrypted DNS server
certificate.  As several DNS servers can be included in the QR code with "D:", each DNS server with
its own certificate using [RFC8792] line folding,

~~~
WIFI:T:WPA;S:example;P:password; \
D:df81dfa6b61eafdffffe1a250240db5d2e6cee25, \
D:28b236db27ff688f919b171e59e2fab81f9e4f2e;;
~~~



# Acknowledgments
{:numbered="false"}

This document was inspired by IETF review comments from Paul Wouters and Wes Eddy.

