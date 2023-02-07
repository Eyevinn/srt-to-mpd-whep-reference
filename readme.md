# FFMPEG -> [SRT-GATEWAY] -> [WHIP-WHEP] -> [MPD-WHEP]

## Requirements
`ffmpeg`
`docker`
`serve`

### Setup

**Start the WHIP-WHEP docker containers**

`curl -SL https://github.com/Eyevinn/whip-whep/releases/download/v0.2.0/docker-compose.yml | docker-compose -f - up -d`

**Start the SRT-GATEWAY docker container** 

First we need to find out which network WHIP-WHEP is on

```
docker network ls

NETWORK ID     NAME                     DRIVER    SCOPE
d8e65abb5e48   host                     host      local
18bd740cfe25   none                     null      local
02eedf7089a9   whip-whep_default        bridge    local
```

In this case they are running on the docker network called `whip-whep_default`

Start the SRT WHIP Gateway on this network:

```
docker run --network whip-whep_default -p 3000:3000 -p 9000-9999:9000-9999/udp eyevinntechnology/srt-whip
```

The SRT Gateway is now available on `http://localhost:3000`

And then use the following WHIP URL in the transmitter: `http://ingest:8200/api/v2/whip/sfu-broadcaster?channelId=test`

**Start the MPD ffmpeg instance**

Start FFMPEG with an an input of SRT in listener mode

`ffmpeg -i "srt://<local-ip>:4141/?mode=listener" -c:v libx264 -c:a aac -f dash live.mpd`

serve the dash manifest and segments on `localhost:1234`

`serve . --cors -l 1234`

**Create the transmitter**

To manage your transmitter you can either use the UI `http://localhost:3000/ui` or the API `http://localhost:3000/api/v1/tx`, we're going to use the [API](http://localhost:3000/api/docs/):

First create a transmitter

`POST /api/v1/tx/` 

```json
{
    "port": 9995,
    "whipUrl": "http://ingest:8200/api/v2/whip/sfu-broadcaster?channelId=srt", // ingest:8200 referes to the WHIP docker container
    "passThroughUrl": "srt://<local-ip>:4141", // PassThrough to the ffmpeg instance that will produce the MPEG-DASH manifest
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

**Ingest**

TODO: Change to use an RTSP stream as input

Now we can start ingesting with ffmpeg
  
```
ffmpeg -re -i <file.mp4> -c copy -f mpegts "srt://localhost:9995"
```

You should now have a working WebRTC stream available through WHEP at `http://localhost:8300/whep/channel/srt`

**Egress**

TODO: Make mpd-whep available somewhere

Run the MPD-WHEP Docker container with the MPD & WHEP env variables set to your WHEP and MPD sources.

`docker run -e MPD=http://10.4.0.150:1234/live.mpd -e WHEP=http://localhost:8300/whep/channel/srt -p 8000:8000 -d mpd-whep`

You now have a MPD with both normal segments and WebRTC 🙌

`http://localhost:8000/manifest.mpd`