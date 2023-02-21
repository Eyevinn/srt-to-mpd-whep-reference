Create a network for all the containers. This is the network that the containers in the docker-compose file will attach to.

```
docker network create -d bridge mpd-whep-network
```

Start the ingest containers:

```
docker-compose -f docker-compose-ingest.yml up
```

Start the MPEG-DASH distribution containers:

```
docker-compose -f docker-compose-mpd-dist.yml up
```

Start the webrtc distribution containers:

```
docker-compose -f docker-compose-wrtc-dist.yml up
```

Create a transmitter using the SRT WHIP gw API:

```
curl -X 'POST' \
  'http://localhost:3000/api/v1/tx' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
    "port": 9995,
    "whipUrl": "http://ingest:8200/api/v2/whip/sfu-broadcaster?channelId=srt",
    "passThroughUrl": "srt://shaka:4141",
    "status": "idle"
}'
```

Start ingesting with ffmpeg:

```
ffmpeg -i "rtsp://<username>:<password>@<rtsp-address>" -c:v libx264 -tune zerolatency -preset ultrafast -c:a aac -f mpegts "srt://localhost:9995"
```