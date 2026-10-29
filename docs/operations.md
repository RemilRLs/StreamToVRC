# Operations and troubleshooting

[Back to README](../README.md) · [HTTP mode](http.md) · [HTTPS mode](https.md)

Run commands from the repository root.

## VRChat playback

Paste a reachable HTTP(S) HLS playlist URL into the world video player. Every viewer must be able to reach that address. This project produces `.m3u8` playlists and `.ts` segments, not a directly playable RTMP URL for VRChat.

If VLC works but VRChat does not:

- Check that the specific world player supports live HLS and your URL scheme.
- Check VRChat's **Allow Untrusted URLs** setting when required for a self-hosted domain.
- Check reachability from the actual viewer device, not only the server.
- Test the target platform: PC and standalone Android/Quest can behave differently.

No player/platform compatibility matrix has been validated for this repository. Record the world/player version, platform, URL scheme and result when testing.

## Encoding profiles

Set `STREAM_PROFILE` in `.env`, then run `docker compose up -d --force-recreate` and restart OBS publishing.

| Profile | Outputs | Intended use |
| --- | --- | --- |
| `full` (default) | 1080p, 720p, 480p, 360p and two 240p variants | Adaptive quality; six simultaneous software encodes |
| `light` | 720p at 30 fps, 2,500 kbit/s video + 128 kbit/s audio | Lower CPU demand for hosts also running OBS, games or VRChat |

Both use the master URL `/hls/<stream-key>.m3u8`. The [`nginx/profiles`](../nginx/profiles) directory contains `transcode.conf` and `variants.conf`; their output suffixes and resolutions must agree.

Encoding uses CPU-based `libx264` with the `superfast` preset. No CPU sizing or maximum viewer count has been measured. Monitor the actual workload:

```sh
docker compose stats rtmp-hls
```

Upload traffic scales with viewers and selected quality. Three viewers at nominal 720p need roughly 7.9 Mbit/s before transport overhead and bitrate peaks. OBS upload to a remote server is additional traffic. Fewer generated variants reduce encoding work, not necessarily traffic per viewer.

### Bitrate and latency

Playlist `BANDWIDTH` estimates include video, audio and 10% transport headroom. These are not measured peak bitrates; variable-rate encoding can exceed them. Measure segment sizes and durations before tuning the advertised bandwidth for production.

NGINX targets one-second segments and a four-second playlist window. End-to-end latency also depends on keyframes, encoding, network conditions and player buffering. Expect several seconds of delay; there is no measured guarantee. Use a visible clock in OBS and the target player to measure it.

## Deployment files

[`docker-compose.yml`](../docker-compose.yml) is the source of truth. It mounts:

- `nginx/nginx.conf.template` into `/etc/nginx/nginx.conf.template`, read-only;
- the chosen profile directory into `/etc/nginx/profile`, read-only;
- the named volume `hls` into `/opt/data`, where HLS files are written.

The image generates `/etc/nginx/nginx.conf` from the template at startup. The optional HTTPS proxy uses its own [template](../reverse-proxy/default.conf.template) and certificate directory.

## Useful commands

```sh
# Validate Compose and the running streaming server's configuration.
docker compose config --quiet
docker compose exec rtmp-hls nginx -t

# Inspect status and logs.
docker compose ps
docker compose logs --tail=100 rtmp-hls

# Stop services, retaining the named HLS volume.
docker compose down
```

After editing streaming NGINX or encoding profile files, run `docker compose restart rtmp-hls` so startup regenerates the configuration. After changing `.env` or Compose, run `docker compose up -d --force-recreate`. Restart OBS publishing after either operation. For proxy changes, see the [HTTPS guide](https.md#update-certificates-or-proxy-configuration).

Images are pinned by digest for reproducibility. To update one, select and resolve the desired upstream image, update its digest in Compose, pull and recreate the service, then validate playback. A pinned image does not automatically receive upstream updates.

## End-to-end validation

1. Compose validation and `nginx -t` succeed.
2. OBS connects to `/stream` using the expected key; click **Start Streaming**.
3. The master playlist returns `#EXTM3U` and the intended quality entries.
4. Variant playlists and referenced `.ts` segments load; VLC plays video and audio.
5. Playback succeeds from another network if public access is required.
6. Playback succeeds in the target VRChat world and platform.
7. Repeat with `STREAM_PROFILE=light` when validating both encoding profiles.

## HTTP endpoints

| Path | Purpose |
| --- | --- |
| `/hls/<key>.m3u8` | Master playlist |
| `/stat` | RTMP statistics, also used by the health check |
| `/stat.xsl` | Statistics stylesheet |

Direct access on port 8080 exposes these endpoints, including statistics. The HTTPS reverse proxy forwards only `/hls/`.

## Troubleshooting

| Symptom | Checks |
| --- | --- |
| Docker cannot connect to its daemon | Start Docker Engine/Desktop and check `docker version`. |
| OBS cannot connect | Address, `/stream`, port 1935, `RTMP_BIND_ADDRESS`, firewall and container status. |
| Playlist returns 404 | OBS is actively streaming, the key matches the filename, and the first segments exist. Inspect logs and `/stat`. |
| Playlist loads but playback fails | Check variant playlists and segments, then try VLC. Inspect encoding load and logs. |
| Local playback works, remote fails | Port forwarding, firewall, public IP, CGNAT, double NAT and WSL forwarding. Test externally. |
| Public URL fails only inside LAN | Check NAT loopback or local DNS. |
| Stuttering or growing delay | CPU usage, OBS dropped frames, server upload and viewer connection. Try `light`. |
| VLC works, VRChat fails | World-player support, platform, URL settings and viewer reachability. |
| NGINX fails after a profile change | `STREAM_PROFILE` must be `full` or `light`; check both profile files and startup logs. |

For certificate, Cloudflare or proxy-port errors, see [HTTPS troubleshooting](https.md#troubleshooting).
