# Introduction to webrtc related terms

There are many concepts in webrtc that will cause understanding obstacles for beginners. If they do not have a general understanding of these concepts, even if these concepts are not important, beginners will still subconsciously think that this matter is difficult. Below I will briefly describe the following frequently touched concepts in vernacular.

# SDP

It is a negotiation protocol that includes whether a data channel needs to be established, whether a video stream needs to be established, whether an audio stream needs to be established, and the format specifications of these streams, such as whether the video encoding format uses H264 or VP8, and whether the audio encoding format uses OPUS, etc. .

The following is an SDP offer and corresponding answer that only establishes a data channel.

```txt
offer
v=0
o=- 8382788528297067006 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0
a=msid-semantic: WMS
m=application 9 UDP/DTLS/SCTP webrtc-datachannel
c=IN IP4 0.0.0.0
a=ice-ufrag:c+zu
a=ice-pwd:jvxc6prZIswRtLdfjJI1GJV4
a=ice-options:trickle
a=fingerprint:sha-256 13:5F:AB:21:80:9F:24:69:48:53:FC:39:5A:A4:5E:FB:31:4B:26:6F:8E:6A :36:01:8F:12:81:F3:60:D8:B9:B3
a=setup:actpass
a=mid:0
a=sctp-port:5000
a=max-message-size:262144

```

And answer:

```txt
v=0
o=- 2411559430796177653 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0
a=msid-semantic: WMS
m=application 9 UDP/DTLS/SCTP webrtc-datachannel
c=IN IP4 0.0.0.0
b=AS:30
a=ice-ufrag:X4yB
a=ice-pwd:ik/QrqW5BOQwML43qbq9/mOo
a=ice-options:trickle
a=fingerprint:sha-256 C9:0D:5D:72:1F:A1:37:D4:06:7C:2C:A0:26:D1:ED:C9:EE:34:0D:EE:2E:05 :D8:06:0B:43:66:0B:CC:2A:F0:12
a=setup:active
a=mid:0
a=sctp-port:5000
a=max-message-size:262144

```

There is no need for beginners to worry about what each field in SDP means and what its specific format is. There are two reasons: First, this is not the point. Second, there is no need, because this is a standard format, and you can read the RFC document specifically when needed.

You only need to pay attention to a few key fields, such as: a=mid:0, which means that this is the first item in the sdp. If there is audio in the SDP that needs to be negotiated, then the audio section is a=mid:1. If there is a video, the part of the video is a=mid:2.

For example: m=application 9 UDP/DTLS/SCTP webrtc-datachannel, it indicates that this is a data channel, using the user mode SCTP protocol, and underneath is the DTLS encrypted UDP protocol.

The active party first creates an SDP through the webrtc interface. This SDP is called OFFER. Then this SDP is transmitted to the passive party through some channels (public network forwarding server), and the passive party responds to the OFFER according to its actual situation. The SDP used to respond is called ANSWER.

In the video streaming chapter, I will continue with examples.

candidate
As the name suggests, candidate means candidate, which refers to an IP address that can be used to establish a network connection.

picture1

In the picture above, there are three PCs and two small subnet segments under a certain LAN. The public IP of this LAN is 23.23.2.1.

When webrtc is preparing to establish a connection, it will collect candidates from both parties. Let's take the connection establishment between PC1 and PC2 as an example, and assume that the socket ports created by PC1 and PC2 are both 8888.

PC1 and PC2 will collect their own candidates respectively. The LAN addresses collected first are 192.168.1.1:8888\10.133.3.1:8888 for PC1 and 192.168.1.2:8888 for PC2.

Every time a candidate is collected, PC1 will send the candidate to PC2 through some means (public network forwarding server). PC2 will also send its own cadidate to PC1.

After the LAN addresses are collected, the public network addresses will be collected. WebRTC does this through the stun protocol. To put it simply, PC1 sends a UDP message to the stun server on the public network, and then the stun server tells PC1 the public network IP and port number (assumed to be 9999) of the UDP message sent by PC1 seen on the stun server side. ). So PC1 obtained its own external network candidate:23.23.2.1:9999.

Then PC1 and PC2 will notify each other of the external network candidate.

In this example, PC1 and PC2 are in the same LAN, and PC1 gets PC2's LAN cadidate:192.168.1.2:8888. PC1 will try to communicate with this address of PC2 (send a message in the specified format):

192.168.1.1:8888 ------> 192.168.1.2:8888 yes!
10.133.3.1:8888 ------> 192.168.1.2:8888 no!
In the same way, when PC2 gets the LAN cadidate of PC1, it will also perform this action. When PC1 and PC2 are connected in both directions, the P2P connection is connected, which is called a "connection".

Every time it receives a candidate from the other party, webrtc will perform the connectivity test mentioned above. If a connectable "connection" appears later, webrtc will switch the "connection". The specific method is to sort through rtt. But no more introduction.
