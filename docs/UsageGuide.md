# HMON Usage Guide

This document is intended to provide guidance on the use of Dyalog's Health Monitor (HMON) protocol. The document makes reference to a demonstration application which uses the protocol to implement a monitor for multiple running APL applications; the code examples in this document are taken from that, the full source code can be found in the [APLSource folder](https://github.com/Dyalog/HMon/blob/main/APLSource.

This document does not go into detail about the protocol itself, instead we refer to the [official documentation for the protocol](https://github.com/Dyalog/HMon/blob/main/docs/Protocol.md). 

## Configuring the Interface

Configuring the HMON interface involves choosing the connection mode, access levels, event gathering Levels, and whether the Application should respond to high-priority request messages.

The connection modes, access and event gathering levels can be set by the application which wants to allow monitoring, using ``112⌶`` to initiate the connection.

```apl
⍝ AplSource/DemoApplication.aplf
[2] 'POLL:localhost:7000'(112⌶)2 1
```

See the [protocol document](https://github.com/Dyalog/HMon/blob/main/docs/Protocol.md) for a detailed description of the right argument. In this example, we enable full access to monitoring information (2) and also ask the interpreter to gather information that will support the ```GetLastKnownState``` request (1). Enabling this has a runtime cost, because the interpreter will keep track of status information and allow HMON to respond even in situations where the interpreter itself cannot respond because it is in the middle of a call to an external library.

Note that in the current implementation, if you want to use ```GetLastKnownState```, it is not sufficient to request state information using ``112⌶``, the application must also enable 'coverage' monitoring using ```⎕PROFILE```.

```apl
     ⎕PROFILE 'start' 'coverage'
```

The left argument has the same format and meaning as the [RIDE_INIT environment setting](https://help.dyalog.com/18.0/Content/UserGuide/Installation%20and%20Configuration/Configuration%20Parameters/RIDE_Init.htm). 

Note that the demonstration client application uses ``POLL``, while the protocol document suggests ``SERVE``. This is is because the easiest way to experiment with the protocol is to have a single interpreter act as the server which you can make a connection to. However, the demo application acts as a monitor for multiple APL interpreters. In this case it is more practical for the client to start a listener (in this case on port 7000) and have each interpreter that wants to be monitored attempt to connect to this port - retrying at regular intervals if the monitor is not present. This allows application processes to come and go - and even allows the monitor to be restarted, if it disappears and comes back each monitored APL process will reconnect to it within a short period of time.

Finally, note that in the current implementation, HMON will only generate timer-driven messages (for example, responses to ```POLLFacts```) when it is active. If no threads are running in the application, HMON will not respond to anything other than high priority requests. The workaround  in the demo client application is to have a thread looping with a delay function:

```apl
⍝ AplSource/DemoApplication.aplf
[4] ⎕FX 'LOOP delay;z' 'z←⎕DL delay' '→1'
[5] LOOP&1
```

## Connecting Client to the monitored Application(s)

A client of HMON must establish a Conga connection to the interpreter which is to be monitored, using using ```BlkText``` mode, as described in the [protocol documentation](https://github.com/Dyalog/HMon/blob/main/docs/Protocol.md). As previously mentioned, both sides can initiate the connection, depending on the requirements of the application. The sample client assumes that the goal is to monitor multiple APL interpreters from a single point.

In this section we describe the design choices taken in the demo client.

### Running the demo application and client

Note that the demonstration client requires Dyalog APL for Microsoft Windows (but the monitored applications can run on any platform).

To start the client, link the APLSource folder and run the function ``Demo`` with out any arguments. For example, if you checked the repository out to ```/tmp/hmon```:

```
      ]link.create # /tmp/hmon/AplSource
      Demo
```

To create some simple processes that you can monitor, run ``LaunchDemoProcess 0``, which starts a new APL process to run the function ```DemoApplication```. The right argument allows for a Ride connection when set to 1, to use this functionality you must set ```RIDE_PATH``` variable to the path of your RIDE executable.

Hopefully you can get an idea of what design choices have been made by playing a bit around with this Demo.

### Setting up the connection

Firstly we need to set up the protocol on the client side. The client copies the Conga class from the distributed "conga" workspace, creates an instance of the Conga protocol, and computes the magic number used for the ```BlkText``` protocol:

```apl
⍝ AplSource/Init.aplf
[16] :If 9≠⎕NC 'Conga'
[17]      'Conga'⎕CY'conga'
[18] :EndIf
[19] iConga←Conga.Init'HMONCLIENT'
[20] {}iConga.SetProp'' 'EventMode' 1
[21] magic←iConga.Magic 'HMON'
```

The Demo creates a listener that all monitored applications can connect to, and runs the monitor function on a new thread.

```apl
⍝ AplSource/Listen.aplf
[4] d←iConga.Srv '' host port 'BlkText' 32768 ('Magic' magic)
[5] :If 0≢⊃d
[6]     ('Unable to start listener on ',host,':',(⍕port),' - ',3⊃d) ⎕SIGNAL 11
[7] :EndIf
...
[11] monitortid←monitor&1
```

The ```monitor``` function handles new connections and each incoming block:

```apl
[ 0]monitor listen;z;rc;con;event;data;ns;i;cid;uid;type;tkn;fn;m;removed;done;cb;port;host;json
[ 1] ⍝ While there is anything in the queue, monitor Conga messages
[ 2] ⍝ and release functions waiting on ⎕TPUT
[ 3]
[ 4] listening←listen
[ 5] :While listening∨0≠≢queue
[ 6]     :If 0=⊃z←iConga.Wait'.' 5000
[ 7]         (rc con event data)←4↑z
[ 8]
[ 9]         :Select event
[10]         :Case 'Timeout' ⋄ :Continue
[11]         :Case 'Connect'
[12]             (host port)←(2⊃iConga.GetProp con'PeerAddr')[2 4]
[13]             cid←AddConnection con((-':'⍳⍨⌽host)↓host)port
[14]             Handshake&cid ⍝ Handshake must run on another thread
[15]
[16]         :Case 'Block'
[17]             :If (≢conns)<i←conns[;2]⍳⊂con
[18]                 Log'*** Message received on unexpected connection'
[19]                 Log'===> ',,⍕z
[20]                 :Continue
[21]             :Else
[22]                 cid←conns[i;1]
[23]             :EndIf
[24]
[25]             uid←0
[26]             :Trap 6 11 ⍝ Might be handshake and not JSON data, or JSON data with no UID
[27]                 json←'UTF-8' ⎕UCS ⎕UCS data
[28]                 data←0 ⎕JSON json
[29]                 Log 'GET ',(⍕cid),': ',json
[30]                 tkn←'uid'GetToken uid←2⊃2⊃⎕VFI data[2].UID
[31]             :Else
[32]                 tkn←'cid'GetToken cid
[33]             :EndTrap
[34]
[35]             :Hold 'hmon'
[36]                 cb←0
[37]                 :If (≢queue)<i←queue[;1 2]⍳cid uid
[38]                     Log'*** Unqueued message received'
[39]                     Log'===> ',,⍕z
[40]                     :Continue
[41]                 :ElseIf cb←(0≠≢fn←⊃queue[i;3])∧'Subscribed'≢⊃data ⍝ Callback fn (ignore for "Subscribe" call)
[42]                     (⍎fn)&data            ⍝ Run it in a separate thread
[43]                 :EndIf
[44]
[45]                 :If done←80≠⎕DR data      ⍝ If namespace, check polling state
[46]                     done←data[2].{6::0 ⋄ Interval=0}⍬  ⍝ Polling done
[47]                 :EndIf
[48]                 :If done∨0=≢fn            ⍝   ... or no callback fn
[49]                     queue←(i≠⍳≢queue)⌿queue
[50]                 :EndIf
[51]                 :If ~cb                   ⍝ No callback done, return result via TPUT
[52]                     data ⎕TPUT tkn
[53]                 :EndIf
[54]             :EndHold
[55]
[56]         :CaseList 'Closed' 'Error'
[57]             Log event,': ',con
[58]             :If con≡listener
[59]                 ∘∘∘
[60]             :EndIf
[61]
[62]             :Hold 'hmon'
[63]                  cid←(conns[;1],¯1)[conns[;2]⍳⊂con]
[64]                  removed←(m←cid=queue[;1])⌿queue
[65]                  :If cid≠¯1
[66]                     queue←(~m)⌿queue
[67]                     conns←(m←conns[;1]≠cid)⌿conns
[68]                     hmongrid.(Values CellTypes)←m∘⌿¨hmongrid.(Values CellTypes)
[69]                     Log'===> ',(⍕≢removed),' queue items removed'
[70]                 :EndIf
[71]             :EndHold
[72]             :For uid :In removed[;2]~0
[73]                 'CONNECTION CLOSED'⎕TPUT'uid'GetToken uid
[74]             :EndFor
[75]         :Else
[76]             ∘∘∘
[77]         :EndSelect
[78]
[79]     :Else
[80]         Log'*** Conga.Wait failed: ',⍕z
[81]         ∘∘∘
[82]     :EndIf
[83] :End
```
