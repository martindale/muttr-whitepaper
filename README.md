Muttr: Secure Distributed Messaging
===================================

Abstract
--------

Messaging is the foundation for higher-order application design in modern
computer science.  Object-oriented programming has historically been utilized as
a mechanism for code organization and for the introduction of metaprogramming
into otherwise complex systems, but the basis in representing state changes
through messaging between them has been lost in various language implementations
over the past several decades.  To complicate matters further, the introduction
of the Internet, and furthermore the World Wide Web, has further blurred the
boundaries between the implicitly strongly-typed actors and participants
therein.

We propose a secure, federated messaging protocol that can be used as a
foundation for applications of all types.  This framework focuses on providing
end-to-end security with regards to message encryption, and in eliminating the
need for any centralized authority to facilitate a robust and reliable network
of messaging actors.

Introduction
------------

Secure messaging is an essential component of a free society – without the
ability to move information in confidence, free speech comes under a natural
pressure resulting from the fear of bad actors being able to intercept and act,
sometimes violently, against the flow of this information.  Whether the
information is being transmitted by humans intending to share their thoughts or
machines attempting to form consensus in a network, the integrity and
reliability of the delivery of the component messages requires the transport
network to be secure and free from censorship.

We can imagine a wildly dystopian future wherein incumbent authorities have an
incentive to have a monopoly on information, and the ability to move information
in an uninhibited fashion threatens their ability to defend that monopoly.  In
such a future, one would expect the emergence of markets for data and
information, especially in regards to the data that is considered "dangerous" to
the established authority.

For a system to be secured against these types of actors, we must ensure a
number of principles.  First and foremost, that while in transit, messages
cannot be read by the parties responsible for their transit.  This means that
network operators should not have any introspection into the content, but also
no awareness of the originator or destination of the data.

This can be partially resolved through a sparse routing network, which Muttr
implements, but can more completely resolved by using asymmetric cryptography.
Muttr has selected the PGP scheme to prevent this type of introspection, which
requires the possession of a private key to decrypt any messages that have been
intercepted.  Furthermore, the specific content to be accessed is addressed by
its hash, which can be communicated securely or out-of-band, further removing
the ability of intermediaries to target specific entities.

Mechanics
---------

The Muttr network is comprised of two distinct types of participants: *seeds*
and *pods*. Seeds are nodes which store PGP encrypted messages on behalf of
other seeds in the network, forming a DHT (distributed hash table). Each seed
possesses a PGP key pair which is used to sign messages to other seeds and
decrypt messages in the DHT that are intended for them.

Pods also participate in the storage of messages but, more importantly, form a
"meta-network" that acts as a federated routing system. Each pod allows seeds
to register *aliases* to their public PGP key, which allows other seeds to
address them by `<some_alias>@<pod_hostname>`. In addition, seeds may use pods
to help deliver messages to other seeds by providing the location (or key) of a
given message in the DHT, which the pod will relay to the recipient directly or
if the recipient is offline, store for "playback" at a later time.

In practice, to join the network a seed will connect to a pod of its choice on
two channels. The first channel is to start discovering other seeds in the DHT
and begin participating in the cooperative storage of messages. The other is a
private channel between the seed and the pod exclusively, for the purpose of
notifying the seed of received message keys that the seed can then go look up
and decrypt.

The typical flow can be described below.

1. Unidentified **seed** connects to **pod** muttr.me and asks for nearest seeds
2. **Pod** responds with nearest seeds and **seed** builds routing table
3. **Seed** identifies itself by sending signed message to **pod** privately
4. **Seed** registers an alias *Alice* with **pod**
5. **Pod** begins pushing message keys addressed to *Alice* over private channel
6. *Alice* asks **pod** for the public key for *Bob* and **pod** replies
7. *Alice* signs and encrypts a message to *Bob*
8. *Alice* stores the message at the appropriate seeds and it propagates the DHT
9. *Alice* informs **pod** that a message for *Bob* is stored at the given key
10. *Bob* gets notified by **pod** of the message
11. *Bob* fetches the message from the DHT
12. *Bob* decrypts and verifies the message from *Alice*

> The relative complexity of the aforementioned flow is entirely abstracted
> from users and implementing applications via *libmuttr*.

Sometimes seeds may not be capable of participating in the DHT directly due to
restrictions imposed by firewalls or configurations of public or corporate
networks. These seeds can optionally operate in "passive mode". In this mode,
a seed only opens a private channel to a pod and uses the RESTful HTTP API that
is exposed by the pod to store and fetch messages to and from the DHT. This
means that the pod is acting on behalf of the seed and the seed is not
contributing to the health of the network.

Protocol
--------

The Muttr protocol defines the message structure for seed-to-seed as well as
seed-to-pod communication. Muttr uses the Kademlia algorithm for routing
messages through the DHT, which will not be discussed here; only the message
structure and required contents will be defined here. For a more in-depth look
at the Kademlia implementation, see the specification used for
[Kad](https://github.com/gordonwritescode/kad), which is used by libmuttr
[here](http://xlattice.sourceforge.net/components/protocol/kademlia/specs.html).

A complete implementation of the protocol in JavaScript can be found at
[libmuttr](https://github.com/muttr/libmuttr). The following protocol
definition should serve as a reference for implementations in other languages.

In it's simplest form, the Muttr network is a distributed database of PGP
encrypted messages that are each retrievable by their respective SHA-1 hash.
The act of "sending" a message to another party is, in actuality, the act of
storing a message across peers in the network and subsequently providing the
recipient with the SHA-1 hash so they may retrieve the message from those peers.

### Seed-to-Seed Communication

Seeds communicate with one another by sending Muttr protocol compliant UDP
packets to indicate some request. These messages are largely related to the
determination of which peers should store some given content. UDP packets are
sent as the binary representation of a UTF-8 JSON string. The JSON structure
and it's properties are what the Muttr protocol enforces.

The JSON structure with all possible properties is:

```js
{
  "type": String(MESSAGE_TYPE),
  "params": {
    "key": String(SHA1_HASH_OF_VALUE),
    "value": String(HEX_ARMORED_PGP_MESSAGE),
    "timestamp": Number(UNIX_TIMESTAMP),
    "publisher": String(NODE_ID_OF_ORIGINAL_PUBLISHER),
    "address": String(YOUR_IP_ADDRESS),
    "port": Number(YOUR_UDP_PORT),
    "nodeID": String(SHA1_HASH_OF_ADDRESS_COLON_PORT),
    "rpcID": String(ID_FOR_MESSAGE),
    "referenceID": String(RPC_ID_IF_REPLY),
    "success": Boolean(OPERATION_SUCEEDED),
    "contacts": [
      {
        "address": String(IP_ADDRESS),
        "port": Number(UDP_PORT),
        "nodeID": String(CONTACT_ID)
      }
    ]
  }
}
```

#### Message Types

The `type` property of the message indicates the intended action or *method* to
execute. The `params` property contents will vary depending on the message
type, but you will **always** include `address`, `port`, `nodeID`, and *one* of
`rpcID` (if you are the initiator) or `referenceID` (if the message is a reply).

The different message types are described below.

* `PING`
  * Used to check if a peer is still active, according to Kademlia rules
  * Required params: *none*
* `PONG`
  * A response to `PING` to indicate you are still an active peer
  * Required params: *none*
* `STORE`
  * Used to request peer to store the given message at the given key
  * Required params: `key`, `value`, `timestamp`, `publisher`
* `STORE_REPLY`
  * A response to `STORE` to indicate whether or not the content was stored
  * Required params: `success`
* `FIND_NODE`
  * Used to ask peer for the 3 contacts in it's routing table closest to the key
  * Required params: `key`
* `FIND_NODE_REPLY`
  * A response to `FIND_NODE` including the 3 closest contacts to the key
  * Required params: `contacts`
* `FIND_VALUE`
  * Used to ask the peer for the value stored at the given key
  * Required params: `key`
* `FIND_VALUE_REPLY`
  * A response to `FIND_VALUE` including either the value *or* 3 close contacts
  * Required params: `value` **or** `contacts`

#### Key/Value Pairs

Values are stored and communicated in a specific format. A valid message is one
that is **both** signed and encrypted using the PGP format. The protocol cannot
prevent the storage and retrieval of unsigned messages. However, a proper
implementation will either ignore messages with invalid signatures after local
decryption or *at minimum* warn the user or application.

Values must be transmitted as the hexadecimal representation of an ASCII
armored PGP message. The corresponding key must be the hexadecimal SHA-1 hash
of the transmitted value.

### Seed-to-Pod Communication

Since pods more or less act as "super seeds", there is no difference in the way
seeds communicate with them in regard to the storage and retrieval of content
as described above. In fact, the additional responsibilities that pods take on
are managed over entirely different channels. This makes their integration into
the network frictionless.

Via these secondary channels, pods provide value to the network by offering the
following services:

* Federated public keyserver
* Identity management (username to public key aliasing)
* Instant message delivery to connected seeds
* Missed message playback for offline seeds
* Interface to network for "passive" seeds

Most of these services are provided through a standardized RESTful API that is
accessible via HTTPS, while instant message delivery is made available over a
WebSockets connection. Both of these interfaces are part of the Muttr protocol
and must be implemented to specification. A complete implementation in Node.js
is available at [MuttrPod](https://github.com/muttr/muttrpod).

#### RESTful Interface

Pods expose a HTTPS API for seeds to take advantage of the aforementioned
services that pods provide the network. This API is part of the Muttr protocol
and is necessary to the ease of operation of the network as a whole.

##### Authentication

Authentication is performed by signing an HTTPS request using a PGP private key
and providing information to the pod so it may verify that the request is
valid. You may authenticate requests to any pod, even if you do not have a
registered identity at the pod with which you are communicating. In this
regard, pods act as a federated routing system.

To authenticate, clients must sign a message with the following parameters, in
the form of a query string:

* `identity`
  * Examples: `4e1243bd22c66e76c2ba9eddc1f91394e57f9f83`, `alice@muttr.me`
* `identityType`
  * Examples: `pubkeyhash`, `href`
* `nonce`
  * Examples: `1437083459995`

If the `identityType` is `pubkeyhash`, the pod must lookup the public key
in it's local database and use it to verify the signature. If the value is
`href`, the pod must fetch the public key from the indicated host, using the
provided alias. In the case of `alice@muttr.me`, this would translate to:

```
https://muttr.me/aliases/alice
```

##### Authorization

Only identities that are registered with the target pod can become authorized.
An authorized client can playback messages she missed while she was offline as
well as purge the from the pod once they are retrieved. She may also register
new aliases that can be used to reach her, like `alice@muttr.me` or
`bob@muttr.me`.

If *authentication* is successful via an `identityType` of `pubkeyhash`, pods
must automatically authorize the client, without requiring another step. For
GET and DELETE requests, clients must provide a token previously acquired from
an authorized POST request to `/tokens` as described below.

##### Errors

All errors are indicated with an appropriate HTTP status code and sent as JSON
in the format of:

```js
{
  "error": ERROR_MESSAGE
}
```

##### Public Endpoints

Several operations are available to the public without the need to authenticate
or register an identity.

##### POST / - (Register Identity)

Registers a public key with the pod, creating an "account". Request body must
be an ASCII armored PGP public key.

Response:

```js
{
  "identity": {
    "pubkeyhash": PUBKEYHASH,
    "registered": UNIX_TIMESTAMP
  }
}
```

##### POST /messages - (Store Message in DHT)

Asks the pod to store a message in the DHT on your behalf. Request body must be
a hexadecimal encoded armored PGP encrypted message.

Response:

```js
{
  "key": SHA1_MESSAGE_HASH
}
```

##### GET /messages/:hash - (Fetch Message from DHT)

Returns the hexadecimal encoded armored PGP encrypted message.

##### GET /aliases/:alias - (Fetch Public Key for Alias)

Returns the ASCII armored PGP public key associated with the given alias.

##### Private Endpoints

##### POST /aliases - (Register an Alias)

Creates an alias for the registered client. Requires parameter: `alias`.

Response:

```js
{
  "alias": {
    "name": ALIAS,
    "created": UNIX_TIMESTAMP
  }
}
```

##### POST /tokens - (Create Auth Token for Request)

Creates an authorization token for GET and DELETE requests. Requires parameters:
`resource` (endpoint path) and `method` (HTTP verb).

Response:

```js
{
  "token": {
    "issued": UNIX_TIMESTAMP,
    "method": HTTP_VERB,
    "resource": RESOURCE_PATH,
    "value": TOKEN_STRING
  }
}
```

##### GET /inboxes - (Playback Missed Messages)

Returns an array of message keys stored on the pod for the caller. Requires
parameter: `token`.

Response:

```js
{
  "messages": [
    {
      "contact": CONTACT_ALIAS,
      "messages: [
        {
          "recipient": {
            "userID": ALIAS_AT_PODHOST,
            "pubkeyhash": SHA1_OF_PUBKEY
          },
          "sender": {
            "userID": ALIAS_AT_PODHOST,
            "pubkeyhash": SHA1_OF_PUBKEY
          },
          "key": SHA1_MESSAGE_KEY,
          "timestamp": UNIX_TIMESTAMP
        }
      ]
    }
  ]
}
```

##### POST /inboxes/:alias - (Store a Message Key for the Alias)

Notifies the recipient of a message key and stores it for playback if offline.
Requires parameters: `from` (your alias) and `key` (SHA-1 message key).

Response:

```js
{
  "message": {
    "sender": {
      "userID": ALIAS_AT_PODHOST,
      "pubkeyhash": SHA1_OF_PUBKEY
    },
    "recipient": {
      "userID": ALIAS_AT_PODHOST,
      "pubkeyhash": SHA1_OF_PUBKEY
    },
    "key": SHA1_MESSAGE_KEY,
    "timestamp": UNIX_TIMESTAMP
  }
}
```

##### DELETE /inboxes - (Purge Stored Message Keys)

Deletes all stored messages from the server for the authorized client. Requires
parameter: `token`.

Response:

```js
{
  "messages": []
}
```

#### WebSockets Interface

The WebSockets interface is reserved for use by registered identities. A user
or application connected via the WebSockets interface will begin receiving
messages after authentication that indicate when another party has stored a
message in the DHT that is intended for her. The pod will provide the SHA-1
key to the subscriber, so that she may then fetch the message from the network
or ask the pod to do so if she is running in passive mode.

##### Connecting

Clients connect to the WebSockets interface at the base URI of the pod. If the
hostname of the pod is `muttr.me`, then the WebSocket URI is `wss://muttr.me`.
Once the connection is open, the client must prove her identity in order to
begin receiving message events.

##### Authentication

For the client to prove her identity, she must send a PGP signed message that
indicates to the pod her public key. The pod will lookup the key in it's known
registered identities and verify the signature. If the signature is verified,
then the client will begin receiving messages. If authentication fails, the pod
must close the WebSockets connection and the client must connect again.

The authentication message expected by the pod should be the hexadecimal
representation of an ASCII armored PGP signed message (cleartext). The message
contents must be in the form of a querystring and contain the following
parameters:

* `identity`
* `identityType`
* `nonce`

The value of `identity` is an identifier that the pod understands as indicated
by the `identityType` parameter. The protocol only requires that pods accept
*pubkeyhash* as an identity type, which is the SHA-1 hash of the ASCII armored
PGP public key. The nonce is included to prevent replay attacks.

An example authentication message (before conversion to hex) is illustrated
below:

```
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA1

identity=4e1243bd22c66e76c2ba9eddc1f91394e57f9f83&identityType=pubkeyhash&nonce=1437075033746
-----BEGIN PGP SIGNATURE-----
Version: GnuPG v1

iQEcBAEBAgAGBQJVqAazAAoJEOMLJDdLN/dUINYH/1/2BDxi+YE1pok/VZdJyEr9
Ue+n0dl5kyk/h+jEvlIA55dbxa/AeZuowDVrhFf4I2sv8uifPHrOCZVwK7l0NHy4
orQqZTd1q4uBVPcj5rW6awjTHUs9eQ6QfdhNFF34xYgRGXz9KZ4XZ1MCnT92qCB7
OW5fWj/BWlJxKlXhLV2E1nRWDNFCf9+K2PXt5408BFxX2f0EDfMf8YRFL5kiJrRT
zMDeWK0ZkA94ONxFNZuQjYsBHie6Vd+XKwz+P7TFI0TmyvJ6Q2/d7ssZxs9iKOwJ
UjGSHthGs5KudXjqHU90+30yU9yZWN7Oc7+v4MOrVjGkVkk40MnkTYTRzIXUFBQ=
=Gs9U
-----END PGP SIGNATURE-----
```

The pod will use the supplied identity information to lookup the corresponding
public key that is registered in it's database. That public key will be used to
verify the signature and therefore authenticate the client.

##### Message Format

Once an authenticated session is established, the pod will begin relaying the
location of messages in the DHT by providing the message's SHA-1 hash, which is
used as the lookup key in the DHT). The messages relayed from the pod are JSON
objects in the form of:

```js
{
  "recipient": {
    "userID": "you@podhost",
    "pubkeyhash": "4e1243bd22c66e76c2ba9eddc1f91394e57f9f83"
  },
  sender: {
    "userID": "them@podhost",
    "pubkeyhash": "5e1243bd22c66e76c2ba9eddc1f91394e57f9f83"
  },
  key: "6e1243bd22c66e76c2ba9eddc1f91394e57f9f83",
  timestamp: 1437075033746
}
```

This provides enough information to the client in order to query the network
for the message, then decrypt and verify the signature.

Implementations
---------------

### libmuttr

The libmuttr project provides a complete implementation of the protocol,
cryptography, and networking. [Source](https://github.com/muttr/libmuttr)

### muttrpod

A standalone, ready to deploy pod, built on libmuttr.
[Source](https://github.com/muttr/muttrpod)

### muttrd

Simple daemon built on libmuttr for developing applications on the muttr
network. [Source](https://github.com/muttr/muttrd)

### muttrchat

Proof of concept chat application, using muttrd.
[Source](https://github.com/muttr/muttrchat)

Comments
--------

Improvements to the Muttr protocol are considered by the community by the
submission of a MIP (Muttr Improvement Proposal). MIPs are submitted by issuing
a pull request the the [MIPS repository](https://github.com/muttr/mips) that
contains a document detailing the improvement. MIPs do not need to be approved
by any authority or central party, instead they are treated as protocol
enhancements that any implementor may choose to support and include in their
implementation. This should be considered when drafting your proposal.

---

This work is licensed under the Creative Commons Attribution-ShareAlike 4.0
International License. To view a copy of this license, visit
http://creativecommons.org/licenses/by-sa/4.0/.
