---
layout: post
title:  "What is ICE, STUN and TURN"
date:   2020-04-18 22:18:41 +0530
categories: jekyll update
---
This post describes how two devices can initiate communication when they are
behind their respective NATs. Specifically, we'll see how a device traverses a NAT to reach out to the other device and establishes a connection to it.
We shall look at how ICE works in conjunction with the STUN and TURN protocols. NAT specifics are out of scope of this post.
   

Problem:
========
Two devices (computers, cellphones, raspberry-pi, etc.) want to communicate with each other and they both are behind respective NATs, hence, having 'private' IP addresses.
&nbsp;
For e.g., one device wants to start a video call with another device using WebRTC and both these devices are behind respective NATs and not directly discoverable to each other.


Solution:
=========
A combination, although not necessary, of the ICE, STUN and TURN protocols.
I say not necessary because you can decide not to use either the STUN or TURN
protocol depending on the network topology.

STUN (Session Traversal Utilities for NAT):
===========================================
STUN, a client-server protocol, is used as a tool in aiding NAT traversal.
Using STUN, a device sitting behind a NAT can figure out what its NAT assigned IP address and port number is. STUN can also be used by a device to check if other device it wants to talk to is reachable or not.

<!-- language: lang-none -->

                               +------+
                               |STUN  |
                               |Server| 
                               +------+
		          STUN server address

		       server-reflexive address
                          +--------------+             Public Internet
          ................|     NAT 2    |.......................
                          +--------------+

		       server-reflexive address
                          +--------------+             Private NET 2
          ................|     NAT 1    |.......................
                          +--------------+

			    host ip address
                              +-------+
                              | STUN  |
                              | Client|
                              +-------+               Private NET 1


                 Figure: One Possible STUN Configuration

How it works?
-------------

A STUN client first sends a request, called a binding request, to the STUN server who's address is known before-hand.  As the request travels through the NATs (NAT 1 and NAT 2 in the above figure), they modify the source IP address and port of the packet. Therefore, when the packet reaches the STUN server, it will see the source IP address and port of NAT 2. This modified source IP address and port is called as the server-reflexive address.   
&nbsp;
The STUN server will then send a response with the server-reflexive address back to the STUN client. As the response travels through the NATs, they modify the destination IP address and port in the IP header of the packet but leave the payload containing the server-reflexive address untouched. When the packet finally reaches the STUN client, it can extract the server-reflexive address allocated by the outermost NAT. It can then share this server-reflexive address with the other endpoint it wants to talk to via signaling (more on this in the ICE section below).
&nbsp;
While STUN is sufficient for use in almost all kinds of NAT settings, it doesn't work in the Symmetric NAT setting. In short, the problem is that if a client sitting behind a Symmetric NAT asks two different STUN servers what its server-reflexive address is, they will respond with the same IP address but different ports.
&nbsp;
This means that every connection a STUN client makes to network addresses outside the symmetric NAT, a different port gets assigned to it.  Therefore, the other client cannot send data to the server-reflexive address of the NAT router.
&nbsp;
<!-- language: lang-none -->

        +------+          +------+
        |STUN  |          |STUN  |
        |Server|          |Server|
        +------+          +------+

       x.x.x.x:1000   x.x.x.x:2000
                +------+
                |Symm. |
                |NAT   |
                +------+

                +------+
                |STUN  |
                |Client|
                +------+

          Figure: A Symmetric NAT

&nbsp;
This is where TURN can solve the problem.

TURN (Traversal Using Relay through NAT):
=========================================

When a direct connection using the STUN provided server-reflexive address can't be made, TURN can be used.  This is an extension to the STUN protocol (TURN understands all STUN requests).  The implementor of the server side TURN protocol can act as a relay server.  What this means is that data passing between two devices is getting relayed through the TURN server. 

<!-- language: lang-none -->

						Peer A
						Server-Reflexive    +---------+
						Transport Address   |         |
						192.0.2.150:32102   |         |
						    |              /|         |
				  TURN              |            / ^|  Peer A |
	    Client's              Server            |           /  ||         |
	    Host Transport        Transport         |         //   ||         |
	    Address               Address           |       //     |+---------+
	   10.1.1.2:49721       192.0.2.15:3478     |+-+  //     Peer A
		    |               |               ||N| /       Host Transport
		    |   +-+         |               ||A|/        Address
		    |   | |         |               v|T|     192.168.100.2:49582
		    |   | |         |               /+-+
	 +---------+|   | |         |+---------+   /              +---------+
	 |         ||   |N|         ||         | //               |         |
	 | TURN    |v   | |         v| TURN    |/                 |         |
	 | Client  |----|A|----------| Server  |------------------|  Peer B |
	 |         |    | |^         |         |^                ^|         |
	 |         |    |T||         |         ||                ||         |
	 +---------+    | ||         +---------+|                |+---------+
			| ||                    |                |
			| ||                    |                |
			+-+|                    |                |
			   |                    |                |
			   |                    |                |
		     Client's                   |            Peer B
		     Server-Reflexive    Relayed             Transport
		     Transport Address   Transport Address   Address
		     192.0.2.1:7000      192.0.2.15:50000     192.0.2.210:49191

			 Figure: A TURN setup

How it works?
-------------

Let's find out what's happening in the above figure.

A TURN client first sends an 'allocation' request to the TURN server which then creates a relay address (192.0.2.15:50000) for it. The TURN server will see the packets coming from the client's server-reflexive address because it is behind a NAT.  So, it knows that when a packet arrives at the relay address, where exactly it should forward it to (192.0.2.1:7000).
&nbsp;
If this client wants to send data to Peer A, it will use Peer A's server-reflexive address (192.0.2.150:32102). 
&nbsp;
TURN client's relay address and Peer A's server-reflexive address were exchanged using signaling (more on the specifics in the ICE section below). 
&nbsp;
Peer A will use the relay address (192.0.2.15:50000) to send data to the TURN client.
&nbsp;
A TURN server, in the symmetric NAT scenario will have a different public port assigned by the NAT server. But that's ok since data from a peer will be routed through the TURN server. 
&nbsp;
Caution: Since data is relayed through the TURN server and there can be multiple connections passing through it, a lot of computing power is needed.  See if you can avoid it and use it only as a fallback option.
&nbsp;
Alright! now let's a take a look at a protocol which coordinates with the TURN and STUN protocols and establishes connection between two devices.

ICE (Interactive Connectivity Establishment):
=============================================
A protocol for NAT traversal for both UDP and TCP transport modes.

How it works?
-------------
Two devices (called ICE Agent L and ICE Agent R in the below figure) want to communicate
with each other. They perform three steps:
a. Gather candidates
b. Exchange the candidates via signaling (like SIP) and test connectivity
c. Nominate candidate pairs

                               +---------+
             +--------+        |Signaling|         +--------+
             | STUN   |        |Server   |         | STUN   |
             | Server |        +---------+         | Server |
             +--------+       /           \        +--------+
                             /             \
                            /               \
                           / <- Signaling -> \
                          /                   \
                   +--------+               +--------+
                   |  NAT   |               |  NAT   |
                   +--------+               +--------+
                      /                             \
                     /                               \
                 +-------+                       +-------+
                 | Agent |                       | Agent |
                 |   L   |                       |   R   |
                 +-------+                       +-------+

                     Figure: ICE Deployment Scenario

A candidate, or more specifically, a candidate transport address, is a a combination of IP address and port for a particular transport protocol (TCP/UDP).
&nbsp;
Note:  We don't need ICE to NAT traverse to the Signaling server. Think about configuring a SIP client - a SIP proxy server IP/domain always has to be entered before registering. This means requests going to the SIP proxy server can traverse NAT naturally.
&nbsp;
A candidate can be one of the three types:
a. local ip address (associated with the network interface - ethernet/wifi)
b. server-reflexive address (read the STUN section above)
c. relay address (read the TURN section above)

What are my addresses?
----------------------
In the candidates gathering process, L first finds out what its set of candidates is. 
If it uses only STUN, then it gets only the server-reflexive address. 
If it uses only TURN, both server-reflexive and relay addresses are fetched.

What are your addresses?
------------------------
In the exchaning step, after gathering all the candidates, L then orders them by highest-to-lowest priority and sends them to R via the signaling server.

R then performs the same candidates gathering steps and shares the candidates list with L.

Can we connect?
---------------
Each agent now has its own list of candidates as well as the other agent's list of candidates. Next, it pairs up these candidates and performs connectivity checks.
&nbsp;
A connectivity check does the following things -
a. Sort the candidate pairs according to their priority.
b. Check connectivity on each candidate pair.
c. Acknowledge connectivity checks from the other agent.
&nbsp;
A connectivity check is a 4-way handshake:

                  L                        R
                  -                        -
                  STUN request ->             \  L's
                            <- STUN response  /  check

                             <- STUN request  \  R's
                  STUN response ->            /  check

                    Figure: Basic Connectivity Check

&nbsp;
Hang on, why are we sending a STUN request? Because, if the remote candidate we are checking connectivity for is a relay address instead of a server-reflex address, then a TURN server should respond to the STUN Binding request since it's an extension of the STUN protocol and thus, supports the Binding request.
&nbsp;
We still aren't done yet. Just one last step.
&nbsp;
Since there can be multiple data streams that can be exchanged by the agents like audio, video, text during a WebRTC meeting, we need to identify which candidate pair will handle which data stream.  

Ok, let's associate candidate pairs with data streams.
------------------------------------------------------
Who decides which pair deals with which data stream though? ICE decides which agent acts as a controlling agent and which agent acts as a controlled agent. Obviously, the controlling agent gets to decide which candidate pair should be used for which data stream. 
&nbsp;
This decision making happens when the connectivity checks are in progress. When the controlling agent finds that a connectivity check for a candidate pair is successful, it then notifies the controlled of the nomination through a STUN request with the USE-CANDIDATE attribute:

             L                        R
             -                        -
             STUN request ->             \  L's
                       <- STUN response  /  check

                        <- STUN request  \  R's
             STUN response ->            /  check

             STUN request + attribute -> \  L's
                       <- STUN response  /  check

                           Figure: Nomination

The controlled agent doens't send the USE-CANDIDATE attribute in it's binding
request.

After this, data should start flowing between the agents on the nominated
pairs.

References:
===========
ICE-UDP https://tools.ietf.org/html/rfc8445
ICE-TCP https://tools.ietf.org/html/rfc6544
TURN https://tools.ietf.org/html/rfc5766
STUN https://tools.ietf.org/html/rfc5389
https://www.frozenmountain.com/developers/blog/webrtc-nat-traversal-methods-a-case-for-embedded-turn
http://help.estos.com/help/en-US/procall/5.1/erestunservice/dokumentation/htm/IDD_FUNCTIONALITY.htm
