# HTTP mode

[Back to README](../README.md) · [HTTPS mode](https.md) · [Operations and troubleshooting](operations.md)

Use this mode to test locally, watch over your LAN, or expose HLS directly over HTTP. No domain or TLS certificate is required.

Run commands from the repository root. On Windows, use Docker Desktop in Linux container mode or a working Docker Engine inside WSL.

## 1. Start the server

Replace `<repository-url>` with this repository's clone URL:

```sh
git clone "<repository-url>" StreamToVRC
cd StreamToVRC
```

Optionally copy [`.env.example`](../.env.example) to `.env` to customize settings. Docker Compose automatically loads `.env`, not `.env.example`. For HTTP mode, leave `COMPOSE_PROFILES` unset. If switching from HTTPS, see [returning to HTTP](https.md#return-to-http-mode).

```sh
docker compose up -d
docker compose ps
```

The first start downloads the image. The service should become `healthy` once NGINX's HTTP statistics endpoint responds. This checks NGINX availability, not that OBS is publishing or that video plays.

## 2. Configure OBS and start streaming

When OBS and Docker run on the same machine, open **Settings → Stream → Custom**:

| Setting | Value |
| --- | --- |
| Server | `rtmp://127.0.0.1:1935/stream` |
| Stream key | `stream` |

Use H.264 video and AAC audio for a predictable input.

**Click Start Streaming in OBS. Keep streaming active during testing and playback.** Wait a few seconds for the first HLS segments to be generated.

The RTMP application name is `/stream`. The key determines the playlist name: `stream` produces `stream.m3u8`, while `demo` produces `demo.m3u8`. Use letters, digits, hyphens or underscores. The key names the stream; it is not authentication.

## 3. Verify local playback

```sh
curl -f http://127.0.0.1:8080/hls/stream.m3u8
```

In Windows PowerShell, use `curl.exe` if `curl` is an alias. Expect a successful response starting with `#EXTM3U`. A `404` is expected before OBS starts and the first segments are generated.

Open the URL in **VLC → Media → Open Network Stream** to verify both video and audio:

```text
http://127.0.0.1:8080/hls/stream.m3u8
```

A browser may download the playlist instead of playing it. A successful playlist request alone does not prove that variant playlists and segments play correctly.

## 4. Watch over the LAN

Find the Docker host's LAN IPv4 address with `ipconfig` on Windows or `ip addr show` on Linux. Choose the active Ethernet/Wi-Fi interface, not loopback or a Docker bridge.

For example, if the server is `192.168.1.26`, viewers on the same LAN use:

```text
http://192.168.1.26:8080/hls/stream.m3u8
```

Port 8080 is published on all host interfaces by default. Allow TCP 8080 through the host firewall. No router port-forward is needed for LAN-only viewing.

Every viewer must be able to reach the URL entered into the VRChat player. `127.0.0.1` refers to each viewer's own device and cannot be used as a shared server address.

## 5. Allow public HTTP playback

Configure port forwarding on the router of the person **hosting Docker**, not on viewers' routers:

| Router setting | Example value |
| --- | --- |
| Protocol | TCP |
| External port | `8080` |
| Destination LAN IP | `192.168.1.26` |
| Internal port | `8080` |

Reserve the server's LAN address in the router and allow access through the host firewall. Find the router's public IP, then test from another network, such as a mobile connection:

```text
http://<public-ip>:8080/hls/stream.m3u8
```

The LAN address `192.168.1.26` is not reachable directly from the Internet. Anyone who can reach the HTTP URL can watch; no viewer authentication is configured. For a public HTTPS domain, follow the [HTTPS guide](https.md).

### Network limitations

- **CGNAT:** ordinary port forwarding cannot usually provide inbound access when your ISP shares a public IPv4 address. Use a reachable public address or another reachable hosting arrangement.
- **Double NAT:** forwarding may be needed on both routers.
- **NAT loopback:** some routers cannot reach their own public address from the LAN. Test externally before diagnosing a failure.
- **Shared URLs:** remote viewers cannot use a LAN-only URL. Suitable local/public DNS can help when all viewers need the same hostname.

## OBS on another machine

The default RTMP binding is `127.0.0.1`. To accept OBS from another machine, set this in `.env`:

```dotenv
RTMP_BIND_ADDRESS=0.0.0.0
```

```sh
docker compose up -d --force-recreate
```

In OBS, use `rtmp://<docker-host-lan-ip>:1935/stream`. Allow TCP 1935 through the host firewall from the OBS machine.

`RTMP_BIND_ADDRESS` selects the Docker host's listening address, not the permitted client's source IP. `0.0.0.0` listens on all host IPv4 interfaces; a specific LAN address restricts the listening address and must belong to the Docker host. Restrict publishers with firewall source-IP rules or NGINX `allow publish` / `deny publish` rules in the `stream` application.

Only forward TCP 1935 on the router if OBS must publish from outside the LAN. The internal `hls` application accepts publishing only from container loopback.

## WSL and Docker Desktop

For OBS on Windows and Docker accessible through WSL, start with:

```dotenv
RTMP_BIND_ADDRESS=127.0.0.1
```

Use `rtmp://127.0.0.1:1935/stream` in Windows OBS, then test from Windows:

```powershell
curl.exe -f http://127.0.0.1:8080/hls/stream.m3u8
```

With Docker Engine directly inside WSL in NAT mode, Windows' LAN address may not exist inside WSL. Binding to it can fail with `can't bind on the specified endpoint` or `cannot assign requested address`.

For remote access, listening on `0.0.0.0` does not itself provide Windows-to-WSL forwarding. Check Docker Desktop integration or WSL networking mode, Windows Firewall and any required Windows port forwarding. A router rule targeting Windows does not automatically reach a standalone Docker Engine inside WSL. Test from another LAN device before testing public access.

## Next steps

- [Configure HTTPS with a domain](https.md).
- [Choose encoding qualities, troubleshoot playback and validate VRChat](operations.md).
