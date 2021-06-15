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
    RFC3552:
    RFC4949:
    RFC4607:
    RFC7258:
    RFC8085:
    RFC8174:
    RFC8815:
    RFC9000:

informative:

    RFC4082:
    RFC6584:
    RFC8446:
    RFC8826:
    AMBI:
        I-D.draft-ietf-mboned-ambi
    webtrans:
        I-D.draft-ietf-webtrans-overview

    huang-w2sp:
        title: "Talking to Yourself for Fun and Profit"
        date: "May 2011"
        seriesinfo:
            Web 2.0 Security and Privacy (W2SP 2011)
        author:
            -
                ins: L-S. Huang
            -
                ins: E.Y. Chen
            -
                ins: A. Barth
            -
                ins: E. Rescorla
            -
                ins: C. Jackson
        target: https://ptolemy.berkeley.edu/projects/truststc/pubs/840/websocket.pdf

--- abstract

Interdomain multicast has unique potential to solve delivery scalability for popular content, but it carries a set of security and privacy issues that differ from those in unicast delivery.
This document analyzes the security threats unique to multicast-based delivery for Internet and Web traffic under the Internet and Web threat models.

--- middle

# Introduction

This document examines the security considerations relevant to the use of multicast for scalable one-to-many delivery of application traffic over the Internet, along with special considerations for multicast delivery to clients constrained by the Web security model.

## Background

This document assumes readers have a basic understanding of some background topics, specifically:

 * The Internet threat model as defined in Section 3 of {{RFC3552}}.

 * The Security Considerations for UDP Usage Guidelines as described in Section 6 of {{RFC8085}}, since application layer multicast traffic is generally carried over UDP.

 * Source-specific multicast, as described in {{RFC4607}}.  This document focuses on interdomain multicast, therefore any-source multicast is out of scope in accordance with the deprecation of interdomain any-source multicast in {{RFC8815}}.

## Web Security Model

The Web security model, while not yet documented authoritatively in a single reference, nevertheless strongly influences Web client implementations, and has generally been interpreted to require certain properties of underlying transports such as:

 * Confidentiality: A passive observer must not be able to identify or access content through simple observation of the bits being delivered, up to the limits of metadata privacy (such as traffic analysis, peer identity, application/transport/security-layer protocol design constraints, etc.).
 * Authenticity: A receiver must be able to cryptographically verify that the delivered content originated from the desired source.
 * Integrity: A receiver must be able to distinguish between original content as sent from the desired source and content modified in some way (including through deletion) by an attacker.
 * Non-linkability: A passive observer must not be able to link a single user across multiple devices or a single client roaming across multiple networks.

For unicast transport, TLS {{RFC8446}} satisfies these requirements, therefore Web Transport {{webtrans}} proposes to require qualifying transport protocols to use "TLS or a semantically equivalent security protocol".

For unicast communication this is sensible and meaningful (if imprecise) for an engineer with a grounding in security, but it is unclear how or whether 'semantic equivalence to TLS' can be directly interpreted in any meaningful way for multicast transport protocols.
This document instead explicitly describes a security and privacy threat model for multicast transports in order to extend the Web security model to accommodate multicast delivery in a way that fits within the spirit of how that model is generally interpreted for unicast.

Although defining the security protections necessary to make multicast traffic suitable for Web Transport is a key goal for this document, many of the security considerations described here would be equally necessary to consider if a higher level multicast transport protocol were used for a more specific use case, for delivery to clients constrained by the Web security model.


# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# Multicast Transport Properties

By contrast with bidirectional unicast transports, multicast transport at the IP or UDP layer is necessarily unidirectional, connectionless, and unreliable.

Although applications compliant with Section 4.1 of {{RFC8085}} will implement congestion control, in the context of a threat model it's important to note that malicious clients might attempt to use non-compliant subscriptions to multicast traffic as part of a DoS attack where possible, and that some applications might not be compliant with the recommendations for congestion control implementations.

TODO: maybe some mention that higher-level protocols can provide reliability (e.g. 5740 (norm)/5775 (alc) and/or flute(6726)/fcast(6968) that use alc, plus draft-pardue-quic-http-mcast), but this may introduce different risks from spoofing if per-packet authenticity is not provided.  For example, in an application with unicast recovery for a reliable object that's constructed out of 1k packets, injecting a single spoofed packet amplifies by 1k * number of receivers, if they all recover the whole reliable object.  Also: determine whether this mention belongs in this section--maybe better as a ref to a different section here?


# Threat Model

At its most general, the goal of authentication is to ensure that data being processed is genuine, while the goal of integrity protection is to ensure that the data is unmodified and complete and (where appropriate) to expose any exceptions to applications for further handling.
In practice, the mechanisms for these two functions overlap to a high degree, so we address them jointly.


## Authentication

The Web security model requires that content be authenticated cryptographically.
In the context of unicast transport security, authentication means that content is known to have originated from the trusted peer, something that is typically enforced via a cryptographic authentication tag:

* Symmetric tags, such as symmetric message authentication codes (MACs) and authentication tags produced by authenticated encryption algorithms.
Because anyone in possession of the keying material may produce valid symmetric authentication tags, such keying material is typically known to at most two parties:
one sender and one receiver.
Some algorithms (such as TESLA, discussed below) relieve this constraint by imposing some different constraint on verification of tagged content.

* Asymmetric tags, typically signatures produced by public key cryptosystems.
These assume that only the sender has access to the signing key, but impose no constraints on dissemination of the signature verification key.

In both cases:

* The receiving party must have a means for establishing trust in the keying material used to verify the authentication tag.

* Instead of directly authenticating the protected content, the tag may protect a root of trust that itself protects cryptographically-linked content.
Examples include:

    * The TLS 1.3 handshake employing an authentication tag to reject MitM attacks against ECDH key agreement.

    * An authentication tag of a Merkle tree root protecting the content represented by the entire tree.

* The authentication tag serves to provide integrity protection over the unit of content to which the tag applies, with additional mechanisms required to detect and/or manage duplication/replay, deletion/loss, and reordering within a sequence of such authenticated content units.

Asymmetric verification of content delivered through multicast is conceptually identical to the unicast case, owing to the asymmetry of access to the signing key;
but the symmetric case does not directly apply given that multiple receivers need access to the same key used for both signing and verification, which in a naïve implementation opens up the possibility of forgery by a receiver on-path or with the ability to spoof the source.

Multiple mechanisms providing for reliable asymmetric authentication of data delivered by multicast have been proposed over the years.

* TESLA {{RFC4082}} achieves asymmetry between the sender and multiple receivers through timed release of symmetric keying material rather than through the assumed computational difficulty of deriving a signing key from a verification key in public key cryptosystems like RSA and ECDSA.
It employs computationally-inexpensive symmetric authentication tagging with release of the keying material to receivers only after they are assumed to have received the protected data, with any data received subsequent to scheduled key release to be discarded by the receiver.
This requires some degree of time synchronization between clients and servers and imposes latency above one-way path delay prior to release of authenticated data to applications.

* Simple per-packet asymmetric signature of packet contents based on out-of-band communication of the signature's public key and algorithm, for example as described in Section 3 of {{RFC6584}}.

* Asymmetric Manifest-based Integrity (AMBI) {{AMBI}} relies on an out-of-band authenticated channel for distribution of manifests containing cryptographic digests of the packets in the multicast stream.
Authentication of this channel may, for instance, be provided by TLS if manifests are distributed using HTTPS from an origin known to the client to be closely affiliated with the multicast stream, such as would be the case if the manifest URL is delivered by the origin of the parent page hosting the media object.
Authenticity in this case is a prerequisite of the out-of-band channel that AMBI builds upon to provide authenticity for the multicast data channel.

Regardless of mechanism, the primary goal of authentication in the multicast context is identical to that for unicast:
that the content delivered to the application originated from the trusted source.
Semantic equivalence to (D)TLS in this respect is therefore straightforwardly achieved by any number of potential mechanisms.


## Integrity

Integrity in the Web security model for unicast is closely tied to the features provided by transports that enabled the Web from its earliest days.
TCP, the transport substrate for the original HTTP, provides in-order delivery, reliability via retransmission, packet de-duplication, and modest protection against replay and forgery by certain classes of adversaries.
SSL and TLS later greatly strengthened those protections.
Web applications universally rely on these integrity assumptions for even the most basic operations.
It is no surprise, then, that when QUIC was subsequently designed with HTTP as the model application, initial requirements included the integrity guarantees provided by TCP at the granularity of an individual stream.

Multicast applications by contrast have different integrity assumptions owing to the multicast transport legacy.
UDP, the transport protocol atop which multicast applications are typically built, provides no native reliability, in-order delivery, de-duplication, or protection against replay or forgery.
Additionally, UDP by itself provides no protection against off-path spoofing or injection.
Multicast has therefore traditionally been used for applications that can deal with a modest loss of integrity through application-layer mitigations such as:

* Packet indexes to reveal duplication/replay and reordering, and to complicate off-path spoofing and injection
* Deletion coding to allow recovery from loss/deletion
* Graceful degradation in response to loss/deletion, exemplified by video codecs designed to tolerate loss

A baseline for multicast transport integrity that makes sense within the Web security model requires that we first define the minimally acceptable integrity requirements for data that may be presented to a user.

TODO: Dangers of manipulation without violating authentication and how to manage this.

Integrity in multicast, as in the unicast case, is partially provided by the authentication mechanism.
As multicast is not connection-oriented at the transport layer, applications relying on multicast must otherwise provide for detection and/or management of packet duplication/replay, loss/deletion, and reordering.

Some of these functions may also be provided by the authentication mechanism. For instance:

* TESLA prevents replay and reveals reordering, but only across time intervals. An application requiring finer-grained countermeasures against duplication/replay or reordering, or indeed any countermeasure to deletion/loss, would need to provide that via custom support (e.g., through the introduction of packet sequence numbers) or via an intermediate-layer protocol providing those functions.

* AMBI by design provides strong protection against duplication/replay and reveals reordering and deletion/loss of content packets through a strict in-order manifest of packet digests.


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
Assuming application-layer unicast control plane traffic is properly-secured, identifiable plaintext control messages are limited to IGMP or MLD messages intercepted by (and not retransmitted verbatim by) a user's upstream router.
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

Forward secrecy (also called "perfect forward secrecy" or "PFS" and defined in {{RFC4949}}) is a countermeasure against attackers that record encrypted traffic with the intent of later decrypting it should the communicating parties' long-term keys be compromised.
Forward secrecy for protocols leveraging time-limited keys to protect a communication session ("session keys") requires that such session keys be unrecoverable by an attacker that later compromises the long-term keys used to negotiate or deliver those session keys.

As noted earlier, confidential content delivered via multicast will necessarily imply delivery of the same keying material to multiple receivers, rather than negotiation of a unique key as is typical in the unicast case.
Presumably, such receivers will need to be individually authenticated and authorized by the content provider prior to delivery of decryption keys.
If this authorization and key delivery channel is a forward secret unicast transport such as TLS 1.3, then so long as these encryption keys are rotated and new keys distributed to clients periodically via this channel, the multicast payloads will also effectively be forward secret beyond the time interval of rotation, which we can consider to be the session duration.


## Non-linkability

Concern about pervasive monitoring of users culminated in the publication of {{RFC7258}}, which states that "the IETF will work to mitigate the technical aspects of [pervasive monitoring]."
One area of particular concern is the ability for pervasive monitoring to track individual clients across changes in network connectivity, such as being able to tell when a device or connection migrates from a wired home network to a cell network.
This has motivated mitigations in subsequent protocol designs, such as those discussed in section 9.5 of {{RFC9000}}.


## Browser-Specific Threats

The security requirements for multicast transport to a browser follow directly from the requirement that the browser's job is to protect the user.  Huang et al. [huang-w2sp] summarize the core browser security guarantee as follows:

> Users can safely visit arbitrary web sites and execute scripts provided by those sites.

The reader will find the full discussion of the browser threat model in section 3 of {{RFC8826}} helpful in understanding what follows.

### Access to Local Resources

This document covers only unidirectional multicast from a server to (potentially many) clients, as well as associated control channels used to manage that communication and access to the content delivered via multicast. As a result, local resource access can be presumed to be limited to that already available within web applications. Note that these resources may include information that can be used to identify or track individuals, such as information about the user agent, viewport size, display resolution, but TODO: reference some other guidance on how this kind of PII is to be handled.

### Injection

### Hostile Origin

A hostile origin could serve a Web application that attempts to join many multicast groups, overwhelming the provider's network with undesired traffic.

### Private Browsing Modes

Browsers that offer a private browsing mode, designed both to bypass access to client-side persistent state and to prevent broad classes of data leakage that can be leveraged by passive and active attackers alike, should require explicit user approval for joining a multicast group given the metadata exposure to network elements of IGMP and MLD messages.


# Security Considerations

This entire document is about security.


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acks
