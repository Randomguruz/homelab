# Homelab Infrastructure

Built and maintained 2025-present. Documented June 2026.

Project writeup for NTU ABA document verification.

---

## Introduction

This homelab was set up initially as a NAS, hence the choice in operating system. However, over time, it has evolved to perform many more functions, as listed below. This is mostly a curiosity project, hence the low budget and repurposing of old hardware. I had wanted a server since I was gaming years ago, and the spare desktop gave me that chance.

## System

I repurposed an old desktop as my home server. Hardware that was necessary was sourced and added, mostly from Shopee, Carousell and friends. The specs are as follows:

| Component  | Detail                                                |
| ---------- | ----------------------------------------------------- |
| CPU        | Intel i5-7500                                         |
| RAM        | 32GB DDR4 2400 (JEDEC spec, mismatched manufacturers) |
| GPU        | Removed (was GTX 1060)                                |
| OS         | TrueNAS Scale                                         |
| Storage    | 5x 1TB HDD (RAIDZ1), 5-wide pool    |
|            | 1x 256GB SSD for OS                                   |
|            | 1x 256GB SSD for Docker containers                    |
| Networking | 1x motherboard 1GbE                                   |
|            | 2x 10GbE (Intel X540-T2 10GBaseT)                     |

Half of the RAM was kindly donated by friends of mine, which led to a manufacturer mismatch. In order to get around this, I had to downclock them to the lowest JEDEC spec.

![](homelab-images/ram-mismatch.jpg)

Hard drives were sourced from Carousell at all under $15 per TB, as consumer drive prices have surged with AI demand. A 5-wide RAID Z1 allows for greater read throughput than a smaller array.

The SSDs were also provided by a friend, as the current SSD prices are too high, and the company he was working for just happened to be getting rid of a bunch of them.

![](homelab-images/truenas-storage.png)

---

## Services

| Service                         | What it does                                                                                                                                                                                                   |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Jellyfin**                    | Media streaming. My family watches shows through it. Our TV is older (no smart features), so I connect my laptop via HDMI. I also set up automated subtitle fetching so they can watch with Chinese subtitles. |
| **Immich**                      | Self-hosted photo backup (iCloud equivalent)                                                                                                                                                                   |
| **Pi-Hole**                     | DNS-wide ad blocking, local DNS records                                                                                                                                                                        |
| **Tailscale**                   | Mesh VPN for remote access                                                                                                                                                                                     |
| **NGINX Proxy Manager + Caddy** | Reverse proxy. Caddy as backup when NPM breaks on updates.                                                                                                                                                     |
| **SearXNG**                     | Self-hosted web search                                                                                                                                                                                         |
| **Crawl4ai**                    | Self-hosted web content extraction                                                                                                                                                                             |
| **MediaMTX**                    | RTSP-to-HLS stream relay (OBS to in-game video players)                                                                                                                                                        |
| **Foundry VTT**                 | Tabletop RPG sessions with friends                                                                                                                                                                             |
| **Filebrowser Quantum**         | Lightweight file sharing with anonymous upload links                                                                                                                                                           |
| **OpenSpeedTest**               | Network diagnostics                                                                                                                                                                                            |
| **Game servers**                | Factorio, Minecraft (on demand)                                                                                                                                                                                |

![](homelab-images/truenas-apps.png)

---

## Problems I faced

### Intel X540-T2: heat, brackets, and 10GBaseT

I found a listing on Shopee for Inspur Intel X540-T2 cards ripped from Chinese servers. The X540 is EOL, so I knew driver support would be a problem. The card shipped with an SFF half height bracket and no fan. I had read about thermal issues but assumed chassis airflow would be enough.

I tried using the card without a bracket. It was unstable and kept pulling out of the PCIe slot. I ordered full height brackets but the port cutouts didn't match my card. Together with my dad, I sawed the full length bracket, flattened the right angle on the half height one, and super glued them together. The card now fits securely when screwed in.

![](homelab-images/jank-pcie-bracket.jpg)

Then the thermal issues hit. The card would overheat and shut down entirely until a reboot. I measured the heatsink, went to Sim Lim Tower for a matching fan, and attached it with thick 3M double sided gel tape. The heatsink is held by plastic push screws that stay cool, so the tape held. Plugged into the motherboard fan header. No thermal shutdowns since.

![](homelab-images/nic.jpg)


### 10GbE without a 10GbE switch

I didn't have a 10GbE switch or a router that supported it. Solution was bridging the two ports on my server. My PC connects to the server over 10GbE, and the server connects to the ISP router over 1GbE. This routes my internet through the 10GbE link.

Getting full speeds took work. OpenSpeedTest, my default testing tool, was not able to hit the full 10 gigabit speeds. I found out that this was due to the nature of how the speed test was structured, and switched to iPerf3 instead, a memory to memory test that nears theoretical speed limits when testing. This test resulted in around 4.5 Gbps. I had to tweak Interrupt Moderation, Flow Control, TCP/UDP checksum offload, Tx/Rx buffers and RSS Queue count on Windows since the card isn't officially supported there. That got the speeds up.

These settings birthed another problem however, my PC was now no longer able to handle real-time audio. Due to how the card was purpose built for servers, when transmissions happened, any music I was playing would have audible "pop" sounds. After some research, I narrowed the cause down to 2 possibilities. A noisy SMBus (System Management Bus), or DPC latency. I quickly ruled that out, as that was something that people reported not allowing their machine to POST at all. A lot of trial and error with LatencyMon got me to settings that solved both problems.

### Split DNS

I set up Jellyfin and wanted my friends overseas to access it, so I bought a domain and set up a reverse proxy. I tested access over the domain and found it was abysmally slow compared to LAN. OpenSpeedTest showed around 0.4 Mbps with 500-1500ms latency.

I assumed it was my reverse proxy config. Tweaked SSL, HSTS, tried different nginx settings:

```
# Disable buffering for the request (client to proxy)
proxy_request_buffering off;

# Disable buffering for the response (proxy to client)
proxy_buffering off;

# Allow Nginx to pass large files without limits
client_max_body_size 0;
```

Nothing worked. Eventually I accepted this was a Cloudflare routing issue I couldn't fix and looked for local solutions.

Two options: Hairpin NAT or Split DNS. I went with Split DNS since I already had Pi-Hole running. I configured it to serve local IPs for my domain internally while forwarding everything else to Cloudflare's 1.1.1.1. Problem solved.

### Reverse proxy: from NPM to Caddy and back

During the DNS troubleshooting, I tried switching from NGINX Proxy Manager to Caddy. Caddy is lightweight and purely config-file driven. Performance improved but I didn't want to reload config files every time I changed an entry. Caddy now serves as backup if an NPM update breaks something.

I also set up a wildcard SSL cert via Let's Encrypt using DNS-01 challenge through Cloudflare. Much easier than per-subdomain certs, and I didn't have to worry about hitting rate limits during testing.

![](homelab-images/npm-ssl-wildcard.png)

### GPU accelerated hardware encoding: not worth it

I wanted to use the GTX 1060 to encode media and save storage. The default HandBrake container from TrueNAS didn't support NVENC. Found one that did, but the NVIDIA driver version it required didn't match what TrueNAS ships. TrueNAS is an appliance OS, I couldn't update the driver. The container didn't even have bash so I couldn't work around it inside either.

To check whether NVENC was worth pursuing, I benchmarked my laptop's 1050 (same encoder chip as the 1060) against my desktop's 3080. The 10-series doesn't even support 10-bit H.265. The quality difference made it not worth pursuing.

After the benchmarks, I decided not to continue with GPU encoding on this server. The hardware and driver constraints made the tradeoff poor.

### The Nextcloud mistake

I set up Nextcloud as a self-hosted cloud platform. It was heavy for my hardware. I stripped down features to make it run. Then Discord embeds stopped working when I switched to HTTPS. After stripping headers through my reverse proxy and examining with curl from the TrueNAS shell, I found the issue: an `X-Robots-Tag: noindex nofollow` header that prevented Discord's crawler from reading the page.

Another problem was the 512MB upload limit. I tried changing php.ini but the container didn't respect my changes. That's when I realised I didn't need a full cloud suite. I just wanted a file server. Switched to Filebrowser, then Filebrowser Quantum. Does what I need without the overhead.

### Physical 5-drive fit

My case had two drive bays. I needed space for five. I used thick 3M double sided gel tape to stick extra drives to the PSU shroud. The tape works as both a mount and vibration dampener. One side has two drives stacked. I keep them at roughly 90 degrees horizontal, which is within spec.

![](homelab-images/hard-drive-bays.jpg)

Temperatures were the next issue. The drives in the dedicated bays had no ventilation and would heat up during resilvering or scrubs. My solution was removing both side panels. Drives now sit in the low 40s, well within operating spec.

### ZFS and SMR

I bought cheap 1TB Seagate drives off Carousell at $12.50 each. The SMART data looked clean, so I grabbed them. They ran fine for a while. Then during a scrub, one of the drives failed.

I had overlooked that these were SMR (Shingled Magnetic Recording) drives. SMR and ZFS are fundamentally incompatible. SMR drives cannot keep up with long continuous operations like scrubbing or resilvering. This eventually caused the drive to fail in my pool.

After more hunting, I found someone selling WD Blue Desktop 1TB CMR drives at $10 per TB. I did the usual checks. Manufacturing date, SMART data, power on hours. This time I also verified they were CMR. I bought several, expanding the pool to 5-wide. They have been fine since.

### MediaMTX: RTSP streaming through a reverse proxy

I wanted to share my OBS screen to games like VRChat through in-game video players. The pipeline was OBS to MediaMTX to reverse proxy to the game. There was a lot of conflicting advice online about RTSP over reverse proxies. The actual fix was simpler: adding `index.m3u8` to the end of the stream URL output an HLS Multivariant Playlist that the video players could read. No RTSP through the proxy needed.

---

## Conclusion

This started as a cheap NAS, but it became the project where I learnt the most about networking, storage, Linux, containers, and hardware limits. Most of the progress came from things breaking. The X540 overheated, SMR drives failed under ZFS, the reverse proxy was not the actual bottleneck, and tuning 10GbE caused audio latency problems on my PC.

I still maintain this server because my family and friends actually use parts of it. It is not a polished enterprise system, but it is real infrastructure that I built, broke, repaired, and improved over time.
