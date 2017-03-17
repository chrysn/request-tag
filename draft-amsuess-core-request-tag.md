---
title: Request-Discriminator option
docname: draft-amsuess-core-request-tag-latest
category: std

ipr: trust200902
area: General
workgroup: CoRE Working Group

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: C. Amsüss
    name: Christian Amsüss
    organization: Energy Harvesting Solutions
    email: c.amsuess@energyharvesting.at

normative:
  RFC7252:

informative:
  I-D.draft-ietf-core-object-security-01:

--- abstract

This memo describes an optional extension to the Constrained Application
Protocol (CoAP, RFC7252) that allows extending the endpoint context of a
request with an additional discriminator. This allows processing
concurrent requests that would otherwise be serialized by proxies in
presence of blockwise transfer, and linking blockwise requests on
transports that do provide authentication but no ordering guarantees.

--- middle

Introduction
============

Blockwise transfer [RFC7959] in the presence of Block1 ties an exchange
to a combination of resource and remote endpoint, keeping any two
exchanges where those parameters match from happening simultaneously.
Wherever either of those parameters get conflated (eg. because a proxy
acts as a single endpoint towards a server when forwarding operations on
the same resource, or in Object Security of CoAP (OSCOAP) where all a
client's request are targetted at the same resource from an outside
point of view), the client needs to queue up requests to keep the
transfers from overriding each other.

(@@@ who else references 'same Endpoint'? other documents could be
affected of profit)

The OSCOAP security layer provides message integrity and replay
protection, but with blockwise, the security context (which augments the
endpoint address) alone is insufficient protection because it does not
(and does not indent to) forbid out-of-order delivery.

Both situations can be mitigated by allowing the client to add an
additional discriminator (the Request-Disciminator option), which must
be treated as a part of the (security context of an) endpoint as defined
in RFC7252 section 1.2.

Terminology
===========

The terms Endpoint, byte, @@@ are used as in RFC7252.

An exchange is @@@ the thing with tokens and not MIDs -- should check
with OSCOAP, i never really considered it there, and it didn't happen in
the tests

Request Discriminator: An opaque identifier chosen by the client. It can
  be absent or a string of up to 8 bytes, including the empty string.
  The phrasing as two words always indicates such a value, while
  Request-Discriminator (with dash) is the CoAP option in which it is
  transferred.

The Request-Discriminator option
================================

A new option is defined for all request methods:

    +-----+---+---+---+---+-----------------------+--------+--------+---------+
    | No. | C | U | N | R | Name                  | Format | Length | Default |
    +-----+---+---+---+---+-----------------------+--------+--------+---------+
    | TBD | x | x | - |   | Request-Discriminator | opaque |    0-8 | (none)  |
    +-----+---+---+---+---+-----------------------+--------+--------+---------+
    
    C=Critical, U=Unsafe, N=NoCacheKey, R=Repeatable

It is critical (because mechanisms may rely on endpoint identities not
to be conflated), unsafe (because the channels implied by endpoint
identities terminate at proxies), not repeatable. (@@@ unsafe makes
nocachekey irrelevant, but should we or not set it to ease proxy
implementations?)

A client may set a Request-Discriminator option to indicate that the
message's Endpoint is to be augmented with the opaque option value. The
server MUST NOT treat the message's endpoint as equal to any other
message's endpoint that does not have the identical
Request-Discriminator. The absence of the option and a zero-length
option are distinct. (@@@ i wouldn't insist on that -- especially with
the inclusion in the AAD it might be better to have absence and h""
equal, so we don't have variable types in the AAD)

The option is not used in responses. Response messages implicitly bear
their corresponding (via the token @@@ or better refer to Exchange
definition?) request's discriminator.

If a request that uses Request-Discriminator is rejected with 4.02 Bad
Option, the client SHOULD retry the request without it, but only if and
when

* no limitations on simultaneous operations on the same endpoint are in
  place, and

* the application does not use distinct Request Discriminators to
  preclude out-of-order delivery on the same endpoint.


For inclusion in OSCOAP
=======================

@@@ if this stays a document of its own, oscoap should make a normative
reference to it and state something like

Whenever the Block1 option is used as inner option, it is associated
with a Request Discriminator. The same Request Discriminator MUST NOT be
reused within a security context unless every single block sent has
elicted a protected response (and its sequence number is consequently
marked used). (@@@ if we're cautious, we could s/unless.*/at all/, but i
don't see any harm in allowing it.) OSCOAP requires that out-of-order
delivery of blockwise transfers is caught by the Request-Discriminator
option, so a client MUST NOT fall back to not using the
Request-Discriminator option if it encounters a 4.02 Bad Option
response.

Note that this does not require the use of the Request-Discriminator in
many cases. An application implementing OSCOAP MUST use that option at
least when a Block1 request has not been losslessly concluded[*], and a
server MUST NOT respond with 4.02 due to the presence of a
Request-Discriminator option. (@@@ or they could declare the complete
context invalid, but i don't think we want contexts to be that volatile)

[*]: Simple package loss will not make a discriminator unusable, because
there is still CoAP retransmission in place. Only when the block
transfer is aborted, or when the same block gets sent with a different
sequence number (@@@ may that be at all?), the discriminator is unusable
for any further blockwise transfer.

A Request-Discriminator option MAY be used when initializing a Block2
transfer as well. A server SHOULD, inside a Request-Discriminator, only
send response messages with matching ETags if they can be expected to
assemble into a consistent representation again. (@@@ imo that's a
requirement that there already is from the ETags themselves. jim, do you
think this is a starting point for your concerns about validating the
whole message?)


@@@ this is more of a verbal patch than something actually includable

The associated Request Discriminator will need to be included in the
AAD. Whether we include that in the request too to keep the AAD the
same both for request and response, or have it in responses only, I
don't care.

The Request-Discriminator option is added to the "E*" category and
like and listed together with Block1/2.


Notes on applications
=====================

@@@ spell out example exchanges: proxy forwarding multiple requests,
oscoap application. include a case with non-piggybacked reponses.

>         Client              Foe         Server
> 
> POST "incarcerate"(1/2) --->  --->
>                        <---  <---   2.31 CONTINUE
> POST "valjean"(2/2)     --->@
>                        <---RST
> 
> (Client: Odd, but let's go on and promote Javert)
> 
> POST "promote"(1/2)    --->  --->
>                       <---  <---    2.31 CONTINUE
>                            @ --->
>                             <---    2.04 Valjean Promoted!
> 
> (The @ indicates a maliciously delayed / wormholed package as used in

this can't happen in oscoap now any more because the incomplete first
POST permanently poisons the first request's request discriminator, and
the second request will have a different one, for which the server will
not accept the delayed "valjean" any more.

Rationale
=========

This part is informative and serves to illustrate why this option is
necessary, and how it is different from similar concepts.

Why not use...

* another port: Proxy implementations can work around the simultaneous
  transfer restrictions by using different ports as a client. This is
  not possible with some constrained implementations (which typically
  get their one static socket from the operating system). Moreover, for
  the OSOCAP inner-bockwise application, the best equivalent would be
  starting another context, which is application dependent and very
  costly.

  Moreover, some transports do not have any such variability (@@@
  over-serial, or is there something more complete that has the same
  limitation?)

* extend the blockwise mechanism with another option: That would not
  make things easier in the author's opinion.

* put a discriminator into OSCOAP: That would have the same effect for
  the OSCOAP inner-blockwise case. It would still not allow different
  OSCOAP requests to happen concurrently on a device (especially in the
  presence of proxying).

How is this identifier different from the...

* message ID and token: Both are defined on the same level as the
  Request Discriminator (that is, from endpoint to endpoint), but the
  discriminator has an even longer lifetime than the token in that it
  at least spans all the requests transferred within a blockwise
  transfer.

* OSCOAP's sequence number: As the above, the sequence number is only
  used for a single exchange, and does not span a blockwise transfer.
  Implementations might, however, opt to use the sequence number of the
  first package of a blockwise transfer as the Request Discriminator for
  the whole transfer.

* ETag: While the Request Discriminator option serves a similar purpose
  in blockwise transfers as the ETag (ETag allows the client to filter
  non-matching Block2 responses, Request-Discriminator allows the server
  to filter non-matching Block1 requests), the ETag is stably determined
  by the server (and can thus be used for caching), while the Request
  Discriminator is an ephemeral label used exclusively during the
  transfer.

The naming of options
=====================

@@@ this section will obviously be removed over time.

The naming of options is a difficult matter -- especially here where the
use to the application (describing which requests of a blockwise bunch
belong together) deviates from what it by its definition (and in
implementation) does, that is, provide a lightweight sub-channel inside
the channel described by the endpoint address.

Alternatives under current consideration are:

* Request-Tag: More on the Block1 application side, in parallel to the
  ETag that the client uses to make sure the responses to its Block2
  request match up.

  Could lead to confusion when used in any context that relies on
  endpoints but is not blockwise.

* Endpoint-Discriminator: On the other end of the spectrum. Technically
  correct, but its use in a blockwise transfer is not immediately
  obvious.

* Request-Discriminator: Some middle ground. I'd understand it not as
  much as something that discriminates between requests, but a
  discriminator introduced by the requester.

Standard hygiene
================

@@@ if this stays in at all, it'll be shortened into some kind of
'additional considerations' -- depending on how much discussion there is
on it.

This draft does something that might be considered dirty in terms of RFC
interaction: It defines, when implemented, additional semantics into the
term "same endpoint" -- one could say that it hijacks the term to be an
extension point. This is done right now because:

* It keeps RFC interdependencies low.

* It is compatible in the sense that whoever does not implement this
  option (and consequently responds 4.02 Bad Option to its use) do
  trivially implement the changed semantics by just not allowing the
  Request-Discriminator option to take any other value than none.

* It allows later or concurrent drafts to use the "same endpoint"
  semantics and optionally utilize this extension without mandating it
  as a dependency.

* RFC7252 already defines "Endpoint" to include the security
  association. Given this protects blockwise transfers against a very
  small range of attacks (those where the attacker can't modify the
  message, but delay it), this can be seen as a security mode and then
  plug into the extension point described in RFC7252 4.1.

Security Considerations
=======================

The Request Discriminator is limited in its used between a pair of end
points. If used conservatively (ie. only when necessary, and with
as-small-as-possible random discriminators), it only indicates that (and
roughly how many) operations that require distinct endpoints are in
simultaneous use or have previously been unsuccessful.

When used to preclude out-of-order situations between endpoints (as for
example in OSCOAP), it is essential that implementations store the
usable state of a discriminator for as long as required (eg. in parallel
with sequence numbers). Failure to do so leads to reuse of a
discriminator, and thus opens up the possibility of replays.

Servers that rely on consistent states set by clients must be aware that
the out-of-order guarantees added by this mechanism only cover
operations that are required by RFC7959 to originate from the same
endpoint (or security association). Block1 requests should therefore be
limited to atomic operations as outlined in that document's security
considerations.

IANA Considerations
===================

@@@ have a number assigned and the option known

References
==========

CoAP

@@@ i think i can get away with only having coap as a normative
reference here (plus 2119 if keywords are used), though blockwise might
be required too due to it updating coap.

Informative References
----------------------

OSCOAP
blockwise
coap-over-serial if referenced

--- back
