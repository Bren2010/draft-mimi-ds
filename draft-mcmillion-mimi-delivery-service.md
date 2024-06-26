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
: Any Service Provider which is not the LSP.

Hub:
: The Service Provider which created and hosts a referenced group. Group
  messages and Welcome messages are always sent to the hub for sequencing and
  fanout.

Follower:
: Service Providers that interact with a hub to allow their users to interact
  with a group which the hub hosts.

{::boilerplate bcp14-tagged}

# Partition Keys

A partition key is a short secret that group members export from the MLS key
schedule. Whenever members send or receive group messages, they provide the
partition key for the current epoch. Messages sent with a given partition key
are only provided to users who request to receive messages with the same
partition key. (Exceptions may be selectively made for external commits /
proposals, given that the sender isn't in the group yet.)

The partition key acts as epoch-level access control, in addition to the group
ID. Users which have been removed from a group may still be able to message the
group, as they know the group ID. However, they will not know the correct
partition key.

Service Providers MUST NOT implement hard-and-fast rules about which partition
keys are acceptable. Instead, it should be considered as a general signal of
abuse or spam.

# Endpoints

The following endpoints are provided by a Delivery Service. All endpoints
exposed by a Delivery Service are accessed by third-party Service Providers, for
the purpose of learning about the Service Provider's users or interacting with a
group for which the Service Provider is the hub.

## Key Packages

It's assumed that users have already interacted with a Discovery Service to
learn the user ID and preferred Service Provider of their contacts. As part of
discovery, users also receive a bearer token associated with each contact. The
bearer token may or may not be unique to the user who requested it, depending on
the RSP's preference.

When a user wishes to fetch a KeyPackage for the purpose of adding a user to a
group, the user submits a `KeyPackageRequest` to the RSP, with its LSP acting as
an OHTTP proxy:

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

## Sending Messages

When a follower has a user participating in a group, and that user wishes to
send a message, the follower issues a `SendRequest` to the hub:

~~~ tls-presentation
opaque PartitionKey<16>;
opaque ServiceProviderId<V>;

struct {
  MLSMessage welcome; // Welcome
  ServiceProviderId service_providers<V>;
} WelcomeData;

struct {
  PartitionKey next_partition_key;
  optional<MLSMessage> group_info; // GroupInfo
  optional<WelcomeData> welcome_data;
} CommitData;

struct {
  MLSMessage message; // PublicMessage or PrivateMessage
  PartitionKey partition_key;
  select (SendRequest.message.content_type) {
    case application:
    case proposal:
    case commit:
      CommitData commit_data;
  }
} SendRequest;
~~~

The hub rejects the message if the `group_id` of `message doesn't match any
known group. Otherwise, the hub sequences the message to the provided
`partition_key` of the group, and pushes the Welcome to any providers.

If the message is a commit, then a `CommitData` is provided which contains the
next partition key in `next_partition_key`, and an optional GroupInfo in
`group_info` to support external joiners. If any Welcome messages need to
be sent, a `WelcomeData` is provided which contains the Welcome in `welcome`,
and a list of service provider ids in `service_providers`. The
`service_providers` array MUST be the same length as the `secrets` array of the
Welcome message. Each provider indicated in `service_providers` corresponds
one-to-one with the user at the same entry in the `secrets` array.

If `group_info` does not have a `ratchet_tree` extension and the hub is unable
to infer a ratchet tree that matches `GroupInfo.group_context.tree_hash`, the
hub rejects the request such that it can be retried with an explicit ratchet
tree provided.

## Welcome

When a user of a non-hub Service Provider is added to a group, the hub first
contacts the Service Provider and lists the `KeyPackageRef` entries that were
indicated as belonging to this Service Provider. If the Service Provider
responds affirmatively, the hub pushes the Welcome message.

The first step is done by sending a `WelcomeInitRequest` to the Service
Provider:

~~~ tls-presentation
struct {
  KeyPackageRef key_package_refs<V>;
} WelcomeInitRequest;
~~~

The Welcome message is provided as an encoded `MLSMessage`.

## Receiving Messages

When a follower has a user participating in a group, and that user wishes to
receive a message, the follower issues a `ReceiveRequest` to the hub:

~~~ tls-presentation
struct {
  PartitionKey partition_key;
  uint32 counter;
} ReceiveRequest;
~~~

The `partition_key` field contains the partition key that the user wishes to
receive messages for, and the `counter` field contains the number of messages
the user has already received in that partition. The hub responds with the
following, which represents a series of messages in a series of epochs:

~~~ tls-presentation
opaque MaskedPartitionKey<V>; // = Hash(partition_key)

struct {
  MLSMessage message; // PublicMessage or PrivateMessage
  select (Message.message.content_type) {
    case application:
    case proposal:
    case commit:
      MaskedPartitionKey next_partition_key;
      optional<MLSMessage> group_info; // GroupInfo
  }
} Message;

struct {
  Message messages<V>;
} Epoch;

struct {
  MaskedPartitionKey masked_partition_key;
  Epoch epoch;
} HintedEpoch;

struct {
  Epoch epoch;
  HintedEpoch hints<V>;
} ReceiveResponse;
~~~

The `epoch` field contains a list of messages sent to the requested
`partition_key`, skipping the first `counter` messages. Messages sent to other
epochs may be provided in `hints`. Each `HintedEpoch` structure contains a
masked partition key and a list of messages sent to that partition key
(skipping none).

The same epoch may not be referenced twice separately in the same
`ReceiveResponse`. No order between epochs may be inferred by the order in
`hints`. However, the order of the `messages` array in each `Epoch` does reflect
the order that the messages were sequenced and should be processed.

The hub MAY proactively push `ReceiveResponse` structures to a following Service
Provider. The hub has complete discretion in which epochs it provides in
`hints`, and when it pushes `ReceiveResponse` structures. In a group with
encrypted handshake messages, the hub may choose to use these features
sparingly. In a group with unencrypted handshake messages, the hub will have a
clearer view of which messages need to be delivered to which follower Service
Providers and can act on that.

When processing Commits, users MUST ignore Commits where `next_partition_key` or
`group_info` does not match what was expected.

## External Joins

A user that wishes to perform an external join to a group may do so by sending
an `ExternalJoinRequest` to the hub:

~~~ tls-presentation
struct {
  MLSMessage message; // PublicMessage
  CommitData commit_data;
} ExternalJoinRequest;
~~~

If the hub recognizes the `group_id` and successfully validates the request, it
sequences the message to the most recent partition key of the group.

A group does not require a published GroupInfo to allow `ExternalJoinRequests`,
however a user can request it as follows:

~~~ tls-presentation
struct {
  opaque group_id<V>;
} GroupInfoRequest;

struct {
  MLSMessage group_info; // GroupInfo;
  optional<RatchetTree> ratchet_tree;
} GroupInfoResponse;
~~~

The hub MAY provide an inferred ratchet tree in `ratchet_tree` if one is not
present in the `group_info`.

# Fork Termination

The endpoints above are designed to gracefully support forks in the group state
through the use of the partition key. Forks can occur when a buggy or malicious
member sends a Commit which members of the group are unable to process, or that
members of the group reject for violating policy. In this case, the
malicious/buggy member would process its own Commit, moving to a new epoch with
a new partition key, while all other group members ignore it and stay in the
current epoch. Forks can also occur when a member attempts to externally join a
group, and the Delivery Service sequences the external Commit to an incorrect
partition key (such as one resulting from the previous case).

When a hub detects that the group state has forked, it MAY decide to terminate
one of the branches by blocking further messages from being sent to its
partition key. When or if to terminate a branch, is left to the discretion of
the hub. Users MUST NOT respond to a branch termination by automatically issuing
an external join to rejoin the group.

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

# Example Flows

## Creating a Group

1. User fetches KeyPackages for all the initial group members it wishes to include. ({{key-packages}})
2. User initiates creation of the group with its LSP and provides the Welcome message.
3. LSP (now hub) fans out Welcome message to all follower Service Providers. ({{welcome}})
4. Followers begin reading messages. ({{receiving-messages}})

## Externally Joining

1. User requests the group's GroupInfo and sends an external Commit. ({{external-joins}})
2. User can immediately begin sending/receiving messages to the new partition key its Commit created. ({{sending-messages}} and {{receiving-messages}})
3. Group members that receive the external Commit will verify it, and if verification succeeds they will begin processing messages from the new partition key.


## Self-Remove

1. User sends a `Remove` proposal to group. ({{sending-messages}})
2. User informs its LSP that it no longer wishes to receive messages for the group.
3. Assuming this is the only user the LSP has in the group, it stops requesting messages for the group / starts rejecting message pushes.

After sending its `Remove` proposal, the user SHOULD continue receiving group
messages for the same epoch to check that it's proposal was sequenced before the
next Commit, and resend it if not. However, this isn't required.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
