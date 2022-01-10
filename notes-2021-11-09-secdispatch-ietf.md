# Notes on Multicast Security Considerations Discussion

At IETF 112 secdispatch, there was a presentation on multicast security
that garnered some comments in chat and at the mic.  This text aims to
capture the comments and serve as a start at discussing them.

# Relevant Links

 * [Draft](https://datatracker.ietf.org/doc/html/draft-krose-multicast-security)
 * [Repo](https://github.com/squarooticus/draft-krose-multicast-security)
 * [Video](https://www.youtube.com/watch?v=vbbFgM761t4&t=1h37m48s)
 * [Slides](https://datatracker.ietf.org/meeting/112/materials/slides-112-secdispatch-multicast-security-privacy-considerations-00)
 * [Minutes](https://notes.ietf.org/notes-ietf-112-secdispatch)
 * [Jabber Log](https://jabber.ietf.org/jabber/logs/secdispatch/2021-11-09.html)
 * [pre-meeting list discussion](https://mailarchive.ietf.org/arch/msg/secdispatch/N1jDh7MRHupuPIf1S5BiLDecGGY/)

# Jabber Topics

I tried to separate the comments into topics for easier digestion, the
original full log is linked above.

## Confidentiality 1:

~~~
[13:48:26] <npd> just as bad -- as in, all network providers already know exactly what videos I'm streaming already?
[13:48:42] <sftcd> multicast confidentiality - good research topic, hard to see it being ready for standardisation though
[13:48:49] <dkg> npd: do they?
[13:49:53] <npd> dkg: I would have guessed that currently network providers would struggle to know which Netflix video I'm streaming, but multicast would make it clearer which content is going to which user. but the speaker suggested that it was no worse than the current situation.
~~~

### Response

There's 2 different senses for the claim "just as bad":

 1. TLS acknowledges vulnerability to traffic analysis attacks in general,
    absent expensive countermeasures from the sender that are typically
    not employed in practice (see Appendix E.3 of RFC 8446).

    For an example practical demonstration on Netflix specifically, see
    "Identifying HTTPS-Protected Netflix Videos in Real-Time" (Andrew Reed,
    Michael Kranch 2017), where they describe 99.5% accuracy.
    <https://dl.acm.org/doi/10.1145/3029806.3029821>
    <https://www.mjkranch.com/docs/pubs/CODASPY17_Kranch_Reed_IdentifyingHTTPSNetflix.pdf>

    Multicast using a shared key for encryption to provide confidentiality
    is not much different in terms of discoverability of the content--an
    on-path observer does have to actively obtain a key and perform
    monitoring (or perform similar traffic analysis if preferred), just as
    they'd have to actively perform traffic analysis for TLS, and in either
    case they can with high confidence determine the traffic contents.

    In light of the low difference in content discoverability, it's an
    actively improved confidentiality tradeoff to use multicast because
    multicast provides an anonymity advantage to the viewer against everyone
    except the controller of the last-hop router, since there is not an
    individualized destination ip.

 2. There is a lot of popular content (e.g. most major sports events)
    that, for scalability reasons, cannot be delivered to all its viewers
    via the internet, and thus is priced exorbitantly for internet delivery.
    Most consumers necessarily watch it via an ISP-provided service that can
    use broadcast capabilities in the ISP's network.  This explicitly hands
    the knowledge of what each end user device is watching to the ISP, since
    they become the only available service provider to deliver that content
    to the user.

    Using multicast sourced from outside the ISP is not worse than this
    situation.  If we assume the ISP can discover the contents by obtaining
    the widely distributed shared group key and can monitor the last-hop
    routers to see which users are joined, they will get back to parity with
    their capability to monitor users who are a captive audience to a
    broadcast service that is only available directly from that ISP.  (The
    user may have a choice of which local ISP their information is exposed
    to, but this is also true for multicast subscriptions.)
   
## Confidentiality 2:

~~~
[14:00:22] <rgmhtt> How do you do confidentiality in a multiparty protocol without something like shared keys were anyone can make claims of sending?  Can someone point me to something that accomplishes this?
[14:00:55] <ekr@jabber.org> @rgmhtt: well, the data origin authentication is just signatures or TESLA or whatever
[14:01:28] <Yoav Nir_web_819> @rgmhtt: shared symmetric key for encryption; PK-signed packets.  There was a draft like that.
[14:01:53] <rgmhtt> @ekr, Authenticity is its own challenge.  PK or TESLA or what?  But the question was confidentiality
[14:02:24] <kaduk@jabber.org/barnowl> And again, I say "confidentiality from what attacker?"
~~~

### Response

With origin authentication I don't think anyone besides origin can make
claims of sending?

But I'm not sure that's relevant to the confidentiality question here.
When there are many receivers and an on-path observer can compromise any
of the receivers or itself find a way to register as a legitimate
receiver, that on-path observer can discover the traffic contents.  If
they also can discover the identity of receivers of that traffic there is
a confidentiality failure.

The claim is that authenticated multicast traffic encrypted with a shared
key (and with sender authentication) provides an advantage to anonymity by
eliminating the receiver-specific destination IP address, traded off with
a disadvantage to content discoverability by relying on an encryption key
that might be shared too widely to be securable.

## Dispatch comments, plus Confidentiality 3

~~~
[13:51:44] <Richard Barnes_web_561> Not clear to me that there's any actual specification work here
[13:52:09] <sftcd> fwiw, my take on the dispatch question: mailing list for now, maybe heading towards where the speaker wanted to go, not sure it'd get there but no harm people trying if they want
[13:52:33] <Watson Ladd_web_281> i think the w3c discussion made clear there was a need for something
[13:52:33] <Richard Barnes_web_561> @sftcd thanks, noted
[13:53:39] <Roman Danyliw_web_774> As a process comment -- Per (c) I don't think the msec mailing list is closed, there just isn't any discussion (since 2017).  List traffic on msec would be a prerequisite for (a).  We'd also need a ML to hold (b).
[13:55:52] <Benjamin Schwartz_web_753> I suggest reducing scope and dispatch to CDNI
[13:56:00] <Eric Orth_web_330> I have enough concerns with this meeting the security needs of the modern web (we really need confidentiality) that I think the overall dispatch result of this should be to drop it until such time as there is a specific proposal with security properties that we can build consensus behind.
[13:56:35] <Kathleen Moriarty_web_782> I agree on Mailing list for now.
[13:57:22] <npd> @orth, it sounds like there is disagreement about whether confidentiality is able to be provided or not
[13:57:58] <Eric Orth_web_330> @npd: And I think this is dead in the water if confidentiality cannot be provided.
[13:58:14] <dkg> it's not even clear what confidentiality means here
[13:58:24] <Martin Thomson_web_890> I think that you need a plausible security model AND a plausible proposal before doing anything.
[13:58:28] <kaduk@jabber.org/barnowl> Confidentiality against what attacker model, though?
[13:58:40] <npd> @orth, I maybe agree! but it just seemed like that was a question/debate that needs to be had
[13:58:54] <kaduk@jabber.org/barnowl> Or rather, "what dkg said"
~~~

### Response

Yes, "confidentiality against what attacker models" is I think a key
question here.

Also relevant is the suitability of comparing to the current protections
TLS provides (including its vulnerability to traffic analysis attacks and
the actual deployment prevalence of the sender-side countermeasures).  But
I think the answer to this depends on the attacker models, probably.

## MLS/ipsecme

~~~
[13:47:04] <rgmhtt> Why isn't MLS mentioned here for the multicast security options?
[13:47:44] <Jonathan Hoyland_web_734> @rgmhtt You'd need to writer anonymity in MLS first.
[13:48:03] <Valery Smyslov_web_871> In ipsecme we have G-IKEv2 draft for securing multicast
[13:48:13] <Jonathan Hoyland_web_734> @rgmhtt Any member of an MLS group IIRC can impersonate any other member.
[13:48:29] <rgmhtt> Use privacy tokens for the writer anonymity?  :)
[13:48:51] <Jonathan Hoyland_web_734> I mean, I'd suggest just 1-1 MACs would suffice.
[13:49:10] <Watson Ladd_web_281> 11 years as a draft. I no longer feel as bad about my 7 year draft
[13:49:41] <dkg> Valery Smyslov: i haven't read G-IKEv2 -- is that for AH or ESP?
[13:49:55] <Jonathan Hoyland_web_734> Wow, that makes my 2 year draft look positively young
[13:50:11] <Chris Inacio> this seems a lot like trying to protect DVD's with a key, but then sending the key to EVERYONE and hoping no one sees they already have the key.  (Except, yeah, streaming…). Am I missing something in the problem definition?
[13:50:14] <Valery Smyslov_web_871> @dkg: AH is mostly dead in real life, but in theory G-IKEvs is for both
[13:50:31] <Paul Wouters_web_653> yes please do not revive AH :)
[13:50:54] <dkg> i'm not reviewing AH, just trying to make sure i understand the scope of G-IKE :)
[13:51:10] <Yoav Nir_web_819> The last session of msec was where g-ikev2 was presented for the first time
[13:51:17] <Valery Smyslov_web_871> @watson: 11 years - that is that. you may also notice that all the orginal authors left and many followinf authors too
[13:52:30] <rgmhtt> @Paul, did you know the grief I got when i pushed ESP NULL when we had AH?  Almost got to IAB involvement, and since I was on the IAB, I would have had to recluse myself....
[13:52:49] <Yoav Nir_web_819> The record is SCEP.  First draft on January 2000. RFC published September 2020
[13:57:26] <Watson Ladd_web_281> while we're on IPSec history: what killed JFK?
[13:57:44] <rgmhtt> hubris
[13:58:13] <Valery Smyslov_web_871> @watson: IKEv2 :-)
~~~

### Response

MLS and ipsec seem not to address the issues raised in
draft-krose-multicast-security about privacy and suitability for transport
of web traffic.

Also, from 5.3 of RFC 3740 it sounded like the source authentication and
integrity is not well addressed for the broadcast use case, or may be more
heavyweight as compared to secured delivery of packet payload hashes (as
in draft-ietf-mboned-ambi) or TESLA.

But thanks and good point, we should probably add a reference to this and
RFC 3740 in draft-krose-multicast-security, and should acknowledge it as
an existing approach that's relevant for the security considerations we're
discussing.

### Refs

 * <https://datatracker.ietf.org/doc/draft-ietf-ipsecme-g-ikev2/>
 * <https://datatracker.ietf.org/doc/html/rfc3740>

## DTLS

~~~
[13:47:26] <Hannes Tschofenig_web_681> Does it add multicast security support to TLS?
[13:47:36] <Hannes Tschofenig_web_681> (DTLS)
~~~

### Response

No.  DTLS relies on 2-way communication per connection and a
connection-specific shared key.

## Blind Caching

~~~
[13:58:41] <ekr@jabber.org> This has some similarities to Blind Caching
[13:59:10] <Martin Thomson_web_890> There are ways in which Blind Caching might help here (OOB encoding specifically).
[14:00:07] <Richard Barnes_web_561> a blind-cachin-like approach also lets you punt all the questions of security model to the key exchange layer
[14:00:24] <Benjamin Schwartz_web_753> Blind caching of popular content is an oxymoron
[14:00:34] <Jeffrey Yasskin_web_812> It does seem to have the same problem as Blind Caching. In particular, that the cache can tell which content a user is fetching.
~~~

### Response

This is interesting and thanks.  I agree there are some similarities.

But I'm not sure all the questions of the security model are punted to the
key exchange layer, particularly the content discoverability for popular
content and the exposure of the multicast group subscription to the
last-hop router, as I think Ben and Jeffrey were pointing out.

### Refs

 * <https://datatracker.ietf.org/doc/draft-thomson-http-bc/>

## Why

~~~
[13:55:05] <Phillip Hallam-Baker_web_105> I don't understand why multicast has security at all. TCP doesn't
[13:55:26] <Phillip Hallam-Baker_web_105> Multicast is merely one way of delivering packets
[13:55:47] <Phillip Hallam-Baker_web_105> To use milticast in a streaming application you need a reliability backup
[13:55:52] <Phillip Hallam-Baker_web_105> \Just run QUIC
~~~

### Response

Content delivered over multicast can't be secured the same way as TLS, so
it's worth discussing what has to happen to offer similar security
properties.  We need to do this for the same reasons we no longer want to
use TCP without TLS.

We think multicast is worth addressing (as opposed to just using quic)
because of the scalability gains.  I agree many applications need
something that could be described as a reliability backup, but we still
need to secure the multicast traffic if we hope to make use of multicast.
Our current proposal does make use of unicast channels to leverage their
security properties, but since linking the multicast and unicast together
requires some caution and specifics to maintain relevant security
properties, it needs some documentation.  I'd encourage reading the draft
for more.

## NAT

~~~
[13:52:27] <ekr@jabber.org> Does anyone know how well this works with NAT?
~~~

### Response

4-to-4 and 6-to-6 nats work fine for the broadcast use case (where the
sender is not natted but receivers are).  I've done receiving thru home
routers, the join is outbound to a non-natted address and the receive uses
the group address and does not need translation.

4-6 and 6-4 nats would probably need to use something like mvpn or an
overlay at the boundary or do something like draft-ietf-mboned-mnat, else
the subscribes won't receive traffic.

I'm not interested in allowing for transmit out from a nat.  I'd be
willing to add a scope statement to that effect if needed, or to consider
PRs to address it if someone else is interested, but I don't have a use
case for it, and the sender authentication model is harder I think.

## Web UI

~~~
[13:53:23] <dkg> "web security" is problematic because it gets into the UI/UX questions: overloading the lock icon to *also* mean this not-really-confidential data seems bad
~~~

### Response

Interesting point, and maybe it's a problem.  I'm curious to see more
discussion on the threat and attacker models here, particularly with
respect to the not-really-confidential data that has been demonstrated to
be discuverable via traffic analysis attacks, such as netflix.  Has anyone
been caught abusing this so far?  What were they doing with it?  (Or why
not?)  How big a role does the lock icon play in the user's understanding
of their vulnerability?  These would be helpful to understand if anyone
has answers.

## Other Group Members

~~~
[14:00:43] <dkg> should everyone involved in the multicast group need to know who else is in the group?
[14:00:53] <Watson Ladd_web_281> the multicast forwarder sure does
[14:00:55] <dkg> if not, how is this in any way "confidential" ?
[14:00:56] <Jake Holland_web_493> @dkg: no.  in general they will not.
[14:01:12] <Jake Holland_web_493> only the last-hop forwarder knows about its next hops, in general.
[14:01:23] <Jake Holland_web_493> however, mnat might have some caveats on that.
~~~

### Response

I'm not sure I understand these comments.  draft-krose-multicast-security
covers a one-to-many broadcast scenario, where receivers who joined a
group will not know about each other.  This seems like a question about a
many-to-many group conference, where it's important to know who's
listening when you say something and there is some kind of trust between
different group members?

My understanding on the intent of the confidentiality discussion was about defense against traffic monitoring, or other threat models involved with leaking information about what people are consuming, if any can be identified.

# Mic Topics

## What's your main question?

 * [Video@1h51m27s](https://www.youtube.com/watch?v=vbbFgM761t4&t=1h51m27s)
 * Notes text:

~~~
(BK): Is your key question “Is there a security model we can use here?”?

(JH): There is lots of non-Browser traffic that uses broadcast now, so we’d like to address this in a generic way. Web isn’t necc. the starting point, but we want to get there eventually.
~~~

### Response

(Live response I think I may have misunderstood the question slightly, and
focused on the web aspect.)

My key question is: I see a lot of nervousness about the confidentiality
aspect and how it's different from existing web confidentiality.  Is there
any identifiable danger?  If not, what are the blockers to establishing
consensus on the properties required for adequate security for generic
multicast traffic delivery on today's internet?

(So far I saw one from Martin and Eric that a specific protocol proposal
is necessary before taking this further, but nothing else yet about the
security requirements.)

## Different problem than you think

 * [Video@1h54m08s](https://www.youtube.com/watch?v=vbbFgM761t4&t=1h54m08s)
 * Notes text:

~~~
Ekr: I’m not sure your problem is what you think it is. There are 2 security models:

FETCH / the origin model
WebRTC/webtransport
No way to map into 1, 2 is likely the place to map into. Think this is not for the IETF, security properties not complex, web is the problem. Not sure what to do

(JH): Chromium is sceptical probably because they don’t think it will work. I don’t think this is as out of reach as people think it is.
~~~

### Response

Extra notes after re-watch:

 * Ekr's comment had a little more: maybe this is not really a security
   problem but a browser interest problem.  If browsers were enthusiastic
   it would be a different answer.

 * Requested adding multicast as a use case for W3C Webtransport:
   - rejected for now as premature due to no underlying protocol support
     and lack of browser interest:
     * <https://www.w3.org/wiki/WebTransport/Meetings#:~:text=still%20need%20to%20implement%20multicast%20in%20quic>
     * <https://github.com/w3c/webtransport/issues/371>

As stated in response to Ben, tentative plan is to pursue deployment
without browser, but would like to start this conversation as we need to
solve web video to solve the scaling problem.

## Requirements or Protocol?

 * [Video@1h56m46s](https://www.youtube.com/watch?v=vbbFgM761t4&t=1h56m46s)
 * Notes text:

~~~
(WL): Not clear what’s needed: does W3C have clear idea, or does this need to be worked out here.

(JH): We think the draft we proposed presents a reasonable model. The W3C’s feedback is that we need to work on the underlying security model.
~~~

### Response

Making a protocol is the next step.  Wanted to check we're not missing
something and discuss it, especially if we are, especially re:
confidentiality.

## Too big to do all at once

 * [Video@1h58m22s](https://www.youtube.com/watch?v=vbbFgM761t4&t=1h58m22s)
 * Notes text:

~~~
(BS): This piece of work is too big to do all out once. Solve the protocol problem and then the broswer problem. Dispatch to CDNI and treat this as a CDN-CDN protocol, and then work on bringing the box closer and closer to the client.

(JH): Don’t think this is viable. We have that part solved.
~~~

 * [mailing list postlude](https://mailarchive.ietf.org/arch/msg/secdispatch/89OkP0dxWGwbL8g5OQ8kHgC2zaE/)

### Response

Suggestion to make a non-multicast datagram webtransport system with
unmodified browser could be useful for testing/debugging if multicast in
webtransport might move on.  CDNI techniques to authenticate objects in
the browser also might be useful after a multicast-based object
construction, similar to blind cache.  However, if multicast is DOA I
don't think this has value, so I would only pursue once we're getting
traction and have non-multicast questions that need answers.

But in webtransport it's a good point it's useful to allow same datagrams,
multicast or not, and the non-multicast could run in unmodified browsers.
Might incorporate this idea, thanks.

