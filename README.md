[![Feedme](https://raw.githubusercontent.com/aarong/feedme-client/master/logo.svg?sanitize=true)](https://feedme.global)

# Feedme Specification

Version 0.1 - Working Draft

This document prescribes the JSON message format and behavior to be used by
Feedme real-time API clients and servers.

[https://feedme.global](https://feedme.global)

<!-- TOC depthFrom:2 -->

- [Introduction](#introduction)
- [Concepts and Terms](#concepts-and-terms)
  - [Transport](#transport)
  - [Handshakes](#handshakes)
  - [Feeds](#feeds)
  - [Actions](#actions)
  - [Action Notifications](#action-notifications)
- [Messages](#messages)
  - [Client-Originating Message Types](#client-originating-message-types)
    - [Handshake](#handshake)
    - [Action](#action)
    - [FeedOpen](#feedopen)
    - [FeedClose](#feedclose)
  - [Server-Originating Message Types](#server-originating-message-types)
    - [Responses to Client Messages](#responses-to-client-messages)
      - [ViolationResponse](#violationresponse)
      - [HandshakeResponse](#handshakeresponse)
      - [ActionResponse](#actionresponse)
      - [FeedOpenResponse](#feedopenresponse)
      - [FeedCloseResponse](#feedcloseresponse)
    - [Feed Notifications](#feed-notifications)
      - [FeedAction](#feedaction)
      - [FeedTermination](#feedtermination)
- [Message Sequencing](#message-sequencing)
  - [Fundamentals](#fundamentals)
  - [Handshakes](#handshakes-1)
    - [Client](#client)
    - [Server](#server)
  - [Actions](#actions-1)
    - [Client](#client-1)
    - [Server](#server-1)
  - [Feeds](#feeds-1)
    - [Client](#client-2)
    - [Server](#server-2)
- [Feed Data](#feed-data)
  - [Feed Deltas](#feed-deltas)
    - [Paths](#paths)
    - [Operations](#operations)
    - [Multi-type Operations](#multi-type-operations)
      - [Set](#set)
      - [Delete](#delete)
      - [DeleteValue](#deletevalue)
    - [String Operations](#string-operations)
      - [Prepend](#prepend)
      - [Append](#append)
    - [Number Operations](#number-operations)
      - [Increment](#increment)
      - [Decrement](#decrement)
    - [Boolean Operations](#boolean-operations)
      - [Toggle](#toggle)
    - [Array Operations](#array-operations)
      - [InsertFirst](#insertfirst)
      - [InsertLast](#insertlast)
      - [InsertBefore](#insertbefore)
      - [InsertAfter](#insertafter)
      - [DeleteFirst](#deletefirst)
      - [DeleteLast](#deletelast)
  - [Integrity Verification](#integrity-verification)
- [Violations of the Specification](#violations-of-the-specification)
  - [Invalid Client-Originating Messages](#invalid-client-originating-messages)
  - [Invalid Server-Originating Messages](#invalid-server-originating-messages)

<!-- /TOC -->

## Introduction

Implementations of this specification enable clients to:

1. Perform operations on a server and be notified about the outcome.

2. Access data on the server and be notified when and changes and why.

3. Subscribe to notifications about operations on the server.

The first objective is achieved using `Actions`; the second and third are
achieved using actions in conjunction with `Feeds`. A Feedme API is a
server-defined set of actions and feeds that clients can interact with.

The specification is agnostic about the platform of the server, the platforms of
the clients, and the communication channel between them. It requires only that
clients and the server be able to exchange and interpret JSON messages.

## Concepts and Terms

### Transport

The specification takes as given that each client is able to establish a
connection to the server through some arbitrary communication channel. This
communication channel is referred to in the abstract as the `Transport`.

Minimal functionality is required of the transport. Once the client has
established a connection to the server via the transport, the transport must
enable the two sides to exchange string JSON messages of sufficient length to
support the API running through it. The transport must ensure that messages are
received by the other side in the order that they were sent.

Feedme APIs may support more than one transport. Supported transports should be
documented for API clients.

### Handshakes

Once a client has connected to the server via the transport, the client performs
a `Handshake` to initiate the Feedme conversation.

### Feeds

Clients access live data and notification streams using `Feeds`. Clients `Open`
feeds by specifying a `Feed Name` and may supply `Feed Arguments` with further
specifics.

When a client opens a feed, the server returns the current `Feed Data` and
commits to notifying the client about future changes to that data. The client
may subsequently `Close` the feed and the server may forcibly `Terminate` it.

The following are determined by the API design and should be documented for API
clients:

- The set of available feeds.

- The required structure of their arguments.

- The structure of their feed data.

- The conditions under which open feeds will be terminated.

### Actions

The core unit of behavior is the `Action`. Actions may be performed explicitly
by clients or may be deemed to have occurred by the server.

In order to perform an action, clients indicate an `Action Name` and may supply
`Action Arguments` with further specifics. If the action is executed
successfully, the server returns `Action Data` to the client describing the
outcome.

The following are determined by the API design and should be documented for API
clients:

- The set of actions that may be invoked by clients.

- The required structure of their arguments.

- The structure of the resulting action data.

### Action Notifications

When an action is invoked by a client or is deemed to have occurred by the
server, the server may transmit an `Action Notification` on one or more feeds.

When the server notifies a client about an action on one its open feeds, the
server specifies an `Action Name` and `Action Data` with further specifics.

The server may also specify a sequence of `Feed Deltas` describing any resulting
changes to the feed data. Feed data can only change as a result of action
notifications.

The following are determined by the API design and should be documented for API
clients:

- The set of actions that can be transmitted as notifications on each feed.

- The structure of any associated action data.

- The manner in which actions modify the feed data.

## Messages

Messages exchanged across the transport must be valid JSON-encoded objects.
Client-originating messages must satisfy the
[client-message](schemas/client-message.json) schema and server-originating
messages must satisfy the [server-message](schemas/server-message.json) schema.

Messages take the following form:

```json
{
    "MessageType": "SOME_MESSAGE_TYPE",
    ...
}
```

Parameters:

- `MessageType` (string) indicates the type of message being transmitted, which
  puts additional requirements on the structure of the message.

### Client-Originating Message Types

Client messages must satisfy the [client-message](schemas/client-message.json)
schema.

Clients can send four types of messages:

- `Handshake` initiates the Feedme conversation.

- `Action` asks to perform an action on the server.

- `FeedOpen` asks to open a feed.

- `FeedClose` closes a feed.

#### Handshake

A `Handshake` message is used to initiate the Feedme conversation and must
satisfy the [handshake](schemas/handshake.json) schema.

The server must respond to a valid `Handshake` message with a
`HandshakeResponse` message.

The server must respond to an invalid `Handshake` message with a
`ViolationResponse` message.

Messages take the following form:

```json
{
  "MessageType": "Handshake",
  "Versions": ["0.1"]
}
```

Parameters:

- `Versions` (array of strings) indicates the Feedme specification versions
  supported by the client. The current and only version of the specification is
  0.1.

#### Action

An `Action` message is used to invoke an action on the server and must satisfy
the [action](schemas/action.json) schema.

The server must respond to a valid `Action` message with an `ActionResponse`
message.

The server must respond to an invalid `Action` message with a
`ViolationResponse` message.

Messages take the following form:

```json
{
  "MessageType": "Action",
  "ActionName": "SOME_ACTION_NAME",
  "ActionArgs": { ... },
  "CallbackId": "SOME_CALLBACK_ID"
}
```

Parameters:

- `ActionName` (string) is the name of the action to be invoked.

- `ActionArgs` (object) contains any action arguments.

- `CallbackId` (string) is an arbitrary, client-determined callback identifier,
  which is returned with the server's response.

  Once a client has transmitted an `Action` message with a given `CallbackId`,
  it must not reuse that `CallbackId` in another `Action` message until it has
  received the associated `ActionResponse` message from the server.

#### FeedOpen

A `FeedOpen` message is used to open a feed and must satisfy the
[feed-open](schemas/feed-open.json) schema.

The server must respond to a valid `FeedOpen` message with a `FeedOpenResponse`
message.

The server must respond to an invalid `FeedOpen` message with a
`ViolationResponse` message.

Messages take the following form:

```json
{
  "MessageType": "FeedOpen",
  "FeedName": "SOME_FEED_NAME",
  "FeedArgs": { ... }
}
```

Parameters:

- `FeedName` (string) is the name of the feed to open.

- `FeedArgs` (object of strings) contains any feed arguments.

#### FeedClose

A `FeedClose` message instructs the server to close a feed and must satisfy the
[feed-close](schemas/feed-close.json) schema.

The server must respond to a valid `FeedClose` message with a
`FeedCloseResponse` message.

The server must respond to an invalid `FeedClose` message with a
`ViolationResponse` message.

Messages take the following form:

```json
{
  "MessageType": "FeedClose",
  "FeedName": "SOME_FEED_NAME",
  "FeedArgs": { ... }
}
```

Parameters:

- `FeedName` (string) is the name of the feed to close.

- `FeedArgs` (object of strings) contains any feed arguments.

### Server-Originating Message Types

Server messages must satisfy the [server-message](schemas/server-message.json)
schema.

The server can send seven types of messages.

Five message types are transmitted in response to client-originating messages:

- `ViolationResponse` responds to a client message that violates the
  specification.

- `HandshakeResponse` responds to a client `Handshake` message.

- `ActionResponse` responds to a client `Action` message.

- `FeedOpenResponse` responds to a client `FeedOpen` message.

- `FeedCloseResponse` responds to a client `FeedClose` message.

Two message types are initiated by the server to notify clients about
developments on open feeds:

- `FeedAction` notifies the client about an action that has occurred on the
  server.

- `FeedTermination` indicates that a previously open feed has been forcibly
  closed.

#### Responses to Client Messages

##### ViolationResponse

A `ViolationResponse` is used to respond to any client message that violates the
specification and must satisfy the
[violation-response](schemas/violation-response.json) schema.

Messages take the following form:

```json
{
  "MessageType": "ViolationResponse",
  "Diagnostics": { ... }
}
```

Parameters:

- `Diagnostics` (object) may contain debugging information.

##### HandshakeResponse

A `HandshakeResponse` message is used to respond to a valid client `Handshake`
message and must satisfy the
[handshake-response](schemas/handshake-response.json) schema.

If returning failure, messages take the following form:

```json
{
  "MessageType": "HandshakeResponse",
  "Success": false
}
```

Failure parameters:

- `Success` (boolean) is set to false, indicating that none of the versions
  specified in the client `Handshake` message are supported by the server.

If returning success, messages take the following form:

```json
{
  "MessageType": "HandshakeResponse",
  "Success": true,
  "Version": "0.1"
}
```

Success parameters:

- `Success` (boolean) is set to true, indicating that one or more of the
  versions specified in the client `Handshake` message is supported by the
  server.

- `Version` (string) is the Feedme specification version that will govern the
  conversation. It must have been selected from the versions specified in the
  client `Handshake` message.

##### ActionResponse

An `ActionResponse` message is used to respond to a valid client `Action`
message and must satisfy the [action-response](schemas/action-response.json)
schema.

If returning failure, messages take the following form:

```json
{
  "MessageType": "ActionResponse",
  "CallbackId": "SOME_CALLBACK_ID",
  "Success": false,
  "ErrorCode": "SOME_ERROR_CODE",
  "ErrorData": { ... }
}
```

Failure parameters:

- `CallbackId` (string) is the value specified in the client `Action` message.

- `Success` (boolean) is set to false, indicating failure.

- `ErrorCode` (string) indicates the nature of the problem.

- `ErrorData` (object) may contain further diagnostic information.

If returning success, messages take the following form:

```json
{
  "MessageType": "ActionResponse",
  "CallbackId": "SOME_CALLBACK_ID",
  "Success": true,
  "ActionData": { ... }
}
```

Parameters:

- `CallbackId` (string) is the value specified in the client `Action` message.

- `Success` (boolean) is set to true, indicating success.

- `ActionData` (object) describes the outcome of the action.

##### FeedOpenResponse

A `FeedOpenResponse` message is used to respond to a valid client `FeedOpen`
message and must satisfy the
[feed-open-response](schemas/feed-open-response.json) schema.

If returning failure, messages take the following form:

```json
{
  "MessageType": "FeedOpenResponse",
  "FeedName": "SOME_FEED_NAME",
  "FeedArgs": { ... },
  "Success": false,
  "ErrorCode": "SOME_ERROR_CODE",
  "ErrorData": { ... }
}
```

Failure parameters:

- `Success` (boolean) is set to false, indicating failure.

- `FeedName` (string) is the feed name specified by the client.

- `FeedArgs` (object of strings) contains any feed arguments specified by the
  client.

- `ErrorCode` (string) indicates the nature of the problem.

- `ErrorData` (object) may contain further diagnostic information.

If returning success, messages take the following form:

```json
{
  "MessageType": "FeedOpenResponse",
  "FeedName": "SOME_FEED_NAME",
  "FeedArgs": { ... },
  "Success": true,
  "FeedData": { ... }
}
```

Success parameters:

- `Success` (boolean) is set to true, indicating success.

- `FeedName` (string) is the feed name specified by the client.

- `FeedArgs` (object of strings) contains any feed arguments specified by the
  client.

- `FeedData` (object) is the current feed data.

##### FeedCloseResponse

A `FeedCloseResponse` message is used to respond to a valid client `FeedClose`
message and must satisfy the
[feed-close-response](schemas/feed-close-response.json) schema.

A `FeedCloseResponse` message indicates that the feed has been closed
successfully. The server is not permitted to reject valid `FeedClose` requests.

Messages take the following form:

```json
{
  "MessageType": "FeedCloseResponse",
  "FeedName": "SOME_FEED_NAME",
  "FeedArgs": { ... }
}
```

Parameters:

- `FeedName` (string) is the feed name specified by the client.

- `FeedArgs` (object of strings) contains any feed arguments specified by the
  client.

#### Feed Notifications

##### FeedAction

A `FeedAction` message is used to notify a client about an action on one of its
open feeds and must satisfy the [feed-action](schemas/feed-action.json) schema.

Messages take the following form:

```json
{
  "MessageType": "FeedAction",
  "FeedName": "SOME_FEED_NAME",
  "FeedArgs": { ... },
  "ActionName": "SOME_ACTION_NAME",
  "ActionData": { ... },
  "FeedDeltas": [ ... ],
  "FeedMd5": "SOME_BASE64_MD5"
}
```

Parameters:

- `FeedName` (string) is the name of the feed on which the client is receiving
  the notification.

- `FeedArgs` (object of strings) contains any arguments for the feed.

- `ActionName` (string) is the name of the action that has occurred.

- `ActionData` (object) is the action data for the action that has occurred. It
  may differ from action data passed with other `FeedAction` and
  `ActionResponse` messages.

- `FeedDeltas` (array of [delta objects](#feed-deltas)) contains a sequence of
  operations to be applied to the feed data. It may differ from the deltas
  passed with other `FeedAction` messages.

- `FeedMd5` (optional string) is a Base64-encoded MD5 hash of the feed data
  after the feed deltas have been applied. It will differ from the hashes passed
  with other `FeedAction` messages if the post-delta feed data is also
  different. It may be omitted if the server does not wish to enable feed data
  verification. If included, it must be exactly 24 characters long.

##### FeedTermination

A `FeedTermination` message is used to notify a client that the server has
forcibly closed a previously open feed and must satisfy the
[feed-termination](schemas/feed-termination.json) schema.

Messages take the following form:

```json
{
  "MessageType": "FeedTermination",
  "FeedName": "SOME_FEED_NAME",
  "FeedArgs": { ... },
  "ErrorCode": "SOME_ERROR_CODE",
  "ErrorData": { ... }
}
```

Parameters:

- `FeedName` (string) is the name of the feed being terminated.

- `FeedArgs` (object of strings) contains any feed arguments.

- `ErrorCode` (string) indicates the reason for termination.

- `ErrorData` (object) may contain further information.

## Message Sequencing

Messages must be sequenced according to the following rules in order to ensure
that the client and server maintain a common understanding of the state of the
conversation.

### Fundamentals

The conversation is controlled by the client. The client initiates the
conversation by transmitting a `Handshake` message to the server and then steers
the interaction by transmitting `Action`, `FeedOpen`, and `FeedClose` messages.

The server must return exactly one message in response to each
client-originating message. The server is not obligated to respond to client
messages in the order that they are received.

### Handshakes

#### Client

Once connected to the server via the transport, the client must treat the
conversation as being in one of three states. The client must initially treat
the conversation as `Not Initiated`.

1. `Not Initiated` - When the conversation is in this state...

- The client must transmit a `Handshake` message to the server and subsequently
  treat the conversation as `Handshaking`. The client must not transmit any
  other type of message to the server.

- If the server is complying with the specification, the client will not receive
  any messages.

2. `Handshaking` - When the conversation is in this state...

- The client must not transmit any messages to the server.

- If the server is complying with the specification, then the client will
  receive only a `HandshakeResponse` message. Once received, the client must
  subsequently treat the conversation as either `Initiated` or `Not Initiated`,
  depending on whether the server indicated success or failure.

3. `Initiated` - When the conversation is in this state...

- The client may transmit `Action`, `FeedOpen`, and `FeedClose` messages to the
  server at its discretion, subject to other sequencing requirements. The client
  must not transmit `Handshake` messages to the server.

- If the server is complying with the specification, then the client may receive
  `ActionResponse`, `FeedOpenResponse`, `FeedCloseResponse`, `FeedAction`, and
  `FeedTermination` messages, subject to other sequencing requirements.

#### Server

The server must treat each client conversation as being in one of three states.
The server must initially treat client conversations as `Not Initiated`.

1. `Not Initiated` - When the conversation is in this state...

- The server must not transmit any messages to the client.

- If the client is complying with the specification, then the server will
  receive only a `Handshake` message. Once received, the server must
  subsequently treat the conversation as `Handshaking`.

2. `Handshaking` - When the conversation is in this state...

- The server must transmit a `HandshakeResponse` message to the client and
  subsequently treat the conversation as either `Initiated` or `Not Initiated`,
  depending on whether it indicated success or failure.

- If the client is complying with the specification, the server will not receive
  any messages.

3. `Initiated` - When the conversation is in this state...

- The server may transmit `ActionResponse`, `FeedOpenResponse`,
  `FeedCloseResponse`, `FeedAction`, and `FeedTermination` messages to the
  client, subject to other sequencing requirements. The server must not transmit
  `HandshakeResponse` messages to the client.

- If the client is complying with the specification, then the server may receive
  `Action`, `FeedOpen`, and `FeedClose` messages, subject to other sequencing
  requirements.

### Actions

#### Client

Once a conversation is `Initiated`, the client may transmit `Action` messages to
the server at its discretion. Clients need not await the response to one
`Action` message before transmitting another.

#### Server

Once a conversation is `Initiated`, the server must respond to each valid
`Action` message by returning an `ActionResponse` message referencing the same
callback identifier. Any associated `FeedAction` messages sent to the client
performing the action may be transmitted either before or after the
`ActionResponse` message.

### Feeds

Feeds are idenitified by a feed name and a set of zero or more key-value feed
arguments. Different feeds have different feed names, or different feed argument
keys, or different feed argument values, or some combination of the preceding.

#### Client

The client must treat each feed as being in one of five states. The client must
initially treat all feeds as `Closed`.

Client feed states:

1. `Closed` - When a feed is in this state...

- The client may, at its discretion, transmit a `FeedOpen` message referencing
  the feed. If it does so, the client must subsequently treat the feed as
  `Opening`.

- The client must not transmit a `FeedClose` message referencing the feed.

- If the server is complying with the specification, then the client will not
  receive any messages referencing the feed.

2. `Opening` - When a feed is in this state...

- The client must not transmit any messages referencing the feed.

- If the server is complying with the specification, then the client will
  receive a `FeedOpenResponse` message referencing the feed. Once received, the
  client must subsequently treat the feed as either `Open` or `Closed`,
  depending on whether the server indicated success or failure.

3. `Open` - When a feed is in this state...

- The client may, at its direction, transmit a `FeedClose` message referencing
  the feed. If it does so, the client must subsequently treat the feed as
  `Closing`.

- The client must not transmit a `FeedOpen` message referencing the feed.

- If the server is complying with the specification, then the client may receive
  an `FeedAction` message referencing the feed. If received, then the client
  must continue to treat the feed as `Open`.

- If the server is complying with the specification, then the client may receive
  a `FeedTermination` message referencing the feed. If received, then the client
  must subsequently treat the feed as `Closed`.

- If the server is complying with the specification, then the client will not
  receive a `FeedOpenResponse` or `FeedCloseResponse` message referencing the
  feed.

4. `Closing` - When a feed is in this state...

- The client must not transmit any messages referencing the feed.

- If the server is complying with the specification, then the client may receive
  an `FeedAction` message referencing the feed. If the server is in compliance,
  then it transmitted the `FeedAction` message before it received the
  `FeedClose` message from the client. If received, then the client must
  continue to treat the feed as `Closing`.

- If the server is complying with the specification, then the client may receive
  a `FeedTermination` message referencing the feed. If the server is in
  compliance, then it transmitted the `FeedTermination` message before it
  received the `FeedClose` message from the client. If received, then the client
  must subsequently treat the feed as `Terminated`.

- If the server is complying with the specification, then the client may receive
  a `FeedCloseResponse` message referencing the feed. Once received, the client
  must subsequently treat the feed as `Closed`.

- If the server is complying with the specification, then the client will not
  receive a `FeedOpenResponse` message referencing the feed.

5. `Terminated` - When a feed is in this state...

- The client must not transmit any messages referencing the feed.

- If the server is complying with the specification, the client will receive a
  `FeedCloseResponse` message referencing the feed. Once received, the client
  must subsequently treat the feed as `Closed`.

- If the server is complying with the specification, then the client will not
  receive a `FeedOpenResponse`, `FeedAction`, or `FeedTermination` message
  referencing the feed.

#### Server

For each client, the server must treat each feed as being in one of five states.
When a client connects, the server must initially treat all feeds as `Closed`.

Server feed states:

1. `Closed` - When a client feed is in this state...

- The server must not transmit any messages referencing the feed.

- If the client is complying with the specification, then the server may receive
  a `FeedOpen` message referencing the feed. If received, then the server must
  subsequently treat the feed as `Opening`.

- If the client is complying with the specification, then the server will not
  receieve a `FeedClose` message referencing the feed.

2. `Opening` - When a client feed is in this state...

- The server must transmit a `FeedOpenResponse` message to the client and
  subsequently treat the feed as either `Open` or `Closed`, depending on whether
  the message indicated success or failure.

- The server must not transmit a `FeedCloseResponse`, `FeedAction`, or
  `FeedTermination` message referencing the feed.

- If the client is complying with the specification, then the server will not
  receive any messages referencing the feed.

3. `Open` - When a client feed is in this state...

- The server may, at its discretion, transmit an `FeedAction` message
  referencing the feed. If sent, the server must continue to treat the feed as
  `Open`.

- The server may, at its discretion, transmit a `FeedTermination` message
  referencing the feed. If sent, the server must subsequently treat the feed as
  `Terminated`.

- The server must not transmit a `FeedOpenResponse` or `FeedCloseResponse`
  message referencing the feed.

- If the client is complying with the specification, then the server may receive
  a `FeedClose` message referencing the feed. If received, the server must
  subsequently treat the feed as `Closing`.

- If the client is complying with the specification, then the server will not
  receive a `FeedOpen` message referencing the feed.

4. `Closing` - When a client feed is in this state...

- The server must transmit a `FeedCloseResponse` message to the client and
  subsequently treat the feed as `Closed`.

- The server must not transmit a `FeedOpenResponse`, `FeedAction`, or
  `FeedTermination` message referencing the feed.

- If the client is complying with the specification, then the server will not
  receive any messages referencing the feed.

5. `Terminated` - When a client feed is in this state...

- The server must not transmit any messages referencing the feed.

- If the client is complying with the specification, then the server may receive
  a `FeedOpen` message referencing the feed. If received, then the server must
  subsequently treat the feed as `Opening`.

- If the client is complying with the specification, then the server may receive
  a `FeedClose` message referencing the feed. If the client is in compliance,
  then it transmitted the `FeedClose` message before receiving the
  `FeedTermination` message from the server. If received, then the server must
  subsequently treat the feed as `Closing`.

- After a period of time has elapsed, the server may deem the feed `Closed` and
  treat it accordingly. The interval of time should be appropriate for the
  latency of the transport.

## Feed Data

When a client successfully opens a feed, the feed data is returned with the
server's `FeedOpenResponse` message and may be modified by subsequent
`FeedAction` messages.

The server may use `FeedAction` messages to notify about actions explicitly
invoked by clients using an `Action` message, as well as actions deemed by the
server to have taken place without an associated client `Action` message.

### Feed Deltas

When the server transmits a `FeedAction` message, it must include a `FeedDeltas`
array describing any resulting changes to the feed data. Each element of the
array must be a valid feed delta object describing an operation to be performed
on the feed data. The server may transmit an empty `FeedDeltas` array to
indicate that the feed data has not changed.

When the client receives a `FeedAction` message, it must apply any feed delta
operations to its local copy of the feed data. Operations must be applied in
array order.

Each delta operation transmitted by the server must be valid given the current
state of client feed data, as determined by the initial `FeedOpenResponse`
message, earlier `FeedAction` messages, and operations located earlier in the
`FeedDeltas` array.

Feed delta objects take the following form:

```json
{
  "Operation": "SOME_OPERATION",
  "Path": [ ... ],
  ...
}
```

Parameters:

- `Operation` (string) is the name of the delta operation to perform. Its value
  may put additional requirements on the feed delta object, including the path.

- `Path` (array) describes the location in the feed data to operate upon.

#### Paths

`Path` arrays are used to reference locations in the feed data object. Paths are
specified according to the following rules:

- `[]` refers to the root of the feed data object.

- `["Foo"]` refers to the `Foo` property of the root feed data object.

- `["Foo", "Bar"]` refers to the `Bar` property of the `Foo` object, which in
  turn is a child of the root feed data object.

- `["Baz", 0]` refers to the first element of the `Baz` array, which in turn is
  a child of the root feed data object.

- Child property names and array indexes may be chained to arbitrary length.

The first element of a `Path` array must be a string if present, as the feed
data root is always an object.

#### Operations

#### Multi-type Operations

##### Set

The `Set` operation writes a value to the specified path and must satisfy the
[feed-delta-set](schemas/feed-delta-set.json) schema.

Delta objects take the following form:

```json
{
  "Operation": "Set",
  "Path": [ ... ],
  "Value": ...
}
```

Parameters:

- `Path` (path array) must point to one of the following:

  1. An existing value of any type. Could refer to the root object, a child
     property of an object, or an element of an array.

  2. A non-existing child property of an existing object.

  3. A non-existing element of an existing array. Because JSON cannot represent
     arrays with missing elements, the referenced element must be directly after
     the last element in the array. If the array is empty, the path must
     reference the zero index.

- `Value` (any JSON value) is written to the referenced path. If the path
  references the root feed data object, then `Value` must be an object.

##### Delete

The `Delete` operation removes an object child property by name or an array
element by index.

Delta objects take the following form:

```json
{
  "Operation": "Delete",
  "Path": [ ... ]
}
```

Parameters:

- `Path` (path array) must point to one of the following:

  1. An existing child property of an object.

  2. An existing element of an array.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "const": "Delete"
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string"
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string"
          },
          {
            "type": "number",
            "multipleOf": 1,
            "minimum": 0
          }
        ]
      }
    }
  },
  "required": ["Operation", "Path"],
  "additionalProperties": false
}
```

##### DeleteValue

The `DeleteValue` operation removes all object child properties or array
elements that deep-equal a specified value.

Delta objects take the following form:

```json
{
  "Operation": "DeleteValue",
  "Path": [ ... ],
  "Value": ...
}
```

Parameters:

- `Path` (path array) must point to one of the following:

  - An existing object (may be the root object).

  - An existing array.

- `Value` (any JSON value) determines the child properties or array elements to
  be deleted.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "const": "DeleteValue"
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string"
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string"
          },
          {
            "type": "number",
            "multipleOf": 1,
            "minimum": 0
          }
        ]
      }
    },
    "Value": {}
  },
  "required": ["Operation", "Path", "Value"],
  "additionalProperties": false
}
```

#### String Operations

##### Prepend

The `Prepend` operation adds a specified string to the beginning of an existing
string.

Delta objects take the following form:

```json
{
  "Operation": "Prepend",
  "Path": [ ... ],
  "Value": "SOME_STRING"
}
```

Parameters:

- `Path` (path array) must point to an existing string value.

- `Value` (string) is prepended to the referenced string.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "const": "Prepend"
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string"
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string"
          },
          {
            "type": "number",
            "multipleOf": 1,
            "minimum": 0
          }
        ]
      }
    },
    "Value": {
      "type": "string"
    }
  },
  "required": ["Operation", "Path", "Value"],
  "additionalProperties": false
}
```

##### Append

The `Append` operation adds a specified string to the end of an existing string.

Delta objects take the following form:

```json
{
  "Operation": "Append",
  "Path": [ ... ],
  "Value": "SOME_STRING"
}
```

Parameters:

- `Path` (path array) must point to an existing string value.

- `Value` (string) is appended to the referenced string.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "const": "Append"
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string"
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string"
          },
          {
            "type": "number",
            "multipleOf": 1,
            "minimum": 0
          }
        ]
      }
    },
    "Value": {
      "type": "string"
    }
  },
  "required": ["Operation", "Path", "Value"],
  "additionalProperties": false
}
```

#### Number Operations

##### Increment

The `Increment` operation increases a number by a specified amount.

Delta objects take the following form:

```json
{
  "Operation": "Increment",
  "Path": [ ... ],
  "Value": 1
}
```

Parameters:

- `Path` (path array) must point to an existing number.

- `Value` (number) is added to the referenced number.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "const": "Increment"
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string"
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string"
          },
          {
            "type": "number",
            "multipleOf": 1,
            "minimum": 0
          }
        ]
      }
    },
    "Value": {
      "type": "number"
    }
  },
  "required": ["Operation", "Path", "Value"],
  "additionalProperties": false
}
```

##### Decrement

The `Decrement` operation decreases a number by a specified amount.

Delta objects take the following form:

```json
{
  "Operation": "Decrement",
  "Path": [ ... ],
  "Value": 1
}
```

Parameters:

- `Path` (path array) must point to an existing number.

- `Value` (number) is subtracted from the referenced number.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "const": "Decrement"
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string"
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string"
          },
          {
            "type": "number",
            "multipleOf": 1,
            "minimum": 0
          }
        ]
      }
    },
    "Value": {
      "type": "number"
    }
  },
  "required": ["Operation", "Path", "Value"],
  "additionalProperties": false
}
```

#### Boolean Operations

##### Toggle

The `Toggle` operation inverts a boolean value.

Delta objects take the following form:

```json
{
  "Operation": "Toggle",
  "Path": [ ... ]
}
```

Parameters:

- `Path` (path array) must point to an existing boolean value.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "const": "Toggle"
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string"
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string"
          },
          {
            "type": "number",
            "multipleOf": 1,
            "minimum": 0
          }
        ]
      }
    }
  },
  "required": ["Operation", "Path"],
  "additionalProperties": false
}
```

#### Array Operations

To set an array element by index, use the `Set` operation.

To delete an array element by index, use the `Delete` operation.

To delete array elements by value, use the `DeleteValue` operation.

##### InsertFirst

The `InsertFirst` operation adds a specified value to the beginning of an array.

Delta objects take the following form:

```json
{
  "Operation": "InsertFirst",
  "Path": [ ... ],
  "Value": ...
}
```

Parameters:

- `Path` (path array) must point to an existing array.

- `Value` (any JSON value) is inserted at the beginning of the referenced array.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "const": "InsertFirst"
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string"
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string"
          },
          {
            "type": "number",
            "multipleOf": 1,
            "minimum": 0
          }
        ]
      }
    },
    "Value": {}
  },
  "required": ["Operation", "Path", "Value"],
  "additionalProperties": false
}
```

##### InsertLast

The `InsertLast` operation adds a specified value to the end of an array.

Delta objects take the following form:

```json
{
  "Operation": "InsertLast",
  "Path": [ ... ],
  "Value": ...
}
```

Parameters:

- `Path` (path array) must point to an existing array.

- `Value` (any JSON value) is inserted at the end of the referenced array.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "const": "InsertLast"
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string"
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string"
          },
          {
            "type": "number",
            "multipleOf": 1,
            "minimum": 0
          }
        ]
      }
    },
    "Value": {}
  },
  "required": ["Operation", "Path", "Value"],
  "additionalProperties": false
}
```

##### InsertBefore

The `InsertBefore` operation inserts a specified value before an element in an
array.

Delta objects take the following form:

```json
{
  "Operation": "InsertBefore",
  "Path": [ ... ],
  "Value": ...
}
```

Parameters:

- `Path` (path array) must point to an existing element of an array.

- `Value` (any JSON value) is inserted before the referenced array element.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "const": "InsertBefore"
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string"
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string"
          },
          {
            "type": "number",
            "multipleOf": 1,
            "minimum": 0
          }
        ]
      }
    },
    "Value": {}
  },
  "required": ["Operation", "Path", "Value"],
  "additionalProperties": false
}
```

##### InsertAfter

The `InsertAfter` operation inserts a specified value after an element in an
array.

Delta objects take the following form:

```json
{
  "Operation": "InsertAfter",
  "Path": [ ... ],
  "Value": ...
}
```

Parameters:

- `Path` (path array) must point to an existing element of an array.

- `Value` (any JSON value) is inserted after the referenced array element.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "const": "InsertAfter"
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string"
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string"
          },
          {
            "type": "number",
            "multipleOf": 1,
            "minimum": 0
          }
        ]
      }
    },
    "Value": {}
  },
  "required": ["Operation", "Path", "Value"],
  "additionalProperties": false
}
```

##### DeleteFirst

The `DeleteFirst` operation removes the first element of an array.

Delta objects take the following form:

```json
{
  "Operation": "DeleteFirst",
  "Path": [ ... ]
}
```

Parameters:

- `Path` (path array) must point to an existing array. The array must not be
  empty.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "const": "DeleteFirst"
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string"
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string"
          },
          {
            "type": "number",
            "multipleOf": 1,
            "minimum": 0
          }
        ]
      }
    }
  },
  "required": ["Operation", "Path"],
  "additionalProperties": false
}
```

##### DeleteLast

The `DeleteLast` operation removes the last element of an array.

Delta objects take the following form:

```json
{
  "Operation": "DeleteLast",
  "Path": [ ... ]
}
```

Parameters:

- `Path` (path array) must point to an existing array. The array must not be
  empty.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "const": "DeleteLast"
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string"
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string"
          },
          {
            "type": "number",
            "multipleOf": 1,
            "minimum": 0
          }
        ]
      }
    }
  },
  "required": ["Operation", "Path"],
  "additionalProperties": false
}
```

### Integrity Verification

When transmitting a `FeedAction` message, the server may optionally enable feed
data integrity verification by including a `FeedMd5` parameter. In that case,
the server must:

1. Generate a JSON encoding of the post-delta feed data object. Canonicalize the
   encoding by serializing object properties in lexicographical order and
   removing all non-meaningful whitespace.

2. Take an MD5 hash of that text, encode it as a Base64 string, and transmit it
   to the client as the `FeedMd5` parameter of the `FeedAction` message.

If the server transmits a feed data hash with an `FeedAction` message, the
client should validate its post-delta copy of the feed data against it. To do
so, the client must:

1. Generate a JSON encoding of the post-delta feed data object. Canonicalize the
   encoding by serializing object properties in lexicographical order and
   removing all non-meaningful whitespace.

2. Take an MD5 hash of that text, encode it as a Base64 string, and validate it
   against the hash string transmitted by the server.

3. If the hash check fails, then the server has violated the specification. The
   client should close the feed and may attempt to re-open it.

## Violations of the Specification

### Invalid Client-Originating Messages

If the server receives a client message that violates any part of the
specification, then the server must respond with a `ViolationResponse` message.
The server must respond in this manner if:

- A client-originating message is not valid JSON.

- A client-originating message is structurally invalid.

- A client-originating message violates the message sequencing requirements.

If the client transmits an invalid message, it is recommended that the server
also disconnect the client, as the state of the conversation has become
ambiguous.

If a client receives a `ViolationResponse` message from the server, it is
recommended that the client disconnect from the server, as the state of the
conversation has become ambiguous. The client may subsequently attempt to
reconnect to the server.

### Invalid Server-Originating Messages

If a client receives a message from the server that violates any part of the
specification, it is recommended that the client disconnect from the server, as
the state of the conversation has become ambiguous. The client should respond in
this manner if:

- A server-originating message is not valid JSON.

- A server-originating message is structurally invalid.

- A server-originating message violates the message sequencing requirements.

- A `FeedAction` message specifies a delta operation that is invalid in the
  context of the current feed data.

- A `FeedAction` message specifies a hash value that does not match the
  post-delta feed data.

The client may subsequently attempt to reconnect to the server.
