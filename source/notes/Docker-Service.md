---
title: Docker-Service
type: notes
cover: img/notes.webp
---

## Minecraft JE Server

```bash
#!/bin/sh

docker run \
  --name minecraft \
  --restart=unless-stopped \
  --network bridge \
  --dns 223.6.6.6 \
  --dns 223.5.5.5 \
  -p 25565:25565 \
  -v /root/minecraft/data:/data \
  -v /root/minecraft/mods:/mods \
  -e TZ="Asia/Shanghai" \
  -e SERVER_NAME="Minecraft" \
  -e ICON="https://www.minecraft.net/content/dam/minecraftnet/games/minecraft/logos/Homepage_Gameplay-Trailer_MC-OV-logo_300x300.png" \
  -e OVERRIDE_ICON="TRUE" \
  -e FORCE_GAMEMODE="TRUE" \
  -e EULA="TRUE" \
  -e MOTD="Welcome to Minecraft" \
  -e MAX_PLAYERS=5 \
  -e INIT_MEMORY="512M" \
  -e MAX_MEMORY="3G" \
  -e VERSION=1.21.1 \
  -e TYPE="NEOFORGE" \
  -e MODS="/mods" \
  -e DIFFICULTY="hard" \
  -e MODE="survival" \
  -e VIEW_DISTANCE=14 \
  -e MAX_BUILD_HEIGHT=512 \
  -e LEVEL_TYPE="minecraft:large_biomes" \
  -d itzg/minecraft-server:java21-graalvm

```

### Server MOD List

- 【Carry on】
- 【Create】
- 【Dungeons and taverns】
- 【Dungeons and Taverns Stronghold Overhaul】
- 【Dungeons and Taverns Ancient City Overhaul】
- 【Grave Stone】
- 【Sit】
- 【Touhou Little Maid】
- 【UnionLib】【Antique Atlas】
- 【The Twilight Forest】
- 【Distant Horizons】
- 【Geophilic】
- 【oωo (owo-lib)】【The Aether】
- 【Subsurface】
- 【Traveler's Backpack】
- 【Structory】
- 【Structory: Towers】
- 【Incendium】
- 【Curios API】
- 【Tutta's Doors】
- 【Another Furniture】
- 【Blueprint / Abnormals Core】【Autumnity】
- 【Farmer's Delight】
- 【Immersive Aircraft】
- 【Storage Drawers】
- 【RoadWeaver】

### Client MOD List

- 【Apple Skin】
- 【Carry On】
- 【Clean Swing】
- 【Create】
- 【Grave Stone】
- 【Iris】【Sodium】
- 【Just Enough Items】
- 【Sit】
- 【Touhou Little Maid】
- 【UnionLib】【Antique Atlas】
- 【The Twilight Forest】
- 【Distant Horizons】
- 【oωo (owo-lib)】【The Aether】
- 【Traveler's Backpack】
- 【Curios API】
- 【Tutta's Doors】
- 【Another Furniture】
- 【Blueprint / Abnormals Core】【Autumnity】
- 【Farmer's Delight】
- 【Immersive Aircraft】
- 【Storage Drawers】
- 【RoadWeaver】
- 【I18nUpdateMod】

## qBittorrent

需要用公网 IPv6 所以不能桥接。不要同时开启 Nikki服务。

```bash
#!/bin/sh

docker run \
	--name qbittorrent \
	--network host \
	--restart unless-stopped \
	-e TZ="Asia/Shanghai" \
	-e WEBUI_PORT=8081 \
	-e TORRENTING_PORT=6881 \
	-v /root/qbittorrent:/config \
	-v /root/share/qbittorrent:/downloads \
	-d linuxserver/qbittorrent:latest

```

