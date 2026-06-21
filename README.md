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
- **Minecraft version:** 1.26.1
- **Memory:** 2 GB
- **Plugins:** WorldEdit
- **Data directory:** `/opt/nijika-deablr/data`

To change settings, edit `compose.yml` and restart the container:

```bash
docker compose up -d
```

## Server Icon

`server-icon.png` is mounted into the container and used as the server icon.

## License

MIT
