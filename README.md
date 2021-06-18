[![Feedme](https://raw.githubusercontent.com/aarong/feedme-client/master/logo.svg?sanitize=true)](https://feedme.global)

# Feedme Specification

Version 0.1 - Working Draft

This document prescribes the JSON message format and messaging behavior to be
used by Feedme real-time API servers and their clients.

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
- [Violations of the Specification](#violations-of-the-specification)
  - [Invalid Client-originating Messages](#invalid-client-originating-messages)
  - [Invalid Server-originating Messages](#invalid-server-originating-messages)

<!-- /TOC -->

## Introduction

Implementations of this specification enable clients to:

1. Perform operations on a server and be notified about the outcome.
2. Access data on the server and be notified when and why it changes.
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
established a connection to the server, the transport must enable the client and
server to exchange string JSON messages of sufficient length to support the API
running through it. The transport must ensure that messages are received by the
other side in the order that they were sent.

Feedme APIs may support more than one transport. Supported transports should be
documented for API clients.

### Handshakes

Once a client has connected to the server via the transport, the client performs
a `Handshake` to initiate the Feedme conversation.

### Actions

Clients perform operations on the server by invoking `Actions`. Clients indicate
the action to be performed by specifying an `Action Name` and may supply
`Action Arguments` with further specifics.

When a client successfully performs an action, the server returns `Action Data`
describing the outcome of the action.

The following are determined by the API design and should be documented for API
clients:

- The set of available actions.

- The required structure of action arguments.

- The structure of resulting action data.

### Feeds

Clients access live data and action notification streams using `Feeds`. Clients
open feeds by specifying a `Feed Name` and may supply `Feed Arguments` with
further specifics.

When a client successfully opens a feed, the server returns the current
`Feed Data` and commits to notifying the client about future changes to that
data.

Once a client has `Opened` a feed, the feed may later be `Closed` by the client
or forcibly `Terminated` by the server.

The following are determined by the API design and should be documented for API
clients:

- The set of available feeds.

- The required structure of their arguments.

- The structure of their feed data.

- The conditions under which open feeds will be terminated.

### Action Revelations

When an action takes place on the server, the server may `Reveal` that action on
one or more feeds. Clients with those feeds open are sent action data describing
the action and a description of any resulting changes to the feed data.

Action revelations can result from:

1. Actions explicitly invoked by clients.

2. Actions deemed by the server to have taken place, but not explicitly
   initiated by a client.

The following are determined by the API design and should be documented for API
clients:

- The actions that are revealed on each feed.

- The rules according to which actions are deemed to have taken place.

### Deltas

When the server reveals an action on a feed, it may specify a sequence of
`Deltas` describing any resulting changes to the feed data.

Feed data only changes when an action is revealed on the feed.

## Messages

Messages exchanged across the transport must be valid JSON-encoded objects.

The server must respond to each client-originating message with exactly one
response message. Clients must not respond to server-originating messages.

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

The server must respond to a compliant `Handshake` message with a
`HandshakeResponse` message.

The server must respond to a non-compliant `Handshake` message with a
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

The server must respond to a compliant `Action` message with an `ActionResponse`
message.

The server must respond to a non-compliant `Action` message with a
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
  it should not reuse that `CallbackId` in other `Action` messages until it has
  received the associated `ActionResponse` from the server.

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

The server must respond to a compliant `FeedOpen` message with a
`FeedOpenResponse` message.

The server must respond to a non-compliant `FeedOpen` message with a
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

The server must respond to a compliant `FeedClose` message with a
`FeedCloseResponse` message.

The server must respond to a non-compliant `FeedClose` message with a
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

Five message types are transmitted in response to client-originating messages:

- `ViolationResponse` responds to a client message that violates the
  specification.

- `HandshakeResponse` responds to a client `Handshake` message.

- `ActionResponse` responds to a client `Action` message.

- `FeedOpenResponse` responds to a client `FeedOpen` message.

- `FeedCloseResponse` responds to a client `FeedClose` message.

Two message types are initiated by the server to notify clients about
developments on open feeds:

- `ActionRevelation` reveals an action on an open feed.

- `FeedTermination` indicates that a previously open feed has been forcibly
  closed.

#### Responses to Client Messages

##### ViolationResponse

A `ViolationResponse` is used to respond to any client message that violates the
part of the specification.

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

A `HandshakeResponse` message is used to respond to a valid client `Handshake`
message.

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
        }
      },
      "required": ["MessageType", "Success", "Version"],
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

An `ActionResponse` message is used to respond to a valid client `Action`
message.

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

A `FeedOpenResponse` message is used to respond to a valid client `FeedOpen`
message.

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

A `FeedCloseResponse` message is used to respond to a valid client `FeedClose`
message and indicates that the feed has been closed successfully. The server is
not permitted to reject valid `FeedClose` requests.

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
the conversation as `Not Ready`.

1. `Not Ready` - When the conversation is in this state...

- The client must transmit a `Handshake` message to the server and subsequently
  treat the conversation as `Handshaking`. The client must not transmit any
  other type of message to the server.

- If the server is complying with the specification, the client will not receive
  any messages.

2. `Handshaking` - When the conversation is in this state...

- The client must not transmit any messages to the server.

- If the server is complying with the specification, then the client will
  receive only a `HandshakeResponse` message. Once received, the client must
  subsequently treat the conversation as either `Ready` or `Not Ready`,
  depending on whether the server indicated success or failure.

3. `Ready` - When the conversation is in this state...

- The client may transmit `Action`, `FeedOpen`, and `FeedClose` messages to the
  server at its discretion, subject to other sequencing requirements. The client
  must not transmit `Handshake` messages to the server.

- If the server is complying with the specification, then the client may receive
  `ActionResponse`, `FeedOpenResponse`, `FeedCloseResponse`, `ActionRevelation`,
  and `FeedTermination` messages, subject to other sequencing requirements.

#### Server

The server must treat each client conversation as being in one of three states.
The server must initially treat client conversations as `Not Ready`.

1. `Not Ready` - When the conversation is in this state...

- The server must not transmit any messages to the client.

- If the client is complying with the specification, then the server will
  receive only a `Handshake` message. Once received, the server must
  subsequently treat the conversation as `Handshaking`.

2. `Handshaking` - When the conversation is in this state...

- The server must transmit a `HandshakeResponse` message to the client and
  subsequently treat the conversation as either `Ready` or `Not Ready`,
  depending on whether it indicated success or failure.

- If the client is complying with the specification, the server will not receive
  any messages.

3. `Ready` - When the conversation is in this state...

- The server may transmit `ActionResponse`, `FeedOpenResponse`,
  `FeedCloseResponse`, `ActionRevelation`, and `FeedTermination` messages to the
  client, subject to other sequencing requirements. The server must not transmit
  `HandshakeResponse` messages to the client.

- If the client is complying with the specification, then the server may receive
  `Action`, `FeedOpen`, and `FeedClose` messages, subject to other sequencing
  requirements.

### Actions

Once a conversation is `Ready`, the client may transmit `Action` messages to the
server at its discretion. Clients need not await the response to one `Action`
message before transmitting another.

The server must respond to each valid `Action` message by returning an
`ActionResponse` message referencing the same callback identifier.

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
  an `ActionRevelation` message referencing the feed. If received, then the
  client must continue to treat the feed as `Open`.

- If the server is complying with the specification, then the client may receive
  a `FeedTermination` message referencing the feed. If received, then the client
  must subsequently treat the feed as `Closed`.

- If the server is complying with the specification, then the client will not
  receive a `FeedOpenResponse` or `FeedCloseResponse` message referencing the
  feed.

4. `Closing` - When a feed is in this state...

- The client must not transmit any messages referencing the feed.

- If the server is complying with the specification, then the client may receive
  an `ActionRevelation` message referencing the feed. If the server is in
  compliance, then it transmitted the `ActionRevelation` message before it
  received the `FeedClose` message from the client. If received, then the client
  must continue to treat the feed as `Closing`.

- If the server is complying with the specification, then the client may receive
  a `FeedTermination` message referencing the feed. If the server is in
  copmliance, then it transmitted the `FeedTermination` message before it
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
  receive a `FeedOpenResponse`, `ActionRevelation`, or `FeedTermination` message
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

- The server must not transmit a `FeedCloseResponse`, `ActionRevelation`, or
  `FeedTermination` message referencing the feed.

- If the client is complying with the specification, then the server will not
  receive any messages referencing the feed.

3. `Open` - When a client feed is in this state...

- The server may, at its discretion, transmit an `ActionRevelation` message
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

- The server must not transmit a `FeedOpenResponse`, `ActionRevelation`, or
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
server's `FeedOpenResponse` message and may be updated by subsequent
`ActionRevelation` messages referencing the feed.

### Action Revelations

The server may use `ActionRevelation` messages to reveal two type of actions:

1. Actions explicitly invoked by clients using an `Action` message.

2. Actions deemed by the server to have taken place without an associated client
   `Action` message.

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

## Violations of the Specification

### Invalid Client-originating Messages

If the server receives a client message that violates any part of the
specification, then the server must respond with a `ViolationResponse` message.
The server must respond in this manner if:

- A client-originating message is not valid JSON

- A client-originating message is structurally invalid

- A client-originating message violates the message sequencing requirements

If the client transmits an invalid message, it is recommended that the server
also disconnect the client, as the state of the conversation has become
ambiguous.

If a client receives a `ViolationResponse` message from the server, it is
recommended that the client disconnect from the server, as the state of the
conversation has become ambiguous. The client may subsequently attempt to
reconnect to the server.

### Invalid Server-originating Messages

If a client receives a message from the server that violates any part of the
specification, it is recommended that the client disconnect from the server, as
the state of the conversation has become ambiguous. The client should respond in
this manner if:

- A server-originating message is not valid JSON

- A server-originating message is structurally invalid

- A server-originating message violates the message sequencing requirements

- An `ActionRevelation` message specifies a delta operation that is invalid in
  the context of the current feed data

- An `ActionRevelation` message specifies a hash value that does not match the
  post-delta feed data

The client may subsequently attempt to reconnect to the server.
