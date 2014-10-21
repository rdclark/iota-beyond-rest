% Beyond REST: Building Reliable Distributed Applications for the Web
% Richard Clark
% October 22, 2014

# Introductions

## Who am I?

- Richard Clark
- Head of Global Training, [Kaazing](http://www.kaazing.com)
- Formerly: Apple, General Magic, HP, Verifone, etc.

## Say a little about yourself

- Who you are
- What's your background
- Any other programming experience?
- What's one thing you'd like from today?

## Why this session?

- "Here be dragons"
- Moving distributed Enterprise systems onto WebSockets
- Developing next-generation tools for IoT
- Sharing what we know (and what we don't) 
- Beginning a community discussion

<div class="notes">
Why are we doing this? We've been helping people build distributed systems across the web for years (using WebSockets.) Until recently, it's been a pretty specialized bunch of uses -- streaming financial data, sports betting, auctions: largely broadcast with QOS requirements, limited return interactions, and people consuming the data. 

The advent of IoT makes the stakes higher: larger demands on scaling, automation replacing people, possible peer-to-peer interactions. We believe our current experience (esp. in mobile) has value in designing for IoT as well.

But, to be clear: we're not the be-all and end-all experts in distributed computing. We're just pioneers who have gone farther down the trail and are here to report back what we've seen. So we want this to be a collaborative workshop where we look at these questions together and try some experiments.
</div>


# The web as a distributed system

## Characteristics of the web

- Largely unidirectional (pull)
- Largely request/response
- Hugely popular

## What makes the web work as a distributed system?

- GET is idempotent
- Existing infrastructure
- Uniform addressing via URLs
- Familiar mental model

## What makes the web unreliable?

- Interactions aren't idempotent
- Existing infrastructure
- No concept of transactions

## When the web fails

- Reloading as a recovery strategy
- The security model (or lack of same)
- When proxies attack
- Who gets priority

<div class="notes">
Talk about the problem of duplicated requests (e.g. in commerce), how people recover from errors (manual reloading), how this exacerbates traffic problems, etc. 

Also discuss the security model, cache-related challenges, lack of traffic prioritization, etc.
</div>

# Fallacies of distributed computing

Derived from [this paper](http://www.rgoarchitects.com/Files/fallacies.pdf

1.	The network is reliable
2.	Latency is zero
3.	Bandwidth is infinite
4.	The network is secure
5.	Topology doesn't change
6.	There is one administrator
7.	Transport cost is zero
8.	The network is homogeneous

<div class="notes">
With the example of the web in mind, let's look at the ways that distributed computing could go wrong.
</div>

## "The network is reliable"

Sources of trouble:

- Overloaded application servers
- DNS failures
- Breaks in peering and other routing errors
- Trouble at the datacenter
- Local transport issues
- Mobile clients

<div class="notes">
Let's think of all the ways a simple GET can go wrong. App servers can become overwhelmed, network links can be broken for multiple reasons, you might even have trouble getting on the network locally. Mobile clients present their own issues as connections can come and go when moving (or even standing still, thanks AT&T)

How might you detect these problems? How would you correct them? (Discussion) Possible answers: ping/pong, retrying at regular intervals, CDNs and other distributed caching, etc.

Another approach is to use a messaging fabric that supports full reliable messaging (though this doens't mean you can ignore network issues!)
</div>

## "Latency is zero"

- When is a break not a break?
- TCP/IP doesn't *always* deliver in order
- (Related) Race conditions between multiple data channels

<div class="notes">
- Enough latency and the other side might start timing out. This can wreak havoc in a distributed system (and often leads to cascading failures.)
- TCP/IP is designed for in-order delivery *in a single session*. But take the example of a mobile client on a train (perhaps reporting entry and exit of each block), if these are individual AJAX sends they could arrive out of order if latency is inconsistent and high enough.
- If you have multiple channels (e.g. a snapshot followed by a stream), there's a risk of a race condition where your snapshot can be outdated relative to the stream. (Discussion approaches to mitigating this.)

What are the possible mitigations? (Discussion) Reduce the number of calls, pass as much as possible per call, have long-lived sessions (e.g. websocket). Timestamps might help, if the data is coming from one source with a relatively consistent clock (i.e. not multiple sources as synchronization is never guaranteed.) A monotonically increasing identifier is an even better bet, especially if it can be used to detect gaps in the message sequence.
</div>

## "Bandwidth is infinite"

Might not seem like a problem, but...

- Congested networks (QED)
- Rural access & other low-bandwidth networks
- Saturating NICs, links, etc.

...

- Volume == latency

<div class="notes">
Bandwidth issues raise their head when you're on a network that's oversubscribed, low-bandwidth, or at its carrying capacity. 

This is one of my largest indictments of REST: the overhead can overwhelm the data being sent.
</div>

## Bandwidth == latency (sometimes)

![COMET benchmark](https://webtide.com/wp-content/uploads/2011/09/http.png)
From [CometD benchmark](https://webtide.com/cometd-2-4-0-websocket-benchmarks/)

<div class="notes">
Talk about latency under conditions of network saturation.
</div>

## (Aside) The same benchmark with WebSockets

![CometD Websocket benchmark](https://webtide.com/wp-content/uploads/2011/09/websocket.png)

## "The network is secure"

Classes of potential exploits:

- HTTP weaknesses (e.g. header injections)
- Denial of Service
- Man in the middle / interception / spoofing
- Weaknesses in device web stacks

<div class="notes">
Brainstorm classes of exploits. What's the likely lifetime of your devices, upgradability, risk from old devices in the field? (And what can you do when endpoints are compromised?)

Mitigations: Minimize number of connection handshakes, use of PKCS, continuous stateful connections (enable synchronized state machines on both ends), device retirement. 
</div>

## "Topology doesn't change"

- Machines move, local networks change, messaging paths change
- What do you do when a node becomes detached?

<div class="notes">
Less of an issue if you rely on DNS, but even local network issues can get you into trouble. How do you maintain connectivity when your protocols change and/or your endpoint names change (e.g. queues, topics.) 

Brainstorm issues and solutions. Possible answers: protocol negotiations, dual infrastructure, monitoring at the hub to identify detached nodes (and some mechanism for following up.)
</div>

## "There is one administrator"

- Do you depend on somebody's WiFi?
- Risks from OS updates (when not embedded)
- Power & other environmental issues

<div class="notes">
</div>

## "Transport cost is zero"

- Bandwidth is not infinite (redux)
- Trade-off around adding layers / protocols / tools

<div class="notes">
Talk again about when happens when you hit NIC saturation. Also, adding a new layer to a protocol often cmoes with overhead; it's an accountable cost. 

</div>

## "The network is homogeneous"

- We're not really falling for this one
- This is why open standards win (lock-in and different speeds of upgrading)

<div class="notes">
Added by James Gosling in 1988. Most architects don't fall into this trap, but beware the risks introduced by proprietary protocols and what happens when deployments don't happen across the whole network at once.
</div>

# Building more reliable distributed systems

## Idempotency

- "[A]n operation that will produce the same results if executed once or multiple times" [http://en.wikipedia.org/wiki/Idempotence#Computer_science_meaning](Wikipedia)

## Hub and spoke vs. peer to peer

- Techniques (and risks) in peer-to-peer
- Connectivity (where is the network likely to break?)
- Message forwarding via hubs

<div class="notes">
We're generally talking about a subset of distributed systems here (as opposed to a distributed database, say) where devices are forwarding results to a central location and receiving commands from there or another fixed hub. But let's talk about true peer to peer and its options, where the network is likely to be severed, and what architetectures could work.
</div>

## Asynchronous operations

- Request/response and head of line blocking

## Handling duplicate transmissions

- Monotonically increasing IDs
- *Not* timestamps!
- Multiple store and forward sources (e.g. Redis under partition)

<div class="notes">
Talk about the simple case of a client forwarding to one reader and holding the value until the eventual ack. Serial numbers as a reliable indicator (not timestamps, and explain why.) What happens when a network partition can mean a new master (e.g. Redis) and why the old master must reject writes as soon as it loses master status. (However, we also have the chance to present values back to the client for confirmation, but we always have a risk of data loss.)
</div>

# WebSockets as a transport mechanism

## Why WebSockets was created

- Inspiration: Oracle forms
- "TCP over the Web" and the HTML5 spec
- Why this is a good thing

# WebSocket overview

## Handshake

- HTTP GET w/ extra fields
- HTTP 101 response
- Optional: Extensions, Protocol

## Packet framing

- Minimal header
- Stream encryption to protect proxies

## Quick demo

- [websocket.org](http://www.websocket.org/echo.html)
- Chrome developer tools

## Protocols and extensions

- Uses for protocols
- Uses for extensions

## Securing the connection

- WS over TLS == WSS

## Programming with WebSockets

```
var ws = new WebSocket("wss://echo.websocket.org");

ws.onopen = function() { ... }

ws.onmessage = function(evt) { ... }

ws.onclose = function() { ... }

ws.onerror = function(err) { ... }

ws.send(data);
```

# Lab: Trivial Client-server

## Echoing off websocket.org

- Hosted demo: [http://www.websocket.org/echo.html](http://www.websocket.org/echo.html)
- Host your own: `python -m SimpleHTTPServer 9999`

## Try it in Node

- Using (ws for node)[http://einaros.github.io/ws/]
1. `npm install ws` 
2. `wscat -c ws://echo.websocket.org -p 13`

## Echoing off your local machine

1. `npm install ws` (if not done before)
2. `python -m SimpleHTTPServer 9999` 
3. `wscat --listen 9998`
4. Copy and modify web page

## Experiments

- Machine to machine (via `.local` addresses)
- Build a simple server in node (echo or chat)

# WebSockets for IoT

## Real-world deployment

![Proxy servers](img/proxy.jpg)

## Work-arounds and alternatives

- Long polling pros and cons
- HTTP streaming pros and cons
- UDP pros and cons
- Relationship to SPDY

## WebSockets and distributed systems

- Recommended architectures
- What you can do with WebSocket that you can't with HTTP
- Risks in practice

## WebSockets in embedded devices

- Requirements for device-level implementation
- Common software stacks: [libwebsiockets](http://libwebsockets.org/trac/libwebsockets), [Minnow Server (commercial)](https://realtimelogic.com/products/sharkssl/minnow-server/)
- Power consumption

## WebSockets, the TCP stack, and data loss

- Potential for losing data in transit
- TCP push-back

# Enterprise messaging

## The overall design of a messaging system

![Messaging system](img/messaging.jpg)

## Common use cases

- Financial (and related) broadcasting
- Extending internal processes

## JMS overview

- Basic building blocks: Queues, Topics, etc.
- STOMP as an unofficial transport layer for JMS

## AMQP overview
- AMQP terminology
- AMQP wireline format

## Enterprise messaging design patterns

- Broadcast
- Broadcast with commands
- Virtual links to specific nodes

## Implementing security in messaging systems

- Wireline security
- Authentication and authorization
- Propogating identity

## Techniques for higher-performance messaging

- Header exclusion
- Calculating message deltas

## Messaging systems behavior under degraded reliability

- Client-broker communication
- Inter-broker communication (e.g. RabbitMQ)

## Recovery strategies for messaging systems

- Restoring subscriptions
- Message retries

## Lab: Enterprise messaging from the client side

- TODO KAAZING Gateway or Node+STOMP

# Experimenting with degraded reliability

## Specific tools

- BPF (*nix general)
- Network link conditioner
- Network link conditioner on iOS
			(? for Android)

## Lab: Experimenting with degraded reliability

# Going deeper

## TODO add additional research

# Conclusion

- TODO write
