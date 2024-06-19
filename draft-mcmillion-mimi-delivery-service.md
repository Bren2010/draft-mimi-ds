---
title: "Robust and Privacy-Preserving Delivery Service"
category: info

docname: draft-mcmillion-mimi-delivery-service-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "More Instant Messaging Interoperability"
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: "More Instant Messaging Interoperability"
  type: "Working Group"
  mail: "mimi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mimi/"
  github: "Bren2010/draft-mimi-ds"
  latest: "https://Bren2010.github.io/draft-mimi-ds/draft-mcmillion-mimi-delivery-service.html"

author:
 -
    fullname: "Brendan McMillion"
    organization: Your Organization Here
    email: "brendanmcmillion@gmail.com"

normative:

informative:


--- abstract

This document describes a federated MLS Delivery Service (DS) for use in the
More Instant Messaging Interoperability (MIMI) protocol. The DS provides for the
delivery of KeyPackages, Welcome messages, and group handshake/application
messages.


--- middle

# Introduction

TODO Introduction

# Conventions and Definitions

The following abbreviations are used where convenient:

Local Service Provider (LSP):
: The Service Provider preferred by the referenced user.

Remote Service Provider (RSP):
: Any Service Provider used by a user which is not the hub or the LSP.

The **hub** is the Service Provider which created a given group. Group messages
and Welcome messages are always sent to the hub for sequencing and fanout. A hub
may interact with several **follower** Service Providers, when those providers
have users that wish to participate in the group.

{::boilerplate bcp14-tagged}

# Endpoints

The following endpoints are provided by a Delivery Service.

## Key Packages

It's assumed that users have already interacted with a Discovery Service to
learn the user ID and preferred Service Provider of their contacts. As part of
discovery, users also receive a bearer token associated with each contact. The
bearer token may or may not be unique to the user who requested it, depending on
the RSP's preference.

When a user wishes to fetch a KeyPackage for the purpose of adding a user to a
group, the user submits a `KeyPackageRequest` to the RSP, with its LSP as the
proxy:

~~~ tls-presentation
struct {
  opaque user_id<V>;
  opaque bearer_token<V>;
  ProtocolVersion version;
  CipherSuite cipher_suite;
} KeyPackageRequest;
~~~

The `user_id` field contains the Service-Specific Identifier of the user,
`bearer_token` contains the bearer token learned during discovery, and `version`
and `cipher_suite` contain the desired MLS group parameters. The RSP responds
with a `KeyPackageResponse` structure that matches the requested parameters:

~~~ tls-presentation
struct {
  KeyPackage key_package;
} KeyPackageResponse;
~~~

### Privacy

Several `KeyPackageRequest` structures MAY be bundled into the same transaction,
if a user needs KeyPackages from many users of the same service and is
comfortable leaking that these users are associated.

Note that the `KeyPackageRequest` and `KeyPackageResponse` structures are
exchanged over OHTTP. This is done primarily to prevent the LSP from learning
who their local user wishes to contact, but it also restricts how much the RSP
can learn about who wants to contact their user:

To prevent abuse (such as constantly exhausting a user's uploaded KeyPackages),
this endpoint is protected with a bearer token learned during discovery. The RSP
can construct the bearer token however it feels is necessary to preserve privacy
and manage abuse. If abuse is a non-issue, a bearer token can be long-lived and
shared by every user. Alternatively, users may be provided a uniquely
identifying bearer token. Bearer tokens that are abused may be invalidated
at-will by the RSP, triggering users to go through the discovery flow again.


# Policy Enforcement

Generally, a hub or follower DS is only able to enforce "negative" policy on the
aspects of the group state that it has visiblity into. "Negative" policy
consists of rules for rejecting messages which are unambiguously invalid.
Examples of such rules might include rejecting application messages from a user
that is muted, or rejecting a commit that contains an Add proposal for a banned
user.

Keeping in mind that the group may fork when there are buggy or malicious
clients, but that clients clearly indicate which fork they are sending their
message to, the DS must always make policy enforcmenent decisions with respect
to the group state as it exists in the indicated fork. If one fork of a group
exists where a proposal was sent that updated the group metadata to ban a
specific user, but another fork exists where this proposal was not sent (or not
accepted), the ban must only apply in the fork that accepted the proposal.

A DS MUST NOT reject messages based on criteria that are not unambiguously known
by the DS to be satisfied. In particular, a DS can not enforce a per-user rate
limit on messages sent to the group for any users that are not local to the DS.
When the DS has a local relationship with a user, the DS can authenticate the
user and unambiguously rate-limit just them. However, a hub DS can not
rate-limit a follower DS's users without speculating on an MLS message's
content.

It's understood that hub servers will make arrangements with follower servers,
where the follower server enforces the hub's policies in situations where the
hub can not, as a precondition to interoperate. This includes rate limits but
also for example, if one of the follower server's users are banned by the hub,
enforcing that the banned user is not allowed in any groups hosted by the hub.
(Noting that the hub would be unable to ban a follower server's user themselves,
in cases where a group's handshake messages are encrypted.)

Ultimately, while a DS can reject *some* abusive messages, clients must be aware
of the policy a group is under and enforce it themselves. Messages which violate
the group's policy MUST be ignored and supressed by the UI, as if they were
cryptographically invalid. Clients SHOULD report to their local DS when the hub
or another follower is not meeting the preconditions to interoperate.

A DS MUST NOT enforce "positive" policy decisions, consisting of rules that
require a given message to be accepted by the members of the group.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
