---
layout: default
title: "Local Media Server"
date: 2025-02-18 14:15:00 -0000
categories: Personal-Projects Containers Cloudflare DNS Self-Hosting
permalink: /posts/local-media-server
---

*this is just a preview, latter images will be posted*

# Background

I have always liked to have my content locally, until now what I did was use a USB to store the files I wanted to see on my TV, but since adding new content was very tedious, I decided to start this project

# The project

Since I wanted it to be local, I decided to use an old laptop I have at home (A Samsung with an Intel Core i3, 8 GB RAM, 240 GB SSD) as a local server using [Debian](https://www.debian.org/) as the OS and the 1TB USB as extra storage.

I used [Docker](https://www.docker.com/) since I wanted to learn how to use it. As a media server I decided to use [Jellyfin](https://jellyfin.org/) since it is open source. To access data remotely while on the network use [Samba](https://www.samba.org/samba/). In order to publish it on the web at the beginning I used [duckdns](https://www.duckdns.org/) and [port forwarding](https://portforward.com/). Then I decided to purchase a [domain](https://nehemiasfeliz.com/) and set up a tunnel using [Cloudflare](https://www.cloudflare.com/es-es/) for more security.

# Setting up Local environment

After a clean installation of Debian, I set a static IP on my local network. After installing Docker I learned how to allow it to access devices such as USB in a easy way, I created a docker compose file with the necessary configurations to download and start the server. After doing the initial configuration of Jellyfin and setting up the media library, I could already access it from any device within the network with the local IP and port. First achievement

# First attempt to go online

The project could have ended in the previous step, but I wanted my family members who don't live with me and my friends to also be able to enjoy this project.

My research on how to remotely access the server led me to investigate DNS, although the public IP address could be enough, I wanted it to be easier to access than putting numbers in the browser, so looking for DDNS providers I found Duckdns, a service that offered just what I needed (Free). I didn't care that it was a subdomain, since it was a simple and personal project. After exposing the ports on my router and redirecting all the traffic from those ports to the laptop, I could now access the server from any network.

Once again, I could have left the project as is, but something didn't sit well with me:
*   It was connected via HTTP (not secure)
*   I was afraid of what could happen in case of DDOS attacks (unlikely but it never hurts to be safe)

So I decided to keep investigating...

# Using the proper technologies

The research led me to this [Youtube](https://youtu.be/ey4u7OUAF3c?si=iqb3qRVzkVIa4Uk1) video, where I learned about Cloudflare and their tunneling system, and after understanding well how the process would be, I had to buy a domain, since although Cloudflare provided what I needed (HTTPS and DDOS protection) for free, I could not continue using DuckDNS, so I had to buy a domain, Cloudflare allows you to buy domains, which made the process of setting it up much easier. After that I simply had to install the tunnel in docker, and thus be able to create subdomains/routes that link to IP addresses on my local network, said subdomains/routes being accessed through HTTPS by SSL certificates without having to open/expose any ports on my router

And with that now, I can say that for now, I have completed the project.

[THE PROYECT](https://media.nehemiasfeliz.com/)