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
  - [Object Settings](#object-settings)
      - [Setting Object](#setting-object)
    - [Channel Settings](#channel-settings)
      - [Setting Specification](#setting-specification)
      - [Standard Settings](#standard-settings)
        - [secret](#secret)
        - [topic](#topic)
        - [no-external](#no-external)
        - [topic-lock](#topic-lock)
        - [moderated](#moderated)
        - [invite-only](#invite-only)
        - [ban](#ban)
        - [exempt](#exempt)
        - [invex](#invex)
        - [quiet](#quiet)
    - [User Settings](#user-settings)
      - [Setting Specification](#setting-specification-1)
      - [Standard Settings](#standard-settings-1)
        - [invisible](#invisible)
        - [cloak](#cloak)
    - [Channel Member Settings](#channel-member-settings)
      - [Setting Specification](#setting-specification-2)
      - [Standard Settings](#standard-settings-2)
        - [op](#op)
        - [voice](#voice)
        - [halfop](#halfop)

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
  - `lang`: Optional. String. An ISO language code specifying the user's
    preferred language for server messages.

Example:

    {
      "cmd": "register",
      "args": {
        "nick": "the-user",
        "user": "theuser",
        "realname": "The User",
        "caps": {},
        "lang": "en-gb"
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

      - `user-settings`: Required. An array of User Setting Specifications.

      - `channel-settings`: Required. An array of Channel Setting Specifications.

      - `channel-member-settings`: Required. An array of Channel Member Setting
        Specifications.

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
          "total-connections": 48,
          "channel-settings": [
            {
              "id": "urn:irc:setting:channel:no-external",
              "local-name": "no-external",
              "legacy-char": "n",
              "title": {"en": "No External Messages"},
              "description: {"en": "Only allow the channel to be messaged by users joined to it."},
              "type": "bool"
            },
            ...
          ],
          "user-settings": [
            {
              "id": "urn:irc:setting:user:invisible",
              "local-name": "invisible",
              "legacy-char": "i",
              "title": {"en": "Invisible"},
              "description": {"en": "Prevents you from showing up in public user listings."},
              "type": "bool"
            },
            ...
          ],
          "channel-member-settings": [
            {
              "id": "urn:irc:setting:channel-member:op",
              "local-name": "op",
              "legacy-char": "o",
              "sigil": "@",
              "sigil-priority": 10,
              "title": {"en": "Channel Operator"},
              "description": {"en": "Authorizes the channel member to configure the channel, and ban and kick other members."},
              "type": "bool"
            },
            ...
          ]
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

  - `settings`: Required. A JSON object mapping the URIs of channel settings
    to Channel Setting Objects expressing the current values of those settings.

Example:

    {
      "from": {
        "server": "a.b.c"
      },
      "cmd": "@channel-welcome",
      "args": {
        "channel": "#foo",
        "settings": {
          "urn:irc:setting:channel:topic": {
            "value": "This is the channel topic.",
            "change-time": "2016-01-01T12:00:00Z",
            "changed-by": {"nick": "the-user", "user": "theuser", "host": "user.example.com"}
          },
          "urn:irc:setting:channel:no-external": {
            "value": true
          },
          "urn:irc:setting:channel:topic-lock": {
            "value": true
          }
        }
      }
    }

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

A Channel Member Specification is any valid User Origin Specification, but may
have the following additional items:

  - `settings`: Required. A JSON object mapping the URIs of channel settings to
    Channel Member Setting Objects expressing the current values of those
    settings.

Example:

    {
      "from": {
        "server": "a.b.c"
      },
      "cmd": "@names",
      "args": {
        "channel": "#foo",
        "names": [
          {
            "nick": "the-user",
            "user": "theuser",
            "host": "user.example.com",
            "settings": {
              "urn:irc:setting:channel-member:op": {
                "value": true
              }
            }
          }
        ],
        "more": false
      }
    }


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

## Object Settings

IRC has the concept of channel and user modes. These are very limited and are problematic for several reasons:

  - They have a single-letter namespace, severely limiting the number of modes
    that can be defined.

  - Somewhat as a consequence of this, different IRC servers may assign subtly
    different or completely different functionality to the same mode character.

  - The use of single characters is not user friendly or intuitive. The meaning
    of modes is not immediately clear.

  - Because the meaning of a mode character can vary between networks,
    GUI clients cannot neccessarily determine the meaning of a mode character
    and thereby offer a graphical interface for mode control.

The following types of channel mode have been observed:

  - Boolean modes. These are either set or unset.

  - Singleton modes with argument. These modes are either unset or set. When
    they are set, they have a single argument. Setting the mode again changes
    the argument, rather than creating an additional setting.

  - Hostmask list modes. These modes express a list of zero or more items,
    namely of hostmasks.

  - Channel member modes. Though these are traditionally expressed in the form
    `channel +o user`, similarly to a hostmask list, they can be better
    considered as modes set on a channel member object, which is an object
    uniquely identified by the tuple (channel, user). These modes are always
    boolean.

Only boolean user modes have been observed.

The mode system is generalised into an *object settings* scheme. *Object* here
means anything which can have settings set on it. The *object settings* schem
is designed to facilitate compatibility, where possible, with legacy clients.

The channel topic is in IRC a special case, but in a generalised object
settings system there is no reason not to treat it as just another setting.
Traditionally servers have tracked who last set the topic and at what time.
Since it may not be desired to track this information for every setting,
servers are allowed to publish last change information for any setting, but are
not required to do so.

A server supports zero or more channel modes and advertises *Channel Setting
Specifications* at registration time in its `@welcome` message.

#### Setting Object

A User, Channel or Channel Member Setting Object is a JSON object communicating
the current state of a User, Channel or Channel Member Setting with regards to
a specific user, channel or (channel, user) tuple.

Each Setting Object has the following items:

  - `value`: Optional. If not specified or specified as `null`, it is assumed
    that the setting is not set.

    For boolean, string and integer values, this is that value.

    For `multi-hostmask` settings, this is an array of JSON objects of the
    following form:

      - `mask`: Required. String. The hostmask.

      - `added-by`: Optional. An Origin Specification. Servers MAY choose not
        to store this data.

      - `add-time`: Optional. String. An ISO 8601 UTC timestamp expressing when
        the mask was added. Servers MAY choose not to store this data.

  - `change-time`: Optional. String. An ISO 8601 UTC timestamp expressing
    when the setting was last changed. Servers MAY choose to store and provide
    this information only for some settings.

  - `changed-by`: Optional. An Origin Specification. Servers MAY choose to store
    and provide this information only for some settings.

### Channel Settings

#### Setting Specification

A Channel Setting Specification is a JSON object specifying information about a
type of channel setting which the server permits to be assigned to channels
created on the server. The server MAY also permit arbitrary settings to be
attached to channels, but the server SHOULD advertise specifications for
all channel settings to which it applies special processing.

Each Channel Setting Specification has the following items:

  - `id`: Required. String. A URI uniquely identifying the **semantic meaning**
    of the channel setting.

  - `local-name`: Required. String. A string which is unique at network scope.
    MUST be comprised only of the letters `[a-zA-Z0-9-]`. This is only guaranteed
    to be unique network-wide, though specifications for standard settings MAY
    specify recommended local names; such names SHOULD be used where possible.

  - `title`: Required. A langstring. This is a human-readable string labelling
    the setting. It should generally be in title case.

  - `description`: Optional. A langstring. This is a human-readable string
    providing additional information on the setting. The string may be
    arbitrarily long, and should be long enough to allow a user to determine
    how to apply the setting, at least in simple cases.

  - `legacy-char`: Optional. A string containing a single ASCII character.
    If this is specified, the setting is mapped to a legacy mode character.
    Settings are not obliged to have legacy characters. Specifications
    for standard settings MAY specify recommended legacy characters; such
    characters SHOULD be used where possible (i.e. where they would not
    conflict with an existing setting using that character).

  - `value-type`: Required. String. Specifies the value type. Supported values:

      - "bool"
      - "string"
      - "int"
      - "multi-hostmask"

  - `max-len`: Optional. Integer. If specified, specifies the maximum length
    for this setting. Only applicable for settings with a value type of
    "string" (in which case it specifies the maximum string length in bytes) or
    "multi-hostmask" (in which case it specifies the maximum number of hostmask
    entries).

#### Standard Settings

##### secret

  - ID: "urn:irc:setting:channel:secret"
  - Recommended Local Name: "secret"
  - Recommended Legacy Character: "s"
  - Suggested Title (`en`): "Secret Channel"
  - Suggested Description (`en`): "Hide the channel from public listings."
  - Type: Boolean.

##### topic

  - ID: "urn:irc:setting:channel:topic"
  - Recommended Local Name: "topic"
  - Recommended Legacy Character: N/A
  - Suggested Title (`en`): "Channel Topic"
  - Suggested Description (`en`): "Set a message to be displayed to anyone joining the channel."
  - Type: String.

##### no-external

  - ID: "urn:irc:setting:channel:no-external"
  - Recommended Local Name: "no-external"
  - Recommended Legacy Character: "n"
  - Suggested Title (`en`): "No External Messages"
  - Suggested Description (`en`): "Only allow the channel to be messaged by users joined to it."
  - Type: Boolean.

This setting, when set, blocks messages sent to channels by any user which is not a member of that channel.

##### topic-lock

  - ID: "urn:irc:setting:channel:topic-lock"
  - Recommended Local Name: "topic-lock"
  - Recommended Legacy Character: "t"
  - Suggested Title (`en`): "Topic Lock"
  - Suggested Description (`en`): "Only allow the channel topic to be changed by operators."
  - Type: Boolean.

This setting, when set, prevents changes to the channel topic by anyone who is not a channel operator.

##### moderated

  - ID: "urn:irc:setting:channel:moderated"
  - Recommended Local Name: "moderated"
  - Recommended Legacy Character: "m"
  - Suggested Title (`en`): "Moderated"
  - Suggested Description (`en`): "Only allow people with voice or channel operators to message the channel."
  - Type: Boolean.

This setting, when set, prevents anyone who is not a channel operator and who does not have voice to message the channel.

##### invite-only

  - ID: "urn:irc:setting:channel:invite-only"
  - Recommended Local Name: "invite-only"
  - Recommended Legacy Character: "i"
  - Suggested Title (`en`): "Invite Only"
  - Suggested Description (`en`): "Only allow people who have been invited to the channel to enter it."
  - Type: Boolean.

##### ban

  - ID: "urn:irc:setting:channel:ban"
  - Recommended Local Name: "ban"
  - Recommended Legacy Character: "b"
  - Suggested Title (`en`): "Ban Hostmask"
  - Suggested Description (`en`): "Ban anyone matching the given hostmask from joining or messaging the channel."
  - Type: Multiple Hostmask.

##### exempt

  - ID: "urn:irc:setting:channel:exempt"
  - Recommended Local Name: "exempt"
  - Recommended Legacy Character: "e"
  - Suggested Title (`en`): "Exempt Hostmask From Bans"
  - Suggested Description (`en`): "Exempt anyone matching the given hostmask from any ban hostmasks."
  - Type: Multiple Hostmask.

##### invex

  - ID: "urn:irc:setting:channel:invex"
  - Recommended Local Name: "invex"
  - Recommended Legacy Character: "I"
  - Suggested Title (`en`): "Exempt Hostmask From Invite-Only Requirement"
  - Suggested Description (`en`): "Exempt anyone matching the given hostmask from any invite-only requirement."
  - Type: Multiple Hostmask.

##### quiet

  - ID: "urn:irc:setting:channel:quiet"
  - Recommended Local Name: "quiet"
  - Recommended Legacy Character: "q"
  - Suggested Title (`en`): "Quiet Hostmask"
  - Suggested Description (`en`): "Prevent anyone matching the given hostmask from messaging the channel."

### User Settings

#### Setting Specification

A User Setting Specification has the same form as a Channel Setting
Specification, but specifies a setting that can be set on a user. The
`legacy-char` namespace (but not the `local-name` namespace) is distinct.

#### Standard Settings

##### invisible

  - ID: "urn:irc:setting:user:invisible"
  - Recommended Local Name: "invisible"
  - Recommended Legacy Character: "i"
  - Suggested Title (`en`): "Invisible"
  - Suggested Description (`en`): "Prevents you from showing up in public user listings."
  - Type: Boolean.

##### cloak

  - ID: "urn:irc:setting:user:cloak"
  - Recommended Local Name: "cloak"
  - Recommended Legacy Character: "x"
  - Suggested Title (`en`): "Hostmask Cloak"
  - Suggested Description (`en`): "Masks your hostname."
  - Type: Boolean.

### Channel Member Settings

#### Setting Specification

A Channel Member Setting Specification has the same form as a Channel Setting
Specification, but specifies a setting that can be set on a channel membership
object, which is the combination of a channel and a user. The `legacy-char` and
`local-name` namespaces are shared with those of the Channel Setting namespace.

Because a Channel Member object is destroyed when a user leaves the channel,
these settings do not survive leaving and rejoining a channel.

The Specification object for a Channel Member Setting may also contain the
following item:

  - `sigil`: Optional. String. A single ASCII character which is used to
    concisely represent the fact that a channel member has this setting set.

  - `sigil-priority`: Optional. Integer. A value which specifies which sigil
    should be used when a channel member has multiple sigil-conferring settings
    set. The setting with the highest value wins.

#### Standard Settings

##### op

  - ID: "urn:irc:setting:channel-member:op"
  - Recommended Local Name: "op"
  - Recommended Legacy Character: "o"
  - Recommended Legacy Sigil: "@"
  - Suggested Title (`en`): "Channel Operator"
  - Suggested Description (`en`): "Authorizes the channel member to configure
    the channel, and ban and kick other members."
  - Type: Boolean.

##### voice

  - ID: "urn:irc:setting:channel-member:voice"
  - Recommended Local Name: "voice"
  - Recommended Legacy Character: "v"
  - Recommended Legacy Sigil: "+"
  - Suggested Title (`en`): "Voiced Member"
  - Suggested Description (`en`): "Allows the channel member to speak when the
    channel is moderated."
  - Type: Boolean.

##### halfop

  - ID: "urn:irc:setting:channel-member:halfop"
  - Recommended Local Name: "halfop"
  - Recommended Legacy Character: "h"
  - Recommended Legacy Sigil: "%"
  - Suggested Title (`en`): "Channel Half-Operator"
  - Suggested Description (`en`): "A more limited type of channel operator. Cannot kick operators."
  - Type: Boolean.

