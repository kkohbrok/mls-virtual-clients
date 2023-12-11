---
title: MLS Virtual Clients
abbrev: MVC
docname: draft-kohbrok-mls-virtual-clients-latest
category: info

ipr: trust200902
area: Security
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -  ins: K. Kohbrok
    name: Konrad Kohbrok
    organization: Phoenix R&D
    email: konrad.kohbrok@datashrine.de
 -  ins: R. Robert
    name: Raphael Robert
    organization: Phoenix R&D
    email: ietf@raphaelrobert.com

--- abstract

This document describes a method that allows multiple MLS clients to emulate a
virtual MLS client. A virtual client allows multiple emulator clients to jointly
participate in an MLS group under a single leaf. Depending on the design of the
application, virtual clients can help hide metadata and improve performance.

--- middle

# Introduction

The MLS protocol facilitates communication between clients, where in an MLS
group, each client is represented by the leaf to which it holds the private key
material. In this document, we propose the notion of a virtual client that is
jointly emulated by a group of emulator clients, where each emulator client
holds the key material necessary to act as the virtual client.

The use of a virtual client allows multiple distinct clients to be represented
by a single leaf in an MLS group. This pattern of shared group membership
provides a new way for applications to structure groups, can improve performance
and help hide group metadata. The effect of the use of virtual clients depends
largely on how it is applied (see {{applications}}).

We discuss technical challenges and propose a concrete scheme that allows a
group of clients to emulate a virtual client that can participate in one or more
MLS groups.

# Terminology

- Client: An MLS client
- Virtual Client: A client for which the key material of which is held by one or
  more (other) clients.
- Emulator Client: A client that collaborates with other emulator clients in
  emulating a virtual client. Emulator clients can themselves be virtual
  clients.

TODO: Terminology is up for debate. We’ve sometimes called this “user trees”,
but since there are other use cases, we should choose a more neutral name. For
now, it’s virtual client emulation.

# Applications

Virtual clients generally allow multiple emulator clients to share membership in
an MLS group, where the virtual client is represented as a single leaf. This is
in contrast to the case where each individual emulator client is a regular
member of the group, each with its own leaf.

Depending on the application, the use of virtual clients can have different
effects. However, in all cases, virtual client emulation introduces a small
amount of overhead for the emulator clients and certain limitations (see
{{limitations}}).

## Virtual clients for performance

If a group of emulator clients emulate a virtual client in more than one group,
the overhead caused by the emulation process can be outweighed by two
performance benefits.

On the one hand, the use of virtual clients makes the higher-level groups (in
which the virtual client is a member) smaller. Instead of one leaf for each
emulator client, it only has a single leaf for the virtual client. As the
complexity of most MLS operations depends on the number of group members, this
increases performance for all members of that group.

At the same time, the virtual client emulation process (see
{{client-emulation}}) allows emulator clients to carry the benefit of a single
operation in the emulation group to all virtual clients emulated in that group.

## Hidden subgroups

Virtual clients can be used to hide the emulator clients from other members of
higher-level groups. For example, removing group members of the emulator group
will only be visible in the higher-level group as a regular group update.
Similarly, when an emulator client wants to send a message in a higher-level
group, recipients will see the virtual client as the sender and won't be able to
discern which emulator client sent the message, or indeed the fact that the
sender is a virtual client at all.

Hiding emulator clients behind their virtual client(s) can, for example, hide
the number of devices a human user has, or which device the user is sending
messages from.

As hiding of emulator clients by design obfuscates the membership in
higher-level groups, it also means that other higher-level group members can't
identify the actual senders and recipients of messages. From the point of view
of other group members, the "end" of the end-to-end encryption and
authentication provided by MLS ends with the virtual client. The relevance of
this fact largely depends on the security goals of the application and the
design of the authentication service.

If the virtual client is used to hide the emulator clients, the delivery service and
other higher-level group members also lose the ability to enforce policies to
evict stale clients. For example, an emulator client could become stale (i.e.
inactive), while another keeps sending updates. From the point of view of the
higher-level group, the virtual client would remain active.

## Transparent subgroups

TODO: The following text assumes that we have some mechanism of adding one or
more additional signatures to MLS messages.

While applications can choose to use virtual clients to hide the corresponding
emulator clients, they don't have to. When using the virtual client to send
messages, the sending emulator client can provide an addition signature using
either its leaf credential in the emulation group, or another AS-provided
credential that allows higher-level group members to authenticate the message.

# Limitations

The use of virtual clients comes with a few limitations when compared to MLS,
where all emulator clients are themselves members of the higher-level groups.

## External remove proposals

In some cases, it is desirable for an external sender (e.g. the messaging
provider of a user) to be able to propose the removal of an individual
(non-virtual) client from a group without requiring another client of the same
user to be online. Doing so would allow another client to commit to said remove
proposal and thus remove the client in question from the group.

This is not possible when using virtual clients. Here, the non-virtual client
would be the emulator client of a virtual client in a higher-level group. While
the server could propose the removal of the client from the emulation group,
this would not effectively remove the client's access to the higher-level groups
in which the virtual client is a member.

For such a removal to take place, another emulator client would have to be
online to update the key material of the virtual client (in addition to the
removal in the emulation group).

Another possibility would be for emulator clients to provision KeyPackages for
which only a subset of emulator clients have access to. The external sender
could then propose the removal of the virtual client, coupled with the immediate
addition of a new one using one of the KeyPackages.

## External joins

When there are no subgroups and all (emulator) clients are members of each
higher-level group, new (emulator) clients would be able to join via external
commit without influencing the operation of any other emulator client and
without requiring another emulator client to be online.

When using virtual clients and a client wishes to externally join the emulator
group, it will not have immediate access to the secrets of the virtual clients
associated with that group.

This can be remedied either by another (online) emulator client providing it
with the necessary secrets, or by the new emulator client having the virtual
client rejoin all higher-level groups.

While the first option has the benefit of not requiring an external commit in
any higher-level groups (thus reducing overhead), it strictly requires another
emulator client to be online.

The second option on the other hand additionally requires the new emulator
client to re-upload all KeyPackages of the virtual client, thus further
increasing the difficulty of coordinating actions between emulation group and
higher-level groups.

# Client emulation

To emulate a virtual client, the emulator clients need to agree on the key
material held by the virtual client and coordinate the actions taken by the
client in any group of which it is a member. The obvious way for the emulator
clients to achieve both of these tasks is to create an MLS group which we call
the emulation group. In contrast, we call any group that the virtual client is a
member of a higher-level group.

## Example protocol flow

TODO: Go through a full flow, where both sender and receiver are virtual
clients. Include both communication in the higher-level group and in the
emulation group.

## DS/AS Details

Virtual client emulation should be largely agnostic to specific details of the
AS and DS of the application. However, a few conditions must be met.

- Access control: All emulator clients must be able to act as the virtual
  client, including, for example, queue access and KeyPackage upload
- Queue compatibility: The queue system must allow all emulator clients to
  retrieve messages for the virtual clients. (Although workarounds like one
  emulator client retrieving messages and then sending them to the emulation
  group are possible.)

## Agreement on/generation of key material

Agreement on key material is one of the primary features of MLS and the emulator
clients can simply export secrets from the emulation group to provide the
randomness for all key material generated by the virtual client throughout its
lifetime, especially any MLS-specific key material such as leaf encryption keys
or init keys.

This leaves the generation of authentication key material, which can also be
based on exporters from the emulation group, but which otherwise depends on the
design of the AS. In the context of virtual clients, one should note, however,
that the specifics of the AS can influence how well the virtual client hides the
metadata of the underlying emulator clients.

TODO: Be more explicit on how the randomness is exported.

## Coordinating virtual client operations

Generally, emulator clients should coordinate the actions taken by the virtual
client. This is, for example, to prevent multiple emulator clients from
uploading KeyPackages simultaneously.

However, emulator clients do not have to coordinate all actions taken by the
virtual client. Specifically, commits in a single higher-level group can be sent
without the risk of desynchronization between the emulator clients, as the
group’s DS will enforce message ordering. Coordination is thus required for
non-commit messages (specifically proposals and application messages), as well
as actions that affect more than one higher-level group (such as credential
updates, depending on the AS).

## Possible message re-ordering

One difficulty in coordinating the use of randomness exported from the emulation
group is that we cannot assume that messages arrive in order. For example, if an
emulator client sends an Update in the emulation group and then immediately
exports randomness from the new epoch to have the virtual client create an
update in the higher-level group, then other emulator clients might receive the
update in the higher-level group first. Since they have not yet received the
other update, they will have to wait for the emulation group update before they
can process it. Depending on the DS, this may or may not require specific care.

TODO: Add a recommendation here, maybe to extract randomness only from emulation
group epochs that are at least X seconds old, which in turn requires clients to
keep around randomness, thus potentially compromising FS somewhat.

In addition, since the epochs in the emulation group and higher level group are
not necessarily correlated, i.e. an emulator client performing an update in the
higher level group might not at the same time perform an update in the emulation
group. The emulator client performing the update in the higher level group thus
has to signal the other emulator clients which emulation group epoch to use to
export the randomness.

TODO: Find a good way to signal this to the rest of the emulation group. Options:

- AAD of the higher-level group update (leaks metadata to higher-level groups,
  could collide with application-level use of AAD)
- Application message to emulation group (subject to message re-ordering as
  described above)
- LeafNode extension (leaks metadata to higher-level groups)

## Adding emulator clients

If a client is added to the emulation group, it has to be provisioned with the
private key material and the group states of all higher-level groups. While the
latter might be able to be provisioned by the higher-level DS, the former has to
be provided by another emulator client.

The other emulator client can provide the secret key material used to derive all
key material relevant to the virtual client (higher-level group secrets,
KeyPackage secrets, etc.)

TODO: This means that all such key material must be derived in a well-separated
and forward-secure way. (See TODO above to specify further details on how to
derive key material for the virtual client.)

Since the new emulator client can only emulate the virtual client if it has
access to those secrets, it cannot join the emulation group via external commit,
except if said secrets are provided asynchronously.

## Sending application messages

Open Question: Key material for encryption/decryption of application messages is
used only once. Since there is no ordering of application messages (per client)
on the DS, emulator clients could have the same virtual client send an
application message at the same time, both using the same key material. When
those messages arrive at a recipient, the recipient won’t be able to decrypt the
second message, because the recipient has already discarded the required key
material. How to avoid this? A few ideas:

- Key generation allotments: Clients in the emulation group use the ratchet
  generation modulo their leaf index in the emulation group. (Leaks metadata to
  higher-level groups, potentially requires all other higher-level group members
  to store lots of intermediary key generations, could confuse applications if
  they derive message ordering from key generations, could make recipients think
  that they missed messages)
- More complex secret tree (or separate secret tree): Instead of a single
  ratchet at the leaf of the secret tree, there is another tree with one ratchet
  for each emulator client. (extra costs for PPRF, leaks metadata, requires an
  extension, either an unsafe one (if the original secret tree is modified), or
  a safe one that defines its own application message (if a new secret tree is
  used))

### PPRF-based Challenges
Key/nonce pairs used to encrypt application messages are derived using a PPRF.
That is, the Secret Tree in the super group is replaced by a GGM-style PPRF
initially keyed by the `encryption_secret`. To send an application messaage,
the sending emulator client choose a 32 byte challenge and includes it in the
`authenticated_data` field in the `PrivateMessage` struct. To derive the nonce
and AEAD key for the application message the PPRF is evaluated (and punctured)
the PPRF at the challenge. Due to the entropy in the challenge nonce-reuse
guards can be dropped.

Advantages:
- Group Structure Agnostic : The construction is agnostic to all aspects of the
group's membership structure. For example, clients can receive application
messages without knowing the sender's leaf index in it's virtual client nor
the relative position of the sender's virtual client in the hiearchy of virtual
clients making up the super group.
- Metadata Hiding: The scheme hides everything about a sender's position in the
groups membership heirachy beyond what is revealed by the signature. For
example, application messages are as unlinkabel as permited by their signatures
(even for other emulation clients in the same virtual client).
- Forward Security Sender Anonymity : Leaking a clients PPRF local state reveals
nothing about who sent previously received application messages. This includes
both hiding both the sending emulation clients and even their virtual clients.

Disadvantages:
- Breaks the RFC9420 specification & requires an un-safe extension.
- Applications messages have an added 28 byte overhead.
- Senders need 28 additional random bytes to send an application message.
- A decryption error occurs when receiving an application message with a
previously used challenge (in the current epoch).

There is a trade-off between cost and error probability when choosing `b`; the
bit-length of challenges. (Above `b = 256` which is very conservative.)
- Sending two application messages with the same challenge in an epoch
immediatly leaks the XOR of the two plaintexts and results in decryption
failures for which ever message a client receives second. The probability a
challenge gets re-used when `k` application messages with `b`-bit challenges
are sent in an epoch (without any coordination) is
`err = 1-[C(2^b,k)/C(2^b+k-1)]`. So larger `b` reduces the probability of
challenge re-use exponentially.
- Evaluating the PPRF requires up to `b` HKDF calls and new 32B values to be
stored locally by the client. Moreover, `b`-bit challenges impost a `8b-4` bytes
of overhead in application messages and added random bits when sending.

## Rotation of authentication key material

If the design of the AS specifies the use of cross-group authentication key
material, emulator clients must coordinate the rotation of said key material in
the emulation group to avoid multiple emulator clients rotating a key at the
same time. Details depend on the design of the AS.

# Security considerations

TODO: Detail security considerations once the protocol has evolved a little
more. Starting points:

Some of the performance benefits of this scheme depend on the fact that one can
update once in the emulation group and “re-use” the new randomness for updates
in multiple higher-level groups. At that point, clients only really recover when
they update the emulation group, i.e. re-using somewhat old randomness of the
emulation group won’t provide real PCS in higher-level groups.

# Privacy considerations

TODO: Specify the metadata hiding properties of the protocol. The details depend
on how we solve some of the problems described throughout this document.
However, using a virtual client should mask add/remove activity in the
underlying emulation group. If it actually hides the identity of the members may
depend on the details of the AS, as well as how we solve the application
messages problem.

# Performance considerations

There are several use cases, where a specific group of clients represents a
higher-level entity such as a user, or a part of an organization. If that group
of clients shares membership in a large number of groups, where its sole purpose
is to represent the higher-level entity, then instead emulating a virtual client
can yield a number of performance benefits, especially if this strategy is
employed across an implementation. Generally, the more emulator clients are
hidden behind a single virtual client and the more clients are replaced by
virtual clients, the higher the potential performance benefits.

## Smaller Trees

As a general rule, groups where one or more sets of clients are replaced by
virtual clients have fewer members, which leads to cheaper MLS operations where
the cost depends on the group size, e.g., commits with a path, the download size
of the group state for new members, etc. This increase in performance can offset
performance penalties, for example, when using a PQ-secure cipher suite, or if
the application requires high update frequencies (deniability).

## Fewer blanks

Blanks are typically created in the process of client removals. With virtual
clients, the removal of an emulator client will not cause the leaf of the
virtual client (or indeed any node in the virtual client’s direct path) to be
blanked, except if it is the last remaining emulator client. As a result,
fluctuation in emulator clients does not necessarily lead to blanks in the group
of the corresponding virtual clients, resulting in fewer overall blanks and
better performance for all group members.

# Emulation costs

From a performance standpoint, using virtual clients only makes sense if the
performance benefits from smaller trees and fewer blanks outweigh the
performance overhead incurred by emulating the virtual client in the first
place.
