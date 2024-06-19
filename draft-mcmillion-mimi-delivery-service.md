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

This document describes a robust and privacy-preserving delivery service
architecture for the MIMI working group.


--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}


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
