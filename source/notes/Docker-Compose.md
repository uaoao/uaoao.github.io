---
title: Docker-Compose
type: notes
cover: img/notes.webp
---

## fedora server compose.yml

```yaml
services:
  fedora-docker:
    image: library/fedora
    network_mode: "host"
    volumes:
      - ./root:/root
      - ./home:/home
    tty: true
    stdin_open: true
    restart: unless-stopped

```

## minecraft server compose.yml

```yaml
services:
  minecraft:
    image: itzg/minecraft-server
    container_name: "MCServer"
    ports:
      - 25565:25565
    volumes:
      - ./minecraft-data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    environment:
      SERVER_NAME: "MCServer"
      ICON: "https://www.minecraft.net/content/dam/minecraftnet/games/minecraft/logos/Homepage_Gameplay-Trailer_MC-OV-logo_300x300.png"
      VERSION: "LATEST"
      OVERRIDE_ICON: "FALSE"
      EULA: "TRUE"
      INIT_MEMORY: "512M"
      MAX_MEMORY: "4G"
      TYPE: "VANILLA"
      MOTD: "Welcome to Minecraft"
      DIFFICULTY: "hard"
      MAX_PLAYERS: 40
      FORCE_GAMEMODE: true
      MAX_BUILD_HEIGHT: 512
      MODE: "survival"
      SETUP_ONLY: false
      VIEW_DISTANCE: 10
      PVP: true
      LEVEL_TYPE: "minecraft:large_biomes"
      USE_AIKAR_FLAGS: false
      NETWORK_COMPRESSION_THRESHOLD: 512
      ENABLE_RCON: true
      RCON_PASSWORD: "1j9fR7_#9~Vs"
      RCON_PORT: 25575
    restart: unless-stopped

  rcon:
    image: itzg/rcon
    container_name: "RCON"
    depends_on:
      - minecraft
    ports:
      - "3000:4326"
      - "4327:4327"
    volumes:
      - ./rcon-data:/opt/rcon-web-admin/db
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    environment:
      RWA_ENV: "TRUE"
      RWA_ADMIN: "TRUE"
      RWA_PASSWORD: "Juf7%n.0pFg1"
      RWA_RCON_HOST: "MCServer"
      RWA_RCON_PASSWORD: "1j9fR7_#9~Vs"
      RWA_RCON_PORT: 25575
    restart: unless-stopped

```

