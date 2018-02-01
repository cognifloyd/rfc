---
domain: rfc.zeromq.org
shortname: 36/ZRE
name: ZeroMQ Realtime Exchange Protocol
status: stable
editor: Pieter Hintjens <ph@imatix.com>
---

The ZeroMQ Realtime Exchange Protocol (ZRE) governs how a group of peers on a network discover each other, organize into groups, and send each other events. ZRE runs over the [ZeroMQ Message Transfer Protocol (ZMTP)](http://rfc.zeromq.org/spec:23/ZMTP).

## Preamble

Copyright (c) 2009-2014 iMatix Corporation

This Specification is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation; either version 3 of the License, or (at your option) any later version. This Specification is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details. You should have received a copy of the GNU General Public License along with this program; if not, see <http://www.gnu.org/licenses>.

This Specification is a [free and open standard](http://www.digistan.org/open-standard:definition) and is governed by the Digital Standards Organization's [Consensus-Oriented Specification System](http://www.digistan.org/spec:1/COSS).

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119).

## Goals

The ZRE protocol provides a way for a set of nodes on a local network to discover each other, track when peers come and go, send messages to individual peers (unicast), and send messages to groups of peers (multicast). Its goals are:

* To work with no centralized services or mediation except those available by default on a network.
* To be robust on poor-quality networks, especially wireless networks.
* To minimize the time taken to detect a new peer's arrival on the network.
* To recover from transient failures in connectivity.
* To be neutral with respect to operating system, programming language, and hardware.
* To allow any number of nodes to run in one process, to allow large-scale simulation and testing.

## Changes Over Version 1

This is version 2 of ZRE. The changes over version 1 are:

* The 'strings' and 'dictionary' types redesigned for better expandability.
* The version number was increased to 2.
* The hard-coded dependency on TCP transport has been removed (merging ipaddress and port into a single endpoint field).
* Removed the logging extension, which is better handled by a separate framework.
* Protocol signature is 1 rather than 0.

## Implementation

### Node Identification and Life-cycle

A ZRE *node* represents a source or a target for messaging. Nodes usually map to applications. A ZRE node is identified by a 16-octet universally unique identifier (UUID). ZRE does not define how a node is created or destroyed but does assume that nodes have a certain durability.

### Node Discovery and Presence

ZRE uses UDP IPv4 *beacon* broadcasts to discover nodes. Each ZRE node SHALL listen to the ZRE discovery service which is UDP port 5670 (ZRE-DISC port assigned by IANA). Each ZRE node SHALL broadcast, at regular intervals, on UDP port 5670 a beacon that identifies itself to any listening nodes on the network.

The ZRE beacon consists of one 22-octet UDP message with this format:

```
+---+---+---+------+  +------+------+
| Z | R | E | %x01 |  | UUID | port |
+---+---+---+------+  +------+------+

       Header               Body
```

The header SHALL consist of the letters 'Z', 'R', and 'E', followed by the beacon version number, which SHALL be %x01.

The body SHALL consist of the sender's 16-octet UUID, followed by a two-byte mailbox port number in network order. If the port is non-zero this signals that the peer will accept ZeroMQ TCP connections on that port number. If the port is zero, this signals that the peer is disconnecting from the network.

A valid beacon SHALL use a recognized header and a body of the correct size. A node that receives an invalid beacon SHALL discard it silently. A node MAY log the sender IP address for the purposes of debugging. A node SHALL discard beacons that it receives from itself.

When a ZRE node receives a beacon from a node that it does not already know about, with a non-zero port number, it SHALL consider this to be a new peer.

When a ZRE node receives a beacon from a known node, with a zero port number, it SHALL disconnect from this peer.

### Interconnection Model

Each node SHALL create a ZeroMQ ROUTER socket and *bind* this to an ephemeral TCP port (in the range %C000x - %FFFFx). The node SHALL broadcast this mailbox port number in all beacons that it sends. Note that a node does not broadcast its IP address as this is provided by the UDP recvfrom function.

This ROUTER socket SHALL be used for all incoming ZeroMQ messages from other nodes. A node SHALL NOT send messages to peers via this socket.

When a node discovers a new peer, it SHALL create a ZeroMQ DEALER socket, set its identity (a byte containing 1, followed by 16-octet UUID) on that socket, and *connect* this to the peer's mailbox port. A node may immediately, after connection, start to send messages to a peer via this DEALER socket.

A node SHALL connect each DEALER sockets to at most one peer. A node may disconnect its DEALER socket if the peer has failed to respond within some time (see Heartbeating).

This DEALER socket SHALL be used for all outgoing ZeroMQ messages to a specific peer. A node SHALL not receive messages on this socket. The sender MAY set a high-water mark (HWM) of, for example, 100 messages per second (if the timeout period is 30 second, this means a HWM of 3,000 messages). The sender SHOULD set the send timeout on the socket to zero so that a full send buffer can be detected and treated as "peer not responding".

Note that the ROUTER socket provides the caller with the UUID of the sender for any message received on the socket, as an identity frame that precedes other frames in the message. A peer can thus use the identity on received messages to look up the appropriate DEALER socket for messages back to that peer. The identity SHALL be a binary 17-octet UUID value where the first byte is always 1.

When a node receives, on its ROUTER socket, a valid message from an unknown node, it SHALL treat this as a new peer in the identical fashion as if a UDP beacon was received from an unknown node.

NOTE: the ROUTER-to-DEALER pattern that ZRE uses is designed to ensure that messages are never lost due to synchronization issues. Sending to a ROUTER socket that does not (yet) have a connection to a peer causes the message to be dropped.

### Protocol Signature

Every ZRE message sent by TCP SHALL start with the ZRE protocol signature, %xAA %xA1. A node SHALL silently discard any message received that does not start with these two octets.

This mechanism is designed particularly for applications that bind to ephemeral ports which may have been previously used by other protocols, and to which there are still nodes attempting to connect. It is also a general fail-fast mechanism to detect ill-formed messages.

### Versioning

A version number octet %x02 shall follow the signature. A node SHALL discard messages that do not contain a valid version number. There is no mechanism for backwards interoperability.

### TCP Protocol Grammar

The following ABNF grammar defines the ZRE protocol:

```
zre             = greeting *traffic
greeting        = hello
traffic         = whisper / shout / join / leave / ping / ping-ok

;     Greet a peer so it can connect back to us
hello           = signature %d1 version sequence endpoint groups status name headers
signature       = %xAA %xA1             ; two octets
version         = number-1              ; Version number (2)
sequence        = number-2              ; Cyclic sequence number
endpoint        = string                ; Sender connect endpoint
groups          = strings               ; List of groups sender is in
status          = number-1              ; Sender groups status value
name            = string                ; Sender public name
headers         = dictionary            ; Sender header properties

;     Send a multi-part message to a peer
whisper         = signature %d2 version sequence content
version         = number-1              ; Version number (2)
sequence        = number-2              ; Cyclic sequence number
content         = msg                   ; Wrapped message content

;     Send a multi-part message to a group
shout           = signature %d3 version sequence group content
version         = number-1              ; Version number (2)
sequence        = number-2              ; Cyclic sequence number
group           = string                ; Group to send to
content         = msg                   ; Wrapped message content

;     Join a group
join            = signature %d4 version sequence group status
version         = number-1              ; Version number (2)
sequence        = number-2              ; Cyclic sequence number
group           = string                ; Name of group
status          = number-1              ; Sender groups status value

;     Leave a group
leave           = signature %d5 version sequence group status
version         = number-1              ; Version number (2)
sequence        = number-2              ; Cyclic sequence number
group           = string                ; Name of group
status          = number-1              ; Sender groups status value

;     Ping a peer that has gone silent
ping            = signature %d6 version sequence
version         = number-1              ; Version number (2)
sequence        = number-2              ; Cyclic sequence number

;     Reply to a peer's ping
ping_ok         = signature %d7 version sequence
version         = number-1              ; Version number (2)
sequence        = number-2              ; Cyclic sequence number

; A list of string values
strings         = strings-count *strings-value
strings-count   = number-4
strings-value   = longstr

; A list of name/value pairs
dictionary      = dict-count *( dict-name dict-value )
dict-count      = number-4
dict-value      = longstr
dict-name       = string

; A msg is zero or more distinct frames
msg             = *frame

; Strings are always length + text contents
string          = number-1 *VCHAR
longstr         = number-4 *VCHAR

; Numbers are unsigned integers in network byte order
number-1        = 1OCTET
number-2        = 2OCTET
number-4        = 4OCTET
```

### ZRE Commands

All commands start with a protocol signature (%xAA %xA1), then a command identifier, then the protocol version number (%d2), and then a sequence number (two octets in network byte order). The first message from a peer (HELLO) MUST have sequence number 1, and every message must have a strictly incrementing sequence number. When a peer detects gaps in the sequence, or an out-of-sequence message, it SHALL treat the peer as invalid, and disconnect the peer.

#### The HELLO Command

Each node SHALL start a dialog by sending HELLO as the first command on an connection to a peer.

When a node receives messages from a new peer it SHALL silently ignore any commands that precede a HELLO command. The sequence number for HELLO SHALL be 1.

The HELLO command contains these fields:

* <tt>endpoint</tt> - ZeroMQ endpoint that the sender will accept connections on.
* <tt>groups</tt> - the list of groups that the sender is present in, as a list of strings.
* <tt>status</tt> - the sender's group status sequence.
* <tt>headers</tt> - zero or more properties set by the sender.

If the recipient has not already connected to this peer it SHALL create a ZeroMQ DEALER socket and connect it to the endpoint specified as "endpoint" (which MAY take the form "tcp://ipaddress:port" when TCP is used).

The "group status sequence" is a one-octet number that is incremented each time the peer joins or leaves a group. Each peer MAY use this to assert the accuracy of its own group management information.

#### The WHISPER Command

When a node wishes to send a message to a single peer it SHALL use the WHISPER command. The WHISPER command contains a single field, which is the message content defined as one 0MQ frame. ZRE does not support multi-frame message contents.

#### The SHOUT Command

When a node wishes to send a message to a set of nodes participating in a group it SHALL use the SHOUT command. The SHOUT command contains two fields: the name of the group, and the the message content defined as one 0MQ frame.

Note that messages are sent via ZeroMQ over TCP, so the SHOUT command is unicast to each peer that should receive it. ZRE does not provide any UDP multicast functionality.

#### The JOIN Command

When a node joins a group it SHALL broadcast a JOIN command to all its peers. The JOIN command has two fields: the name of the group to join, and the group status sequence number *after* joining the group. Group names are case sensitive.

#### The LEAVE Command

When a node leaves a group it SHALL broadcast a LEAVE command to all its peers. The LEAVE command has two fields: the name of the group to leave, and the group status sequence number *after* leaving the group.

#### The PING Command

A node SHOULD send a PING command to any peer that it has not received a UDP beacon from within a certain time (typically five seconds). Note that UDP traffic may be dropped on a network that is heavily saturated. If a node receives no reply to a PING command, and no other traffic from a peer within a somewhat longer time (typically 30 seconds), it SHOULD treat that peer as dead.

Note that PING commands SHOULD be used only in targeted cases where a peer is otherwise silent. Otherwise, the cost of PING commands will rise exponentially with the number of peers connected, and can degrade network performance.

#### The PING-OK Command

When a node receives a PING command it SHALL reply with a PING-OK command.

## Node Discovery and Presence

ZRE uses UDP IPv4 *beacon* broadcasts to discover nodes and track their presence. This works as follows:

* A ZRE node SHALL listen to the ZRE discovery service which is UDP port 5670 (ZRE-DISC port assigned by IANA).
* Each ZRE node SHALL broadcast, at regular intervals, a UDP beacon that identifies itself to any listening nodes on the network.
* When a ZRE node receives a beacon from a node that it does not already know about, it SHALL consider this to be a new peer.
* When a ZRE node stops receiving beacons from a peer that it knows about, after a certain interval it SHALL consider this peer to be disconnected or dead.

The ZRE beacon consists of one 22-octet UDP message with this format:

```
+---+---+---+------+  +------+------+
| Z | R | E | %x01 |  | UUID | port |
+---+---+---+------+  +------+------+

       Header               Body
```

Notes for implementors:

* The header SHALL consist of the letters 'Z', 'R', and 'E', followed by the beacon version number, which SHALL be %x01.
* The body SHALL consist of the sender's 16-octet UUID, followed by a two-byte mailbox port number in network order.
* A valid beacon SHALL: use a recognized header; use a body of the right size; and provide a non-zero mailbox port number.
* A node that receives an invalid beacon SHALL discard it silently. A node MAY log the sender IP address for the purposes of debugging.
* A node SHALL discard beacons that it receives from itself.

## Security Aspects

ZRE security is handled by the underlying transport. For ZMTP v2 and v1, there is no security model and all information is exchanged in clear text. For ZMTP v3, any of the defined security mechanism may be used.
