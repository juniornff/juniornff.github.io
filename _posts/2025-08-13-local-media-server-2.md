---
layout: post
title: "Local Media Server part 2"
date: 2025-08-13 15:20:00 -0000
categories: Personal-Projects Docker Containers Proxy-Server Caddy-Server Transcoding Self-Hosting  
permalink: /posts/local-media-server-2
---

# Updates

Since the [first post](/posts/local-media-server) about the server, there have been several changes to the server and its environment. In the following posts, I'll explain what those changes were.

# From CloudFlare Tunnels to a Personal DDNS provider

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

# Caddy Server for reverse proxy and SSL/TLS

Cloudflare's standard workflow ensures secure connections between browsers and servers. In the previous configuration, this meant securing connections between browsers and Cloudflare as an intermediary, and then between Cloudflare and the server.

![Cloudflare](/assets/images/posts/localmediaserver2/Cloudflare.PNG)

Since we're no longer using Cloudflare Tunnels, and the DDNS provider only tells Cloudflare which public IP to route browser requests to (in this state, accessing media.nehemiasfeliz.com or example.nehemiasfeliz.com redirects to the device's public IP, but doesn't know which port each requested service runs on), the connection between Cloudflare and the server remains unsecured.

To solve these issues, we implemented [Caddy Server](https://caddyserver.com/). This will handle redirecting server requests based on the requesting subdomain, while also securing the connection between Cloudflare and the server by managing the [SSL/TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security) certificates provided by Cloudflare.

## Reverse proxy
Caddy Server can also be implemented via Docker. It only requires a *Caddyfile* specifying how to handle incoming external requests.

This is how I implemented the *docker-compose.yml* for the container:
```yaml
services:
  caddy:
    image: caddy:latest
    container_name: caddy-reverse
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    restart: unless-stopped
volumes:
  caddy_data:
  caddy_config:
```

And this is how the *caddyfile* is configured to handle requests. In this case, it simply converts all HTTP requests to HTTPS (secure connection) and redirects requests for the *media.* subdomain to the server's local IP and the port where Jellyfin is running:
```caddyfile
:80 {
  redir https://{host}{uri} permanent
}

media.nehemiasfeliz.com {
    reverse_proxy 192.168.1.19:8096
}
```

Part of the configuration involves opening ports 80 (HTTP connections) and 443 (HTTPS connections) on the router (as previously done for the initial [Jellyfin setup](/posts/local-media-server#port-forwarding)) so Caddy can manage these requests.

![Ports](/assets/images/posts/localmediaserver2/Ports.png)


## SSL/TLS

To address security, we implemented SSL/TLS - essentially encrypting the connection using certificates generated from "cryptographic files." Cloudflare provides these files ([Cloudflare's Origin CA](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/)) for cases like this where Cloudflare Tunnels aren't used, maintaining protection between Cloudflare and the server.

![TLS](/assets/images/posts/localmediaserver2/TLS.PNG)

Caddy manages these certificates files by simply specifying their location both in the container and in the caddyfile.

*docker-compose.yml*
```yaml
services:
  caddy:
    image: caddy:latest
    container_name: caddy-reverse
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./cloudflare-origin/cert.pem:/etc/caddy/cloudflare-origin/cert.pem:ro
      - ./cloudflare-origin/key.pem:/etc/caddy/cloudflare-origin/key.pem:ro
      - caddy_data:/data
      - caddy_config:/config
    restart: unless-stopped
volumes:
  caddy_data:
  caddy_config:
```

*caddyfile*
```caddyfile
:80 {
  redir https://{host}{uri} permanent
}

media.nehemiasfeliz.com {
    tls /etc/caddy/cloudflare-origin/cert.pem /etc/caddy/cloudflare-origin/key.pem
    reverse_proxy 192.168.1.19:8096
}
```

With this implementation, we can configure Cloudflare to use "Full (Strict)" encryption between Cloudflare and the server.
> Full (Strict): Enable encryption end-to-end and enforce validation on origin certificates. Use Cloudflare’s Origin CA to generate certificates for your origin.

Through the combination of cloudflare-ddns and Caddy Server, we can now manage both subdomains and server connection security without Cloudflare Tunnels' limitations. This provides greater control and scalability, allowing quick addition/removal of subdomains without manual Cloudflare portal access.

# Hardware Acceleration on Jellyfin

## from Docker Desktop to Docker "native"

###### TODO
I need to talk about the changes that have been made in the media server
* Change the the docker container from Docker Desktop to Docker Engine in the system
* Using the Transcoding options of the CPU by the Video Acceleration API (VA-API) to convert video files to reproduce them in different devices