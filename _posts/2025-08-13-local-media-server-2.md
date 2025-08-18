---
layout: post
title: "Local Media Server part 2"
date: 2025-08-13 15:20:00 -0000
categories: Personal-Projects Docker Containers Proxy-Server Caddy-Server Transcoding Self-Hosting  
permalink: /posts/local-media-server-2
---

# Updates

Since the [first post](/posts/local-media-server) about the server, there have been several changes to the server and its environment. In the following posts, I'll explain what those changes were.

# From CloudFlare Tunnels to DDNS provider

One of the first changes I made to the server was switching from Cloudflare Tunnels to a custom [DDNS](https://en.wikipedia.org/wiki/Dynamic_DNS) provider.

The primary reason for this change was my interest in installing [Nextcloud](https://nextcloud.com/) and their documentation about the [limitations](https://github.com/nextcloud/all-in-one?tab=readme-ov-file#notes-on-cloudflare-proxytunnel) when using Cloudflare Tunnels. While these limitations don't currently affect my project, I wanted to learn how to use DDNS.

My first step was researching which DDNS provider to implement. I decided to use [timothymiller/cloudflare-ddns](https://github.com/timothymiller/cloudflare-ddns), a Cloudflare-focused DDNS provider with multiple features packaged as a Docker container. This made it relatively easy to implement.

This is the *docker-compose.yml* with the implemented configuration:

```yaml
services:
  cloudflare-ddns:
    image: timothyjmiller/cloudflare-ddns:latest
    container_name: cloudflare-ddns
    network_mode: "host"
    security_opt:
      - no-new-privileges:true
    volumes:
      - ./config.json:/config.json:ro
    restart: unless-stopped
```

And the JSON configuration file:

```json
{
  "cloudflare": [
    {
      "authentication": {
        "api_token": <CLOUDFLARE API TOKEN>
      },
      "zone_id": <CLOUDFLARE ZONE ID>,
      "subdomains": [
        { "name": "media", "proxied": true },
      ]
    }
  ],
  "a": true,
  "aaaa": true,
  "purgeUnknownRecords": false,
  "ttl": 300
}
```

For this implementation, it was necessary to generate an API KEY capable of creating/modifying DNS records in Cloudflare. The DDNS service's job is to periodically inform Cloudflare of the public IP address where requests for that subdomain/domain should be directed.

With this in place, we can now:
* Stop the Cloudflare Tunnels container
* Delete the DNS records created by the tunnel
* Start the cloudflare-ddns container to generate/modify DNS records in Cloudflare

![CloudDnsRecords](/assets/images/posts/localmediaserver2/CloudflareDnsRecords.PNG)

We can observe that in this implementation, there's mention of proxy redirection. Since we're no longer using Cloudflare Tunnels, we must implement our own reverse proxy to handle requests to different subdomains (in this case, for the Jellyfin server). For this purpose, along with cloudflare-ddns, we implemented Caddy Server.

# Caddy Server for reverse proxy and SSL

# from Docker Desktop to Docker "native"

# Hardware Acceleration on Jellyfin

###### TODO
I need to talk about the changes that have been made in the media server
* The implementation of Caddy Server for reverse proxy and SSL
* Change the the docker container from Docker Desktop to Docker "native" in the system
* Using the Transcoding options of the CPU by the Video Acceleration API (VA-API) to convert video files to reproduce them in different devices