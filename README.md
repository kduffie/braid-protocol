##Braid Protocol
This is a proposed specification for a federated system for offline/online/real-time collaboration.  

It is intended to support a wide variety of implementors that can interoperate.  The protocol has
not yet been offered to any standards body, but our intention is to make this protocol freely 
available to all who want to participate and to standardize appropriately as that community grows.

#Design Goals
Collaboration applications today are increasingly dominated by proprietary client-server solutions.
While 'cloud' has positive connotations for many people, it has the downside that you are sharing
your private information with a cloud provider and are at the mercy of one provider.  Consider 
Facebook, Twitter, Google, Dropbox, Slack, Asana, etc.  From large to small, many of these are excellent
services, but in each case, they are vertically-integrated and proprietary.  While many have APIs,
these are not intended to give consumers greater choice but to create larger and larger hegemonies.

Our goal is to define a new protocol that allows for any of these kinds of collaboration services
to be implemented by a variety of different providers on a level playing field, and to allow 
different implementations to interoperate.  This will empower consumers with more choice and 
encourage more innovation.

We want to support a wide variety of collaboration scenarios from simple message and
file exchange to online chats and business applications to real-time media applications.  And,
mostly importantly, we want to allow new collaboration apps to be able to be constructed on top
of this infrastructure without requiring changes to the infrastructure itself.

Another major goal is to enable application developers to create new collaborative apps as easily
as possible -- by avoiding a large block of complexity that comes with creating any networked application.
That includes many aspects of identity, contacts, groups, authentication, security, persistence,
media exchange and state synchronization.  Without having to deal with these, app developers can
focus on features that are specific to the value they are offering. 

#Summary
Before we go into the details, here is a broad outline of how braid works.

A braid client (think of your phone or laptop) opens a websocket to a server responsible for your
domain.  You exchange JSON-encoded messages over that websocket.  These messages have a simple
consistent structure that includes addressing allowing the messages to be sent to one or more
other clients.  Braid servers are federated -- meaning that when your message is going to someone
in a different domain, your server uses a websocket to the server responsible for that other
domain and delivers the message in that way.  If and when clients need real-time communication,
braid messages are used between clients to negotiate a standard webRTC session, resulting in
a direct socket between the clients.

If we stop at this point, braid has essentially superceded SMTP with a simpler protocol that
eliminates all of the legacy problems with that older protocol, while enabling real-time
communication.  But one of braid's goals is to simplify the world for developers of collaboration
apps as well.  So it takes it one step further.

The braid protocol establishes the notion of a "tile".  Braid clients can share tiles with each
other.  Tiles are completely generic entities that support a wide variety of collaboration apps.
Tiles have a notion of distributed state.  Think of the tile as the shared data model for the app. 
For example, an app developer might create a new calendar app so that a set of braid users can work 
on a calendar at the same time.  The app maintains its state -- say a list of appointments --
as tile records.  As one client makes a change to that state, all other clients sharing that tile
are kept in sync with those changes.  An offline client can make changes, and these will be 
synchronized when they come back online.

It's vital to understand that there is no central repository for this tile state.  Braid is a 
distributed peer-to-peer state protocol.  All clients have a copy of the state (or can find out
where to get it from others).  The protocol works out the details for synchronization and
conflict resolution, so that most of that complexity is hidden from the app developer.

#Messages
To understand braid, a good place to start is with the messages.  All braid messages (except
file exchanges and media streams) have a consistent structure.  They are all JSON-encoded 
with UTF-8 character encoding.  All messages have this structure:

` { id: <id>,
    to: <recipient_array>,
    from: <sender>,
    type: <cast | request | reply | error>,
    request:  <request type>,
    code: <http-style error code>,
    message: <http-style error message>,
    data: <request-specific object>
  }`
  
Some of these fields will be absent depending on the scenario.

The `id` field is a way for the sender to identify the message and is primarily used for 
associating an `error` or `reply` message with the `request` message to which it refers.
The `id` needs only be unique in the context of a sender session and is typically an incrementing
integer.

The `to` field is an array of addresses.  The `from` field is an address.  Braid addresses are
encoded as JSON objects with the following structure:

`  { domain: <domain>,
     user: <userid>,
     resource:  <resource>
   }`
   
The `domain` field is an internet domain, e.g., "hivepoint.com".  The `user` field identifies a user
with an identity in that domain, e.g., "kduffie".  The `resource` field identifies a specific 
active session with that user.  These resources are ephemeral.  They only last as long as a user
has an active connection established with their domain.  If they close their connection and open a 
new one, they will get a new resource.

When sending braid messages, some or all of the address may be omitted depending on the scenario.
For example, when talking directly to its braid server, the client can omit the `to` address entirely
to imply that the server is the intended recipient of the message.  The client never needs to
include a `from` field in messages, because it is implied based on the socket on which it is sent.
(The server will add the from field on behalf of the sender when it delivers the message.)  Since
braid supports multicast, when sending a message to all active resources for a user, the resource
field in an address may be omitted, and the server will deliver the message to all active resources
for that user.  Alternatively, the client can target a message to a specific resource by including
that resource in the address.

When any entity receives a braid message of type `request`, they are expected to promptly send a message in
return, either a `reply` or an `error`.  If the recipient doesn't understand the message, they must
send an `error` indicating this.  The `code` and `message` fields are used only when type is `error`.

The `cast` type is for messages that do not require a response.  Since websockets support 
guaranteed delivery, only messages where information in the reply is needed are sent using `request`.
(Various failure modes are discussed later.)

#Network Topology
Braid clients talk to the server responsible for their domain using a secure websocket (wss).  The
client discovers the server location using a DNS lookup for an SRV record which determines the 
appropriate IP address and port number to use to contact an appropriate server.

Braid servers talk to other braid servers responsible for different domains using secure websockets
as well.  They also use a DNS lookup to find the appropriate server.  They open a websocket and
use a callback mechanism or certificate to ensure bilateral identity authentication.  That socket
can then be held open for as long as there is a flow of messages between those domains.

Braid supports a variety of different scenarios.  The most common is a public braid network, where
anyone on any braid server can talk to anyone else, regardless of which domain they are on.  Braid
never uses forwarding (like SMTP), so there is no issue of circular routing or various other 
similar problems.

Another scenario is the "off-the-grid" scenario.  For example, a corporation can create a braid
domain that is completely hidden behind a firewall and not accessible to the public braid network.
Clients on this isolated domain can communicate with each other, but not with anyone on the public
network.  Extending this, in a large corporation, one could have multiple braid domains that allow
users to communicate between these domains, but not connect with anyone in the public network.

Another hybrid scenario is where a braid domain is exposed to the public network, but all clients
must access that domain from behind a firewall.  The braid server sits in the DMZ and accepts
client connections only from internal to the network.  In that way, communication between
internal clients is guaranteed never to leave the firewall.

#Authentication
When a websocket is opened, only a limited set of request types are accepted:

*hello*:  The sender offers information about itself and requests information in return.  This consists
of extensible capabilities so that each side knows more about what is supported.  (This message can
also be sent end-to-end once both clients are on authenticated connections.)

*auth*:  The sender is a client is requesting to authenticate the session and provides credentials.  The recipient
sends a reply or error.  If a reply, then the session is then authenticated, and the recipient address
contains the resource that has been assigned for the session.

*register*:  The sender is a client that wants to register a new identity on the domain.  If the server
supports registration, it responds with a reply and the session is authenticated.

*federate*:  The sender is a braid domain server requesting a connection to this domain.  It includes a
callback token and expects the server to establish a reverse connection and send a *callback* message to
complete the connection.  Alternatively, the *federate* request carries a certificate proving the identity
of the caller, and the current connection is authenticated without a callback.

*callback*:  On a new connection, the sender is a braid domain server calling back in response to a 
*federate* request.  Once this is done, the original connection carrying the *federate* request is closed.

All websockets involved in message delivery use WSS -- meaning full encryption consistent with the HTTPS
protocol.  Therefore, end-to-end communication is as secure as the clients and servers involved.

#Presence and Subscription
An important part of collaboration is understanding who is online and how to reach them.  Braid borrows
from the presence model established by XMPP, although somewhat simplified.

*subscribe*:  Once a connection is authenticated, an entity can send a subscribe message to another
entity.  This tells the recipient that the sender is now offering their presence information to that entity.
It also tells their server to deliver that presence information.  A *presence* message will be sent by
the server to all subscribers whenever a new connection is authenticated or an authenticated connection is
closed by that entity.  This one-way subscription does not imply anything about subscription in the other
direction.  The recipient can choose to send a subscribe message in return, or not.  Subscriptions are
completely unidirectional as far as the braid protocol is concerned.

*unsubscribe*:  This is the inverse of *subscribe*.  It tells the recipient that presence information will
no longer be delivered, and asks the server to stop sending presence information to that entity.

*presence*:  This message is sent from a domain to a recipient (or another domain) informing them that a 
session with someone who has subscribed them to their presence has started or ended.  When received by
a domain server, they are responsible for delivering that information to subscribers in their domain.

*roster*:  This is a request to get the presence information from their domain about all of the users who have
offered to subscribe the sender to their presence.  A list of active resources is returned for each
subscribed entity.

#Extensible messages
Braid defines a set of request types as part of the protocol.  But it also anticipates extensibility.
Like XMPP, braid servers and clients are expected to deliver messages without necessarily understanding 
the nature of those messages.  Therefore, the request type is effectively unconstrained.  However, to
avoid naming conflicts, non-standard request types should be defined within a namespace controlled by
the person defining that request type.  In these cases, the request type should be in the form of a
complete URL -- typically pointing to additional information about the request.  For example, if one
were in control of the example.org domain, one could define a new request type, `http://example.org/braid/request/foobar`.
It is recommended that that URL point to a page where the request is documented.

#Contacts
Braid clients may wish to exchange contact information with one another.  Alternatively, in some
organizational scenarios, another entity (like an LDAP server) may provide a central resource for
contact information.  An entity requests contacts from another entity, and they may or may not provide
information in return.

*contact-query*:  This request is sent to ask the recipient for contact information.  Depending on the
data provided in the query, the recipient may responds with zero, one, or several contacts.  Information
about each contact is JSON-encoded and extensible.  But all contacts contains at least these fields:
fullName, braidAddress (a structure described above), lastChanged, and may contain emailAddress, imageUrl, imageFileId,
and many more.

The response to this request carries a list of matches.

*contact-inventory*:  This request offers to synchronize contact information with the recipient.  It 
includes information so that changes can be efficiently exchanged based on last-changed timestamps.

#Tiles
Braid makes it possible for entities to share distributed state information, without requiring any
centralized repository for this information as is typical for most collaboration services today.  Each
client maintains their own copy of this state information and the protocol deals with synchronizing these
copies.

Of course, there is a problem when some clients are online while others are offline and vice versa.  How
will they ever synchronize if they are never online at the same time?  The solution to this are what we 
call "braid bots".  These are optional client proxies that typically built into braid servers, but need
not be.  A braid bot acts like another endpoint with the user's identity, but one that is always online.
As with any other endpoint, it maintains a copy of shared state, and because it is always online, it can
ensure that state will always be available when needed.  But it's important to understand that braid bots 
are not necessary to the protocol.

A "tile" is a name we give to a conceptual object that is shared between a set of entities.  These entities
may be different people on the same domain or on different domains.  And these entities can be different
resources with the same identity -- such as your phone and your laptop.  And these entities can represent
real people, or bots that act on behalf of those people, or even services that are brought in 
to provide added value, such as backups or printing, or anything else.

A "tile" has a unique identifier.  (Every unique identifier in braid is a UUID that is exchanged using a
standard human-readable string display, e.g., de305d54-75b4-431b-adb2-eb6b9e546014.)  It has associated
with it an "app" that is the agreement between its members on how it will be used.  That app is identified
using a app ID (another UUID) and a version.  The tile also has a set of members, each of which is 
identified by a userId and domain.

Beyond this, a tile is two things:  a context to help entities understand the context in which they are
communicating when sending messages; and a set of shared state information.

Tiles "contain" the following state information:

- members:  The list of members on the tile using their braid identities
- properties:  A list of name-type-value tuples containing simple shared information.  Types can only be {string, boolean, number}.
- records:  A set of JSON-encoded objects, each of which is associated with a collection (by name), with a unique ID (UUID), and with a sort order (number).
- files:  A set of files, each of which has a name (with may include a path), a content-type, and binary data;

Based on our experience to date, we have found that most collaborative applications can be designed easily
based on a data model that is based on this structure.  The "records", in particular, provide a simple but 
effective form of what is essentially a distributed NOSQL database.

What makes braid work is that all information exchanged between entities about a tile are expressed as 
"mutations".  A mutation is a message that describes how the state information for a tile is to be modified.
For example, there is a mutation that sets a tile property to a given value.  There is another that
sets the value of a record.  There is another that adds or updates a file.  Of particular note is that
membership in the tile is also managed through mutations.  So when you share the tile with someone else,
you are adding a mutation that changes the tile's membership.

Because all mutations are designed to be both reversible and idempotent, it is possible to guarantee
eventual consistency between the states of all copies of the tile state -- even when some entities are
temporarily out of touch with one another.  This eventual consistency is the key to braid.  It means that
one does not need any central authority to store the "official" state of the tile.   

It is beyond the scope of this document to explain this mutation model, but the approach is meant to be
openly available and there is an open-source reference implementation.

#Mutations
Following are the list of tile mutations:

- *add-member*:  Add a member to the tile, providing the base address of the user, if they are not already a member
- *remove-member*:  Remove a member from a tile, if they are currently a member
- *set-property*:  Set the type and value of a named property (or remove by setting value to null)
- *set-record*:  Add or update a record, providing a collection name, record ID, sort order, and value (a JSON-encoded object)
- *delete-record*:  Delete a record by collection and ID, if it exists
- *reorder-record*:  Update the sort order of an existing record, if it exists
- *set-file*:  Add or update a file, providing the name (including path), content type, and file ID (files are delivered separately, referenced by this ID)
- *delete-file*:  Delete a file by name if it exists

As you will see, these mutations are all reversible and idempotent.  To ensure consistent state, all entities
are required to apply mutations based on chronological order (with originator as a secondary sort), and must
use rollbacks when mutations are received out of order.


