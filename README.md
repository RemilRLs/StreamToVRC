# RTMP → HLS Streaming for VRChat

This project allows you to **stream your desktop, gameplay, or media content** from OBS Studio directly into a **VRChat world video player**.  
VRChat does not support RTMP directly, so the stream is converted into **HLS (`.m3u8` + `.ts` segments)**, which VRChat video players can read.

---

## Overview

**Workflow:**

1. **OBS Studio** captures your screen and sends it as an RTMP stream: rtmp://<host>:1935/live/stream
2. **NGINX with RTMP module** receives the RTMP input and converts it into **HLS segments**.
3. The HLS segments are served over HTTP and can be accessed via: http://<host>:8080/hls/stream.m3u8
4. Paste this URL into a **VRChat Video Player** → your live stream will play inside VRChat.

---

## Protocols

### RTMP (Real-Time Messaging Protocol)
- **Definition**: Low-latency protocol for live streaming (port `1935`).
- **Limitation**: VRChat and browsers cannot play RTMP directly.

### HLS (HTTP Live Streaming)
- **Definition**: Splits live video into `.ts` segments coordinated by an `m3u8` playlist.
- **Limitation**: Adds some latency.

---

## Implementation Pipeline

- **OBS Studio** → Send RTMP stream
- **NGINX + RTMP Module** → Convert RTMP to HLS (`.ts` + `m3u8`)
- **HTTP Server** → Serve HLS over port `8080`
- **VRChat** → Play the HLS stream inside the world video player

---


## Networking / Ports

To make the HLS stream accessible in **VRChat** (or to other viewers),  
you must **at least open port 8080 (TCP)** on your machine and forward it from your router/firewall to the host running Docker.

- **8080 (TCP)** → Serves the HLS files (`.m3u8` and `.ts`) → **Required** for playback.  
- **1935 (TCP)** → Only used for RTMP ingest from OBS → You don’t need to open this port if you are streaming **only from your local machine**.  
  If you want other people to be able to publish RTMP streams remotely, open **1935 (TCP)** as well and secure it using `on_publish` or `allow/deny`.

---

## Ports

- **1935** → RTMP ingest (OBS → NGINX)  
- **8080** → HLS output (NGINX → VRChat)

If others need to view your stream, **port forwarding on 8080 (TCP)** is required.

---


## Docker Setup

We use the [`alfg/nginx-rtmp`](https://hub.docker.com/r/alfg/nginx-rtmp) image which includes NGINX compiled with the RTMP module.

```yaml
services:
rtmp-hls:
 image: alfg/nginx-rtmp:latest
 container_name: rtmp-hls
 restart: unless-stopped
 ports:
   - "1935:1935"   # RTMP ingest from OBS
   - "8080:80"     # HTTP for HLS playback
 volumes:
   - ./nginx/nginx.conf:/etc/nginx/nginx.conf
   - hls:/var/www/hls

volumes:
hls:
```

---

## Usage

1 -  Start Docker Compose

```bash
docker-compose up -d
```

2 - Configure OBS to stream to:

1. Open **OBS Studio**.  
2. Go to **Settings → Stream → Services → Custom**:  
   - **Server**: `rtmp://<your-local-ip>:1935/stream`  
   - **Stream Key**: `stream` or whatever you want.
3. Start streaming. 


3 - Find your local IP address


On windows (Powershell or CMD) :

```bash
ipconfig
```

Look for the section Ethernet adapter (IPv4 Address)

On Linux

```bash
ip addr show
```

You’ll get your LAN IP (e.g., 192.168.1.26).

4 - Access your stream

From the same network (LAN) :

```bash
http://<your-local-ip>:8080/hls/stream.m3u8
```

5 - Share to other people (Public access)

If you want people outside your network to watch your stream:

1. Find your public IP address (visit https://whatismyip.com)
2. On your router/box, configure port forwarding:
  * TCP 8080 → 8080 on your machine running Docker
  * (Optional) TCP 1935 → 1935 if you want others to publish RTMP streams to your server
3. Share the HLS link with others:

```bash
http://<your-public-ip>:8080/hls/stream.m3u8
```

6 - Play on the VideoPlayer

Put thet link :

```bash
http://<your-public-ip-address>:8080/hls/stream.m3u8
```

---

## Notes

- HLS introduces ~5–10 seconds of latency compared to RTMP.
- If you expose port 1935 for remote ingest, secure it with `allow/deny` rules or `on_publish` validation.
- VRChat players only accept HTTP(S) `.m3u8` links, **not** RTMP URLs.
