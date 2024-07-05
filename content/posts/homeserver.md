---
title: "Switching to a self hosted paradigm."
date: 2024-07-05T00:00:00+00:00
tags: ["home server", "docker"]
showToc: false
TocOpen: false
draft: false
hidemeta: true
comments: true
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "images/serverdocker.webp"
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/erdeicodrut/personal-website/blob/master/content/"
    Text: "Suggest Changes"
    appendFilePath: true # to append file path to Edit link
---

In the past years I've always wanted to have a home server for some reason, but my formation as a programmer who did leetcode style problems from the age of 10 made it really hard for me to bear the inefficiency of having a Linux machine running 24/7 in my home.

Having been semi-cured of that issue let's jump in the actual config.

### The config

Starting out with the computer I chose for a server: a Dell OptiPlex 7050 Micro that I bought refurbished from a local company for around $140, plus I had to get an SSD because the one that came with it had too many bad sectors;
so I got an SSD for ~$35 and that was that.
![dell](images/DellOptiPlex.webp)

### Services

I started gathering all the needed software that was going to be useful around the internet.

#### Media

What I needed was a replacement for:

- All the Movies/TV Series streaming services because most of the stuff I wanna watch is either not available in my country or not on the streaming platforms.
- Music because I recently completed my listening setup and streaming services don't satisfy anymore. I still keep my YouTube Music subscription and use it, but when it comes to listening to full albums FLACs are the way to go.
- Some kind of iCloud replacement.
- Password manager
- Pocket replacement. I would've said bookmark manager, but it's not even that, I've been using Pocket for a long while and it just got worse and worse over the years.

And some nice-to-haves

- Something to monitor everything and get notified if they're down.
- A reverse proxy to be able to go to `cloud.erdeiserver.us` rather than `192.162.0.10:8080`
- A dashboard to access everything from

> Small note: I have a Synology NAS already, that handles all my photo backup stuff with a 2x4TB NAS grade HDDs in raid.

**PSA: A very important thing when you have a home server is protection in case of power loss.** For this I have a very dumb UPS connected that holds my NAS and server up and running for at least 2 hours. In my use case, it's the perfect setup because I have frequent power spikes and drops for 2-3 minutes and it comes back.

So the dashboard looks like this

![homarr](images/homarr.png)


This being said everything I have is here.

Now get ready for the whole docker-compose file.

```yaml
services:
  nginxproxymanager:
    image: "jc21/nginx-proxy-manager:github-pr-3121"
    container_name: nginxproxymanager
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - ${HOME}/storage/nginx/data:/data
      - ${HOME}/storage/nginx/letsencrypt:/etc/letsencrypt
      - ${HOME}/storage/nginx/config.json:/app/config/production.json

  nextcloud:
    image: lscr.io/linuxserver/nextcloud:latest
    container_name: nextcloud
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Bucharest
    volumes:
      - ${HOME}/storage/nextcloud/appdata:/config
      - ${HOME}/storage/nextcloud/data:/data
    restart: unless-stopped

  homeassistant:
    image: lscr.io/linuxserver/homeassistant:latest
    container_name: homeassistant
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Bucharest
      - TRUSTED_PROXIES=172.18.0.12

    volumes:
      - ${HOME}/storage/hass/config:/config
    restart: unless-stopped

  plex:
    image: lscr.io/linuxserver/plex
    container_name: plex
    environment:
      - PUID=1000
      - PGID=1000
      - VERSION=docker
      - PLEX_CLAIM=claim-B7znMtSSnu-YWd-Rv9Jy
    volumes:
      - ${HOME}/storage/configs:/config
      - ${HOME}/storage/tv:/tv
      - ${HOME}/storage/movies:/movies
    restart: unless-stopped
    ports:
      - 32400:32400
      - 1900:1900/udp
      - 5353:5353/udp
      - 8324:8324
      - 32410:32410/udp
      - 32412:32412/udp
      - 32413:32413/udp
      - 32414:32414/udp
      - 32469:32469

  overseerr:
    image: lscr.io/linuxserver/overseerr:latest
    container_name: overseerr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - ${HOME}/storage/overseerr/config:/config
    ports:
      - 5055:5055
    restart: unless-stopped

  homarr:
    container_name: homarr
    image: ghcr.io/ajnart/homarr:latest
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ${HOME}/storage/homarr/configs:/app/data/configs
      - ${HOME}/storage/homarr/icons:/app/public/icons
      - ${HOME}/storage/homarr/data:/data
    ports:
      - "7575:7575"

  radarr:
    image: linuxserver/radarr
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Bucharest
      - UMASK_SET=022
    volumes:
      - ${HOME}/storage/configs/Radarr:/config
      - ${HOME}/storage/downloads:/downloads
      - ${HOME}/storage/movies:/movies
    ports:
      - 7878:7878
    restart: unless-stopped

  lidarr:
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Eruope/Bucharest
    volumes:
      - ${HOME}/storage/configs/lidarr:/config
      - ${HOME}/storage/music:/music
      - ${HOME}/storage/downloads:/downloads
    ports:
      - 8686:8686
    restart: unless-stopped

  slskd:
    ports:
      - 5030:5030/tcp
      - 5031:5031/tcp
      - 50300:50300/tcp
    volumes:
      - ${HOME}/storage/configs/slskd:/app:rw
      - ${HOME}/storage/music:/music:rw
    user: 1000:1000
    image: slskd/slskd:latest

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Eruope/Bucharest
    volumes:
      - ${HOME}/storage/configs/prowlarr:/config
    ports:
      - 9696:9696
    restart: unless-stopped

  sonarr:
    image: linuxserver/sonarr
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Bucharest
      - UMASK_SET=022
    volumes:
      - ${HOME}/storage/configs/Sonarr:/config
      - ${HOME}/storage/downloads:/downloads
      - ${HOME}/storage/tv:/tv
    ports:
      - 8989:8989
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - WEBUI_PORT=8080
      - TORRENTING_PORT=6881
    volumes:
      - ${HOME}/storage/qbittorrent/appdata:/config
      - ${HOME}/storage/downloads:/downloads
      - ${HOME}/storage/tv:/tv
      - ${HOME}/storage/movies:/movies
    ports:
      - 8080:8080
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped

  navidrome:
    image: deluan/navidrome:latest
    ports:
      - "4533:4533"
    environment:
      ND_SCANSCHEDULE: 1h
      ND_LOGLEVEL: info
    volumes:
      - ${HOME}/storage/configs/navidrome:/data
      - ${HOME}/storage/music:/music:ro

  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    volumes:
      - ${HOME}/storage/configs/vw-data/:/data/
    restart: unless-stopped

  wallabag:
    image: wallabag/wallabag
    environment:
      - MYSQL_ROOT_PASSWORD=wallaroot
      - SYMFONY__ENV__DATABASE_DRIVER=pdo_mysql
      - SYMFONY__ENV__DATABASE_HOST=db
      - SYMFONY__ENV__DATABASE_PORT=3306
      - SYMFONY__ENV__DATABASE_NAME=wallabag
      - SYMFONY__ENV__DATABASE_USER=wallabag
      - SYMFONY__ENV__DATABASE_PASSWORD=wallapass
      - SYMFONY__ENV__DATABASE_CHARSET=utf8mb4
      - SYMFONY__ENV__DATABASE_TABLE_PREFIX="wallabag_"
      - SYMFONY__ENV__MAILER_DSN=smtp://127.0.0.1
      - SYMFONY__ENV__DOMAIN_NAME=https://wall.erdeiserver.us
      - SYMFONY__ENV__SERVER_NAME="Wallabag"
    ports:
      - 9010:80
    volumes:
      - ${HOME}/storage/wallabag/images:/var/www/wallabag/web/assets/images
    healthcheck:
      test:
        [
          "CMD",
          "wget",
          "--no-verbose",
          "--tries=1",
          "--spider",
          "https://wall.erdeiserver.us",
        ]
      interval: 1m
      timeout: 3s
    depends_on:
      - db
      - redis
  db:
    image: mariadb
    environment:
      - MYSQL_ROOT_PASSWORD=wallaroot
    volumes:
      - ${HOME}/storage/wallabag/data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 20s
      timeout: 3s
  redis:
    image: redis:alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 20s
      timeout: 3s

  uptime-kuma:
    container_name: uptime-kuma
    image: louislam/uptime-kuma:latest
    volumes:
      - ${HOME}/storage/configs/kuma/data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - 3001:3001
      - 3307:3306

  ### And some stuff my wife asked me for and I didn't get to organize nicely
  proxy: # The proxy must not be removed. If needed, point your own proxy to this container, rather than removing this
    container_name: recipesage_proxy
    image: julianpoy/recipesage-selfhost-proxy:v4.0.0
    ports:
      - 7270:80
    depends_on:
      - static
      - api
      - pushpin
    restart: unless-stopped
  static: # Hosts frontend assets
    container_name: recipesage_static
    image: julianpoy/recipesage-selfhost:static-v2.14.6
    restart: unless-stopped
  api: # Hosts backend API
    container_name: recipesage_api
    image: julianpoy/recipesage-selfhost:api-v2.14.6
    depends_on:
      - postgres
      - typesense
      - pushpin
      - browserless
    command: sh -c "npx prisma migrate deploy; npx ts-node --swc --project packages/backend/tsconfig.json packages/backend/src/bin/www"
    environment:
      - STORAGE_TYPE=filesystem
      - FILESYSTEM_STORAGE_PATH=/rs-media
      - NODE_ENV=selfhost
      - VERBOSE=false
      - VERSION=selfhost
      - POSTGRES_DB=recipesage_selfhost # If changing this, make sure to update the postgres container and the DATABASE_URL below accordingly
      - POSTGRES_USER=recipesage_selfhost # If changing this, make sure to update the postgres container and the DATABASE_URL below accordingly
      - POSTGRES_PASSWORD=recipesage_selfhost # If changing this, make sure to update the postgres container and the DATABASE_URL below accordingly
      - POSTGRES_PORT=5432 # If changing this, make sure to update the postgres container and the DATABASE_URL below accordingly
      - POSTGRES_HOST=postgres # If changing this, make sure to update the postgres container and the DATABASE_URL below accordingly
      - POSTGRES_SSL=false
      - POSTGRES_LOGGING=false
      - DATABASE_URL=postgresql://recipesage_selfhost:recipesage_selfhost@postgres:5432/recipesage_selfhost
      - GCM_KEYPAIR
      - SENTRY_DSN
      - GRIP_URL=http://pushpin:5561/
      - GRIP_KEY=changeme
      - SEARCH_PROVIDER=typesense
      - 'TYPESENSE_NODES=[{"host": "typesense", "port": 8108, "protocol": "http"}]'
      - TYPESENSE_API_KEY=recipesage_selfhost
      - STRIPE_SK # Value should not be set.
      - STRIPE_WEBHOOK_SECRET # Value should not be set
      - BROWSERLESS_HOST=browserless
      - BROWSERLESS_PORT=3000
      - INGREDIENT_INSTRUCTION_CLASSIFIER_URL=http://ingredient-instruction-classifier:3000/
      - OPENAI_API_KEY # Please follow the instructions in the README if you decide to supply a value here
    volumes:
      - ${HOME}/storage/food/apimedia:/rs-media
    restart: unless-stopped
  typesense: # Provides the fuzzy search engine
    container_name: recipesage_typesense
    image: typesense/typesense:0.24.1
    volumes:
      - ${HOME}/storage/food/typesensedata:/data
    command: "--data-dir /data --api-key=recipesage_selfhost --enable-cors"
    restart: unless-stopped
  pushpin: # Provides websocket support
    container_name: recipesage_pushpin
    image: julianpoy/pushpin:2023-09-17
    entrypoint: /bin/sh -c
    command:
      [
        'sed -i "s/sig_key=changeme/sig_key=$$GRIP_KEY/" /etc/pushpin/pushpin.conf && echo "* $${TARGET},over_http" > /etc/pushpin/routes && pushpin --merge-output',
      ]
    environment:
      - GRIP_KEY=changeme
      - TARGET=api:3000
    restart: unless-stopped
  postgres: # Database
    container_name: recipesage_postgres
    image: postgres:16
    environment:
      - POSTGRES_DB=recipesage_selfhost # If you change this, make sure to change both POSTGRES_DB and DATABASE_URL on the API container
      - POSTGRES_USER=recipesage_selfhost # If you change this, make sure to change both POSTGRES_USER and DATABASE_URL on the API container
      - POSTGRES_PASSWORD=recipesage_selfhost # If you change this, make sure to change both POSTGRES_PASSWORD and DATABASE_URL on the API container
    volumes:
      - ${HOME}/storage/food/postgresdata:/var/lib/postgresql/data
    restart: unless-stopped
  browserless: # A headless browser for scraping websites with the auto import tool
    container_name: recipesage_browserless
    image: browserless/chrome:1.61.0-puppeteer-21.4.1
    environment:
      - MAX_CONCURRENT_SESSIONS=3
      - MAX_QUEUE_LENGTH=10
    restart: unless-stopped

  ingredient-instruction-classifier:
    container_name: recipesage_classifier
    image: julianpoy/ingredient-instruction-classifier:1.4.11
    environment:
      - SENTENCE_EMBEDDING_BATCH_SIZE=200
      - PREDICTION_CONCURRENCY=2
    restart: unless-stopped

volumes:
  apimedia:
    driver: local
  typesensedata:
    driver: local
  postgresdata:
    driver: local
```

I suggest you use Portainer to create this docker-compose stack because it's a bit easier to manage than with the CLI, but Portainer itself will have to be started with the CLI.

That looks like this
![portainer](images/portainer.png)


I left all passwords as default in this file so you can change them when you copy and paste everything, but it will run like this as well, it's a matter of security only.

Now, how I suggest you go about these things is to check each container one by one and try to look up on Google what it does and how you have to set them up. For most of them, it is as easy as either looking up the default credentials or setting them up yourself.

The more complicated part is setting up nginx proxy manager, but [here is the article](https://notthebe.ee/blog/easy-ssl-in-homelab-dns01/) that helped me set it up in less than 20 minutes.

The R suite (`overseerr` `radarr` `lidarr` `prowlarr` `sonarr`), is basically what will help you get all your media downloaded, needs a torrent client so there is `qBitTorrent`. Easy to set up and get working.

Some things to keep in mind to make your life easy:
- Docker compose stacks have a network of their own, if you want to refer to a service it has an internal IP address that you see in Portainer. You don't need to have the ports forwarded if you need to connect 2 containers. Just use their internal IP to refer them.
- Anything can be deleted and things will still work, it's not necessarily always true, but it's true enough for you to play with things.
- It will take a long time to start because you have to download lots of containers, be patient, or add each service after you read about it a bit.
- You will need to SSH into your system often so it's great to set up an SSH key for easier authentication.
- Please change your passwords from the default ones.
