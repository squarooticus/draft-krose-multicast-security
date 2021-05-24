---
docname: draft-krose-multicast-security-latest
title: Security and Privacy Considerations for Multicast Transports
abbrev: Multicast Transport Security
category: std

ipr: trust200902
area: General
workgroup: TODO Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: K. Rose
    name: Kyle Rose
    organization: Akamai Technologies, Inc.
    email: krose@krose.org

normative:
    RFC2119:

informative:

    webtrans:
        title: "The WebTransport Protocol Framework"
        date: "18 October 2020"
        seriesinfo:
            Internet-Draft: draft-ietf-webtrans-overview-01
        author:
            ins: V. Vasiliev
            name: Victor Vasiliev
            org: Google, Inc.

    RFC4082:

    AMBI:
        title: "Asymmetric Manifest-based Integrity"
        date: "30 October 2020"
        seriesinfo:
            Internet-Draft: draft-ietf-mboned-ambi-01
        author:
            -
                ins: J. Holland
                name: Jake Holland
                org: Akamai Technologies, Inc.
            -
                ins: K. Rose
                name: Kyle Rose
                org: Akamai Technologies, Inc.


--- abstract

Interdomain multicast has unique potential to solve delivery scalability for popular content, but it carries a set of security and privacy issues that do not affect unicast delivery.
These merit a full analysis of the risks and mitigations before a determination can be made about whether interdomain multicast can reasonably fit within the Web security model.


--- middle

# Introduction

The Web security model, while not yet documented authoritatively in a single reference, has generally been interpreted to require certain properties of underlying transports, such as:

* Confidentiality: A passive observer must not be able to identify or access content through simple observation of the bits being delivered, up to the limits of metadata privacy (such as traffic analysis, peer identity, application-layer protocol design, etc.).
* Authenticity: A receiver must be able to cryptographically verify that the delivered content originated from the desired source.
* Non-linkability: A passive observer must not be able to link a single user across multiple devices or a single client roaming across multiple networks.

Web Transport {{webtrans}}, for instance, proposes to interpret these requirements in part to mean that any qualifying transport protocol "MUST use TLS or a semantically equivalent security protocol".
For unicast communication this is sensible and meaningful (if imprecise) for an engineer with a grounding in security, but it is unclear how or whether 'semantic equivalence to TLS' can be directly interpreted in any meaningful way for multicast transport protocols.
This document instead explicitly describes a security and privacy threat model for multicast transports as a proposal to extend the Web security model to accommodate multicast delivery in a way that fits within the spirit of how that model is generally interpreted for unicast.


# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# Multicast Transport Properties

By contrast with bidirectional unicast transports, multicast transports are necessarily unidirectional, connectionless, unreliable, and not congestion-controlled.


# Threat Model

## Authenticity

In general, the goal of authentication is to ensure that data being processed is genuine.
In the context of unicast transport security, this means that content is known to have originated from the trusted peer, something that is typically enforced via a cryptographic authentication tag:

* With a symmetric message authentication code (MAC), this assumes only the two parties to the communication have access to the keying material.
* With an asymmetric signature (that may be repeated to multiple clients by the same origin, and potentially even relayed through otherwise untrusted parties), this assumes that only the sender has access to the signing key.

In both cases, the receiving party must have a means for establishing trust in the keying material used to verify the authentication tag.

Asymmetric authentication of content delivered through multicast is identical to the unicast case, owing to the asymmetry of access to the signing key; but the symmetric authentication case does not directly apply given that multiple receivers need access to the same key used for both signing and verification, which opens up the possibility of forgery by a receiver on-path or with the ability to spoof the source.

Multiple mechanisms providing for symmetric authentication of data delivered by multicast have been proposed over the years.
TESLA {{RFC4082}} employs computationally-inexpensive symmetric authentication via time-released keys at the costs of requiring loose time synchronization between clients and servers and of imposing latency above one-way path delay prior to release of authenticated data to applications.
Effectively, TESLA achieves asymmetry between the sender and multiple receivers through timed release of keying material rather than through the computational difficulty of deriving a signing key from a verification key.

Asymmetric Manifest-based Integrity (AMBI) {{AMBI}}, by contrast to TESLA, assumes the existence of an out-of-band, authenticated channel for distribution of manifests containing cryptographic digests of those packets.
Authentication of this channel may, for instance, be provided by TLS if manifests are distributed using HTTPS from an origin known to the client to be closely affiliated with the multicast stream, such as would be the case if the manifest URL is delivered by the origin of the parent page hosting the media object.
Asymmetry in this case is a prerequisite of the out-of-band channel rather than a property of the AMBI protocol itself.

Regardless of mechanism, however, the primary goal of authentication in the multicast context is identical to that for unicast:
that the content delivered to the application originated from the trusted source.
Semantic equivalence to TLS is therefore straightforwardly achieved by any number of potential mechanisms.


## Confidentiality

In the unicast transport security context, confidentiality implies that a passive observer without access to keying material must not be able to decrypt the bytes on the wire or identify the content being transferred, even if that adversary has access to the decrypted content via other means.
In practice, the former is trivially achieved through the use of key exchange and modern symmetric ciphers, but the latter is an ideal that is rarely possible owing to the substantial metadata in the clear on the public Internet.
Traffic analysis can make use of packet sizes and timing, endpoint identities, biases in application-layer protocol designs, side channels, and other such metadata to reveal an often surprising amount of information about the encrypted payload without needing access to any keying material.

To this problem, multicast introduces the complication that all receivers of a stream, even if encrypted, receive the same payload (loss and duplication notwithstanding).
As a result, if a passive observer knows what the content is and can observe the same payload being delivered to another receiver, the observer knows what content the receiver is consuming.

That having been said, however, there are factors and practices that help mitigate this limitation:

* Multicast delivery is one-way and has no protocol-level end-to-end unicast control channel association.
Identifiable plaintext control messages are limited to IGMP messages intercepted by (and not retransmitted verbatim by) a user's upstream router.
A passive observer with access to tap a particular path element can thus only directly determine that some entity served by that path element has joined a particular SSM multicast source and group (hereafter "(S,G)"):
increasing the specificity of user identification would require a tap point closer to the user or some way to manipulate a user into revealing metadata out-of-band that the observer can tie to the user via traffic analysis or other means.

* There is in general no way to link a multicast (S,G) to other related content:
in particular, if a multicast stream is encrypted using a key delivered out-of-band, there is no standard mechanism in the multicast protocol ecosystem for identifying the location of the keying material.
Thus, for a passive observer to know what content is actually being consumed, beyond simply identifying the multicast (S,G) or metadata such as the cohort of possible group members revealed by tracking (S,G) membership at the observer's set of network tap points, they would need to already know what content is available via that (S,G), either via traffic analysis such as in the case of passive observation of unicast TLS, or via access to the related content that references the (S,G). TODO: This explanation needs to be seriously cleaned up.
A dragnet cataloging all content available through a particular origin would defeat this, but could be further mitigated via controlled access to index information, or via periodic changes in multicast group or keying material or both.


## Non-linkability


## Forward Secrecy


# Security Considerations

This entire document is about security.


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acks
