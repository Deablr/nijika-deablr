# Nijika Deablr Minecraft Server

A Docker Compose setup for the Nijika Deablr Paper Minecraft server.

## Overview

This repository contains the configuration to run a Paper Minecraft server using the [`itzg/minecraft-server`](https://github.com/itzg/docker-minecraft-server) Docker image.

## Requirements

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)

## Quick Start

```bash
docker compose up -d
```

The server will be available on the default Minecraft port:

```
25565
```

## Configuration

Server settings are defined in [`compose.yml`](compose.yml). Key settings include:

- **Server type:** Paper
- **Minecraft version:** 26.3
- **Memory:** 2 GB
- **Plugins:** WorldEdit, Floodgate
- **Data directory:** `/opt/nijika-deablr/data`

To change settings, edit `compose.yml` and restart the container:

```bash
docker compose up -d
```

## Bedrock Support

Bedrock Edition players will be served by a shared [Geyser](https://geysermc.org/) instance running alongside mc-router (see [`handoff.md`](handoff.md); not deployed yet). This server only runs [Floodgate](https://geysermc.org/wiki/floodgate/), which lets Bedrock players join without a Java account. It does nothing until the shared Geyser is live.

Bedrock player names are prefixed with `.` (e.g. `.Steve`). To op a Bedrock player, use their in-game name: `/op .Steve`.

## Server Icon

`server-icon.png` is mounted into the container and used as the server icon.

## License

MIT
