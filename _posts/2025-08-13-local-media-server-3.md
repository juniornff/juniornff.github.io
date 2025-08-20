---
layout: post
title: "Local Media Server part 3"
date: 2025-08-13 15:20:00 -0000
categories: Personal-Projects Docker Containers Proxy-Server Caddy-Server DNS Shoko-Server Automating Torrent Networking Self-Hosting  
permalink: /posts/local-media-server-3
---

Now's a good time to say that I consider myself someone who enjoys [anime](https://anilist.co/user/juniornff/) quite a bit, and therefore, over the course of my life, I've built up a collection of anime on my computers. So, since the [second post](/posts/local-media-server-2), I decided to add another server dedicated to better managing my anime series on Jellyfin.

# Shoko Server

Although jellyfin can "[detect](https://jellyfin.org/docs/general/server/plugins/#metadata-plugins)" which series the files belong to based on the name of the folder/files themselves using plugins, it doesn't always work and [organizing](https://jellyfin.org/docs/general/server/media/shows/) series with multiple seasons requires a specific and tedious structure and configuration, and since the seasons of an anime are not named numerically/sequentially ([example](https://anidb.net/anime/14491/relation/graph)), determining the order is entirely up to you.

Based on that need for organization, I began researching how to organize my series more efficiently and automatically. The research led to the solution: [Shoko](https://shokoanime.com/). An open-source system that automates series organization for media centers like Jellyfin or Plex.

The main characteristic of Shoko is the following:
> Shoko streamlines your anime collection by hashing your files and matching them against AniDB’s comprehensive database. It automatically fills your collection with detailed information about each series and episode, while also pulling in metadata from other sources.[1](https://shokoanime.com/#:~:text=Effortless%20Organization-,Why%20Use%20Shoko%3F,-Shoko%20streamlines%20your)

In addition to handling duplicates, missing, corrupt/incomplete files, among other things. But what interests us about Shoko is that it will organize the series in the way that Jellyfin expects them (structure and metadata), allowing us to get rid of the task of manual organization.

### Instaling the server

Shoko offers a [Docker image](https://hub.docker.com/r/shokoanime/server) for installation on Linux systems, which is very convenient since we've been working with containers so far. As always, we will create a docker compose file to create the container (the docs offers a [Docker Compose Builder](https://docs.shokoanime.com/getting-started/installing-shoko-server#docker-compose-builder)).

```yaml
# docker-compose.yml
services:
  shoko_server:
    shm_size: 256m
    container_name: shoko_server
    image: ghcr.io/shokoanime/server:v5.1.0
    restart: always
    environment:
      - "PUID=$UID"
      - "PGID=$GID"
      - "TZ=UTC-4"
    ports:
      - "8111:8111"
    volumes:
      - "config:/home/shoko/.shoko"
      - "/mnt/storage:/mnt/anime"
volumes:
  config:
```

With that we can access the server through the browser by accessing http://localhost:8111

### Configuring the server

The [documentation](https://docs.shokoanime.com/getting-started/running-shoko-server) details how to configure the server, create users, select the folder with the files to analyze, and connect to AniDB.

![ShokoInitSetup](https://docs.shokoanime.com/images/shoko-server/shoko-server-first-run-welcome.jpg)

I'll only explain how I configured the folder to use with Shoko. The [documentation](https://docs.shokoanime.com/getting-started/running-shoko-server#import-folders) discusses the different ways Shoko interacts with the folder and its files:
* **Watch**: It only watches and analyzes the files without making any changes to them or their folders.
* **Drop Source**: Analyzes the files, modifies them (renames and changes the folder structure), and sends them to the Drop Destination.
* **Drop Destination**: Responsible for storing the files/folders for consumption.

When adding/modifying which folders to use, Shoko offers the following options:

[![ShokoFolders](/assets/images/posts/localmediaserver3/ShokoFolders.PNG)]([https://](https://docs.shokoanime.com/shoko-server/dashboard#import-folder-options))

In my case, since I don't want to modify the names or structure of the files (I'll explain why later), I configured my folder like this:

![ShokoFoldersActual](/assets/images/posts/localmediaserver3/ShokoFoldersActual.PNG)

With that, Shoko is now able to automatically analyze and organize my series. All that's left is to integrate Shoko into Jellyfin so it can take advantage of Shoko's organization to present the series efficiently. But first, I want to talk a little about AniDB.

#### AniDB

As Shoko's documentation mentions:
> [AniDB](https://anidb.net/) is a free and comprehensive anime database and file tracker, heavily utilized by Shoko to provide metadata for the series and files in your collection. Currently, AniDB is required to use Shoko, so you'll need to create an account if you don't already have one.

But generally speaking, AniDB is one of the sites known as "trackers" for anime series. But unlike [Anilist](https://anilist.co/home) or [Myanimelist](https://myanimelist.net/), AniDB focuses on the files themselves.

The same episode of a series can have differences (quality, subtitles, length, size, etc.). These differences are reflected in what we call [HASH](https://en.wikipedia.org/wiki/Hash_function). An episode released by Netflix will have a different HASH than one released by Crunchyroll or a particular individual. An example of this can be seen with episode 1 of the first season of [Dandadan](https://anidb.net/anime/18290):
* [Netflix](https://anidb.net/file/3548589): 9aa6b701
* [Crunchyroll](https://anidb.net/file/3548590): 934805ac
* [WZF](https://anidb.net/file/3731429): 44c378b0

So in the end, apart from keeping track of the anime you've watched, you can also know exactly where those files you have are from. Shoko analyzes the files you own to find their hashes and compares them with files known to AniDB. This way, it doesn't depend on the file name, but rather on the file hash.

Since Shoko uses your AniDB account to analyze files, when it detects them, it can add them to your anime list and even report if they've been viewed or delete them if they're no longer available.

**Note:** Not all of your files may be registered in AniDB and therefore not [recognized](https://docs.shokoanime.com/shoko-server/unrecognized-files) by Shoko, but Shoko helps you register them in AniDB (Registering files has its rules [1](https://wiki.anidb.net/Content:Files) [2](https://wiki.anidb.net/Tutorial:How_to_Add_Files_for_Dummies!) and is not always recommended unless you know what you are doing) and thus be recognized by Shoko. Or simply manually indicate which anime/episode the file belongs to (This registers them in AniDB as [generic](https://wiki.anidb.net/Filetype) files).

I only mention this because I'm a big fan of keeping track of the things I watch/own, and I've added files to AniDB myself.

Now, let's integrate Shoko into Jellyfin.

### Integrating Shoko with Jellyfin

To enable Jellyfin, there are a few steps involved, but generally it involves telling Jellyfin where the folder is, and Shoko will virtually restructure the folders and files the way Jellyfin needs them.

To implement this integration, Shoko offers a Jellyfin plugin called [Shokofin](https://github.com/ShokoAnime/Shokofin) that acts as a bridge between Shoko and Jellyfin.

#### Installing the Plugin

Shoko's [documentation](https://docs.shokoanime.com/jellyfin/installing-shokofin) shows how to install the plugin on Jellyfin. For that we go to *Dashboard -> Catalog -> Adjust (Gear) -> Add (+)*. We add an identifying name, in my case "Shokofin Stable" and the link to the JSON file provided by Shoko. We save the changes. This shows the repository and the plugin in the catalog:

![Shokofin](/assets/images/posts/localmediaserver3/shokofin.PNG)

![Shokofin2](/assets/images/posts/localmediaserver3/shokofin2.PNG)

After installing it, we restart the Jellyfin server so that the plugin finishes installing.

#### Configuring the Plugin

After the restart we access *Dashboard -> My plugins -> Shokofin (the 3 points) -> settings*.

![Shokofin3](/assets/images/posts/localmediaserver3/shokofin3.PNG)

Shoko's [documentation](https://docs.shokoanime.com/jellyfin/configuring-shokofin) explains all the plugin's options quite well. But to simplify things, I'll explain the options I selected to adapt it to my needs.

![ShokofinConfig1](/assets/images/posts/localmediaserver3/ShokofinConfig1.PNG)

Here I simply indicated the local IP and the port on which Shoko is working as well as the username and password for authentication.

![ShokofinConfig2](/assets/images/posts/localmediaserver3/ShokofinConfig2.PNG)

In this section, I've set the main title of the series/season/episode/description to be adapted to Shoko's specifications (in my case, Spanish/English/Romanji) and the alternate title to be decided by AniDB (mainly Romanji). I've also set useful links.

![ShokofinConfig3](/assets/images/posts/localmediaserver3/ShokofinConfig3.PNG)

In this section, the most important thing is to enable [VFS](https://en.wikipedia.org/wiki/Virtual_file_system) for libraries created using Shokofin. The plugin itself describes what it consists of:
> Enabling this feature allows you to disregard the underlying disk file structure while automagically meeting Jellyfin's requirements for file organization. It also ensures that no unrecognized files appear in your library and allows us to fully leverage Jellyfin's native features better than we otherwise could without it. This enables us to effortlessly support trailers, special features, and theme videos for series, seasons and movies, as well as merge partial episodes into a single episode. All this is possible because we disregard the underlying disk file structure to create our own using symbolic links. [1](https://docs.shokoanime.com/jellyfin/configuring-shokofin#library:~:text=Use%20the%20Virtual%20File%20System)

![ShokofinConfig4](/assets/images/posts/localmediaserver3/ShokofinConfig4.PNG)

This section is not so interesting, but basically it is what elements are considered in the VFS

![ShokofinConfig5](/assets/images/posts/localmediaserver3/ShokofinConfig5.PNG)

In this section, a Jellyfin user is synchronized with a Shoko user, allowing synchronization of the progress of the episodes watched with Shoko (and Shoko with AniDB).

![ShokofinConfig6](/assets/images/posts/localmediaserver3/ShokofinConfig6.PNG)

For this section, I prefer to use the description from the documentation:
> SignalR is a feature that allows for real-time communication with your running Shoko Server so that Shokofin has the ability to react to certain types of events such as file events and refresh events. This means that when Shoko Server identifies new media or updates metadata for your media, Shokofin can immediately import new or update existing media in your library. [1](https://docs.shokoanime.com/jellyfin/configuring-shokofin#signalr)

#### Creating a Library with Shokofin

The documentation also explains how to create the library using the Shokofin plugin as the metadata and image provider (also the VFS structure, but Shokofin does this in parallel).

Something worth mentioning is that the library must be created as a Shows/Movies or Mixed Movies and Shows library, but Shows is [recommended](https://docs.shokoanime.com/jellyfin/configuring-shokofin#content-type).

In my case, I created the Show library. I indicated where the series folder is, and Shoko as the metadata and image provider. Note that below the "Original" folder is the folder created by Shokofin's VFS; this is the one Jellyfin primarily uses to obtain series information and display them.

![ShokoLibraryFolder](/assets/images/posts/localmediaserver3/ShokofinLibraryFolder.png)

# Getting new anime series

### Direct Download

### Qbittorrnet

#### Automation of new episodes of the series currently being broadcast

# Unifying different folders into one

# Access servers/services directly without leaving the local network

notes:
I need to talk about the changes that have been made in the media server
* Automating the import/download of new files for use in Shoko/Jellyfin by Qbittorrent/Direct Download
* Unifying different folders into one for easier access and scalability
* Implementing a personal DNS/reverse proxy to redirect requests to Jellyfin/Shoko directly to the server without leaving the local network