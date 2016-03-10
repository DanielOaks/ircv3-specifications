---
title: IRCv3.3 Metadata
layout: spec
copyrights:
  -
    name: "Daniel Oaks"
    period: "2016"
    email: "daniel@danieloaks.net"
---
## Description

This document describes the `metadata` capability, and updates to the
[Metadata framework](./metadata-3.2.html) so clients know that specific keys
are set by servers. This allows clients who have negotiated the `metadata`
client capability to rely on these keys as being server-set.

This client capability MUST be named `metadata`.

## Reserved key prefixes

The following namespaces, or key prefixes, are only changeable by servers.
Servers implementing the `metadata` capability MUST NOT let any client set a
key with the following prefixes:

* `server.`
* `conn.`

If any client (a client having negotiated `metadata` or not) tries to
`METADATA SET` a key prefixed with either of these values, the server MUST
respond with the `ERR_KEYNOPERMISSION` numeric and the command MUST fail.

If any client tries to `METADATA CLEAR`, any keys with these prefixes MUST NOT
be unset, and the server MUST omit or respond with the `ERR_KEYNOPERMISSION`
numeric for them.
