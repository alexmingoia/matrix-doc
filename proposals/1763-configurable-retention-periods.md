# Proposal for specifying configurable retention periods for messages.

A major shortcoming of Matrix has been the inability to specify how long
events should stored by the servers and clients which participate in a given
room.

This proposal aims to specify a simple yet flexible set of rules which allow
users, room admins and server admins to determine how long data should be
stored for a room, from the perspective of respecting the privacy requirements
of that room (which may range from "burn after reading" ephemeral messages,
through to FOIA-style public record keeping requirements).

As well as enforcing privacy requirements, these rules provide a way for server
administrators to better manage disk space (e.g. to enforce rules such as "don't
store remote events for public rooms for more than a month").

## Problem:

Matrix is inherently a protocol for storing and synchronising conversation
history, and various parties may wish to control how long that history is stored
for.

 * Users may wish to specify a maximum age for their messages for privacy
   purposes, for instance:
   * to avoid their messages (or message metadata) being profiled by
     unscrupulous or compromised homeservers
   * to avoid their messages in public rooms staying indefinitely on the public
     record
   * because of legal/corporate requirements to store message history for a
     limited period of time
   * because of legal/corporate requirements to store messages forever
     (e.g. FOIA)
   * to provide "ephemeral messaging" semantics where messages are best-effort
     deleted after being read.
 * Room admins may wish to specify a retention policy for all messages in a
   room.
   * A room admin may wish to enforce a lower or upper bound on message
     retention on behalf of its users, overriding their preferences.
   * A bridged room should be able to enforce the data retention policies of the
     remote rooms.
 * Server admins may wish to specify a retention policy for their copy of given
   rooms, in order to manage disk space.

Additionally, we would like to provide this behaviour whilst also ensuring that
users generally see a consistent view of message history, without lots of gaps
and one-sided conversations where messages have been automatically removed.

At the least, it should be possible for people participating in a conversation
to know the expected lifetime of the other messages in the conversation **at the
time they are sent** in order to know how best to interact with them (i.e.
whether they are knowingly participating in a future one-sided conversation or
not).

We would also like to discourage users from setting low message retention as a
matter of course, as it can result in very antisocial conversation patterns to
the detriment of Matrix as a useful communication mechanism.

This proposal does not try to solve the problems of:
 * GDPR erasure (as this involves retrospectively changing the lifetime of
   messages)
 * Bulk redaction (e.g. to remove all messages from an abusive user in a room,
   as again this is retrospectively changing message lifetime)
 * Limiting the number (rather than age) of messages stored per room (as this is
   more a question of quotaing rather than empowering privacy)

## Proposal

### User-specified per-message retention

Users can specify per-message retention by adding the following fields to the
event within its content.  Retention is only considered for non-state events.

`max_lifetime`:
	the maximum duration in seconds for which a server must store
	this event.  Must be null or in range [0, 2<sup>31</sup>-1]. If absent, or null,
        should be interpreted as 'forever'.

`min_lifetime`:
	the minimum duration for which a server should store this event.
	Must be null or in range [0, 2<sup>31</sup>-1]. If absent, or null, should be
        interpreted as 'forever'.

`self_destruct`:
	the duration in seconds after which servers (and optionally clients) must
	remove this event after seeing an explicit read receipt delivered for it.
	Must be null or in range [0, 2<sup>31</sup>-1]. If absent, or null, this
	behaviour does not take effect.
	

`expire_on_clients`:
	a boolean for whether clients must expire messages clientside
	to match the min/max lifetime and/or self_destruct semantics fields. If absent,
  or null, should be interpreted as false.

For instance:

```json
{
	"max_lifetime": 86400,
}
```

The above example means that servers receiving this message should store the
event for a only 86400 seconds (1 day), as measured from that event's
origin_server_ts, after which they MUST purge all references to that event ID
(e.g. from their db and any in-memory queues).

We consciously do not redact the event, as we are trying to eliminate
metadata here at the cost of deliberately fracturing the DAG, which will
fragment into disparate chunks.  (See "Issues" below in terms of whether this
is actually valid)

```json
{
	"min_lifetime": 2419200,
}
```

The above example means that servers receiving this message SHOULD store the
event forever, but MAY choose to purge their copy after 28 days (or longer) in
order to reclaim diskspace.

```json
{
	"self_destruct": 5,
	"expire_on_clients": true,
}
```

The above example describes 'self-destructing message' semantics where both server
and clients MUST purge/delete the event and associated data, 5 seconds after a read
receipt for that message is received from the recipient.  In other words, the
recipient(s) have 5 seconds to view the message after receiving it.

Clients and servers MUST send explicit read receipts per-message for
self-destructing messages (rather than for the most recently read message,
as is the normal operation), so that messages can be destructed as requested.

XXX: this means that self-destruct only really makes sense for 1:1 rooms. is this
adequate? should self-destruct messages be removed from this MSC entirety to
simplify landing it?

These retention fields are preserved during redaction, so that even if the event
is redacted, the original copy can be subsequently purged appropriately from the
DB.

XXX: This may change if we end up redacting rather than purging events (see
Issues below)

TODO: do we want to pass these in as querystring params when sending, instead of
putting them inside event.content?

### User-advertised per-message retention

If we had extensible profiles, users could advertise their intended per-message
retention in their profile (in global profile or per-room profile) as a useful
social cue.  However, this would be purely informational.

### Room Admin-specified per-room retention

We introduce a `m.room.retention` state event, which room admins can set to
override the retention behaviour for a given room.  This takes the same fields
described above.  It follows the default PL semantics for a state event (requiring
PL of 50 by default to be set)

If set, these fields replace any per-message retention behaviour
specified by the user - even if it means forcing laxer privacy requirements on
that user.  This is a conscious privacy tradeoff to allow admins to specify
explicit privacy requirements for a room.  For instance, a room may explicitly
disable self-destructing messages by setting `self_destruct: null`, or may
require all messages in the room be stored forever with `min_lifetime: null`.

In the instance of `min_lifetime` or `max_lifetime` being overridden, the
invariant that `max_lifetime >= min_lifetime` must be maintained by clamping
max_lifetime to be equal to `min_lifetime`.

If the user's retention settings conflicts with those in the room, then the
user's clients MUST warn the user when participating in the room.  A conflict
exists if the user sets retention fields on their messages which are specified
with differing values on the `m.room.retention` state event.  This is particularly
important to warn the user if the room's retention is longer than their requested
retention period.

### Server Admin-specified per-room retention

Server admins have two ways of influencing message retention on their server:

1) Specifying a default `m.room.retention` for rooms created on the server, as
defined as a per-server implementation configuration option which inserts the
state events after creating the room (effectively augmenting the presets used
when creating a room).  If a server admin is trying to conserve diskspace, they
may do so by specifying and enforcing a relatively low min_lifetime (e.g. 1
month), but not specify a max_lifetime, in the hope that other servers will
retain the data for longer.

XXX: is this the correct approach to take? It's how we force E2E encryption on,
but it feels very fragmentory to have magical presets which do different things
depending on which server you're on.

2) By adjusting how aggressively their server enforces the the `min_lifetime`
value for message retention.  For instance, a server admin could configure their
server to attempt to automatically remote purge messages in public rooms which
are older than three months (unless min_lifetime for those messages was set
higher).

A possible configuration here could be something like:
 * target_lifetime_public_remote_events: 3 months
 * target_lifetime_public_local_events: null # forever
 * target_lifetime_private_remote_events: null # forever
 * target_lifetime_private_local_events: null # forever

...which would try to automatically purge remote events from public rooms after
3 months (assuming their individual min_lifetime is not higher), but leave
others alone.

These config values would interact with the min_lifetime and max_lifetime values
of a message (either per-message or per-room) in the different classes of room
by decreasing the effective max_lifetime to the proposed value (whilst
preserving the `max_lifetime >= min_lifetime` invariant).  However, the precise
behaviour would be up to the server implementer.

XXX: should this configuration be specced or left as an implementation-specific
config option?

Server admins could also override the requested retention limits (e.g. if resource
constrained), but this isn't recommended given it may result in history being
irrevocably lost against the senders' wishes.

## Client-side behaviour

Clients which persist conversation history must calculate the retention of a message
based on the event fields and the room state.  If a message has a finite lifespan
that fact MUST be indicated clearly in the timeline
to allow users to interact with the message in an informed manner.  (The details of the
lifespan can be shown on demand, however).

If `expire_on_clients` is true, then clients should also calculate expiration for
said events and delete them from their local stores as required.

## Pruning algorithm

To summarise, servers and clients must implement the pruning algorithm as
follows:

If we're a client, apply the algorithm if:
  * if specified, the `expire_on_clients` field in the `m.room.retention` event for the room is true.
  * otherwise, if specified, the message's `expire_on_clients` field is true.
  * otherwise, don't apply the algorithm.

The maximum lifetime of an event is calculated as:
  * if specified, the `max_lifetime` field in the `m.room.retention` event for the room.
  * otherwise, if specified, the message's `max_lifetime` field.
  * otherwise, the message's maximum lifetime is considered 'forever'.

The minimum lifetime of an event is calculated as:
  * if specified, the `min_lifetime` field in the `m.room.retention` event for the room.
  * otherwise, if specified, the message's `min_lifetime` field.
  * otherwise, the message's minimum lifetime is considered 'forever'.
  * for clients, `min_lifetime` should be considered to be 0 (as there is no
    requirement for clients to persist events).

If the calculated max_lifetime is less than the min_lifetime then the max_lifetime
is set to be equal to the min_lifetime.

The server/client then selects a lifetime of the event to lie between the
calculated values of minimum and maximum lifetime, based on their implementation
and configuration requirements.  The selected lifetime MUST not exceed the
calculated maximum lifetime. The selected lifetime SHOULD not be less than the
calculated minimum lifetime, but may be less in case of constrained resources,
in which case the server should prioritise retaining locally generated events
over remote generated events.

Server/clients then set a maintenance task to remove ("purge") the event and
references to its event ID from their DB and in-memory queues after the lifetime
has expired (starting timing from the absolute origin_server_ts on the event).

As a special case, servers and clients should purge the event N seconds after observing
a read receipt for that specific event ID, if:
  * if specified, the `self_destruct` field in the `m.room.retention` event for
    the room is set to N where N is not null.
  * otherwise, if specified, the message's `self_destruct` field is true.
  
The device emitting the read receipt for a self-destructing message must give the
user sufficient time to view the message after op

If possible, servers/clients should remove downstream notifications of a message
once it has expired (e.g. by cancelling push notifications).

## Tradeoffs

This proposal deliberately doesn't address GDPR erasure or mega-redaction scenarios,
as it attempts to build a coherent UX around the use case of users knowing their
privacy requirements *at the point they send messages*.  Meanwhile GDPR erasure is
handled elsewhere (and involves hiding rather than purging messages, in order to
avoid annhilating conversation history), and mega-redaction is yet to be defined.

## Issues

It's debatable as to whether we're better off applying the redaction algorithm
to expired events (and thus keep the integrity of the DAG intact, at the expense
of leaking metadata), or whether to purge instead (as per the current proposal),
which will punch holes in the DAG and potentially break the ability to backpaginate
the room.

How do we handle scenarios where users try to re-backfill in history which has
already been purged?  This should presumably be a server admin option on whether
to allow it or not, and if allowed, configure how long the backfill should persist
for before being purged again?

How do we handle retention of media uploads (especially for E2E rooms)?  It feels
the upload itself might warrant retention values applied to it.

Should room retention be announced in a room per-server?  The advantage is full
flexibility in terms of servers announcing their different policies for a room
(and possibly letting users know how likely history is to be retained, or conversely
letting servers know if they need to step up to retain history).  The disadvantage
is that it could make for very complex UX for end-users: "Warning, some servers in
this room have overridden history retention to conflict with your preferences" etc.

## Security considerations

There's scope for abuse where users can send abusive messages into a room with a
short max_lifetime and/or self_destruct set true which promptly self-destruct.

One solution for this could be for server implementations to implement a quarantine
mode which initially marks purged events as quarantined for N days before deleting
them entirely, allowing server admins to address abuse concerns.

## Conclusion

Previous attempts to solve this have got stuck by trying to combine together too many
disparate problems (e.g. reclaiming diskspace; aiding user data privacy; self-destructing
messages; mega-redaction; clearing history on specific devices; etc) - see
https://github.com/matrix-org/matrix-doc/issues/440 and https://github.com/matrix-org/matrix-doc/issues/447
for the history.

This proposal attempts to simplify things to strictly considering the question of
how long servers should persist events for (with the extension of self-destructing
messages added more to validate that the design is able to support such a feature).