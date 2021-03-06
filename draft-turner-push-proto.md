---
title: Simply Push Protocol
abbrev: SPP
docname: draft-turner-push-proto-latest
date: 2014
category: std

ipr: pre5378Trust200902
area: General
workgroup:
keyword: Internet-Draft

stand_alone: yes
pi:
  rfcedstyle: yes
  toc: yes
  tocindent: yes
  sortrefs: yes
  symrefs: yes
  strict: yes
  comments: yes
  inline: yes
  text-list-symbols: -o*+
author:
 -
       ins: D. Turner
       name: Douglas Turner II
       organization: Mozilla
       email: dougt@mozilla.com

normative:
  RFC2119:


--- abstract

This page describes the protocol used for communication by the PushServer and the UserAgent.

A reference server is available on [https://github.com/dougt/go-push-server github].
--- middle


# Introduction

The SimplePush protocol is closely based on [http://static.googleusercontent.com/external_content/untrusted_dlcp/research.google.com/en/us/pubs/archive/37474.pdf Thialfi: A Client Notification Service for Internet-Scale Applications] and makes the same delivery guarantees, soft server state and client driven recovery. It is a signaling and not a data carrying system.

The goal: To notify clients of changes to application server state in a reliable manner by ensuring that the client will always eventually learn of the latest version of an object for which it has expressed interest.

# Definitions

{:br: vspace="0"}

PushServer
: A publicly accessible server that implements the server side of the Push Protocol and exposes an HTTP API for AppServer's to notify it.

UserAgent
: A device or program that implements the client side of the Push Protocol.

UAID
: A globally unique UserAgent ID. Used by the PushServer to associate channelIDs with a client. Stored by the UserAgent, but opaque to it.

Channel
: The flow of information from AppServer through PushServer to UserAgent.

ChannelID
: Unique identifier for a Channel. Generated by UserAgent for a particular application. Opaque identifier for both UserAgent and PushServer. This MUST NOT be exposed to an application.

Endpoint
: A REST-ful HTTP URL uniquely associated to a channel. Requests to this URL should update the PushServer state for the channel. MUST be exposed to applications.

Version
: Monotonically increasing 64-bit integer describing the application state. This
holds meaning as a primary key or similar only to the AppServer. The PushServer
and UserAgent and App should use this only for detecting changes. MUST be exposed to applications.

Application
: A program which requires access to push notifications. The UserAgent acts on behalf of applications.
{: br}

# Protocol Overview

The SimplePush protocol defines how UserAgents and PushServers communicate to ensure reliable delivery of the latest version of a channel from the PushServer to the UserAgent.

The SimplePush communication channel is WebSockets, and the wire protocol is JSON, with messages defined below.
The WebSocket specific HTTP header `Sec-WebSocket-Protocol` MUST be set to `push-notification` by the UserAgent. The PushServer SHOULD reject the connection if the header is not set. (FIXME: MUST/SHOULD paradox?)

All messages MUST use TLS (wss:// protocol for WebSockets). In addition, Endpoints generated by PushServer's MUST use HTTPS URLs.

The PushServer should maintain a mapping of channelIDs and their versions and pushEndpoints.
SimplePush is backed by a best effort delivery mechanism over WebSocket

The protocol has some request-response driven actions, where the UserAgent MUST make requests, and the PushServer MAY respond. One message, `notification`, MAY be sent at any time by the PushServer.

# Considerations 

Since SimplePush is not a data carrying system, there are a few things developers should keep in mind.

1. *Do not send a SimplePush update message for every bit of exchanged data.*

This service is a way for you to wake your remote app and have it reconnect back to you. If it's already connected, sending a SimplePush service is wasteful. (It's a bit like sending a fax to make sure the person on the phone got your email.)

2. *While it's possible to have more than one Channel per App, it helps to be frugal about them.*

Remember, that each Channel has an associated action, and that they're not really "free", and they can be lossy. A single channel that allows the system to start your app, or a separate channel that advises that a screen widget needs to be updated are all fine uses. Having one channel per bullet path for a FPS game is not, since this will pretty much guarantee data will be lost. 

3. *Don't flood the channel.*

Nobody likes having something constantly remind you to do something, particularly after you've already done it. In the same way, be conscious that channel updates invoke actions on your user's devices, and may lead to users removing your app. Once your app connects, you'll have a far more efficient and faster pipe than SimplePush. 

4. *Pay attention to endpoint reassignments.*

When a given user's endpoint changes, the previous endpoint is no longer valid. Updates sent to that endpoint will no longer have any effect and are a waste of your bandwidth. If you send a great many updates to endpoints that are invalid, the server may have a very difficult time determining that you're not being hostile. 

# ChannelID, UAIDs and Endpoint

Since the UAID is the only unique identifier of a UserAgent, with no authentication information, it is important that UAIDs are not compromised. Only the UserAgent to which the UAID was assigned, and the PushServer, should know the UAID. An attacker can use a leaked UAID to pretend to be the UserAgent with that UAID. Although SimplePush does not transfer any private information, the attacker would still receive the notifications meant for the victim. If any party detects a compromise, they SHOULD reset the UAID. For this, the UserAgent may send a blank UAID and get a new one from the PushServer. Similarly the PushServer may send a new UAID during a handshake.

ChannelIDs are also unique. It is RECOMMENDED that channelIDs be associated with a UAID, so that an attacker cannot receive notifications for a compromised channelID without also having accessed the UAID.

It is REQUIRED that both UAID and channelIDs be UUIDv4 tokens.

The format of the Endpoint is not specified by this protocol. There are two requirements that MUST be satisfied:

* PushServers MUST use HTTPS, so the Endpoint URL will begin with "https://".
* ChannelIDs and Endpoints MUST have a one-one mapping and Endpoints may not 'expire' while their channelID is still active.

It is RECOMMENDED that Endpoints simply be a prefix followed by the channelID or some transformation of the channelID which is referentially transparent. PushServers SHOULD avoid exposing UAIDs in the Endpoints since they are sent to AppServers.

# Messages

All messages are encoded as JSON. All messages MUST have the following fields:

messageType (string)
: Defines the message type
{: br}

## Handshake

After the WebSocket is established, the UserAgent begins communication by
sending a `hello` message.  The hello message contains the UAID if the
UserAgent has one, either generated by the UserAgent for the first handshake or
returned by the server from an earlier handshake. The UserAgent also transmits
the channelIDs it knows so the server may synchronize its state.

The server MAY respect this UAID, but it is at liberty to ask the UserAgent
to change its UAID in the response.

If the UserAgent receives a new UAID, it MUST delete all existing channelIDs
and their associated versions. It MAY then wake up all registered applications
immediately or at a later date by sending them a `push-register`
message.

The handshake is considered ''complete'', once the UserAgent has received a reply.

An UserAgent MUST transmit a `hello` message *only once* on its
WebSocket.  If the handshake is not completed in the first try, it MUST
disconnect the WebSocket and begin a new connection.

*NOTE:* Applications may request registrations or unregistrations from the
UserAgent, before or when the handshake is in progress. The UserAgent MAY
buffer these or report errors to the application. But it MUST NOT send these
requests to the PushServer until the handshake is completed.

### UserAgent -> PushServer

messageType = "hello"

: Begin handshake

uaid string (REQUIRED)

: If the UserAgent has a previously assigned UAID, it should send it. Otherwise send an empty string.

channelIDs list of strings (REQUIRED)

: If the UserAgent has a list of channelIDs it wants to be notified of, it must pass these, otherwise an empty list.
{: br}



Extra fields:
The UserAgent MAY pass any extra JSON data to the PushServer. This data may include information required to wake up the UserAgent out-of-band. The PushServer MAY ignore this data.

#### Example

~~~~~
  {
    "messageType": "hello",
    "uaid": "fd52438f-1c49-41e0-a2e4-98e49833cc9c",
    "channelIDs": ["431b4391-c78f-429a-a134-f890b5adc0bb",
                   "a7695fa0-9623-4890-9c08-cce0231e4b36"]
  }
~~~~~

### PushServer -> UserAgent

PushServers MUST only respond to a hello once.
UserAgents MUST ignore multiple hello replies.

messageType = "hello"
: Responses generally have the same messageType as the request

uaid string (REQUIRED)
: If the UserAgent sent no UAID, generate a new one. If the UserAgent send a valid UAID and the PushServer is in sync with the UserAgent, send back the same UAID, otherwise the PushServer should generate a new UAID.
{: br}



#### Example

~~~~~
  {
    "messageType": "hello",
    "uaid": "fd52438f-1c49-41e0-a2e4-98e49833cc9c"
  }
~~~~~

## Register

The Register message is used by the UserAgent to request that the PushServer notify it when a channel changes. Since channelIDs are associated with only one UAID, this effectively creates the channel, while unregister destroys the channel.

The channelID is chosen by the UserAgent because it also acts like a nonce for the Register message itself. Because of this PushServers MAY respond out of order to multiple register messages or messages may be lost without compromising correctness of the protocol. 

The request is considered successful only after a response is received with a status code of 200. On success the UserAgent MUST:

* Update its persistent storage based on the response
* Notify the application of a successful registration. 
* On error, the UserAgent MUST notify the application as soon as possible.

NOTE: The register call is made by the UserAgent on behalf of an application. The UserAgent SHOULD have reasonable timeouts in place so that the application is not kept waiting for too long if the server does not respond or the UserAgent has to retry the connection.

### UserAgent -> PushServer

;messageType = "register"

;channelID string (REQUIRED)
:A unique identifier generated by the UserAgent, distinct from any existing channelIDs it has registered. It is RECOMMENDED that this is a UUIDv4 token.

#### Example

~~~~
  {
    "messageType": "register",
    "channelID": "d9b74644-4f97-46aa-b8fa-9393985cd6cd"
  }
~~~~

### PushServer -> UserAgent

messageType = "register"

channelID string (REQUIRED)
:This MUST be the same as the channelID sent by the UserAgent in the register request that this message is a response to.

status number (REQUIRED)
: Used to indicate success/failure. MUST be one of:
* 200 - OK. Success. Idempotent: If the PushServer receives a register for the same channelID from a UserAgent which already has a registration for the channelID, it should still respond with success.
* 409 - Conflict. The chosen ChannelID is already in use and NOT associated with this UserAgent. UserAgent SHOULD retry with a new ChannelID as soon as possible.
* 500 - Internal server error. Database out of space or offline. Disk space full or whatever other reason due to which the PushServer could not grant this registration. UserAgent SHOULD avoid retrying immediately.

pushEndpoint string (REQUIRED)
: Should be the URL sent to the application by the UserAgent. AppServers will contact the PushServer at this URL to update the version of the channel identified by channelID.
{: br}

#### Example

~~~~
  {
    "messageType": "register",
    "channelID": "d9b74644-4f97-46aa-b8fa-9393985cd6cd",
    "status": 200,
    "pushEndpoint": "http://pushserver.example.org/d9b74644"
  }
~~~~

## Unregister

Unregistration is an optional procedure.

PushServers MUST support it.
UserAgents SHOULD support it.

The unregister is required only between the App and the UserAgent, so that the
UserAgent stops notifying the App when the App is no longer interested in a pushEndpoint.

The unregister is also useful to the AppServer, because it should stop
sending notifications to an endpoint the App is no longer monitoring. Even
then, it is really an optimization so that the AppServer need not have some
sort of garbage collection mechanism to clean up endpoints at intervals of time.

The PushServer MUST implement unregister, but need not rely on it. Well behaved
AppServers will stop notifying it of unregistered endpoints automatically. Well behaved
UserAgents won't notify their apps of unregistered updates either. So the PushServer
can continue to process notifications and pass them on to UserAgents, when it has
not been told about the unregistration.

When an App calls `unregister(endpoint)` it is RECOMMENDED that the UserAgent follow these steps:

* Remove its local registration first, for example from the database.
  This will allow it to immediately start ignoring updates.
* Notify the App that unregistration succeeded.
* Fire off an unregister message to the PushServer

### UserAgent -> PushServer

messageType = "unregister"

channelID string (REQUIRED)
: This is sort of obvious isn't it? :)
{: br}

#### Example

~~~~
  {
    "messageType": "unregister",
    "channelID": "d9b74644-4f97-46aa-b8fa-9393985cd6cd"
  }
~~~~

### PushServer -> UserAgent

messageType = "unregister"

channelID string (REQUIRED)
: This MUST be the same as the channelID sent by the UserAgent in the unregister request that this message is a response to.

status number (REQUIRED)
: Used to indicate success/failure. MUST be one of:
  * 200 - OK. Success. Idempotent: If the PushServer receives a unregister for a non-existent channelID it should respond with success. If the channelID is associated with a DIFFERENT UAID, it MUST NOT delete the channelID, but still MUST respond with success to this UserAgent.
* 500 - Internal server error. Database offline or whatever other reason due to which the PushServer could not grant this unregistration. UserAgent SHOULD avoid retrying immediately.
{: br}

#### Example

~~~~
  {
    "messageType": "unregister",
    "channelID": "d9b74644-4f97-46aa-b8fa-9393985cd6cd",
    "status": 200
  }
~~~~

## Ping

The connection to the Push Server may be lost due to network issues. When the UserAgent detects loss of network, it should reconnect. There are situations in which the TCP connection dies without either end discovering it immediately. The UserAgent should send a ping approximately every 30 minutes andexpect a reply from the server in a reasonable time (The Mozilla UserAgent uses 10 seconds). If no data is received, the connection should be presumed closed and a new connection started.

The UserAgent should consider normal communications as an indication that the socket is working properly. It SHOULD send the ping packet only if no activity has occurred in the past 30 minutes.

#### UserAgent -> PushServer

The 2-character string `{}` is sent. This is a valid JSON object that requires no alternative processing on the server, while keeping transmission size small.

### PushServer -> UserAgent

The PushServer may reply with any data. The UserAgent is only concerned about the state of the connection. The PushServer may deliver pending notifications or other information. If there is no pending information to be sent, it is RECOMMENDED that the PushServer also reply with the string `{}`.

## Notification

### AppServer -> PushServer

The AppServer MUST make a HTTP *PUT* request to the Endpoint received from the App.

If no request body is present, the server MAY presume the version to be the current server UTC. 

If the request body is present, the request MUST contain the string "version=N" and the Content-Type MUST be `application/x-www-form-urlencoded`.

### PushServer -> AppServer

The HTTP response status code indicates if the request was successful.

* 200 - OK. The PushServer will attempt to deliver a notification to the associated UserAgent.
* 500 - Something went wrong with the server. Rare, but the AppServer should try again.

The HTTP response body SHOULD be empty.

### PushServer -> UserAgent

Notifications are *acknowledged* by the UserAgent.  PushServers should
retry unacknowledged notifications every 60 seconds.  If the version of an
unacknowledged notification is updated, the PushServer MAY queue up a new
notification for this channelID and the new version, and remove the old
notification from the pending queue.

messageType: "notification"

updates list  REQUIRED
: The list contains one or more {"channelID": "id", "version": N } pairs.
{: br}

##### Example

~~~~
  {
    "messageType": "notification",
    "updates": [
      { "channelID": "431b4391-c78f-429a-a134-f890b5adc0bb",
        "version": 23 },
      { "channelID": "a7695fa0-9623-4890-9c08-cce0231e4b36",
        "version": 42 } ]
  }
~~~~

#### UserAgent -> PushServer

It is RECOMMENDED that the UserAgent try to batch all pending acknowledgements into fewer messages.

messageType ="ack"

updates list
: The list contains one or more {"channelID": channelID, "version": N} pairs.
{: br}

##### Example

~~~~
  {
    "messageType": "ack",
    "updates": [
       { "channelID": "431b4391-c78f-429a-a134-f890b5adc0bb",
         "version": 23 },
       { "channelID": "a7695fa0-9623-4890-9c08-cce0231e4b36",
         "version": 42 } ]
  }
~~~~

# Synchronization of server and client state

# Achieving reliable delivery

At any time, if the PushServer reaches an inconsistent condition, due to database failure, network failure, or any other reason, it MAY drop all state. It MUST then disconnect all active clients, forcing them to reconnect and begin a handshake to get synchronized.

# Garbage collection

The PushServer MAY perform occasional garbage collection to reduce its disk/memory usage.
UserAgents that have not contacted the PushServer for a long period of time (a few days. TODO decide minimum) are eligible for garbage collection.
To do this, the PushServer simply deletes the UAID and all associated ChannelIDs and other associated data (including versions and Endpoints).
The deletion of the UAID should be done first, and ''atomically''. Other details may be deleted when the server has spare cycles. Now if the UserAgent with said UAID connects to the PushServer, it will be assigned a new UAID, the handshake will occur and Apps on the server will learn the latest version (TODO: really need NotifyUnknown!)

The PushServer MUST NOT delete channelIDs without deleting the associated UAID, except when one of the following occurs:

* The UserAgent explicitly sends an unregister message.
* or, When a previously known UserAgent reconnects, in the handshake it will transmit channelIDs which are registered on the UserAgent's side. If the PushServer finds that it has registrations for channelIDs that are not in the list transmitted by the UserAgent, it may delete these extra channelIDs.

# Alternative communication

In environments or devices where maintaining an always alive WebSocket is difficult, UserAgents and PushServers MAY implement alternative means to notify UserAgents about updates. This out of band notification SHOULD be restricted to request the UserAgent to establish the WebSocket. SimplePush state SHOULD NOT be transmitted out of band, since reliable delivery may not be guaranteed.


# Security Considerations

[TODO]


# IANA Considerations

[TODO]
--- back
