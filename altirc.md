<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents** 

- [AltIRC](#altirc)
  - [Transport Protocol Binding](#transport-protocol-binding)
    - [TCP Binding](#tcp-binding)
    - [TLS Binding](#tls-binding)
    - [WebSocket Binding](#websocket-binding)
  - [Message Format](#message-format)
    - [Message Origin Specification](#message-origin-specification)
    - [Command String](#command-string)
    - [Logical Argument Specification](#logical-argument-specification)
    - [Sequential Argument Specification](#sequential-argument-specification)
    - [Message Body](#message-body)
      - [Message Body: Standard Extensions](#message-body-standard-extensions)
    - [Langstrings](#langstrings)
  - [Messages](#messages)
    - [User Registration](#user-registration)
      - [register](#register)
      - [@welcome](#@welcome)
    - [Channel Subscription](#channel-subscription)
      - [join](#join)
      - [part](#part)
      - [@channel-welcome](#@channel-welcome)
      - [@names](#@names)
    - [User-to-User Messaging](#user-to-user-messaging)
      - [privmsg](#privmsg)
      - [notice](#notice)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# AltIRC

The AltIRC specification specifies a new transport protocol for Internet Relay
Chat (IRC) communications. The new transport protocol is intended to allow IRC
to be advanced to facilitate new capabilities while still affording
compatibility, where possible, with legacy RFC1459 clients.

The transport protocol is an extensible, JSON-based wire protocol. The protocol
is designed to be transmissible over TCP, TLS or WebSockets. The protocol is a
bidirectional protocol between client and server. Each side transmits a number
of messages. Each message is a JSON object. UTF-8 encoding MUST be used.

## Transport Protocol Binding

### TCP Binding

Messages are exchanged over a TCP connection in the following format:

    4 bytes  uint32     Length
    ...                 Data

The length value is little endian-encoded. The length value specifies the
length of the data field exactly, which contains the JSON-encoded object.

### TLS Binding

The TLS binding operates identically to the TCP binding in terms of the
encapsulated wire protocol. Implementations SHOULD support TLS ALPN using a
string of "altirc". However, servers MUST NOT require client support for TLS
ALPN.

### WebSocket Binding

Each message shall be transmitted in a WebSocket frame. A text-type frame
SHOULD be used, but implementations SHOULD accept peers which use binary-type
frames. The content of the frame is the JSON-encoded object.

Servers MAY, via protocol handshake detection, support both WebSockets (and
therefore HTTP requests) and non-WebSocket connections over TLS on the same
port. Servers MAY use TLS ALPN to help identify connections but MUST NOT
require it.

## Message Format

Each protocol message is a JSON object containing keys "cmd", "args" and
optionally "from" and/or "id".

The "cmd" item MUST have as its value a string which is a valid Command String.

The "args" item MUST have as its value either an array which is a valid
Sequential Argument Specification or an object which is a valid Logical
Argument Specification.

The "from" item, if present, MUST have as its value an object which is a valid
Message Origin Specification or be specified as `null`.

The "id" item, if present, MUST have a string or an integer value, or be
specified as `null`. Integer values must not exceed the range `[0,2**32)` and
string values must not exceed a length of 40 bytes when UTF-8 encoded.

  {
    "from": {
    },
    "cmd": "privmsg",
    "args": ...
  }

The "id" item is used for request/response correlation by clients. Where servers
receive commands with a valid "id" item, servers MUST incorporate an identical
"id" item in any messages they issue in response. Where a server issues a response
to several users, only one of which is the requester, servers SHOULD remove the
"id" item from the version sent to other users.

### Message Origin Specification

A Message Origin Specification is a JSON object. A valid Message Origin Specification
must be a valid Server Origin Specification or a valid User Origin Specification.

A valid Server Origin Specification has a key "server" with, as its value, a string
containing the canonical name of the server, which MUST be a valid hostname. A Server
Origin Specification MUST NOT have keys named "nick", "user", "host" or "account".
Transmitting implementations SHOULD NOT place other keys in a Server Origin Specification
but receiving implementations MUST ignore any such keys.

A valid User Origin Specification has the following items:

    "nick": Required. String. Specifies the current nickname of the user.
    "user": Required. String. Specifies the current username of the user.
    "host": Required. String. Specifies the current hostname of the user.
    "account": Optional. String. Specifies the account name for the user, if any.

A User Origin Specification MUST NOT have a "server" key, but receiving
implementations MUST ignore the presence of such a key. A Message Origin
Specification with both "server" and "nick" keys MUST be treated as a User
Origin Specification.

### Command String

A Command String specifies a command name. Command strings are, by convention,
expressed in lowercase. IRC protocol commands which were previously expressed
in uppercase are now expressed in lowercase and these are valid command
strings.

The use of numeric-based notifications is eschewed in favour of meaningful
names. Where previously a numeric was assigned, a command string will be
assigned of the form "@" followed by a meaningful name.

Newly assigned command and notification names SHOULD use a lowercase and
hyphens naming convention (`[0-9a-z-]`).

### Logical Argument Specification

A Logical Argument Specification is a JSON object. The contents are
command-specific. However, some key names have standardised semantic meaning
and MUST NOT be used where their values do not bear this meaning:

    "target":   A string containing a target specification, a user or channel name.

    "message":  A message conveyed to or from a user. Must be a Message Body.

    "channel":  A string specifying a channel name that the command is in regard of.
                Not used when the channel is also the `target` of the command.

### Sequential Argument Specification

The preferred format for a message's arguments is a Logical Argument Specification.
This format SHOULD be used wherever possible. 

In some cases, a client may need to marshal a command which it does not
understand, such as a raw command typed by the user. In this case, the
sequential argument specification may be used.

A Sequential Argument Specification is an array of zero or more items, each of
which MUST be a string. Each item in the array is an argument specified by the
user. Clients SHOULD support the use of double quotes by users to delimit
argument bounds.

Servers which recognise a command which was issued with Sequential Argument
Specification SHOULD convert the message to use Logical Argument Specification
as early as possible. The mapping from Sequential to Logical modes of
expression shall be defined on a per-command basis. If a command does not
explicitly define a mapping, no mapping is available.

### Message Body

A Message Body is a JSON object relayed from one user to one or several other
users. It is an opaque value that, by default, is not inspected by any server.

The key `text` in a Message Body is assigned a standard meaning: the
communication of a short (generally one-line) textual message. Clients SHOULD
convey received Message Bodies which contain this key to users, even if the
Message Body also contains other keys which are not understood.

A Message Body without the key `text` is a no-op for a client which does not
understand it. Clients MAY allow users to view the raw data of these messages,
but the general expectation is that they will not bother the user about such
messages.

As a special case, a Message Body MAY be a string instead of a JSON object.
This is equivalent to a JSON object with a single key `text` and the string as
its value. Servers MUST convert this form, wherever seen, to the equivalent
object form.

#### Message Body: Standard Extensions

Support for the following standard keys is RECOMMENDED:

  - `intent`: Optional. String. If specified, expresses a message intent.
    The only supported value is "action". This is used to support the `/me`
    command.

  - `notice`: Optional. Boolean. If specified, indicates the message was
    generated automatically and that automatic responses to it should not be
    made, to avoid message loops.

### Langstrings

A langstring is a standard element used in some places in the protocol.

It is a JSON object. Each item has a key of an ISO language code in lowercase,
with the value being a human-readable string in that language. At least an item
with key "en" SHOULD always be included.

## Messages

### User Registration

User registration is the process of registering with a server upon connection.
It must be completed before any other action can be performed.

#### register

The `register` command is sent by a client immediately upon connection, before
any other command.

The arguments shall be as follows:

  - `nick`: Required. String. The user's desired nickname.
  - `user`: Required. String. The user's desired username.
  - `realname`: Required. String. The user's desired realname.
  - `caps: Required. Object. Reserved; set to an empty object.

Example:

    {
      "cmd": "register",
      "args": {
        "nick": "the-user",
        "user": "theuser",
        "realname": "The User",
        "caps": {}
      }
    }

#### @welcome

The `@welcome` notification is sent by a server upon successful registration.

The arguments shall be as follows:

  - `user`: Required. A User Origin Specification. This informs the client
    about its assigned nickname, username, account name and hostname,
    which may differ from what it requested.

  - `server`: Required. An object containing a Server Welcome.

      - `name`: Required. String. The server's canonical hostname. MUST be a valid
        hostname.

      - `version`: Required. String The server software identification and
        version string. SHOULD state the server software name, followed by the
        version.

      - `start-time`: Optional. String. ISO 8601 UTC timestamp. The time at
        which the server started operating.

      - `num-clients`: Optional. A numstat. The number of users connected to
        this server.

      - `num-servers`: Optional. A numstat. The number of servers connected to
        this server.

      - `total-connections`: Optional. A non-negative integer. The number of
        connections which have been made to this server in total.

      - LIMITS

      - SUPPORT

      - PREFIXES

      - USER MODES

      - CHANNEL MODES

  - `network`: Required. An object containing the following:

      - `name`: Required. String. The network short name. This MUST
        use only the characters `a-zA-Z0-9-` and is intended to uniquely
        identify the network.

      - `title`: Optional. A langstring. A human-readable network name.

      - `num-users`: Optional. A numstat. The number of users connected to the network.

      - `num-servers`: Optional. A numstat. The number of servers comprising the network.

      - `num-operators`: Optional. A numstat. The number of operators currently connected
        to the network. `num-users` counts operators.

      - `num-channels`: Optional. A numstat. The number of channels currently in existence
        on the network.

      - `case-mapping`: Required. String. Must be "rfc1459".

      - `nick-len`: Required. Positive integer. Specifies the maximum number of
        bytes a UTF-8 nickname may comprise.

      - `max-nick-len`: ...

      - `channel-name-len`: Required. Positive integer. Specifies the maximum
        number of bytes a UTF-8 channel name may comprise.

      - `topic-len`: Required. Positive integer. Specifies the maximum number
        of bytes a UTF-8 topic may comprise.

      - `user-len`: Required. Positive integer. Specifies the maximum number
        of bytes a UTF-8 username may comprise.

      - `realname-len`: Required. Positive integer. Specifies the maximum
        number of bytes a UTF-8 realname may comprise.

  - `motd`: Optional. String. Contains the Message of the Day, which is a
    (potentially multiline) server-specific greeting message.

A numstat is a JSON object as follows:

  - `cur`: Required. Integer. The current value of the statistic.

  - `highest`: Optional. Integer. The high water mark of the statistic,
    specifying the highest value that `cur` has ever had.

Example:

    {
      "from": {
        "server": "x.y.z"
      },
      "cmd": "@welcome",
      "args": {
        "user": {
          "nick": "the-user",
          "user": "user",
          "host": "user.example.com"
        },
        "server": {
          "name": "x.y.z",
          "version": "fooserve-1.2.3",
          "start-time": "2016-01-01T12:00:00Z",
          "num-clients": {"cur":2,"highest":2},
          "num-servers": {"cur":0,"highest":0},
          "total-connections": 48
        },
        "network": {
          "name": "FOONET",
          "title": {"en": "Foo Network"},
          "num-users": {"cur":10, "highest":42},
          "num-servers": {"cur":2, "highest":3},
          "num-operators": {"cur":3, "highest":9},
          "num-channels": {"cur":7, "highest":11},
          "case-mapping": "rfc1459",
          "nick-len": 30,
          "user-len": 30,
          "realname-len": 100,
          "channel-len": 50,
          "topic-len": 390
        },
        "motd": "Welcome.\n"
      }
    }

### Channel Subscription

#### join

The `join` command is used by a client to join a channel. It is also issued by a server
as a notification to indicate that a channel join has occurred.

The arguments shall be as follows:

    - `target`: Required. Standard `target` item specifying a channel name.

    - `password`: Optional. String. Used if a channel requires a password to join. Ignored
      otherwise.

A Sequential Argument Specification is mapped to a Logical Argument
Specification by taking the first argument to be the `target` and the second
argument, if present, to be the `password`. Additional arguments are ignored.

Example request from a client:

    {
      "cmd": "join",
      "args": {
        "target": "#chan1"
      }
    }

Example join notification:

    {
      "from": {
        "nick": "the-user",
        "user": "theuser",
        "host": "user.example.com",
        "account": "the-user"
      },
      "cmd": "join",
      "args": {
        "target": "#chan1"
      }
    }

#### part

The `part` command is used by a client to part a channel. It is also issued by a server
as a notification to indicate that a channel part has occurred.

The arguments shall be as follows:

    - `target`: Required. Standard `target` item specifying a channel name.

    - `reason`: Optional. String. May be used to specify a reason for parting
      the channel.

A Sequential Argument Specification is mapped to a Logical Argument
Specification by taking the first argument to be the `target` and the second
argument, if present, to be the `reason`. Additional arguments are ignored.

#### @channel-welcome

The `@channel-welcome` notification is issued when a user has successfully
joined a channel. It provides initial state information about the channel.

The following arguments are used:

  - `channel`: Required. String. The channel which has been joined. This might
    not be exactly the same as the channel specified in an earlier `join`
    command. The case might be different, or the client may have been redirected
    to another channel. The case specified in this item is the official case
    for the channel.

  - `topic`: Optional. Object. If a topic is set, this MUST be present and set to
    a valid Topic Specification. Absence, or specification as `null` indicates that
    a topic is not set.

  - `mode`: Required. Object. [INDICATES CHANNEL MODES TODO]

A Topic Specification is a JSON object with the following items:

  - `message`: Required. String. The topic.

  - `author`: Optional. If present, MUST be a valid Message Origin Specification.
    Indicates who last changed the topic.

  - `time`: Optional. String. If present, MUST be an ISO 8601 UTC timestamp.

#### @names

The `@names` notification is issued to provide a complete list of channel members.
It is sent on request, or when a user joins a channel.

The following arguments are used:

  - `names`: Required. Array. Each item in the array shall be a Channel Member
    Specification.

  - `more`: Optional. Boolean, default false. If present and set to true, this
    indicates that the `@names` notification will be complemented by a further
    `@names` notification. This may be used if the list of channel members is
    very large and a server desires to send it in multiple chunks. In this
    case, all but the final `@names` notifications for a channel have `more`
    set to true.

A Channel Member Specification is any valid User Origin Specification.

### User-to-User Messaging

#### privmsg

The `privmsg` command is used for user-to-user, user-to-channel and
channel-to-user messaging.

The Logical Argument Specification of a `privmsg` command MUST contain the
standard `target` item, with any valid target specification (i.e. a valid user
or channel name). The standard `message` item MUST be present.

The `privmsg` command expresses the transmission of a message from one user to
another use, or from one user to one or more users via a channel. The
user-to-user(s) payload is the Message Body specified as the value of the
`message` item, and, by default, servers MUST NOT impose constraints other than
length limitations on the nature of the Message Body, unless requested by the
relevant channel or user,

A Sequential Argument Specification is mapped to a Logical Argument
Specification by taking the first two arguments and treating the first as the
target and the second as the Message Body. Any additional arguments are
ignored.

Example of a privmsg command transmitted by a user:

    {
      "cmd": "privmsg",
      "args": {
        "target": "#chan1",
        "message": {
          "text": "Ahoy."
        }
      }
    }

Example of a privmsg command transmitted by a user via Sequential Argument Specification:

    {
      "cmd": "privmsg",
      "args": ["#chan1", "Ahoy."]
    }

Note that the message body is specified as a string, which is converted automatically.

Example of a privmsg received by a user via a channel:

    {
      "from": {
        "nick": "the-user",
        "user": "theuser",
        "host": "user.example.com",
        "account": "the-user"
      },
      "cmd": "privmsg",
      "args": {
        "target": "#chan1",
        "message": {
          "text": "Ahoy."
        }
      }
    }

Example of a `/me` message received by a user via a channel:

    {
      "from": {
        "nick": "the-user",
        "user": "theuser",
        "host": "user.example.com",
        "account": "the-user"
      },
      "cmd": "privmsg",
      "args": {
        "target": "#chan1",
        "message": {
          "text": "thinks hard",
          "intent": "action"
        }
      }
    }

CTCP is deprecated.

#### notice

The `notice` command is retired. Instead of issuing a `notice`, issue a
`privmsg`, placing a `"notice": true` item inside the message body. Bots MUST
use the `notice` item to avoid causing recursive messaging loops.

    {
      "cmd": "privmsg",
      "args": {
        "target": "#chan1",
        "message": {
          "text": "I don't support that command.",
          "notice": true
        }
      }
    }


