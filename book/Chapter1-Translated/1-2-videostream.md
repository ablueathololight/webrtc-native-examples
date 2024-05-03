# webrtc video streaming

This chapter introduces how to use webrtc to send and receive video streams.

Similarly, two peerconnections are still used in the same process to send and receive video streams.

In order not to involve operations such as "camera acquisition", and in order to allow the sample code to run on all platforms, I manually constructed the RGB data of red, orange, yellow, green, cyan, blue, and purple as each frame of the video stream.

**Note: It is actually video data in I420 format. In order not to cause more understanding obstacles for novices, it is collectively referred to as RGB data in the following**

## Video Codec Factory

As mentioned in the previous section, creating a PeerConnection requires corresponding peerconnectionfactory (factory for short).

The factory is created through the webrtc::CreatePeerconnctionFactory(params) function, which has many parameters. Here are its 7th and 8th parameters, which are VideoEncoderFactory and
VideoDecoderFactory gives a brief introduction.

When we create a video stream on peerconnection, webrtc will query the factory for the video encoder types it supports and write the supported encoders to sdp.

And when webrtc sends RGB data, webrtc uses the encoder inside the factory for encoding.

When webrtc receives the "encoded" video frame sent from the remote end, webrtc uses the decoder inside the factory to decode it.

VideoEncoderFactory is an interface class specified by webrtc. Users can customize the implementation or use the default implementation provided by webrtc.

```C++
//Omit some non-key functions
class VideoEncoderFactory {
   // Returns a list of supported video formats in order of preference, to use
   // for signaling etc.
   virtual std::vector<SdpVideoFormat> GetSupportedFormats() const = 0; //Return supported encoder type (SDP format)

   // Creates a VideoEncoder for the specified format.
   virtual std::unique_ptr<VideoEncoder> CreateVideoEncoder( //Returns a video encoder of the specified type (after negotiation, supported by both parties)
       const SdpVideoFormat& format) = 0;
};
```

For those who are new to webrtc, it is enough to know this. Later I will introduce how to implement custom encoders and decoders (hardware acceleration).

## Add video stream to PeerConnection

Use the following code to add a "video source" to the peerconnection.

```C++

video_track = factory->CreateVideoTrack("video", video_souce);
peerconnection->AddTrack(video_track);
```

This video source is an abstract class, and users must implement the specified virtual functions.

```C++
template <typename VideoFrameT>
class VideoSourceInterface {
  public:
   virtual ~VideoSourceInterface() = default;

   virtual void AddOrUpdateSink(VideoSinkInterface<VideoFrameT>* sink,
                                const VideoSinkWants& wants) = 0;
   // RemoveSink must guarantee that at the time the method returns,
   // there is no current and no future calls to VideoSinkInterface::OnFrame.
   virtual void RemoveSink(VideoSinkInterface<VideoFrameT>* sink) = 0;
};
```

addxxxSink actually sets a key callback object, VideoSinkInterface:

```C++
template <typename VideoFrameT>
class VideoSinkInterface {
  public:
   virtual ~VideoSinkInterface() = default;

   virtual void OnFrame(const VideoFrameT& frame) = 0;

   // Should be called by the source when it discards the frame due to rate
   //limiting.
   virtual void OnDiscardedFrame() {}
};
```

When we generate a frame of video data, as long as we set the recent sink and use VideoSinkInterface::OnFrame(frame) on it to notify webrtc, webrtc will encode and transmit it.

The pseudo code is as follows:

```C++

class VideoSourceMock : public rtc::VideoSourceInterface<webrtc::VideoFrame> {
public:
     void start(frame)
     {
         while(true) {
             std::this_thread::sleep_for(10ms);
             auto video_frame = get_frame(); //Assume get_frame() can return a frame of video image
             broadcaster_.OnFrame(video_frame);//Submit to webrtc
         }
     }
private:
     void AddOrUpdateSink(rtc::VideoSinkInterface<webrtc::VideoFrame>* sink,
                                  const rtc::VideoSinkWants& wants) override {
         broadcaster_.AddOrUpdateSink(sink, wants);
         (void) video_adapter_; //we willn't use adapter at this demo
     }
     void RemoveSink(rtc::VideoSinkInterface<webrtc::VideoFrame>* sink) override {
         broadcaster_.RemoveSink(sink);
         (void) video_adapter_; //we willn't use adapter at this demo
     }
private:
     rtc::VideoBroadcaster broadcaster_;
     cricket::VideoAdapter video_adapter_;
};
```

On the sending end, set a video track to the peerconnection and add a video source as above on the video track. Then the "video frame" can be delivered to webrtc, and webrtc will automatically complete the remaining encoding, transmission and other work.

## Changes to sdp

When we add a video stream to a peerconnection, when creating offer, there will be one more item in sdp:

```txt

v=0
o=-5204290053496113649 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0
a=msid-semantic: WMS stream1
m=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 100 101 127 120 125 119 124 107 108 109 123 118 122 117 114
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:qE9e
a=ice-pwd:vrZjsfbZ8njxNamShhas60+L
a=ice-options:trickle
a=fingerprint:sha-256 59:BC:FB:DA:29:42:CB:FF:99:BD:BC:92:C1:3B:2A:D1:EC:6E:1A:2F:17:CB :87:56:B7:9B:A4:81:54:59:66:31
a=setup:actpass
a=mid:0
### The following are some fields related to video formats
a=extmap:1 urn:ietf:params:rtp-hdrext:toffset
a=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:3 urn:3gpp:video-orientation
a=extmap:4 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:5 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay
a=extmap:6 http://www.webrtc.org/experiments/rtp-hdrext/video-content-type
a=extmap:7 http://www.webrtc.org/experiments/rtp-hdrext/video-timing
a=extmap:8 http://www.webrtc.org/experiments/rtp-hdrext/color-space
a=extmap:9 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=sendrecv
a=msid:stream1 video
a=rtcp-mux
a=rtcp-rsize
### vp8
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 goog-remb
a=rtcp-fb:96 transport-cc
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96
### vp9
a=rtpmap:98 VP9/90000
a=rtcp-fb:98 goog-remb
a=rtcp-fb:98 transport-cc
a=rtcp-fb:98 ccm fir
a=rtcp-fb:98 nack
a=rtcp-fb:98 nack pli
a=fmtp:98 profile-id=0
a=rtpmap:99 rtx/90000
a=fmtp:99 apt=98
a=rtpmap:100 VP9/90000
a=rtcp-fb:100 goog-remb
a=rtcp-fb:100 transport-cc
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=fmtp:100 profile-id=2
a=rtpmap:101 rtx/90000
a=fmtp:101 apt=100
### h264
a=rtpmap:127 H264/90000
a=rtcp-fb:127 goog-remb
a=rtcp-fb:127 transport-cc
a=rtcp-fb:127 ccm fir
a=rtcp-fb:127 nack
a=rtcp-fb:127 nack pli
a=fmtp:127 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42001f
a=rtpmap:120 rtx/90000
a=fmtp:120 apt=127
a=rtpmap:125 H264/90000
a=rtcp-fb:125 goog-remb
a=rtcp-fb:125 transport-cc
a=rtcp-fb:125 ccm fir
a=rtcp-fb:125 nack
a=rtcp-fb:125 nack pli
a=fmtp:125 level-asymmetry-allowed=1;packetization-mode=0;profile-level-id=42001f
a=rtpmap:119 rtx/90000
a=fmtp:119 apt=125
a=rtpmap:124 H264/90000
a=rtcp-fb:124 goog-remb
a=rtcp-fb:124 transport-cc
a=rtcp-fb:124 ccm fir
a=rtcp-fb:124 nack
a=rtcp-fb:124 nack pli
a=fmtp:124 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f
a=rtpmap:107 rtx/90000
a=fmtp:107 apt=124
a=rtpmap:108 H264/90000
a=rtcp-fb:108 goog-remb
a=rtcp-fb:108 transport-cc
a=rtcp-fb:108 ccm fir
a=rtcp-fb:108 nack
a=rtcp-fb:108 nack pli
a=fmtp:108 level-asymmetry-allowed=1;packetization-mode=0;profile-level-id=42e01f
a=rtpmap:109 rtx/90000
a=fmtp:109 apt=108
a=rtpmap:123 AV1X/90000
a=rtcp-fb:123 goog-remb
a=rtcp-fb:123 transport-cc
a=rtcp-fb:123 ccm fir
a=rtcp-fb:123 nack
a=rtcp-fb:123 nack pli
a=rtpmap:118 rtx/90000
a=fmtp:118 apt=123
a=rtpmap:122 red/90000
a=rtpmap:117 rtx/90000
a=fmtp:117 apt=122
a=rtpmap:114 ulpfec/90000
a=ssrc-group:FID 3124827794 1259095527
a=ssrc:3124827794 cname:DDQzIip6txmbJUKJ
a=ssrc:3124827794 msid:stream1 video
a=ssrc:3124827794 mslabel:stream1
a=ssrc:3124827794 label:video
a=ssrc:1259095527 cname:DDQzIip6txmbJUKJ
a=ssrc:1259095527 msid:stream1 video
a=ssrc:1259095527 mslabel:stream1
a=ssrc:1259095527 label:video
```

Likewise, we still selectively ignore many fields. Only focus on the field "a=rtpmap:96 vp8/90000", as well as vp9 and h264.

It indicates that the sender supports vp8, vp9 and h264 encoding (h265 is not supported due to technical constraints). These are the three encoding methods supported by webrtc by default.

When the receiving end receives this offer, it needs to give the sender an answer based on its own decoding capabilities. The answer will include which of the three encoding formats the receiving end can support.

If all three are supported, the sender will choose the format listed first in the answer to create the encoder.

In this example, the sender and receiver use the same webrtc code, and all three formats are supported, so the vp8 encoder is created, and the "answer" is not listed.

## Transmit video stream

In the previous section of DataChannel, I introduced the interaction process of sdp and candidate in detail. When these processes are successfully executed, the channel will naturally be established.

At this time, webrtc will automatically help us open the path between the encoder and "video sink". The video frames delivered in "broadcaster_.onframe()" will reach the encoder, and then be transferred down layer by layer, and finally divided Fragment into multiple UDP (maybe TCP) messages in RTP format and send them to the network.

How this process is specifically performed will be introduced in subsequent chapters. Source code analysis is always boring. If you just use webrtc, there is no need to read it.

## Receive video frames

Return to the sdp interaction process. When the receiving end confirms the sender's offer, webrtc will tell the user through a callback: the sender has created a video stream.

```C++
     void OnAddStream(rtc::scoped_refptr<webrtc::MediaStreamInterface> stream) override{
         std::cout<<"[info] on add stream, id:"<<stream->id()<<std::endl;
     }
     //The key is this function
     void OnAddTrack(rtc::scoped_refptr<webrtc::RtpReceiverInterface> receiver,
             const std::vector<rtc::scoped_refptr<webrtc::MediaStreamInterface>>& streams) override {
         auto track = receiver->track().get();
         if(track->kind() == "video" && video_receiver_) {
             auto cast_track = static_cast<webrtc::VideoTrackInterface*>(track);
             cast_track->AddOrUpdateSink(video_receiver_.get(), rtc::VideoSinkWants());
         }
     }
```

In the "OnAddTrack" function, webrtc passes a receiver object, on which we can get a "track (corresponding to the sender)".

If we want to receive this video stream, then we need to add a "sink" (video_receiver_) to the "track". This "sink" needs to be implemented by itself according to the interface specified by webrtc:

```C++
class VideoStreamReceiver : public rtc::VideoSinkInterface<webrtc::VideoFrame> {
public:
     void OnFrame(const webrtc::VideoFrame& frame) override {
         //You can get binary data (RGB) from the frame object, just check this data structure
     }
};
```

webrtc will notify the user of every frame received through this interface.

If we write the data in the example to a file and open it with the corresponding video software, you can see "red, orange, yellow, green, blue, purple" played at 30 frames.

This example does not show how to render.

## end

Source code: <https://github.com/MemeTao/webrtc-native-samples/blob/master/src/video-channel>

In this example, in order to allow novices to better understand the video stream interaction process of webrtc:

* There is no screen capture, screen recording, rendering and other processes
* webrtc stipulates that the input video data format is I420 format, which I described as RGB. Regarding the format conversion between RGB and I420, there is also corresponding code in this example. Readers need to read the relevant information to understand this code.
