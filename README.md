Go Kurento communication package
================================

[Kurento](http://www.kurento.com/) is a WebRTC Media Server built over Gstreamer. It allows stream management between Web browser and Server.

> WebRTC is a project that provides browser and mobiles integration with Reat Time Communication capabilities via simple APIs.
> -- http://www.webrtc.org/

Without Kurento, you will only be able to connect 2 browsers each other. That enougth for a lot of applications but you will need more when you want to stream from one browser to several clients, stream a video to several readers, and so on. For that need, you probably want to have client-server capabilities.

WebRTC is fully implemented in Kurento Media Server. 

Kurento with Go
---------------

With Go at server side, you only need to implement browser to application message system (websocket, xhttpRequest...) to pass information to KMS or to clients. 

Your application will communicate streams information to KMS via Kurento Go Package and to clients.

See doc at https://godoc.org/github.com/forkwhilefork/kurento-go

Example:
--------

This snippet shows a simple way to have master (sender) and viewer (reciever). Master MUST be created at first. Note that there is no "close" management, this is only to show you how it works.

Launch KMS server, or if you have docker, run this command:
    
    docker start --rm -it -p 8888:8888 kurento/media-server-64:5.1.0-trusty

You will be able to connect to 127.0.0.1:8888 that is the JSONRPC over WebSocket port that opens Kurento.

In you Go application:

```go
import "github.com/forkwhilefork/kurento-go"

// Create master pipeline and master WebRtcEnpoint
// that will be shared to viewers
var pipeline = new(kurento.MediaPipeline)
var master = new(kurento.WebRtcEndpoint)
var server = kurento.NewConnection("ws://127.0.0.1:8888")

...

// Somewhere 
// in a websocket handler:

message := ReadWebSocket()

if message["id"] == "master" {
    pipeline.Create(master, nil)
    answer, _ := master.ProcessOffer(message["sdpOffer"])

    // need to connect one sink, use self connection
    master.Connect(master, "", "", "") // empty options

    // return the answer to sender
    SendtoClient(json.Marshal(map[string]string{
        "id"        : "masterResponse",
        "response"  : "accept",
        "sdpAnswer" : answer,
    }))
}

if message["id"] == "viewer" {
    viewer := new(kurento.WebRtcEndpoint)
    pipeline.Create(viewer, nil)
    answer, _ := viewer.ProcessOffer(message["sdpOffer"])

    //connect master to viewer
    master.Connect(viewer, "", "", "")

    //return answer to viewer
    SendtoClient(json.Marshal(map[string]string{
        "id"        : "viewerResponse",
        "response"  : "accept",
        "sdpAnswer" : answer,
    }))
}
```

Note that SendToClient() and ReadWebsocket() should be implemented by yourself, you may want to use standard websocket handler, or gorilla websocket.

The browser side:

```javascript
 var ws = new WebSocket("ws://127.0.0.1:8000/ws"),
        master = null,
        viewer = null;

    ws.onmessage = function(resp) {
        r = JSON.parse(resp.data);
        switch(r.id) {
            case "masterResponse":
                if (r.response == "accepted") {
                    master.processSdpAnswer(r.sdpAnswer);
                }
                break;
            case "viewerResponse" :
                if (r.response == "accepted") {
                    viewer.processSdpAnswer(r.sdpAnswer);
                }
                break;
            default:
                console.error(r);
                break;
        }
    };

```

To launch master:

```javascript
function startMaster(){
    var video = $("#mastervideo");
    master = kurentoUtils.WebRtcPeer.startSendOnly(video, function(offer){
            var message = {
                id : "master",
                sdpOffer : offer
            };
            ws.send(JSON.stringify(message));
            console.log("sent offer from master to viewer")
    });
}

```

And the viewer:

```javascript
function startViewer(){
    var video = $("#clientvideo");
    viewer =  kurentoUtils.WebRtcPeer.startRecvOnly(video, function(offerSdp) {
        var message = {
            id : 'viewer',
            sdpOffer : offerSdp
        };
        ws.send(JSON.stringify(message));
        console.log("sent offer from viewer to master")
    });
}
```

You have to create video element and manage call to startMaster() and startViewer().

Remember that the "onclose" process is not implemented on this snippet.
