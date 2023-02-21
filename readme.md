# SRT to MPD-WHEP Reference implementation

![](./assets/diagram.png)

## Requirements
- docker

### Setup

**Start containers**

```sh
docker-compose -p mpd-whep up
```

**Create the transmitter**

To manage your transmitter you can either use the UI `http://localhost:3000/ui` or the API `http://localhost:3000/api/v1/tx`, we're going to use the [API](http://localhost:3000/api/docs/):

*There is a POSTMAN collection [here](https://github.com/Eyevinn/srt-whip-gateway/blob/main/docs/SRT-GATEWAY.postman_collection.json)*

First create a transmitter

`POST /api/v1/tx/` 

```json
{
    "port": 9995,
    "whipUrl": "http://ingest:8200/api/v2/whip/sfu-broadcaster?channelId=srt", // ingest:8200 referes to the WHIP docker container
    "passThroughUrl": "srt://srtrecv:4141", // PassThrough to the shaka-packager instance that will produce the MPEG-DASH manifest
    "status": "idle"
}
```

Then start the transmitter

`PUT /api/v1/tx/9995/state`

```json
{
    "desired": "running"
}
```

**Ingest RTSP**

Now we can start ingesting with ffmpeg
  
```sh
ffmpeg -i "rtsp://<username>:<password>@<rtsp-address>" -c:v libx264 -tune zerolatency -preset ultrafast -c:a aac -f mpegts "srt://localhost:9995"
```

You should now have a working WebRTC stream available through WHEP at `http://localhost:8300/whep/channel/srt`

**Egress**

The `eyevinntechnology/mpd-whep` container now running provides you with an MPD with both normal segments and WebRTC 🙌

`http://localhost:8000/manifest.mpd`


**Play the DASH stream**

WebRTC in dash is still in early stages and as such there are no players that supports it, you can however use an experimental branch of dash.js, `feature/webrtc`, available [here](https://github.com/Dash-Industry-Forum/dash.js/tree/feature/webrtc).

Once you've setuped everything you'll be able to play the stream on `http://localhost:3000/samples/dash-if-reference-player/index.html?mpd=http://localhost:8000/manifest.mpd`. Use the quality switcher to switch between dash & webrtc.
