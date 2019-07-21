# Feedme Specification

Version 0.1 - Working Draft

This document (the "specification") prescribes the JSON message format and
messaging behavior to be used by Feedme real-time API servers and their clients.

[https://feedme.global](https://feedme.global)

<!-- TOC depthFrom:2 -->

- [Introduction](#introduction)
- [Concepts and Terms](#concepts-and-terms)
  - [Transport](#transport)
  - [Handshakes](#handshakes)
  - [Actions](#actions)
  - [Feeds](#feeds)
  - [Action Revelations](#action-revelations)
  - [Deltas](#deltas)
- [Messages](#messages)
  - [Client-originating Message Types](#client-originating-message-types)
    - [Handshake](#handshake)
    - [Action](#action)
    - [FeedOpen](#feedopen)
    - [FeedClose](#feedclose)
  - [Server-originating Message Types](#server-originating-message-types)
    - [Responses to Client Messages](#responses-to-client-messages)
      - [ViolationResponse](#violationresponse)
      - [HandshakeResponse](#handshakeresponse)
      - [ActionResponse](#actionresponse)
      - [FeedOpenResponse](#feedopenresponse)
      - [FeedCloseResponse](#feedcloseresponse)
    - [Feed Notifications](#feed-notifications)
      - [ActionRevelation](#actionrevelation)
      - [FeedTermination](#feedtermination)
- [Message Sequencing](#message-sequencing)
  - [Fundamentals](#fundamentals)
  - [Handshakes](#handshakes-1)
    - [Client](#client)
    - [Server](#server)
  - [Actions](#actions-1)
  - [Feeds](#feeds-1)
    - [Client](#client-1)
    - [Server](#server-1)
- [Feed Data](#feed-data)
  - [Action Revelations](#action-revelations-1)
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

<!-- /TOC -->

## Introduction

Implementations of this specification enable clients to:

1. Perform operations on a server and be notified about the result.
2. Access live data on the server and be notified when and why it changes.
3. Subscribe to notifications about operations on the server.

The first objective is achieved using `Actions`; the second and third are
achieved using actions in conjunction with `Feeds`. A Feedme API is a
server-defined set of actions and feeds that clients can interact with.

The specification is agnostic about the platform of the server, the platforms of
the clients, and the communication channel between them. It requires only that
the clients and server be able to exchange and interpret JSON messages.

## Concepts and Terms

### Transport

The specification takes as given that each client is able to establish a
connection to the server through some arbitrary communication channel. This
communication channel is referred to in the abstract as the `Transport`.

Minimal functionality is required of the transport. Once connected, it must
enable the client and server to exchange string JSON messages of sufficient
length to support the API running through it. The transport must also ensure
that messages are received by the other side in the order that they were sent.

Feedme APIs may support more than one transport. Supported transports should be
documented for API clients.

### Handshakes

Once a client has connected to the server via the transport, the client performs
a `Handshake` to initiate the Feedme conversation.

### Actions

Clients perform operations on the server by invoking `Actions`. Clients indicate
the action to be performed by specifying an `Action Name` and may supply
`Action Arguments` with further specifics.

When a client performs an action successfully, the server returns `Action Data`
describing the outcome of the action.

The following are determined by the API design and should be documented for API
clients:

- The set of available actions.

- The required structure of action arguments.

- The structure of resulting action data.

### Feeds

Clients access live data and action notification streams by opening `Feeds`. The
client indicates the feed to be opened by specifying a `Feed Name` and may
supply `Feed Arguments` describing the specifics of the feed.

When a client opens a feed successfully, the server returns the current
`Feed Data` and commits to notifying the client about future changes to that
data. The data for a given feed is the same across all clients, but feed
arguments can be used to provision client-specific feeds.

Once a client has `Opened` a feed, the feed may later be `Closed` by the client
or forcibly `Terminated` by the server.

The following are determined by the API design and should be documented for API
clients:

- The set of available feeds.

- The required structure of their arguments.

- The structure of returned feed data.

- The conditions under which open feeds will be terminated.

### Action Revelations

Actions and feeds are linked via `Action Revelations`. When an action takes
place on the server, the server may `Reveal` that action on any number of feeds.
Clients with those feeds open are said to `Perceive` the action on those feeds.

When an action is revealed on a feed, clients with that feed open are sent a
copy of the action data and a description of any resulting changes to the feed
data.

Action revelations can result from:

1. Actions explicitly invoked by clients using an `Action` message.

2. Actions `Deemed` by the server to have taken place, but not initiated by a
   client `Action` message.

The following are determined by the API design and should be documented for API
clients:

- The feeds on which each action is revealed.

- The rules according to which actions may be deemed to have taken place.

### Deltas

When the server reveals an action on a feed, it may specify a sequence of
`Deltas` describing any resulting changes to the feed data.

A feed's data can only change when an action is revealed on the feed.

## Messages

Messages exchanged in either direction across the transport must be valid
JSON-encoded objects.

Each client-originating message must receive exactly one response message from
the server. The client must not respond to server-originating messages.

Messages take the following form:

```json
{
    "MessageType": "SOME_MESSAGE_TYPE",
    ...
}
```

Parameters:

- `MessageType` (string) indicates the type of message being transmitted and
  puts additional requirements on the structure of the message.

### Client-originating Message Types

Clients can send four types of messages:

- `Handshake` initiates the Feedme conversation.

- `Action` asks to perform an action on the server.

- `FeedOpen` asks to open a feed.

- `FeedClose` closes a feed.

#### Handshake

A `Handshake` message initiates the Feedme conversation.

The server must respond to compliant `Handshake` messages with a
`HandshakeResponse` message.

The server must respond to non-compliant `Handshake` messages with a
`ViolationError` message.

Messages take the following form:

```json
{
  "MessageType": "Handshake",
  "Versions": ["0.1"]
}
```

Parameters:

- `Versions` (array of strings) indicates the Feedme specification versions
  supported by the client. At this stage, the current and only version of the
  specification is 0.1.

Messages must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "MessageType": {
      "type": "string",
      "enum": ["Handshake"]
    },
    "Versions": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "string"
      }
    }
  },
  "required": ["MessageType", "Versions"],
  "additionalProperties": false
}
```

#### Action

An `Action` message asks to invoke an action on the server.

The server must respond to compliant `Action` messages with an `ActionResponse`
message.

The server must respond to non-compliant `Action` messages with a
`ViolationError` message.

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

- `CallbackId` (string) is an arbitrary, client-determined callback identifier.
  It is returned with the server's response.

Messages must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "MessageType": {
      "type": "string",
      "enum": ["Action"]
    },
    "ActionName": {
      "type": "string",
      "minLength": 1
    },
    "ActionArgs": {
      "type": "object"
    },
    "CallbackId": {
      "type": "string",
      "minLength": 1
    }
  },
  "required": ["MessageType", "ActionName", "ActionArgs", "CallbackId"],
  "additionalProperties": false
}
```

#### FeedOpen

A `FeedOpen` message asks to open feed.

The server must respond to compliant `FeedOpen` messages with a
`FeedOpenResponse` message.

The server must respond to non-compliant `FeedOpen` messages with a
`ViolationError` message.

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

Messages must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "MessageType": {
      "type": "string",
      "enum": ["FeedOpen"]
    },
    "FeedName": {
      "type": "string",
      "minLength": 1
    },
    "FeedArgs": {
      "type": "object",
      "additionalProperties": {
        "type": "string"
      }
    }
  },
  "required": ["MessageType", "FeedName", "FeedArgs"],
  "additionalProperties": false
}
```

#### FeedClose

A `FeedClose` message instructs the server close a feed.

The server must respond to compliant `FeedClose` messages with a
`FeedCloseResponse` message.

The server must respond to non-compliant `FeedClose` messages with a
`ViolationError` message.

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

Messages must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "MessageType": {
      "type": "string",
      "enum": ["FeedClose"]
    },
    "FeedName": {
      "type": "string",
      "minLength": 1
    },
    "FeedArgs": {
      "type": "object",
      "additionalProperties": {
        "type": "string"
      }
    }
  },
  "required": ["MessageType", "FeedName", "FeedArgs"],
  "additionalProperties": false
}
```

### Server-originating Message Types

The server can send seven types of messages.

Five message types are responses to client-originating messages:

- `ViolationResponse` responds to a client message that violates the
  specification.

- `HandshakeResponse` responds to a client `Handshake` message.

- `ActionResponse` responds to a client `Action` message.

- `FeedOpenResponse` responds to a client `FeedOpen` message.

- `FeedCloseResponse` responds to a client `FeedClose` message.

Two message types are initiated by the server to notify clients about
developments on open feeds:

- `ActionRevelation` reveals an action on an open feed.

- `FeedTermination` indicates that an open feed has been forcibly closed.

#### Responses to Client Messages

##### ViolationResponse

A `ViolationResponse` message responds to a client message that violates the
specification.

Messages take the following form:

```json
{
  "MessageType": "ViolationResponse",
  "Diagnostics": { ... }
}
```

Parameters:

- `Diagnostics` (object) may contain debugging information.

Messages must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "MessageType": {
      "type": "string",
      "enum": ["ViolationResponse"]
    },
    "Diagnostics": {
      "type": "object"
    }
  },
  "required": ["MessageType", "Diagnostics"],
  "additionalProperties": false
}
```

##### HandshakeResponse

A `HandshakeResponse` message responds to a client `Handshake` message.

If returning failure, messages take the following form:

```json
{
  "MessageType": "HandshakeResponse",
  "Success": false
}
```

Failure parameters:

- `Success` (boolean) is set to false, indicating no supported versions.

If returning success, messages take the following form:

```json
{
  "MessageType": "HandshakeResponse",
  "Success": true,
  "Version": "0.1",
  "ClientId": "SOME_CLIENT_ID"
}
```

Success parameters:

- `Success` (boolean) is set to true, indicating success.

- `Version` (string) is the Feedme specification version that will govern the
  conversation. It must have been selected from the list of supported versions
  indicated in the client `Handshake` message.

- `ClientId` (string) is a unique identifier assigned to the client by the
  server.

Messages must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "oneOf": [
    {
      "type": "object",
      "properties": {
        "MessageType": {
          "type": "string",
          "enum": ["HandshakeResponse"]
        },
        "Success": {
          "type": "boolean",
          "enum": [true]
        },
        "Version": {
          "type": "string",
          "minLength": 1
        },
        "ClientId": {
          "type": "string",
          "minLength": 1
        }
      },
      "required": ["MessageType", "Success", "Version", "ClientId"],
      "additionalProperties": false
    },
    {
      "type": "object",
      "properties": {
        "MessageType": {
          "type": "string",
          "enum": ["HandshakeResponse"]
        },
        "Success": {
          "type": "boolean",
          "enum": [false]
        }
      },
      "required": ["MessageType", "Success"],
      "additionalProperties": false
    }
  ]
}
```

##### ActionResponse

An `ActionResponse` message responds to a client `Action` message.

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

- `ActionData` (object) describes the action result.

Messages must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "oneOf": [
    {
      "type": "object",
      "properties": {
        "MessageType": {
          "type": "string",
          "enum": ["ActionResponse"]
        },
        "CallbackId": {
          "type": "string",
          "minLength": 1
        },
        "Success": {
          "type": "boolean",
          "enum": [true]
        },
        "ActionData": {
          "type": "object"
        }
      },
      "required": ["MessageType", "CallbackId", "Success", "ActionData"],
      "additionalProperties": false
    },
    {
      "type": "object",
      "properties": {
        "MessageType": {
          "type": "string",
          "enum": ["ActionResponse"]
        },
        "CallbackId": {
          "type": "string",
          "minLength": 1
        },
        "Success": {
          "type": "boolean",
          "enum": [false]
        },
        "ErrorCode": {
          "type": "string",
          "minLength": 1
        },
        "ErrorData": {
          "type": "object"
        }
      },
      "required": [
        "MessageType",
        "CallbackId",
        "Success",
        "ErrorCode",
        "ErrorData"
      ],
      "additionalProperties": false
    }
  ]
}
```

##### FeedOpenResponse

A `FeedOpenResponse` message responds to a client `FeedOpen` message.

If returning failure, messages take the following form:

```json
{
  "MessageType": "FeedOpenResponse",
  "Success": false,
  "FeedName": "SOME_FEED_NAME",
  "FeedArgs": { ... },
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
  "Success": true,
  "FeedName": "SOME_FEED_NAME",
  "FeedArgs": { ... },
  "FeedData": { ... }
}
```

Success parameters:

- `Success` (boolean) is set to true, indicating success.

- `FeedName` (string) is the feed name specified by the client.

- `FeedArgs` (object of strings) contains any feed arguments specified by the
  client.

- `FeedData` (object) is the current feed data.

Messages must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "oneOf": [
    {
      "type": "object",
      "properties": {
        "MessageType": {
          "type": "string",
          "enum": ["FeedOpenResponse"]
        },
        "Success": {
          "type": "boolean",
          "enum": [true]
        },
        "FeedName": {
          "type": "string",
          "minLength": 1
        },
        "FeedArgs": {
          "type": "object",
          "additionalProperties": {
            "type": "string"
          }
        },
        "FeedData": {
          "type": "object"
        }
      },
      "required": [
        "MessageType",
        "Success",
        "FeedName",
        "FeedArgs",
        "FeedData"
      ],
      "additionalProperties": false
    },
    {
      "type": "object",
      "properties": {
        "MessageType": {
          "type": "string",
          "enum": ["FeedOpenResponse"]
        },
        "Success": {
          "type": "boolean",
          "enum": [false]
        },
        "FeedName": {
          "type": "string",
          "minLength": 1
        },
        "FeedArgs": {
          "type": "object",
          "additionalProperties": {
            "type": "string"
          }
        },
        "ErrorCode": {
          "type": "string",
          "minLength": 1
        },
        "ErrorData": {
          "type": "object"
        }
      },
      "required": [
        "MessageType",
        "Success",
        "FeedName",
        "FeedArgs",
        "ErrorCode",
        "ErrorData"
      ],
      "additionalProperties": false
    }
  ]
}
```

##### FeedCloseResponse

A `FeedCloseResponse` message responds to a client `FeedClose` message and
indicates that the feed has been closed successfully. The server may not return
failure.

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

Messages must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "MessageType": {
      "type": "string",
      "enum": ["FeedCloseResponse"]
    },
    "FeedName": {
      "type": "string",
      "minLength": 1
    },
    "FeedArgs": {
      "type": "object",
      "additionalProperties": {
        "type": "string"
      }
    }
  },
  "required": ["MessageType", "FeedName", "FeedArgs"],
  "additionalProperties": false
}
```

#### Feed Notifications

##### ActionRevelation

An `ActionRevelation` message notifies a client that an action is being revealed
on one of its open feeds.

Messages take the following form:

```json
{
  "MessageType": "ActionRevelation",
  "ActionName": "SOME_ACTION_NAME",
  "ActionData": { ... },
  "FeedName": "SOME_FEED_NAME",
  "FeedArgs": { ... },
  "FeedDeltas": [ ... ],
  "FeedMd5": "SOME_BASE64_MD5"
}
```

Parameters:

- `ActionName` (string) is the name of the action being revealed.

- `ActionData` (object) is the action data for the action being revealed.

- `FeedName` (string) is the name of the feed that the action is being revealed
  on.

- `FeedArgs` (object of strings) contains any arguments for the feed that the
  action is being revealed on.

- `FeedDeltas` (array of delta objects) contains a sequence of operations to be
  applied to the feed data.

- `FeedMd5` (optional string) is a Base64-encoded MD5 hash of the feed data
  after the feed deltas have been applied. It may be omitted if the server does
  not wish to enable feed data verification. If included, it must be exactly 24
  characters long.

Messages must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "MessageType": {
      "type": "string",
      "enum": ["ActionRevelation"]
    },
    "ActionName": {
      "type": "string",
      "minLength": 1
    },
    "ActionData": {
      "type": "object"
    },
    "FeedName": {
      "type": "string",
      "minLength": 1
    },
    "FeedArgs": {
      "type": "object",
      "additionalProperties": {
        "type": "string"
      }
    },
    "FeedDeltas": {
      "type": "array",
      "items": {
        "type": "object"
      }
    },
    "FeedMd5": {
      "type": "string",
      "minLength": 24,
      "maxLength": 24
    }
  },
  "required": [
    "MessageType",
    "ActionName",
    "ActionData",
    "FeedName",
    "FeedArgs",
    "FeedDeltas"
  ],
  "additionalProperties": false
}
```

Each element in the `FeedDeltas` array must also satisfy one of the feed delta
schemas specified below.

##### FeedTermination

A `FeedTermination` message notifies a client that the server has forcibly
closed a previously open feed.

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

Messages must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "MessageType": {
      "type": "string",
      "enum": ["FeedTermination"]
    },
    "FeedName": {
      "type": "string",
      "minLength": 1
    },
    "FeedArgs": {
      "type": "object",
      "additionalProperties": {
        "type": "string"
      }
    },
    "ErrorCode": {
      "type": "string",
      "minLength": 1
    },
    "ErrorData": {
      "type": "object"
    }
  },
  "required": ["MessageType", "FeedName", "FeedArgs", "ErrorCode", "ErrorData"],
  "additionalProperties": false
}
```

## Message Sequencing

Messages must be sequenced according to the following rules in order to ensure
that the client and server maintain a common understanding of the state of the
conversation.

### Fundamentals

The client steers the conversation. It initiates interaction by transmitting
`Handshake`, `Action`, `FeedOpen`, and `FeedClose` messages.

The server must transmit exactly one message in response to each
client-originating message. The server need not respond to client messages in
the order that they are received.

- If a client message violates any part of the specification, including the
  sequencing requirements, then the server must respond with a
  `ViolationResponse` message.

- If a client message complies with the specification, then the server must
  respond with a `HandshakeResponse`, `ActionResponse`, `FeedOpenResponse`, or
  `FeedCloseResponse` as appropriate.

When the client has opened a feed, the server may also transmit
`ActionRevelation` and `FeedTermination` messages referencing that feed. The
client must not respond to these messages.

The client should discard server messages that violate the specification.

### Handshakes

#### Client

Once connected to the server via the transport, the client must treat the
conversation as being in one of three states.

The client must initially treat the conversation as `Not Ready`.

Client conversation states:

1. `Not Ready` - When the conversation is not ready...

- The client must transmit a `Handshake` message to the server and subsequently
  treat the conversation as `Handshaking`.

- The client must not transmit any other type of message to the server.

- The client should discard all server messages (server violation).

2. `Handshaking` - When the conversation is handshaking...

- The client must not transmit any messages to the server.

- The client must expect a `HandshakeResponse` message from the server. When
  that message is received, the client must subsequently treat the conversation
  as either `Ready` or `Not Ready` according to whether the server returned
  success or failure.

- The client should discard all other server messages (server violation).

3. `Ready` - When the conversation is ready...

- The client may transmit `Action`, `FeedOpen`, and `FeedClose` messages to the
  server, subject to other sequencing requirements.

- The client must not transmit `Handshake` messages to the server.

- The client must expect `ActionResponse`, `FeedOpenResponse`,
  `FeedCloseResponse`, `ActionRevelation`, and `FeedTermination` messages from
  the server, subject to other sequencing requirements.

- The client should discard `HandshakeResponse` messages from the server (server
  violation).

- The client should review its transmission history and debug if it receives a
  `ViolationResponse` message from the server.

#### Server

Once a client has connected to the server via the transport, the server must
treat the conversation as being in one of three states.

The server must initially treat each client conversation as `Not Ready`.

Server conversation states:

1. `Not Ready` - When a conversation is not ready...

- The server must not transmit any messages.

- The server must expect a `Handshake` message from the client. When that
  message is received, it must subsequently treat the conversation as
  `Handshaking`.

- The server must respond to any other client messages with a
  `ViolationResponse` message.

2. `Handshaking` - When a conversation is handshaking...

- The server must transmit a `HandshakeResponse` message.

  - If the server response indicates success, the server must subsequently treat
    the conversation as `Ready`.

  - If the server response indicates failure, the server must subsequently treat
    the conversation as `Not Ready`.

- The server must not transmit any other types of messages to the client.

- The server must respond to all client messages with a `ViolationResponse`
  message.

3. `Ready` - When a conversation is ready...

- The server must expect `Action`, `FeedOpen`, and `FeedClose` messages from the
  client, subject to other sequencing requirements. It must respond to such
  messages by transmitting `ActionResponse`, `FeedOpenResponse`, and
  `FeedCloseResponse` messages, respectively.

- The server must respond to client `Handshake` messages with a
  `ViolationResponse` message.

- The server may transmit `ActionRevelation` and `FeedTermination` messages,
  subject to other sequencing requirements.

### Actions

Once a conversation is `Ready`, the client may transmit `Action` messages at its
discretion. Clients need not await the response to one `Action` message before
transmitting another.

The server must respond to each such message by transmitting an `ActionResponse`
indicating success or failure. The server's response must reference the callback
identifier from the client message.

If the client receives an `ActionResponse` message referencing an unrecognized
callback identifier, then it should discard the message (server violation).

### Feeds

#### Client

Once a transport conversation is `Ready`, the client may transmit `FeedOpen` and
`FeedClose` messages, subject to sequencing requirements.

The client must treat each feed name-argument combination (feed) as having one
of five states. The state of a feed determines the set of client-originating
messages that may reference that feed.

The client must initially treat all feeds as `Closed`.

Client feed states:

1. `Closed` - When a feed is closed...

- The client must transmit only `FeedOpen` messages referencing that feed. If
  the client transmits a `FeedOpen` message, it must subsequently treat the
  specified feed as `Opening`.

- The client should discard all server messages referencing that feed (server
  violation).

2. `Opening` - When a feed is opening...

- The client must not transmit any messages referencing that feed.

- The client must expect a `FeedOpenResponse` message referencing that feed.
  When that message is received, the client must subsequently treat the feed as
  either `Open` or `Closed` according to whether the server returned success or
  failure.

- The client should discard `FeedCloseResponse`, `ActionRevelation`, and
  `FeedTermination` messages referencing that feed (server violation).

3. `Open` - When a feed is open...

- The client must transmit only `FeedClose` messages referencing that feed. If
  the client transmits a `FeedClose` message, it must subsequently treat the
  specified feed as `Closing`.

- The client must expect a sequence of `ActionRevelation` messages referencing
  that feed.

- The client must expect a `FeedTermination` message referencing that feed. If
  received, the client must subsequently treat the feed as `Closed`.

- The client should discard `FeedOpenResponse` and `FeedCloseResponse` messages
  referencing that feed (server violation).

4. `Closing` - When a feed is closing...

- The client must not transmit any messages referencing that feed.

- The client must expect a sequence of `ActionRevelation` messages referencing
  that feed.

- The client must expect a `FeedTermination` message referencing that feed. If
  received, the client must subsequently treat the feed as `Terminated`.

- The client must expect a `FeedCloseResponse` message referencing that feed.
  When that message is received, the client must subsequently treat the feed as
  `Closed`.

- The client should discard all `FeedOpenResponse` messages transmitted by the
  server (server violation).

5. `Terminated` - When a feed is terminated...

- The client transmitted a `FeedClose` message to the server, but received a
  `FeedTermination` message before the `FeedCloseResponse`.

- The client must not transmit any messages referencing that feed. The client
  must await a `FeedCloseResponse` message, and when received must subsequently
  treat the feed as `Closed`.

- The client should discard `FeedOpenResponse`, `ActionRevelation`, and
  `FeedTermination` messages referencing that feed (server violation).

#### Server

For each client, the server must treat each feed name-argument combination
(feed) as having one of five states. The state of a feed determines the set of
server-originating messages that may reference that feed.

When a client connects, the server must initially treat all feeds as `Closed`.

Server feed states:

1. `Closed` - When a feed is closed...

- The server must not transmit any messages referencing that feed.

- The server must expect a `FeedOpen` message referencing that feed. If the
  client transmits such a message, the server must subsequently treat the feed
  as `Opening`.

- The server should discard all `FeedClose` messages referencing that feed
  (client violation).

2. `Opening` - When a feed is opening...

- The server must transmit a `FeedOpenResponse` message to the client.

  - If the `FeedOpenResponse` message indicates success, the server must
    subsequently treat the feed as `Open`.

  - If the `FeedOpenResponse` message indicates failure, the server must
    subsequently treat the feed as `Closed`.

- The server must not transmit any other messages referencing that feed.

- The server should discard all client messages referencing that feed (client
  violation).

3. `Open` - When a feed is open...

- The server may transmit a sequence of `ActionRevelation` messages referencing
  that feed.

- The server may transmit a `FeedTermination` message referencing that feed. If
  such a message is transmitted, the server must subsequently treat the feed as
  `Terminated`.

- The server must expect a `FeedClose` message referencing that feed. If the
  client transmits such a message, the server must subsequently treat the feed
  as `Closing`.

- The server should discard all `FeedOpen` messages referencing that feed
  (client violation).

4. `Closing` - When a feed is closing...

- The server must transmit a `FeedCloseResponse` message and subsequently treat
  the feed as `Closed`.

- The server may transmit a sequence of `ActionRevelation` messages referencing
  that feed.

- The server must not transmit `FeedTermination` or `FeedCloseResponse` messages
  referencing that feed.

- The server should discard all `FeedOpen` and `FeedClose` messages referencing
  that feed (client violation).

5. `Terminated` - When a feed is terminated...

- The server must not transmit any messages referencing that feed.

- The server must expect either a `FeedOpen` or a `FeedClose` message
  referencing the feed.

  - If the server receives a `FeedOpen` message, it must respond with a
    `FeedOpenResponse` message indicating success or failure and, according to
    that result, subsequently treat the feed as either `Open` or `Closed`.

  - If the server receives a `FeedClose` message, it must respond with a
    `FeedCloseResponse` message and subsequently treat the feed as `Closed`. If
    the client is adhering to the specification, then it transmitted the
    `FeedClose` message before receiving the server's `FeedTermination` message.

- After a period of time has elapsed, the server may deem the feed `Closed` and
  treat it accordingly. The period of time should be appropriate for the latency
  of the transport.

## Feed Data

Data for a given feed name-argument combination (feed) is returned with the
server's successful `FeedOpenResponse` message and may be updated by subsequent
`ActionRevelation` messages referencing that feed.

### Action Revelations

The server may use `ActionRevelation` messages to reveal two type of actions:

1. `Client Actions` explicitly invoked using `Action` messages.

2. `Deemed Actions` that the server considers to have taken place without a
   triggering `Action` message.

When revealing an action on a feed, the server must send all clients with that
feed open exactly one `ActionRevelation` message. The same message must be sent
to all clients.

If a client opens a feed and then performs an action that is revealed on that
feed, it must receive an `ActionRevelation` message like any other client with
that feed open. The server may transmit the `ActionResponse` and
`ActionRevelation` messages to that client in any order.

### Feed Deltas

When the server transmits an `ActionRevelation` message, it must include a
`FeedDeltas` array describing any resulting changes to the feed data. Each
element of the array must be a valid feed delta object describing an operation
to be performed on the feed data. The server may transmit an empty `FeedDeltas`
array to indicate that the feed data has not changed.

When the client receives an `ActionRevelation` message, it must apply any feed
delta operations to its local copy of the feed data. Operations must be applied
in array order.

Each delta operation transmitted by the server must be valid given the current
state of client feed data, as determined by the initial `FeedOpenResponse`
message, earlier `ActionRevelation` messages, and earlier operations in the
`FeedDeltas` array.

If the sequence of delta operations received from the server is invalid then the
client should close the feed (server violation). The client may subsequently
attempt to re-open the feed.

Feed delta objects take the following form:

```json
{
  "Path": [ ... ],
  "Operation": "SOME_OPERATION",
  ...
}
```

Parameters:

- `Path` (array) describes the location in the feed data to operate upon.

- `Operation` (string) is the name of the delta operation to perform. Its value
  may put additional requirements on the feed delta object (including the path).

#### Paths

`Path` arrays are used to reference locations in the feed data object. Paths are
specified according to the following rules:

- `[]` refers to the root of the feed data object.

- `["Something"]` refers to the `Something` property of the root feed data
  object.

- `["Parent", "Something"]` refers to the `Something` property of the `Parent`
  object, which in turn is a child of the root feed data object.

- `["MyArray", 0]` refers to the first element of the `MyArray` array, which in
  turn is a child of the root feed data object.

- Child property names and array indexes may be chained to arbitrary length.

The first element of a `Path` array must be a string if present, as the feed
data root is always an object.

#### Operations

#### Multi-type Operations

##### Set

The `Set` operation writes a value to the specified path.

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
     reference a zero index.

- `Value` (any JSON value) is written to the referenced path. If the path
  references the root feed data object, then `Value` must be an object.

Delta objects must satisfy the following JSON Schema:

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["Set"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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

##### Delete

The `Delete` operation removes an object child property (by name) or an array
element (by index).

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
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["Delete"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["DeleteValue"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["Prepend"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["Append"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["Increment"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["Decrement"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["Toggle"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["InsertFirst"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["InsertLast"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["InsertBefore"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["InsertAfter"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["DeleteFirst"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "Operation": {
      "type": "string",
      "enum": ["DeleteLast"]
    },
    "Path": {
      "type": "array",
      "items": [
        {
          "type": "string",
          "minLength": 1
        }
      ],
      "additionalItems": {
        "oneOf": [
          {
            "type": "string",
            "minLength": 1
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

When transmitting an `ActionRevelation` message, the server may optionally
enable feed data integrity verification by including a `FeedMd5` parameter. In
that case, the server must:

1. Generate a JSON encoding of the post-delta feed data object. Canonicalize the
   encoding by serializing object properties in lexicographical order and
   removing all non-meaningful whitespace.

2. Take an MD5 hash of that text, encode it as a Base64 string, and transmit it
   to the client as the `FeedMd5` parameter of the `ActionRevelation` message.

If the server transmits a feed data hash with an `ActionRevelation` message, the
client should validate its post-delta copy of the feed data against it. To do
so, the client must:

1. Generate a JSON encoding of the post-delta feed data object. Canonicalize the
   encoding by serializing object properties in lexicographical order and
   removing all non-meaningful whitespace.

2. Take an MD5 hash of that text, encode it as a Base64 string, and validate it
   against the hash string transmitted by the server.

3. If the hash check fails, then the server has violated the specification. The
   client should close the feed and may attempt to re-open it.
