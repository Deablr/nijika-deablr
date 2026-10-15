# Handoff: Shared Geyser in front of mc-router

## Goal

Let Bedrock players join **any** Deablr server by typing its normal hostname (e.g. `nijika.deablr.com`) with no port, while:

- adding a new server stays "edit mc-router `MAPPING`, done", with no per-server Bedrock wiring
- there's no third system to manage (everything lives in the existing mc-router Dokploy project)

## Architecture

```
Bedrock client ──UDP 19132──▶ geyser-deablr  (Geyser Standalone, in the mc-router compose project)
                                   │ joins as a Java client, forwarding the hostname the player typed
                                   ▼
                               mc-router-deablr:25565 ──▶ routes by hostname, same as Java today
                                   ▼
                     nijika-deablr-mc / richard-deablr-mc / ...  (each runs the Floodgate plugin + shared key)
```

### Why this works (verified 2026-10-15)

- **Geyser `java.forward-hostname: true`** sends the hostname the Bedrock client connected with (`session.joinAddress()`) in the Java handshake, instead of Geyser's own target address. See `GeyserSessionAdapter.java` in GeyserMC/Geyser.
- **mc-router strips Floodgate's payload.** Floodgate appends its encrypted data after a `\0`, and mc-router already does `strings.Split(serverAddress, "\x00")[0]` (meant for Forge) before matching a route (`server/routes.go:279`). There's no hostname length cap in its `ReadString`.
- **Tested end to end locally:** a Bedrock bot (gophertunnel) went through Geyser, then mc-router, and spawned on two separate Paper 26.3 backends by hostname:
  ```
  [BedrockNijika]  SPAWNED via nijika.deablr.com:19132
  [BedrockRichard] SPAWNED via richard.deablr.com:19132
  ```
  mc-router logged the correct backend per hostname, and Floodgate on each backend logged `.BedrockX joined the game` with the real client IP.

### Things that did NOT work / aren't options

- **Dokploy domains / Traefik:** domains are HTTP routers matched by hostname. Bedrock is UDP (RakNet) and carries no hostname at the transport level, so there's nothing to route on.
- **mc-router itself:** Java/TCP only. The maintainer declined Bedrock/UDP support (itzg/mc-router #14, #189, #337).
- **Geyser as a plugin on each server:** only one container can bind host UDP 19132, so only one server would get portless Bedrock.

### Test-only caveat

The test bot had no Xbox account, so the test ran with Geyser `validate-bedrock-login: false` and Floodgate `player-link.enabled: false`. **Production keeps both at their defaults (`true`).** Real Bedrock clients always have an Xbox account/XUID, so this doesn't affect real players, but Xbox sign-in itself wasn't exercised in the test. Verify it with a real Bedrock client after deploying.

---

## Implementation

### 1. Shared Floodgate key (once, on the VPS)

Floodgate's `key.pem` is just 16 random bytes. Geyser and every backend must have the **same** file.

```bash
sudo mkdir -p /opt/deablr/floodgate
sudo openssl rand -out /opt/deablr/floodgate/key.pem 16
sudo chmod 644 /opt/deablr/floodgate/key.pem   # containers run as uid 1000; must be readable
```

This is a secret. **Never commit it.** Back it up somewhere safe; if it's lost, generate a new one and restart Geyser plus all backends.

### 2. mc-router repo (`Deablr/mc-router-deablr`)

New files:

```
geyser/
├── Dockerfile
└── config.yml
```

**`geyser/Dockerfile`** (`eclipse-temurin:25-jre` has no curl/wget, so `ADD` fetches the jar at build time. The build is pinned for reproducibility; bump the URL to upgrade.)

```dockerfile
FROM eclipse-temurin:25-jre
ADD https://download.geysermc.org/v2/projects/geyser/versions/2.12.0/builds/1258/downloads/standalone /opt/geyser/Geyser-Standalone.jar
COPY config.yml /opt/geyser/config.yml
WORKDIR /geyser
# Geyser rewrites its config on boot, so copy the repo version in fresh every start
CMD ["sh", "-c", "cp /opt/geyser/config.yml /geyser/config.yml && exec java -Xms256M -Xmx1G -jar /opt/geyser/Geyser-Standalone.jar --nogui"]
```

> To find a newer build: `curl -s https://download.geysermc.org/v2/projects/geyser/versions/latest/builds/latest | jq '{version, build}'`. Geyser must support the backends' Java version (check the "Java" row in the Geyser README).

**`geyser/config.yml`**: generate the default, then edit it. Don't hand-write it; the schema changes between versions (`config-version: 10` as of 2.12.0).

```bash
mkdir /tmp/geyser-gen && cd /tmp/geyser-gen
curl -sL -o Geyser-Standalone.jar https://download.geysermc.org/v2/projects/geyser/versions/2.12.0/builds/1258/downloads/standalone
docker run --rm -it -v "$PWD":/g -w /g eclipse-temurin:25-jre java -jar Geyser-Standalone.jar --nogui
# wait for "Done", type `geyser stop` (or Ctrl-C), then copy /tmp/geyser-gen/config.yml into geyser/config.yml
```

Then change these keys:

| Key | Default | Set to | Why |
| --- | --- | --- | --- |
| `java.address` | `127.0.0.1` | `mc-router-deablr` | Connect through the router |
| `java.port` | `25565` | `25565` | (unchanged) |
| `java.auth-type` | `online` | `floodgate` | Bedrock players don't need Java accounts |
| `java.forward-hostname` | `false` | `true` | **The key setting.** Lets mc-router route by hostname |
| `motd.primary-motd` | `Geyser` | e.g. `Deablr` | Shown in Bedrock server list |
| `motd.secondary-motd` | `Another Geyser server.` | e.g. `nijika / richard .deablr.com` | |
| `motd.passthrough-motd` | `true` | `false` | Ping has no hostname, so mc-router can't route it and Geyser spams `Failed to ping the remote Java server!` |
| `motd.passthrough-player-counts` | `true` | `false` | Same reason |
| `gameplay.server-name` | `Geyser` | e.g. `Deablr` | |
| `advanced.floodgate-key-file` | `key.pem` | `key.pem` | (unchanged; mounted at `/geyser/key.pem`) |

Leave `advanced.bedrock.validate-bedrock-login: true`.

**`compose.yml`**: add the service and a volume (the existing `mc-router` service is unchanged except for the header comment, if you like):

```yaml
  geyser:
    build: ./geyser
    container_name: geyser-deablr
    tty: true
    stdin_open: true
    restart: unless-stopped
    depends_on:
      - mc-router
    # Bedrock is UDP; neither mc-router nor Traefik can carry it, so Geyser binds it directly
    ports:
      - "19132:19132/udp"
    volumes:
      - geyser-data:/geyser
      - /opt/deablr/floodgate/key.pem:/geyser/key.pem:ro
    networks:
      - dokploy-network

volumes:
  geyser-data:
```

**`README.md`**: add the Bedrock flow, port `19132/udp`, the shared key location, and the backend requirements below.

**Host firewall:** open **UDP 19132** on the VPS (and in the Hostinger panel firewall, if enabled).

### 3. Each backend server (start with this repo, `nijika-deablr`)

> **State of this repo:** already on 26.3 / WorldEdit 7.4.6 with the Floodgate plugin installed and no Geyser. The only change left here is the key mount. It was held back on purpose: if `/opt/deablr/floodgate/key.pem` doesn't exist on the host yet, Docker creates a *directory* at that path and Floodgate breaks. Create the key (step 1) first.

In `compose.yml`, **add** the shared key mount:

```yaml
volumes:
  - "/opt/nijika-deablr/data:/data"
  - "./server-icon.png:/server-icon.png:ro"
  - "/opt/deablr/floodgate/key.pem:/data/plugins/floodgate/key.pem:ro"
```

Keep `ENFORCE_SECURE_PROFILE: "false"` (needed for Bedrock chat).

README: update the "Bedrock Support" section to drop "not deployed yet" and mention the shared key mount. Bedrock players use `nijika.deablr.com` with no port.

Repeat for `richard-deablr-mc`: add the Floodgate line to `PLUGINS`, plus the key mount.

### 4. Deploy order

1. Create the key on the VPS (step 1).
2. Back up `/opt/nijika-deablr/data` (and Richard's data dir). The 26.1.2 → 26.3 world upgrade is one-way.
3. Redeploy each backend in Dokploy (Floodgate installs, reads the shared key).
4. Redeploy mc-router-deablr in Dokploy (builds and starts Geyser).
5. Open UDP 19132.

### 5. Verify

- `docker logs geyser-deablr` shows `Started Geyser on UDP port 19132`, with no `XBOX AUTHENTICATION IS DISABLED` error.
- Backend logs show Floodgate loading with no key errors.
- From a real Bedrock client: Servers → Add Server → address `nijika.deablr.com`, port `19132` (default). It should join Nijika. Repeat with `richard.deablr.com` and confirm it lands on Richard's world.
- `docker logs mc-router-deablr` shows `server=nijika.deablr.com ... backendHostPort=nijika-deablr-mc:25565` for the Bedrock join.
- Java joins still work unchanged.

### Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Bedrock "Unable to connect to world" immediately | UDP 19132 not open on the VPS / provider firewall, or Geyser not running |
| `An internal error happened while handling Floodgate data` | Backend has a different `key.pem` than Geyser, or the backend lacks Floodgate. Check the mount on both sides |
| Disconnected with no backend log entry | Hostname has no `MAPPING` entry in mc-router (same as Java) |
| `Failed to ping the remote Java server!` spam in Geyser | `passthrough-motd` / `passthrough-player-counts` still `true` |
| Player lands on wrong server | DNS/hostname typo; check mc-router log `server=` field |

## Known limitations

- Bedrock's server list shows one fixed MOTD/player count for all hostnames (Geyser's ping can't carry a hostname).
- All Bedrock traffic shares one Geyser JVM; fine at this scale (-Xmx1G).
- New servers that should accept Bedrock must add Floodgate plus the key mount. Servers without it reject Bedrock joins, which is harmless.
