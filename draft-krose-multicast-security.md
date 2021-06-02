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
    street: 145 Broadway
    city: Cambridge, MA 02144
    country: United States of America
    email: krose@krose.org
 -
    ins: J. Holland
    name: Jake Holland
    organization: Akamai Technologies, Inc.
    street: 145 Broadway
    city: Cambridge, MA 02144
    country: United States of America
    email: jakeholland.net@gmail.com

normative:
    RFC2119:

informative:

    RFC4082:

    RFC4607:

    RFC8826:

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

    webtrans:
        title: "The WebTransport Protocol Framework"
        date: "18 October 2020"
        seriesinfo:
            Internet-Draft: draft-ietf-webtrans-overview-01
        author:
            ins: V. Vasiliev
            name: Victor Vasiliev
            org: Google, Inc.


--- abstract

Interdomain multicast has unique potential to solve delivery scalability for popular content, but it carries a set of security and privacy issues that do not affect unicast delivery.
These merit a full analysis of the risks and mitigations before a determination can be made about whether interdomain multicast can reasonably fit within the Web security model.


--- middle

# Introduction

The Web security model, while not yet documented authoritatively in a single reference, has generally been interpreted to require certain properties of underlying transports, such as:

* Confidentiality: A passive observer must not be able to identify or access content through simple observation of the bits being delivered, up to the limits of metadata privacy (such as traffic analysis, peer identity, application/transport/security-layer protocol design constraints, etc.).
* Authenticity: A receiver must be able to cryptographically verify that the delivered content originated from the desired source.
* Integrity: A receiver must be able to distinguish between original content as sent from the desired source and content modified in some way (including through deletion) by an attacker.
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

## Authenticity and Integrity

At its most general, the goal of authentication is to ensure that data being processed is genuine, while the goal of integrity protection is to ensure that the data is unmodified and complete and (where appropriate) to expose any exceptions to applications for further handling.
In practice, the mechanisms for these two functions overlap to a high degree, so we address them together.

In the context of unicast transport security, authentication means that content is known to have originated from the trusted peer, something that is typically enforced via a cryptographic authentication tag:

* With a symmetric message authentication code (MAC), this assumes only the two parties to the communication have access to the keying material.
* With an asymmetric signature (that may be repeated to multiple clients by the same origin, and potentially even relayed through otherwise untrusted parties), this assumes that only the sender has access to the signing key.

In both cases:

* The receiving party must have a means for establishing trust in the keying material used to verify the authentication tag.
* The authentication tag serves to provide integrity protection over the unit of content to which the tag applies, with additional mechanisms required to provide protection against duplication (replay), deletion, and reordering.

Asymmetric signing of content delivered through multicast is identical to the unicast case, owing to the asymmetry of access to the signing key; but the symmetric MAC case does not directly apply given that multiple receivers need access to the same key used for both signing and verification, which opens up the possibility of forgery by a receiver on-path or with the ability to spoof the source.

Multiple mechanisms providing for reliable asymmetric authentication of data delivered by multicast have been proposed over the years.

* TESLA {{RFC4082}} employs computationally-inexpensive symmetric authentication via time-released keys at the costs of requiring loose time synchronization between clients and servers and of imposing latency above one-way path delay prior to release of authenticated data to applications.
Effectively, TESLA achieves asymmetry between the sender and multiple receivers through timed release of keying material rather than through the computational difficulty of deriving a signing key from a verification key.
TODO: Insert something about the downsides of TESLA here.

* Asymmetric Manifest-based Integrity (AMBI) {{AMBI}}, in contrast with TESLA, assumes the existence of an out-of-band, authenticated channel for distribution of manifests containing cryptographic digests of those packets.
Authentication of this channel may, for instance, be provided by TLS if manifests are distributed using HTTPS from an origin known to the client to be closely affiliated with the multicast stream, such as would be the case if the manifest URL is delivered by the origin of the parent page hosting the media object.
Asymmetry in this case is a prerequisite of the out-of-band channel rather than a property provided by the AMBI protocol itself.

Regardless of mechanism, however, the primary goal of authentication in the multicast context is identical to that for unicast:
that the content delivered to the application originated from the trusted source.
Semantic equivalence to (D)TLS in this respect is therefore straightforwardly achieved by any number of potential mechanisms.

Integrity is similarly partially provided by the authentication mechanism.
As multicast is not connection-oriented at the transport layer (e.g., multicast protocols typically employ UDP), applications using multicast must already mitigate or tolerate duplication/replay, loss/deletion, and reordering, irrespective of authenticity.


## Confidentiality

In the unicast transport security context, confidentiality implies that an observer (passive or active) without pre-existing access to keying material must not be able to decrypt the bytes on the wire or identify the content being transferred, even if that adversary has access to the decrypted content via other means.
In practice, the former is trivially achieved through the use of authenticated key exchange and modern symmetric ciphers, but the latter is an ideal that is rarely possible owing to the substantial metadata in the clear on the public Internet:
traffic analysis can make use of packet sizes and timing, endpoint identities, biases in application-layer protocol designs, side channels, and other such metadata to reveal an often surprising amount of information about the encrypted payload without needing access to any keying material.

Multicast additionally introduces the complication that all receivers of a stream, even if encrypted, receive the same payload (loss and duplication notwithstanding). This introduces novel privacy concerns that do not apply to unicast transports.

### Privacy

In contrast to (say) unicast TLS, on-path monitoring can trivially prove that identical content was delivered to multiple receivers, irrespective of payload encryption.
Furthermore, since those receivers all require the same keying material to decrypt the received payload, a compromise of any single receiver's device exposes decryption keys, and therefore the plaintext content, to the attacker.

That having been said, however, there are factors and practices that help mitigate these additional risks:

* Multicast delivery is unidirectional from content provider to consumer and has no end-to-end unicast control channel association at the transport-layer, though such associations are possible and even likely at the application layer.
Assuming application-layer unicast control plane traffic is properly-secured, identifiable plaintext control messages are limited to IGMP messages intercepted by (and not retransmitted verbatim by) a user's upstream router.
Notwithstanding linkability via data or metadata from application-layer control flows, a passive observer with monitoring capability at a particular path element can thus only directly determine that some entity served by that path element has joined a particular multicast channel (in SSM {{RFC4607}}, identified by the (source, group) pair of IP addresses).
Increasing the specificity of user identification would require a monitoring point closer to the user or some way to manipulate a user into revealing metadata out-of-band that the observer can tie to the user via traffic analysis or other means.

* There is in general no way to link a multicast channel to other related content:
in particular, if a multicast stream is encrypted using a key delivered out-of-band, there is no standard mechanism in the multicast protocol ecosystem for identifying the location of the keying material.
Thus, for a passive observer to know what content is actually being consumed, beyond simply identifying the multicast channel or metadata such as the cohort of possible group members revealed by tracking channel membership at the observer's set of network tap points, they would need to already know what content is available via that channel, either via traffic analysis such as in the case of passive observation of unicast TLS, or via access to the related content that references the channel. TODO: This explanation needs to be made comprehensible.
A dragnet cataloging all content available through a particular origin would defeat this, but could be further mitigated via controlled access to index information, or via periodic changes in multicast source, group, or keying material or some combination of the three.

* As covered in {{personaldata}}, multicast is not generally suitable for transport of personal data: consequently, while the attack surface of a pool of receivers increases with the number of receivers, the corresponding impact of a compromise goes down as more broadly-distributed content is less likely to contain revelatory information. TODO: This doesn't account for the dissidents-watching-a-prohibited-stream scenario.

### Personal Data {#personaldata}

A sender has responsibility not to expose personal information broadly. This is not a consideration unique to multicast delivery:
an irresponsible service could publish a web page with Social Security numbers or push its server TLS private key into the certificate transparency log as easily as it could multicast personal data to a large set of receivers.

The Web security model partially mitigates irresponsible senders by mandating the use of secure transports:
prohibiting the fetching of mixed content on a single page prevents a server from sending private data to a browser in the clear.
The main effect is to raise the bar closer to requiring bad faith on the part of senders in revealing personal information.

Multicast by its very nature is not generally suitable for transport of personal data:
since the main value of leveraging a multicast transport is to deliver the same data to a large pool of receivers, such content must not include confidential personal information.
Senders already have a responsibility to handle private information in a way that respects the privacy of users:
the availability of multicast transports does not further complicate this responsibility.

### Forward Secrecy

TODO: Confidentiality of past payloads, e.g. via key rotation.


## Non-linkability

TODO: Re: passively linking a user across user agents or roaming devices.


## Browser-Specific Threats

Shamelessly paraphrasing {{RFC8826}}, the security requirements for multicast transports to a browser follow directly from the requirement that the browser's job is to protect the user.

> Huang et
> al. [huang-w2sp] summarize the core browser security guarantee as
> follows:
>
>> Users can safely visit arbitrary web sites and execute scripts
>> provided by those sites.

The reader will find the full discussion of the browser threat model in section 3 of {{RFC8826}} helpful in understanding what follows.

### Access to Local Resources

This document covers only unidirectional multicast from a server to (potentially many) clients, as well as associated control channels used to manage that communication and access to the content delivered via multicast. As a result, local resource access can be presumed to be limited to that already available within web applications. Note that these resources may include information that can be used to identify or track individuals, such as information about the user agent, viewport size, display resolution, but TODO: reference some other guidance on how this kind of PII is to be handled.

### Injection

### Hostile Origin



# Security Considerations

This entire document is about security.


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acks