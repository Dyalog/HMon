# Introduction

The Health Monitor (HMON) interface to the Dyalog interpreter allows a separate
user application or tool to connect and monitor its activity, memory usage etc.,
to gauge its "state of health".

Dyalog does not provide any application that uses the interface other than a
sample.

The interface is for a _monitor_ and only allows the client to _observe_
what is going on inside the interpreter to which it is attached, not
control it in any way.

THE HEALTH MONITOR INTERFACE IS UNDER CONTINUING DEVELOPMENT AND HAS BEEN
INCLUDED IN DYALOG 19.0 AS AN EXPERIMENTAL FEATURE. FURTHER DEVELOPMENT IS
PLANNED FOR SUBSEQUENT RELEASES AND THIS MAY RESULT IN CHANGES TO THE INTERFACE
WHICH ARE INCOMPATIBLE WITH THE SPECIFICATION DESCRIBED HERE.

By making use of this feature now and providing your feedback you will help
shape its ongoing development. Please do get in touch to discuss your experiences,
either through your normal communication channels or by emailing support@dyalog.com.

# Permitting connections and controlling access levels

The interpreter will not permit connections unless explicitly configured
to do so using either the `HMON_INIT` config setting (prior to startup)
or [`112⌶`](#112) (from within the interpreter).

The level of information made available by the interpreter can be controlled
by setting an appropriate Access Level:

- 0 - No connections permitted.
- 1 - Permit connections, with restricted information provided once
  connected. Restricted information includes memory usage, number of
  running threads etc. but nothing which exposes the code of the
  application being run, such as the SI stack.
- 2 - Permit connections, with full information provided once
  connected.

Some information which the interpreter provides requires it to perform additional
work (which slows it down) even when a connected Health Monitor application
is not requesting it. Whether it does this or not is controlled by setting
the Event Gathering Level:

- 0 - Do not gather information for [`GetLastKnownState`](#getlastknownstate) requests.
- 1 - Gather information for [`GetLastKnownState`](#getlastknownstate) requests - has a
  runtime performance impact.

The address and port on which the interpreter should listen for connections
(the Interface Configuration) is determined by a character sequence of the form:

`SERVE:address:port`

where:

- _address_ is the address of the interface on which the machine running
  the APL process should listen (that is, the address of the machine that is
  running the interpreter). Valid values are:
  - `<empty>` – listen on all loopback interfaces, that is, the interpreter
    only accepts connection from the local machine.
  - `*` – listen on all local machine interfaces, that is, the interpreter
    listens for connections from any (local or remote) machine/interface.
  - The host/DNS name of the machine/interface running the interpreter –
    listen on that specific interface on the local machine.
  - The IPv4 address of the machine/interface running the interpreter –
    listen on that specific interface on the local machine.
  - The IPv6 address of the machine/interface running the interpreter –
    listen on that specific interface on the local machine.
- _port_ is the TCP port to listen on.

The HMON_INIT configuration setting can be used to provide this, for example:

`HMON_INIT="SERVE:localhost:4512"`

When the interpreter starts and HMON_INIT is set, it will start the Health Monitor
interface, and with the following Access and Event Gathering Levels:

- Access Level 1 (runtime interpreters) or 2 (development interpreters).
- Event Gathering Level 0.

These levels may be altered from within the interpreter using [`112⌶`](#112). Alternatively,
[`112⌶`](#112) may be used to set the interface configuration _and_ the Levels without, or
to replace, any HMON_INIT setting.

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
establishing a connection. These constitute the _handshake_ and are not JSON-encoded.
Their payloads are:

```
SupportedProtocols=2
UsingProtocol=2
```

Messages may be sent by the client to the interpreter and the
interpreter will usually provide a response. This response may be the
information requested, confirmation of receipt, or an error report
(indicating e.g. invalid syntax or invalid message type etc.) No
response is ever guaranteed: the interpreter may be in a state where it
is unable to provide one.

The interpreter may also send messages to the client at any time, to
inform of a particular event or to provide regular updates on its
condition. These messages are only sent if either:

- The client has first subscribed to them, _or_
- The application that the interpreter is running initiates them.

The interpreter will never _expect_ a response to any message it sends.

The interpreter will process most messages it receives when it is
between execution of APL code or otherwise in a position where it can
safely access its workspace, and may therefore not respond immediately.
Some messages, however, are handled as soon as they arrive.

The different varieties of message are classified as follows:

- _Request_, sub-divided into:
  - _Standard request_: requests sent by the client, which are
    handled when the interpreter is able to safely access its workspace.
  - _High-priority request_: requests sent by the client, which
    are handled by the interpreter as soon as they arrive.
- _Response_: messages sent by the interpreter in reply to request
  messages.
- _Notification_: messages sent by the interpreter at any time.

Messages must be syntactically valid JSON text and of the 2-element
array form described above; messages received by the interpreter which
do not conform are rejected and an [`InvalidSyntax`](#invalidsyntax) response is sent. Messages
must have a valid Message Name; messages which do not are rejected and an
[`UnknownCommand`](#unknowncommand) response is sent.

Named items in the payload are described in the relevant documentation
for each message type. The interpreter will ignore any unexpected named
item (so long as it is described using syntactically valid JSON). Except
as noted, request messages may include the named item "UID" (a
string value) and if present the response will echo the same UID value.
If a UID is present in the messages to subscribe to notification
messages, it will be present in the response and in the notification
messages themselves.

The payload object in high-priority request messages must contain at
most one named item, which must be "UID" and have a string value if
present. If the message does not conform to this requirement, but is
otherwise syntactically valid, it will be handled as a standard request
message, rejected, and a [`MalformedCommand`](#malformedcommand) response will be sent.

## Standard request messages

### GetFacts

Requests zero or more "facts" about the application state, and will be
responded to with a [`Facts`](#facts) message. Each available fact has a numeric and
alphanumeric (string) identifier, and either may be used for the
request. Example:

```json
["GetFacts",{"Facts":[1,"Workspace"]}]
```

The [`Facts`](#facts) response will contain, for each requested fact,

- An object named \"Value\" containing the items described in the sections
  below, _or_
- An array named \"Values\" containing zero or more objects each
  containing the items described below.

The following facts may be requested:

| Fact ID | Fact name            | "Value" object or "Values" array in response |
| ------- | -------------------- | -------------------------------------------- |
| 1       | "Host"               | Value                                        |
| 2       | "AccountInformation" | Value                                        |
| 3       | "Workspace"          | Value                                        |
| 4       | "Threads"            | Values (one per thread)                      |
| 5       | "SuspendedThreads"   | Values (one per thread)                      |
| 6       | "ThreadCount"        | Value                                        |

The contents of the "Value" object or "Values" objects are:

#### "Host" fact

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

- "CommsLayer" - an object containing facts about the interpreter comms layer
  servicing the Health Monitor:
  - "Version" - the Comms Layer (Conga) version.
  - "Address" - the interpreter's network IP address.
  - "Port4" - the interpreter's network port number.
  - "Port6" - an alternate port number.

- "RIDE" - an object containing facts about the interpreter comms later servicing RIDE:
  - "Listening" - a Boolean value indicating whether the interpreter is listening
    for RIDE connections. There are no further entries in this object if the value is 0.
  - "HTTPServer" - a Boolean value indicating whether the interpreter is
    running as a RIDE HTTP server ("Zero footprint" RIDE).

#### "AccountInformation" fact

- "UserIdentification", "ComputeTime", "ConnectTime", "KeyingTime" - elements from `⎕AI`.

#### "Workspace" fact

- "WSID" - the workspace name.
- "Available", "Used", "Compactions", "GarbageCollections", "GarbagePockets",
  "FreePockets", "UsedPockets", "Sediment", "Allocation", "AllocationHWM",
  "TrapReserveWanted", "TrapReserveActual" - statistics from `2000⌶`.

#### "Threads" fact

- "Tid" - thread ID
- "Stack" - SIstack, as an array of objects each containing:
  - "Restricted": a Boolean value indicating whether some information is restricted (missing)
    because the Access Level does not permit it.
  - "Description" - a line of SIstack information - only if "Restricted" is 0.
- "Suspended" - a Boolean value indicating whether the thread is suspended.
- "State" - a string indicating the current location of the thread.
- "Flags" - e.g. Normal, Paused or Terminated.

If "Suspended" is 1 the object will also contain:

- "DMX" - an object containing elements of `⎕DMX` in that thread, either `null` or:
  - "Category", "DM", "EM", "EN", "ENX", "InternalLocation", "Vendor"
    "Message", "OSError" - only if "Restricted" is 0.
  - "Restricted": a Boolean value indicating whether some information is restricted (missing)
    because the Access Level does not permit it.
- "EXCEPTION" - an object containing elements of `⎕EXCEPTION` in that thread, either `null` or:
  - "Source", "StackTrace", "Message" - only if "Restricted" is 0.
  - "Restricted": a Boolean value indicating whether some information is restricted (missing)
    because the Access Level does not permit it.

#### "SuspendedThreads" fact

As ["Threads"](#threads-fact) with the exception that:

- Only suspended threads are enumerated.
- The "Suspended" item is omitted as it would always have the value 1.

#### "ThreadCount" fact

- "Total" - the total number of threads.
- "Suspended" - the number of threads which are suspended.

### PollFacts

`PollFacts` behaves in the same way as [`GetFacts`](#getfacts) except that it polls -
that is, the [`Facts`](#facts) message response will be sent immediately and then
repeat after specified or implied intervals until a new request is made.

The interval defaults to 1000ms but any value of 500ms or more may be
specified. Values less than 500 will be taken as 500.

Example:

```json
["PollFacts",{"Facts":[1,"Workspace"],"Interval":750}]
```

If a UID is specified it will appear in every message that it sent.

Messages will continue at the requested frequency until either a new
request is made (which will supersede any already established) or a
[`StopFacts`](#stopfacts) message is sent.

**Note:** polling messages may stop temporarily when the main interpreter
thread is inactive - that is, it is not running APL code or responding
to external input such as HMON requests or keyboard events.

### StopFacts

A `StopFacts` message will stop polling messages from being sent.

A UID should not be included in the message.

Example:

```json
["StopFacts",{}]
```

A [``Facts``](#facts) message will be sent back, with an empty list of facts and
Interval set to 0.

### BumpFacts

A `BumpFacts` message will cause a polling [`Facts`](#facts) message to be sent,
regardless of the time remaining until the next message is due. Messages
will then continue at the normal frequency.

A UID should not be included in the message.

Example:

```json
["BumpFacts",{}]
```

### Subscribe

The `Subscribe` message tells the interpreter to send notification
messages when certain, specifiable, events occur. The interpreter will
confirm the settings with a [`Subscribed`](#subscribed
) message in response, and a
[`Notification`](#notification) message whenever the subscribed events occur. If the
`Subscribe` message contains a UID then the [`Subscribed`](#subscribed) response and all
subsequent [`Notification`](#notification) messages of all types will echo that UID.

Each subscribable event has a numeric and alphanumeric (string)
identifier, and either may be used for the request. Example:

```json
["Subscribe",{"UID":"XX","Events":["WorkspaceCompaction",4]}]
```

No event notifications are enabled by default.

The following events may be subscribed to:

| Subscription ID | Subscription name   | Event                                                                             | Additional values provided in notification |
| --------------- | ------------------- | --------------------------------------------------------------------------------- | ------------------------------------------ |
| 1               | WorkspaceCompaction | A workspace compaction has occurred                                               | "Tid" (thread ID) and "Stack" (SIstack)    |
| 2               | WorkspaceResize     | The workspace size (the amount of memory committed by the OS) has grown or shrunk | "Size" (new size, in bytes)                |
| 3               | UntrappedSignal     | An APL exception has been signalled which has not been trapped                    | "Tid", "Stack", "DMX" and "Exception"      |
| 4               | TrappedSignal       | An APL exception has been signalled which has been trapped                        | "Tid", "Stack", "DMX" and "Exception"      |

Sending a `Subscribe` message resets the list of subscribed events to those specified - that is, it replaces any existing subscriptions. The list may be empty.

"Tid", "Stack", "DMX" and "Exeption" appear in the [`Notification`](#notification) reponse in
the same format as the ["Threads" fact](#threads-fact).

**Note:** following a WSFULL exception the interpreter may be unable to send
either a TrappedSignal or UntrappedSignal. [`GetLastKnownState`](#getlastknownstate) will
reliably report the last time a WSFULL event occurred.

## High-priority request messages

Responses to well-formed high-priority request messages are produced
immediately regardless of what interpreter is otherwise doing at the
time. To make this possible, the interpreter handles these incoming
messages on a separate thread, where it has no access to the interpreter
workspace and instead provides information from a repository which is continuously
maintained by the interpreter during its normal operation on the off-chance that it
might be asked for at any time. This introduces an overhead so is only
done if the Event Gathering Level is set to 1. Maintaining this repository is
independent of whether a Health Monitor is connected at the time or not.

[`112⌶`](#112) controls the Event Gathering Level.

In addition, `⎕PROFILE` must be started to provide full information,
e.g.:

`⎕PROFILE 'start' 'coverage'`

**Note:** High-priority requests are intended to be used when a
monitored interpreter becomes otherwise unresponsive. In "normal" use,
standard requests should be used.

### GetLastKnownState

Requests the last known state of the interpreter, and will be responded
to with a [`LastKnownState`](#lastknownstate) message.

Examples:

#### No UID provided:

```json
["GetLastKnownState",{}]
```

#### UID provided:

```json
["GetLastKnownState",{"UID":"123"}]
```

#### Bad syntax - will receive a (not necessarily immediate) [`InvalidSyntax`](#invalidsyntax) response:

```json
["GetLastKnownState"]
```

#### Good syntax, but not precisely conforming to specification - will receive a (not necessarily immediate) [`MalformedCommand`](#malformedcommand) response:

```json
["GetLastKnownState",{"id":"12345"}]
```

## Response messages

### InvalidSyntax

The response to a syntactically invalid JSON message, or a message which
does not strictly define a two-element array with a string name in the
first array element and an object in the second array element. Because
it is a response to an "unintelligible" message, it will never contain
a UID response.

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

The response to a syntactically correct message which contains a message
with an unrecognised name. The unrecognised name, and the UID if
provided, are included in the message.

Example:

```json
["UnknownCommand",{"UID":"ABC","Name":"Hatstand"}]
```

### MalformedCommand

The response to a syntactically correct high-priority request message
which does not exactly conform to specification. The name, and the UID
if provided, are included in the message.

Example:

```json
["MalformedCommand",{"Name":"GetLastKnownState"}]
```

### Facts

Reports one or more "facts" about the application state, corresponding
to a [`GetFacts`](#getfacts), [`PollFacts`](#pollfacts), [`StopFacts`](#stopfacts) or [`BumpFacts`](#bumpfacts) request.

Example:

```json
["Facts",{"UID":"xx","Interval":5000,"Facts":[{"ID":6,"Name":"ThreadCount","Value":{"Total":1,"Suspended":0}}]}]
```

The facts are returned as an array of objects in the same order as in
the request. The content of the "Value" or "Values" item
within each object depends on the fact type, and is described in the
[`GetFacts`](#getfacts) section.

"Interval" is only present in the response if polling.

### Subscribed

The response to [`Subscribe`](#subscribe). The message lists all subscription events and
whether they are enabled or disabled.

Example (showing an incomplete list of subscription events):

```json
["Subscribed",{"UID":"XX","Events":[{"ID":1,"Name":"WorkspaceCompaction","Value":1},{"ID":2,"Name":"WorkspaceResize","Value":0}]}
```

### LastKnownState

The (immediate) response to [`GetLastKnownState`](#getlastknownstate), containing:

- The UID, if provided in the request.
- The interpreter\'s current UTC clock setting, so that times elapsed
  since the other timings can be computed.
- The line currently being executed by the interpreter and the UTC
  time that this line started or resumed execution, if enabled with
  `112⌶` and `⎕PROFILE` is running (otherwise this information is
  omitted).
- An activity code and the UTC time that this activity started, if
  enabled with [`112⌶`](#112) (otherwise omitted).
- The time a trapped or untrapped WSFULL event last occurred, if
  enabled with [`112⌶`](#112) (otherwise omitted).

UTC times are in ISO format with millisecond precision, e.g.
20231231T235959.999 for the very last millisecond of 2023.

Activity codes are:

| Code | Meaning                                                         |
| ---- | --------------------------------------------------------------- |
| 1    | Anything not specifically listed below                          |
| 2    | Performing a workspace allocation                               |
| 3    | Performing a workspace compaction                               |
| 4    | Performing a workspace check                                    |
| 222  | Sleeping due to the use of [`222⌶`](#222) (an internal testing feature) |

Examples:

#### No UID provided, and [`112⌶0`](#112) set:

```json
["LastKnownState",{"TS":"20230111T144700.132Z"}]
```

#### UID provided, [`112⌶2 1`](#112) set and `⎕PROFILE` started:

```json
["LastKnownState",{"UID":"123","TS":"20230111T144700.132Z","Activity":{"Code":1,"TS":"20230111T144700.132Z"},"Location":{"Function":"#.f","Line":2,"TS":"20230111T144700.132Z"},"WS FULL":{"TS":"20230111T144620.723Z"}}]
```

Note: "Location" is updated by the interpreter whenever execution of a
line begins or resumes. If program execution stops for any reason (e.g.
exception or program termination) it will report the last executed line.
"location" does not report anything about inactive threads - full
thread/stack info is available with [`GetFacts`](#getfacts), so long as the interpreter
is responsive.

## Notification messages

### Notification

A `Notification` message indicates that a specified event, for which
notifications have been subscribed, has taken or is taking place. The
[`Subscribe`](#subscribe) message is used to select notified event types. The
`Notification` message always reports a single event and contains the
event ID and name, along with any specific detail pertaining to that
event type - the specific detail is listed in the table shown for the
[`Subscribe`](#subscribe) message.

Example:

```json
["Notification",{"UID":"XX","Event":{"ID":2,"Name":"WorkspaceResize"},"Size":894213}]
```

### UserMessage

Sent by the interpreter under user/application control using [`111⌶`](#111).

Example:

```json
["UserMessage",{"UID":"123","Messge":"Hello"}]
```

# Interpreter I-beam functions

## 110

`{R}←(110⌶) Y`

Specifies the interpreter "description", which will appear in [`Facts`](#host-fact) messages
sent to the Health Monitor.

Y is a character vector or scalar containing the free-form text.

The shy result is the value 1.

## 111

`{R}←{X} (111⌶) Y`

Will cause the interpreter to send a [`UserMessage`](#usermessage) notification message to
the client, if one is connected.

Y is a character vector or scalar containing the free-form message text.

X is an optional character vector or scalar containing the UID.

The shy result is the value 1.


## 112

`R←X (112⌶) Y`

Starts and stops the Health Monitor, specifies the Interface Configuration and
controls the Access and Event Gathering Levels. See the section
"Permitting connections and controlling access levels" for an
explanation of what these are and their permitted values.

Y is a 1 or 2-element numeric array consisting of:

- Settings for Access Level, _and optionally_
- Event Gathering.

X may be omitted, or scalar zero, or an empty character vector, or
character vector specifying the Interface Configuration.

### Starting the HMON comms layer

X should be a character vector containing either:

- The Interface Configuration, _or_
- Nothing (i.e. empty), in which case the previous Interface Configuration
  (if any) is reused.

Y should contain:

- The Access Level values of 1 or 2, _and optionally_
- The Event Gathering setting; defaults to 0 if not specified.

### Updating the Access Level and Event Gathering settings

The function should be called monadically or with 0 in X.

Y should contain:

- The Access Level values of 1 or 2, _and optionally_
- The Event Gathering setting, which defaults to 0 if not specified.

The settings can be changed whether or not a client is currently
attached and take effect immediately.

### Stopping the HMON comms layer

The function should be called monadically or with 0 in the left
argument.

Y should contain:

- The Access Level value 0.

### Errors and return values

No action will be taken and 0 will be returned if:

- A request is made to start the HMON comms layer when it is already
  started.
- A request is made to update the Access Level and Event Gathering
  settings, and the HMON comms layer is not started.
- A request is made to stop the HMON comms layer when it is already
  stopped.

Otherwise, the comms layer is stopped or started as requested and then:

- An error will be signalled if the operation fails.
- The shy value 1 will be returned if the operation succeeds.

## 222

`{R}←(222⌶) Y`

Y is an integer time value, in seconds.

The interpreter will sleep for the specified time. Unlike `⎕DL`, the
interpreter will not thread switch or respond to any events on its main
thread during this time. This function exists for internal testing of
High-priority request messages.
