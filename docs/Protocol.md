# Introduction 

The Health Monitor (HMON) interface to the Dyalog interpreter allows a client
application or tool to connect and monitor its activity, memory usage etc.,
to gauge its "state of health". The communications protocol between the
interpreter and client is described in this document.

THE HEALTH MONITOR INTERFACE IS UNDER CONTINUING DEVELOPMENT AND IS
INCLUDED IN DYALOG 19.0 AND 20.0 AS AN EXPERIMENTAL FEATURE. FURTHER DEVELOPMENT IS
PLANNED FOR SUBSEQUENT RELEASES AND THIS MAY RESULT IN CHANGES TO THE INTERFACE
WHICH ARE INCOMPATIBLE WITH THE SPECIFICATION DESCRIBED HERE.

> [!NOTE]
> Features which differ between different versions of Dyalog are highlighted like this.

# Establishing connections and controlling access levels

The interpreter to be monitored may either listen for incoming client
connections or initiate a connection to a client. It will not do either unless
explicitly configured to do so using either the `HMON_INIT` config setting
(prior to startup) or [`112⌶`](#112) (from within the interpreter).

The connection mode, address and port on which the interpreter should listen 
or connect (the Interface Configuration) is determined by a character sequence
of the form:

`mode:address:port`

where:

- _mode_ is the connection mode - either SERVE (listen for incoming connections)
  or POLL (initiate outgoing connection).
- _address_ is the address of an interface on the local machine on which the
  the interpreter should listen (SERVE mode) or the address of a machine
  running a client to which the interpreter should connect (POLL mode).
- _port_ is the TCP port to listen on or connect to.

In SERVE mode _address_ may be specified as follows:

- `<empty>` - listen on all loopback interfaces, that is, the interpreter only
  accepts connection from the local machine.
- `*` – listen on all local machine interfaces, that is, the interpreter
  listens for connections from any (local or remote) machine/interface.
- The host/DNS name of the machine/interface running the interpreter –
  listen on that specific interface on the local machine.
- The IPv4 address of the machine/interface running the interpreter –
  listen on that specific interface on the local machine.
- The IPv6 address of the machine/interface running the interpreter –
  listen on that specific interface on the local machine.

In POLL mode _address_ may be specified as follows:

- `<empty>` – connect to a client on the the local machine.
- The host/DNS name of the machine/interface running the client.
- The IPv4 address of the machine/interface running the client.
- The IPv6 address of the machine/interface running the client.

For example:

`HMON_INIT="SERVE:localhost:4512"`

The level of information made available by the interpreter is determined by an
Access Level:

- 0 - No connections permitted.
- 1 - Permit connections, with restricted information provided once connected.
  Restricted information includes memory usage, number of running threads etc.
  but nothing which exposes the code of the application being run, such as the
  SI stack.
- 2 - Permit connections, with full information provided once connected.

The Access Level defaults to 1 with runtime interpreters and 2 with development interpreters.

Some information which the interpreter provides requires it to perform
additional work (which slows it down) even when a connected client is not
requesting it. Whether it does this or not is controlled by setting an Event
Gathering Level.

- 0 - Do not gather information for [`GetLastKnownState`](#getlastknownstate)
  requests.
- 1 - Gather information for [`GetLastKnownState`](#getlastknownstate)
  requests - has a runtime performance impact.

The Event Gathering Level defaults to 0.

The Access Level and Event Gathering Level may be altered from within the
interpreter using [`112⌶`](#112). Alternatively, [`112⌶`](#112) may be used to
set the interface configuration and the Levels without setting `HMON_INIT` prior
to starting the interpreter, or to override any existing `HMON_INIT` setting.

# Messages

## Overview

A message starts with a 4-byte big-endian _total length_ field, followed by
the four ASCII-encoded characters `HMON` and a UTF-8-encoded payload:

```
 Total length        "HMON"              Payload
┌───────────────────┬───────────────────┬─────~─────┐
│0x00 0x00 0x00 0x0B│0x48 0x4D 0x4F 0x4E│    ...    │
└───────────────────┴───────────────────┴─────~─────┘
```

_Total length_ is therefore 8 + the byte length of the payload.

The payload is almost always a 2-element JSON array consisting of a Message
Name and arguments as key/value pairs:

```json
["MessageName",{"key1":"value1","key2":222,"key3":[3,4,5]}]
```

The only exception are the first two messages that each side sends upon
establishing a connection. These constitute the _handshake_ and are not
JSON-encoded. Their payloads are:

```
SupportedProtocols=2
UsingProtocol=2
```

Messages sent by the client to the interpreter are _request_ messages. Messages
sent by the interpreter to the client are _response_ messsages.

The interpreter will usually respond to a request message by sending a response
containing either the information requested, confirmation of receipt, or an
error report (indicating e.g. invalid syntax or invalid message type etc.) It
may also send response messages at any time - for example, to inform of a
particular event or to provide regular updates on its condition, or if the
application that the interpreter is running initiates them.

The interpreter will process most request messages it receives when it is
between execution of APL code or otherwise in a position where it can safely
access its workspace, and may therefore not react immediately. The special
request message [`GetLastKnownState`](#getlastknownstate) is designed to remain
active and provide an immediate response in the event that the the interpreter
is unable to respond at all to the others.

Messages must be syntactically valid JSON text and of the 2-element array form
described above.

Named items in the payload are described in the relevant documentation for each
message type. The interpreter will ignore any unexpected named item (so long as
it is described using syntactically valid JSON). Except as noted, request
messages may include the named item "UID" (a string value) and if present the
response will echo the same UID value. UID strings may be used by a client as
it wishes - for example, to track requests and responses; the interpreter does
not require them.

## Request messages

### GetFacts

Requests zero or more "facts" about the application state, and will be
responded to with a [`Facts`](#facts) message.

```json
["GetFacts",{"Facts":["Host","Workspace"]}]
```

The following facts may be requested:

| Fact ID | Fact name            |
| ------- | -------------------- |
| 1       | "Host"               |
| 2       | "AccountInformation" |
| 3       | "Workspace"          |
| 4       | "Threads"            |
| 5       | "SuspendedThreads"   |
| 6       | "ThreadCount"        |

See the [`Facts`](#facts) response message for the information provided.

Fact may be requested using either their numeric Fact ID or their alphanumeric
(string) Fact name, so the following `GetFacts` requests are equivalent to the
one above:

```json
["GetFacts",{"Facts":[1,3]}]
```

```json
["GetFacts",{"Facts":["Host",3]}]
```

### PollFacts

`PollFacts` behaves in the same way as [`GetFacts`](#getfacts) except that it
polls - that is, the [`Facts`](#facts) message response will be sent
immediately and then repeat after specified or implied intervals.

The interval defaults to 1000ms but any value of 500ms or more may be
specified. Values less than 500 will be taken as 500.

Example:

```json
["PollFacts",{"Facts":[1,"Workspace"],"Interval":750}]
```

If a UID is specified it will appear in every message that is sent.

Messages will continue at the requested frequency until either a new request
is made (which will supersede any already established) or a
[`StopFacts`](#stopfacts) message is sent.

**Note:** polling responses may occasionally stop when the interpreter is
waiting on an external event such as a file operation, `⎕NA` call, etc.  It is
currently also a limitation that polling messages may also stop when the
interpreter is inactive - that is, when it is not running APL code or
responding to external input such as HMON requests or keyboard events.

### StopFacts

`StopFacts` cancels [`PollFacts`](#pollfacts).

A UID may not be included in the message.

Example:

```json
["StopFacts",{}]
```

A [``Facts``](#facts) message will be sent back, with an empty list of facts and
Interval set to 0.

### BumpFacts

A `BumpFacts` message will cause a polling [`Facts`](#facts) message to be sent,
regardless of the time remaining until the next message is due. Messages will
then continue at the normal frequency.

A UID may not be included in the message.

Example:

```json
["BumpFacts",{}]
```

### Subscribe

The `Subscribe` message tells the interpreter to send notification messages
when certain, specifiable, events occur. The interpreter will confirm the
settings with a [`Subscribed`](#subscribed) message in response, and a
[`Notification`](#notification) message whenever the subscribed events occur.
If the `Subscribe` message contains a UID then the [`Subscribed`](#subscribed)
response and all subsequent [`Notification`](#notification) messages of all
types will echo that UID.

Each subscribable event has a numeric and alphanumeric (string) identifier, and
either may be used for the request. Example:

```json
["Subscribe",{"UID":"XX","Events":[1,4]}]
```

No event notifications are enabled by default.

The following events may be subscribed to:

| Subscription ID | Subscription name   |
| --------------- | ------------------- |
| 1               | WorkspaceCompaction |
| 2               | WorkspaceResize     |
| 3               | UntrappedSignal     |
| 4               | TrappedSignal       |

When a subscribed event takes place, the [`Notification`](#notification)
message will include details of that event, as documented there.

Sending a `Subscribe` message resets the list of subscribed events to those
specified - that is, it replaces any existing subscriptions. The list may be
empty.

**Note:** following a WSFULL exception the interpreter may be unable to send
either a TrappedSignal or UntrappedSignal.
[`GetLastKnownState`](#getlastknownstate) will reliably report the last time a
WSFULL event occurred.

### GetLastKnownState

Requests the last known state of the interpreter, and will be responded to with
a [`LastKnownState`](#lastknownstate) message.


Examples:

```json
["GetLastKnownState",{}]
```

```json
["GetLastKnownState",{"UID":"123"}]
```

`GetLastKnownState` requests can be used when a monitored interpreter becomes
otherwise unresponsive. In "normal" use, [`GetFacts`](#getfacts) should be used.

The "Last Known State" is provided from a repository which must be continuously
maintained by the interpreter during its normal operation on the off-chance
that it might be asked for at any time. This introduces an overhead so is only
done if the Event Gathering Level is set to 1. Maintaining this repository is
independent of whether a Health Monitor is connected at the time or not.

[`112⌶`](#112) controls the Event Gathering Level.

In addition, `⎕PROFILE` must currently be started to provide full
information, e.g.:

`⎕PROFILE 'start' 'coverage'`

## Response messages

### Facts

Reports one or more "facts" about the application state, corresponding to a
[`GetFacts`](#getfacts), [`PollFacts`](#pollfacts), [`StopFacts`](#stopfacts)
or [`BumpFacts`](#bumpfacts) request.

The facts are presented as an array of objects in the same order as in the
request. Each will contain a "Value" object or a "Values" array of objects,
the contents of which depend on the fact type.

Example:

```json
["Facts",{"UID":"xx","Interval":5000,"Facts":[{"ID":6,"Name":"ThreadCount","Value":{"Total":1,"Suspended":0}}]}]
```

"Interval" is only present in the response if polling.

#### "Host" fact

The "Value" object contains:

- "Machine" - an object containing facts about the host machine:
  - "Name" - the name of the machine.
  - "User" - the name of the user account.
  - "PID" - the interpreter process ID.
  - "Desc" - an application-specific description - see [`110⌶`](#110).
  - "AccessLevel" - the level of rights the Health Monitor is permitted.

- "Interpreter" - an object containing facts about the host interpreter:
  - "Version" - the interpreter version, in the form "A.B.C".
  - "BitWidth" - the interpreter edition word size, either 32 or 64 (bits).
  - "IsUnicode" - a Boolean value indicating whether the interpreter is a
    Unicode edition or not (i.e. is Classic).
  - "IsRuntime" - a Boolean value indicating whether the interpreter is a
    Runtime edition or not (i.e. is a development version).
  - "SessionUUID" - a String value containing a unique Session UUID in
    [RFC 9562](https://datatracker.ietf.org/doc/html/rfc9562) format.

> [!NOTE]
> "SessionUUID is not provided by Dyalog 19.0.

- "CommsLayer" - an object containing facts about the interpreter comms layer
  servicing the Health Monitor:
  - "Version" - the Comms Layer (Conga) version.
  - "Address" - the interpreter's network IP address.
  - "Port4" - the interpreter's network port number.
  - "Port6" - an alternate port number.

- "RIDE" - an object containing facts about the interpreter comms later
  servicing RIDE:
  - "Listening" - a Boolean value indicating whether the interpreter is
    listening for RIDE connections. There are no further entries in this object
    if the value is 0.
  - "HTTPServer" - a Boolean value indicating whether the interpreter is
    running as a RIDE HTTP server ("Zero footprint" RIDE).
  - "Version" - the Comms Layer (Conga) version.
  - "Address" - the interpreter's network IP address.
  - "Port4" - the interpreter's network port number.
  - "Port6" - an alternate port number.

#### "AccountInformation" fact

The "Value" object contains:

- "UserIdentification", "ComputeTime", "ConnectTime", "KeyingTime" - elements
  from `⎕AI`.

#### "Workspace" fact

The "Value" object contains:

- "WSID" - the workspace name.
- "Available", "Used", "Compactions", "GarbageCollections", "GarbagePockets",
  "FreePockets", "UsedPockets", "Sediment", "Allocation", "AllocationHWM",
  "TrapReserveWanted", "TrapReserveActual" - statistics from `2000⌶`.

#### "Threads" fact

The "Values" array contains one or more objects (one per thread), each containing:

- "Tid" - thread ID
- "Stack" - SIstack, as an array of objects each containing:
  - "Restricted": a Boolean value indicating whether some information is
    restricted (missing) because the Access Level does not permit it.
  - "Description" - a line of SIstack information - only if "Restricted" is 0.
- "Suspended" - a Boolean value indicating whether the thread is suspended.
- "State" - a string indicating the current location of the thread.
- "Flags" - e.g. Normal, Paused or Terminated.

If "Suspended" is 1 the object will also contain:

- "DMX" - an object containing elements of `⎕DMX` in that thread, either `null`
  or:
  - "Category", "DM", "EM", "EN", "ENX", "InternalLocation", "Vendor"
    "Message", "OSError" - only if "Restricted" is 0.
  - "Restricted": a Boolean value indicating whether some information is
    restricted (missing) because the Access Level does not permit it.
- "Exception" - an object containing elements of `⎕EXCEPTION` in that thread,
  either `null` or:
  - "Source", "StackTrace", "Message" - only if "Restricted" is 0.
  - "Restricted": a Boolean value indicating whether some information is
    restricted (missing) because the Access Level does not permit it.

#### "SuspendedThreads" fact

The "Values" array contains one or more objects (one per suspended thread), each
containing values as descrived for ["Threads"](#threads-fact) with the exception
that the "Suspended" item is omitted as it would always have the value 1.

#### "ThreadCount" fact

The "Value" object contains:

- "Total" - the total number of threads.
- "Suspended" - the number of threads which are suspended.

### Subscribed

The response to [`Subscribe`](#subscribe). The message lists all subscription
events and whether they are enabled or disabled.

Example (showing an incomplete list of subscription events):

```json
["Subscribed",{"UID":"XX","Events":[{"ID":1,"Name":"WorkspaceCompaction","Value":1},{"ID":2,"Name":"WorkspaceResize","Value":0}]}
```
### Notification

A `Notification` message indicates that a specified event, for which
notifications have been subscribed, has taken or is taking place. The
[`Subscribe`](#subscribe) message is used to select notified event types. The
`Notification` message always reports a single event and contains the event ID
and name, along with any specific detail pertaining to that event type.

Example:

```json
["Notification",{"UID":"XX","Event":{"ID":2,"Name":"WorkspaceResize"},"Size":894213}]
```

#### "WorkspaceCompaction" event

Occurs when a workspace compaction has occurred. The additional values provided are:

- "Tid" - thread ID.
- "Stack" - SIstack, as an array of objects each containing:
  - "Restricted": a Boolean value indicating whether some information is
    restricted (missing) because the Access Level does not permit it.
  - "Description" - a line of SIstack information - only if "Restricted" is 0.

#### "WorkspaceResize" event

Occurs when the workspace size (the amount of memory committed by the OS) has
grown or shrunk. The additional values provided are:

- "Size" - new size, in bytes.

#### "UntrappedSignal" event

Occurs when an APL exception has been signalled which has not been trapped. The additional values provided are:

- "Tid" - thread ID.
- "Stack" - SIstack, as an array of objects each containing:
  - "Restricted": a Boolean value indicating whether some information is
    restricted (missing) because the Access Level does not permit it.
  - "Description" - a line of SIstack information - only if "Restricted" is 0.
- "DMX" - an object containing elements of `⎕DMX` in that thread, either `null`
  or:
  - "Category", "DM", "EM", "EN", "ENX", "InternalLocation", "Vendor"
    "Message", "OSError" - only if "Restricted" is 0.
  - "Restricted": a Boolean value indicating whether some information is
    restricted (missing) because the Access Level does not permit it.
- "Exception" - an object containing elements of `⎕EXCEPTION` in that thread,
  either `null` or:
  - "Source", "StackTrace", "Message" - only if "Restricted" is 0.
  - "Restricted": a Boolean value indicating whether some information is
    restricted (missing) because the Access Level does not permit it.

#### "TrappedSignal" event

Occurs when an APL exception has been signalled which has not been trapped. The
additional values provided are as described for
[`UntrappedSignal`](#untrappedsignal-event)

### LastKnownState

The response to [`GetLastKnownState`](#getlastknownstate), containing:

- The UID, if provided in the request.
- The interpreter\'s current UTC clock setting, so that times elapsed since the
  other timings can be computed.
- The line currently being executed by the interpreter and the UTC time that
  this line started or resumed execution, if enabled with `112⌶` and `⎕PROFILE`
  is running (otherwise this information is omitted).
- An activity code and the UTC time that this activity started, if enabled with
  [`112⌶`](#112) (otherwise omitted).
- The time a trapped or untrapped WSFULL event last occurred, if enabled with
  [`112⌶`](#112) (otherwise omitted).

UTC times are in ISO format with millisecond precision, e.g.
20231231T235959.999 for the very last millisecond of 2023.

Activity codes are:

| Code | Meaning                                 |
| ---- | --------------------------------------- |
| 1    | Anything not specifically listed below  |
| 2    | Performing a workspace allocation       |
| 3    | Performing a workspace compaction       |
| 4    | Performing a workspace check            |
| 222  | Sleeping (an internal testing feature)  |

_This functionality is at an early stage of development. It is anticipated that
this list will be significantly extended in future._

Examples:

#### No UID provided, and [`112⌶0`](#112) set:

```json
["LastKnownState",{"TS":"20230111T144700.132Z"}]
```

#### UID provided, [`112⌶2 1`](#112) set and `⎕PROFILE` started:

```json
["LastKnownState",{"UID":"123","TS":"20230111T144700.132Z","Activity":{"Code":1,"TS":"20230111T144700.132Z"},"Location":{"Function":"#.f","Line":2,"TS":"20230111T144700.132Z"},"WS FULL":{"TS":"20230111T144620.723Z"}}]
```

Note: "Location" is updated by the interpreter whenever execution of a line
begins or resumes. If program execution stops for any reason (e.g.  exception
or program termination) it will report the last executed line. "location" does
not report anything about inactive threads - full thread/stack info is
available with [`GetFacts`](#getfacts), so long as the interpreter is
responsive.

### InvalidSyntax

The response to a syntactically invalid JSON message, or a message which does
not strictly define a two-element array with a string name in the first array
element and an object in the second array element. Because it is a response to
an "unintelligible" message, it will never contain a UID response.

Example:

```json
["InvalidSyntax",{}]
```

### DisallowedUID

The response to a message which contains a UID when it should not -
specifically, [`StopFacts`](#stopfacts) and [`BumpFacts`](#bumpfacts) message.

Example:

```json
["DisallowedUID",{"UID":"xx","Name":"StopFacts"}]
```

### UnknownCommand

The response to a syntactically correct message which contains a message with
an unrecognised name. The unrecognised name, and the UID if provided, ares
included in the message.

Example:

```json
["UnknownCommand",{"UID":"ABC","Name":"Hatstand"}]
```

### MalformedCommand

The response to a syntactically correct request message which does not exactly
conform to specification. The name, and the UID if provided, are included in
the message.

Example:

```json
["MalformedCommand",{"Name":"GetLastKnownState"}]
```

### UserMessage

Sent by the interpreter under user/application control using [`111⌶`](#111).

Example:

```json
["UserMessage",{"UID":"123","Message":"Hello"}]
```

# Interpreter I-beam functions

## 110

`{R}←(110⌶) Y`

Specifies the interpreter "description", which will appear in
[`Facts`](#host-fact) messages sent to the Health Monitor.

Y is a character vector or scalar containing the free-form text.

The shy result is the value 1.

## 111

`{R}←{X} (111⌶) Y`

Will cause the interpreter to send a [`UserMessage`](#usermessage) notification
message to the client, if one is connected.

Y is a character vector or scalar containing the free-form message text.

X is an optional character vector or scalar containing the UID.

The shy result is the value 1.

## 112

`R←X (112⌶) Y`

Starts and stops the Health Monitor, specifies the Interface Configuration and
controls the Access and Event Gathering Levels. See the section
[`Permitting connections and controlling access levels`](#permitting-connections-and-controlling-access-levels)
for an explanation of what these are and their permitted values.

Y is a 1 or 2-element numeric array consisting of:

- Settings for Access Level, _and optionally_
- Event Gathering.

X may be omitted, or scalar zero, or an empty character vector, or character
vector specifying the Interface Configuration.

### Starting the HMON comms layer

X should be a character vector containing either:

- The Interface Configuration, _or_
- Nothing (i.e. empty), in which case the previous Interface Configuration (if
  any) is reused.

Y should contain:

- The Access Level values of 1 or 2, _and optionally_
- The Event Gathering setting; defaults to 0 if not specified.

### Updating the Access Level and Event Gathering settings

The function should be called monadically or with 0 in X.

Y should contain:

- The Access Level values of 1 or 2, _and optionally_
- The Event Gathering setting, which defaults to 0 if not specified.

The settings can be changed whether or not a client is currently attached and
take effect immediately.

### Stopping the HMON comms layer

The function should be called monadically or with 0 in the left argument.

Y should contain:

- The Access Level value 0.

### Errors and return values

No action will be taken and 0 will be returned if:

- A request is made to start the HMON comms layer when it is already started.
- A request is made to update the Access Level and Event Gathering settings,
  and the HMON comms layer is not started.
- A request is made to stop the HMON comms layer when it is already stopped.

Otherwise, the comms layer is stopped or started as requested and then:

- An error will be signalled if the operation fails.
- The shy value 1 will be returned if the operation succeeds.
