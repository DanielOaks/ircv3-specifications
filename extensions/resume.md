---
title: IRCv3 `resume` extension
layout: spec
work-in-progress: true
copyrights:
  -
    name: Daniel Oakley
    period: 2017
    email: daniel@danieloaks.net
---
## Introduction

Occasionally, clients disconnect from IRC. What happens these days is that the client connects with a different nick, joins all their old channels again, waits for the old connection to time out (or manually kills it using services), and then changes back to their original nickname.

This feature intends to vastly simplify this form of reconnection, and reduce the amount of nick-switching, `JOIN` and `QUIT` notices, and general disruption that other clients see when this happens. At the same time, one vital piece of information to convey on reconnection is that chat history may have been lost. This feature conveys this and also makes such a notice more specific than it currently is (by allowing reconnecting clients to specify exactly how much history they may have lost).


## Architecture

This feature is enabled using the `draft/resume` capability, and uses various new messages and numerics to convey state about reconnecting clients.

### Capabilities

The `draft/resume` and `sasl` capabilities are advertised by servers that support resuming connections. The `draft/resume` capability is requested by clients that support parsing the `RESUMED` message described in this specification, as well as those using the `RESUME` command to resume their previous connection. Once the `draft/resume` capability has been negotiated for a client, servers MAY extend the "ping timeout" length for that client (with the expectation that if the client disconnects, it will try to resume its' connection as described here).

### Messages

There are two new messages defined by this specification:

- `RESUME`: Indicates that the client wishes to resume a previous connection.
- `RESUMED`: Indicates that a client has disconnected+reconnected to the server.

Below, we go through both of these.

#### `RESUME` Command

This command is sent before starting SASL authentication, and indicates that the client wishes to resume a previous connection to the server which may still be active. The command is sent with the following format:

    RESUME oldnick [timestamp]

`oldnick` is the nickname the client had when they last disconnected. `timestamp`, if it is given, is a timestamp which indicates when they received the last message from the server on the old connection. This timestamp is used to indicate to other clients how long the disconnection was for, and uses the same format as the IRCv3 `server-time` extension (i.e. `YYYY-MM-DDThh:mm:ss.sssZ`, or in UTC using extended format as specified by ISO 8601:2004(E) 4.3.2).

This command is sent before SASL authentication has successfully completed. After sending this command, the client continues connection registration as usual.

Once SASL reg completes, if the client's able to successfully resume their connection the server sends them a `RESUMED` message (and their nickname is set to `<oldnick>`). If they cannot resume, the server sends the `ERR_CANNOT_RESUME` numeric (and optional additional error numerics).

If successful, the `RESUMED` message is sent to the connecting user before they receive `RPL_WELCOME (001)`. When this message is sent out, the connecting client's nickname is set to be the old client's nick. From this point forward, the combination of old nickname and new connecting client's username+hostname is used to identify the client from that point forward.

Connection resumption is done on a best-effort basis. If the resumption fails clients SHOULD continue with a regular reconnection.

#### `RESUMED` Message

This message is sent when a client successfully resumes a connection. For clients that have not received numerics indicating that their connection registration is completed, this numeric means that their `RESUME` attempt was successful. For registered clients, this indicates that the given client disconnected and reconnected to the server.

This message has the following format:

    :nick!olduser@oldhost RESUMED nick user host [timestamp]

`nick` is the nickname of the client whose connection has been resumed. The `user` and `host` params are the username and hostname that the new connecting client has. The client receiving this message MUST update the username and hostname they have stored for the given client, in the same way that the `CHGHOST` message is parsed. `timestamp` if given, shows when the reconnecting client received their last message from the server. The timestamp may be used as a rough indication of how much message history has been lost.

When sent to other clients, the `RESUMED` message MUST have the source of the old client's nickmask. This is illustrated in the examples below.

### Numerics

Here are the new numerics that this extension defines:

| No. | Label                   | Format                                                           |
| --- | ----------------------- | ---------------------------------------------------------------- |
| ??? | `ERR_CANNOT_RESUME`     | `:<server> ??? <oldnick> :Cannot resume connection[, <details>]` |
| ??? | `ERR_HISTORY_TRUNCATED` | `:<server> ??? <client/channel> :Chat history may be truncated`  |

`ERR_CANNOT_RESUME` is used to indicate a failure to resume a connection for a specific reason. The purpose of this numeric is to indicate that resumption was not successful and provide a human-readable explanation. If a more appropriate numeric for conveying the issue already exists (for example, `ERR_NOSUCHNICK` may indicate that the old nickname is no longer being present on the network), then it should be sent after this numeric is sent (i.e. both numerics will be sent).

`ERR_HISTORY_TRUNCATED` is used to indicate that chat history may be truncated for a given client/channel. It may be sent during a `chathistory` batch or otherwise during history playback. Bouncers that reconnect may also find this useful to send to their connected clients, to indicate that there may be missed history.


## Connection Registration

Upon connection, clients request the `draft/resume` capability. If they receive both this capability and the SASL capability, they can attempt to resume their old connection.

After successfully completing SASL authentication, clients wishing to resume an old connection send an appropriate `RESUME` command indicating the nickname they wish to take over and when the client received the last message from the server (to assist in replaying history and let other clients know how long they've been disconnected for). If resuming is successful, clients are sent the `RESUMED` message and then connection registration continues, eventually ending in the registration burst (`001`-`005` and all).

Once the registration burst has been sent to the client, the server sends the channel `JOIN` messages and the usual numerics that are sent on joining a channel and dispatches the relevant messages and numerics to other clients in those channels (i.e. either just the `RESUMED` message, or a `QUIT`+`JOIN`+etc if they don't support the `draft/resume` cap). The new client is also given the user modes that were active on the old client, as appropriate.

Other clients that have the reconnecting user `MONITOR`'d MUST be sent one `RPL_MONOFFLINE` numeric and one `RPL_MONONLINE` numeric indicating that the user reconnected. The timestamps on both these messages, if sent, SHOULD be the current time.

Upon resuming the connection, servers close the old connection.

### TLS

Clients may initially connect with TLS, only to later attempt to `RESUME` from a connection without TLS. In this situation, servers SHOULD deny the attempt with a message such as _"Cannot resume connection, client is no longer using TLS"_.

This is intended to ensure that TLS-only channel modes are not bypassed, and that clients prefer continuing to use TLS.


## Examples

A client with the nick `dan` reconnecting. The old connection used the username `~old` and the host `192.168.0.5`, and the new connection uses the username `~d` and the host `127.0.0.1`:

    C1 - C: PING 12345678
    C1 - S: :irc.example.com PONG 12345678
    C1 - C: PING 87654321
    ... C1 - Server does not send a response ...
    ... C1 disconnects and reconnects as C2 ...

    C2 - C: CAP LS
    C2 - C: NICK dan-backup-nick
    C2 - C: USER d * 0 :An example user!
    C2 - S: :irc.example.com CAP * LS :sasl draft/resume
    C2 - C: CAP REQ :sasl draft/resume
    C2 - S: :irc.example.com CAP dan-backup-nick ACK :sasl draft/resume
    C2 - C: RESUME dan 2017-04-13T15:12:51.620Z
    C2 - C: AUTHENTICATE PLAIN
    C2 - S: :irc.example.com AUTHENTICATE +
    C2 - C: AUTHENTICATE YnVubnkAYnVubnkAYnVubnk=
    C2 - S: :irc.example.com 900 dan-backup-nick * bunny :You are now logged in as bunny
    C2 - S: :irc.example.com 903 dan-backup-nick :SASL authentication successful
    C2 - S: :irc.example.com RESUMED dan ~d 127.0.0.1 2017-04-13T15:12:51.620Z
    ... C1's connection is closed and C1's attributes are applied to C2 ...
    C2 - C: CAP END
    C2 - S: :irc.example.com 001 dan :Welcome to the Internet Relay Network dan
    ... C2 receives regular registration burst ...
    C2 - S: :irc.example.com 376 dan :End of MOTD command
    C2 - S: :dan!~d@127.0.0.1 JOIN #test
    C2 - S: :irc.example.com 332 dan #test :Example topic
    C2 - S: :irc.example.com 333 dan #test george 1442060874
    C2 - S: :irc.example.com 353 dan @ #test :@dan @george +violet roger
    C2 - S: :irc.example.com MODE #test +o dan

Here is this reconnection seen by `george`, a client that does not have the `draft/resume` capability enabled:

    S: :dan!~old@192.168.0.5 QUIT :Client reconnected
    S: :dan!~d@127.0.0.1 JOIN #test
    S: :irc.example.com MODE #test +o dan

And here is this reconnection seen by `violet`, a client that has the `draft/resume` capability:

    S: :dan!~old@192.168.0.5 RESUMED dan ~d 127.0.0.1 2017-04-13T15:12:51.620Z


## Implementation Considerations

This section notes considerations software authors will need to take into account while implementing this specification. This section is non-normative.

Right now, when clients detect that their connection to the server may have dropped they tend to send a `QUIT` command, close their current connection and then create a new connection to the server. In cases where the server supports resuming connections and SASL is configured for this server, clients may find it more useful to attempt to establish a new link to the server and resume the connection before closing their old one. If this is done, clients should be able to better take advantage of connection resumption.

In addition, users sometimes manually reconnect when they see that there is lag on their connection. In these cases, clients may also wish to do the above rather than closing the connection and then reconnecting. However, it could be good to make this configurable.

When clients see a `RESUMED` message for another client which contains a timestamp, they can calculate how much time has passed since the timestamp and the current time and then display this next to the reconnect notice. Displaying this can assist users in knowing how much message history has been lost in private queries and channels.

A client that disconnects and reconnects to the server should explicitly display the reconnection, even if they're able to resume the connection. This is so that the user knows why they may be missing message history and similar issues.

Servers may wish to check the new hostmask of resuming clients, to ensure that it does not fall under their list of banned hosts or hostmasks.


## Security Considerations

This section notes security-specific considerations software authors will need to take into account while implementing this specification. This section is non-normative.

Without this specification, if you know a client's account credentials you can typically close their connection (using something like `/NS GHOST` or `/NS REGAIN`), and find out which non-hidden channels they're joined to with a regular `WHOIS`.

With this specification, if you have a client's account credentials you're able to see which hidden channels they've joined and essentially take-over their connection. This is a security concern that's not addressed by this specification (and is very difficult to address for explicitly unprepared reconnections such as the ones this specification focuses on). Clients should display incoming `RESUMED` messages in such a way that users are explicitly aware that the given client has reconnected, and servers may choose to limit connection resumption to cases where TLS is used on both the old and new connections.