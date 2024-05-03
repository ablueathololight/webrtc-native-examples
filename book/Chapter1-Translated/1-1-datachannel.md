# webrtc data channel

In addition to transmitting audio and video in webrtc, some ordinary data, such as binary protocols, can also be transmitted through "datachannle".

In addition to introducing datachannle, this chapter focuses on describing the process of webrtc "establishing a connection" and describing it through pseudo code.

For the convenience of description, here we assume that two users A and B want to use webrtc to interconnect, and A actively initiates a request to establish a connection. There is no video and audio stream, only a data channel is established.

## Signaling Server

webrtc is point-to-point communication, designed to establish communication between two webrtc clients located under different networks. Before the p2p connection is established, A does not know anything about B.

webrtc needs to use a third-party service that can be accessed by both AB and AB to help AB exchange information. This third-party service is called a signaling server.

webrtc does not make any specifications for the signaling server, and users can implement it freely. However, signaling servers usually use the websocket protocol to establish a connection with the browser (javascript).

The structure is like this:

![picture1](../materials/pictures/1-1-framework.png)

## PeerConnection

A peerconnection can be created as follows:

```c++
auto factory = new PeerConnectionFacotry(params);
auto peer_connection = factory->create_peer_connection(params);
```

After all, webrtc is a very large modern C++ project, and it is necessary to use some kind of design pattern to manage the source code. The factory pattern runs through the entire webrtc source code.

It can be said that among all the classes and interfaces provided by webrtc to users, PeerConnection is the most critical class.

Use PeerConnection to create a DataChannel and register an Observer for this DataChannel:

```c++
auto data_channel = peer_connection.CreateDataChannel("data"); //line 1
auto observer = new Observer();
data_channel.RegisterObserver(observer); //line 2
peer_conection.RegisterObserver(observer);

class Observer {
public:
     void OnSuccess(SessionDescriptionInterface* sdp) {
         peer_connection.SetLocalDescription(observer, sdp); //line3
         signal_server.send_to("B", sdp); //Inform the peer of the local sdp through the signaling server
     }
     void OnStateChange() {
         if(data_channel.state() == DataState::kOpen) {
             DataBuffer buf("hello,I'm A!");
             data_channel->send(buf);
         }
     }
}
```

When we execute line1, peerconnection knows that a datachannel needs to be created, and when creating the Offer, writes "datachannel" in the offer.

```C++
peer_connection.CreateOffer(observer, options); //Write "datachannel" in offer
```

After webrtc creates the offer, Observer.OnSuccess(sdp) will be called back.

In this callback, the user executes SetLocalDescription(sdp), officially uses this sdp as "local description", and sends this sdp through the signaling server
to the peer.


Here we ignore the signaling server forwarding process. Then for remote B, a peerconnection also needs to be created. After receiving the SDP from A, use this SDP as
"Remote Description", and respond to A with an Answer.

The pseudo code is as follows:

```c++
auto factory = new PeerConnectionFacotry(params);
auto peer_connection = factory->create_peer_connection(params);

void OnRemoteSdp(webrtc::SessionDescriptionInterface* sdp) { #line1
     peer_connection.SetRemoteDescription(observer, sdp); #line2
     peer_connection.CreateAnswer(observer, options); #line3
}

class Observer {
public:
     void OnSuccess(SessionDescriptionInterface* sdp) {
         peer_connection.SetLocalDescription(observer, sdp);
         signal_server.send_to("A", sdp); //Inform the peer of the local sdp through the signaling server
     }
     //Callback after the data channel of A is opened
     void OnDataChannel(rtc::scoped_refptr<DataChannelInterface> data_channel) {
         data_channel_ = data_channel;
         data_channel_->RegisterObserver(this);
     }
     void OnStateChange() {
         if(data_channel_->state() == DataState::kOpen) {
             DataBuffer buf("hello,I'm B!");
             data_channel->send(buf);
         }
     }
private:
     rtc::scoped_refptr<DataChannelInterface> data_channel_ = nullptr;
}
```

Assume that B receives a message from the signaling server (offer from A) and calls line1. Create the corresponding Answer through line3. Similarly, after the sdp (answer) is created,
webrtc will call back observer.OnSuccess(sdp).

In this callback, this sdp is officially used as the "local description" of B, and this sdp is forwarded to A through the signaling server.

When A receives the message from the signaling server, it sends B to its own sdp as "remote description".

```C++
void OnRemoteSdp(webrtc::SessionDescriptionInterface* sdp) { #line1
     peer_connection.SetRemoteDescription(observer, sdp); #line2
}

```

So far, the offer\answer interaction has been completed. Next, we introduce candidates.

When the application layer calls SetXXXDescription(), webrtc begins to collect "**candidate addresses**".

### candidate (candidate address)

Host A under the LAN has two local network devices: Ethernet and 802.11 (wifi). The IP addresses are 192.168.1.2\10.133.1.2 respectively, and its public network address is 200.100.50.1. In theory, host A has three udp candidates, which are the above three addresses. Print a candidate as a string, which looks like this:

``` shell
"candidate":"candidate:1779684516 1 udp 2122260223 192.168.29.185 56370 typ host
generation 0 ufrag 7XGL network-id 1","sdpClientType":"cpp",
"sdpMLineIndex":0,"sdpMid":"0","type":"candidates"
```

type=host indicates that this is a local candidate, and the ip is 192.168.29.185.

Every time a candidate is collected, webrtc will call back OnIceCandidate. At this time, the application layer needs to send this candidate to the peer:

```c++
class Observer {
public:
     void OnIceCandidate(const webrtc::IceCandidateInterface* candidate) {
         signal_server.send_to("B", candidate); //Inform the peer of our candidate through the signaling server
     }
```

When one end receives a candidate from the other end, it needs to tell webrtc about the candidate:

```c++
auto peer_connetion = webrtc::PeerConnectionInterface();
class B {
     void on_recv_candidate(const webrtc::IceCandidateInterface* candidate) {
         peer_connection.AddIceCandidate(candidate);
     }
}
```

Immediately afterwards, webrtc will match the local candidate and the remote candidate to create a virtual connection (**connection**).

### connection

Host A under a certain LAN has only one network device, the local IP is 192.168.1.2, and its external network address is 123.1.1.2.

Host B in another LAN has only one network device, the local IP is 10.133.1.2, and its external network address is 200.1.1.2.

When B's two candidates (local and external) are sent to A, A will create the following **connection**:

* 192.168.1.1 <-> 10.133.1.2
* 192.168.1.1 <-> 200.1.1.2

**connection** is a virtual "connection" that serves as a possible candidate for connectivity.

webrtc will perform connectivity testing on all connections, similar to "ping". Being able to "ping" generally indicates that the network is connected, and sorts all "connections" to select the best connection (the rtt is the shortest) and use it Final streaming.

## Datachannle callback

After this, the dtls handshake will be carried out, and then the user-mode sctp protocol handshake will be carried out, and the data channel will be established.

For the party (A) who actively creates the datachannel, webrtc will notify the user through the OnStateChange(state) callback.

For B, webrtc will tell the user through the OnDataChannel(channle) callback.

## C++ examples that can be compiled and run

In src/datachannel/main.cpp, I created 2 peerconnections as active and passive parties.

The main purpose of the example is to show the interaction process of webrtc, so the signaling server forwarding action is ignored and the other party directly uses sdp and candidate:

```C++
     virtual void OnSuccess(webrtc::SessionDescriptionInterface* desc) override {
         peer_connection_->SetLocalDescription(DummySetSessionDescriptionObserver::Create(), desc);
        
         //Need to convert sdp into a string and forwarded by the signaling server
         std::string sdp_str;
         desc->ToString(&sdp_str);
         /* sending sdp to remote...
          *----------------------------------------> 1 ms
          *----------------------------------------> 2 ms
          * ....
          *----------------------------------------> 1/2 rtt
          */
         //The receiving end needs to re-convert the string to sdp
         auto sdp_cp = webrtc::CreateSessionDescription(desc->GetType(), sdp_str, nullptr);
         other_->received_sdp(sdp_cp.release());
     }
```

Although A and B are in the same process, they are still two normal independent webrtc clients. I hope this will not cause confusion to readers.

The above is a normal webrtc connection process.

**Complete source code**

https://github.com/MemeTao/webrtc-native-samples/blob/master/src/datachannel/main.cpp