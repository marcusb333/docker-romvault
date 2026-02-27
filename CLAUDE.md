# CLAUDE.md

## Project Overview

Docker container for [ROMVault](https://www.romvault.com) — a ROM management tool. The GUI is served via web browser or VNC client using `jlesage/baseimage-gui` as the base image.

## Repository Structure

```
Dockerfile              # Main build (Ubuntu 20.04 + Mono, auto-detects arch)
Dockerfile.alpine       # Alpine-based variant
Dockerfile.ubuntu       # Ubuntu-based variant
Dockerfile.template     # Template used by update.py to generate Dockerfile
docker-compose.yml      # Local build compose config (ARM64/Mac)
setup.sh                # Local build and container management script
update.py               # CLI tool to build/run/push versioned Docker images
requirements.txt        # Python deps for update.py (typer, rich)
rootfs/                 # Files copied into the container image
  startapp.sh           # Container entrypoint that launches ROMVault
  etc/cont-init.d/      # Container initialization scripts
  etc/openbox/          # Openbox window manager config
```

## Common Commands

### Local Development (setup.sh)

```bash
# Download ROMVault*.zip from romvault.com and place it in the project root, then:
./setup.sh run       # Create dirs, build image, start container
./setup.sh build     # Build image only
./setup.sh start     # Start existing stopped container
./setup.sh stop      # Stop running container
./setup.sh restart   # Restart container
./setup.sh status    # Show container status
./setup.sh logs      # Tail container logs
./setup.sh clean     # Remove container and image
```

### update.py (versioned builds for publishing)

```bash
pip install -r requirements.txt

python update.py 3.7.4              # Process template, build image, run container
python update.py 3.7.4 --no-run    # Build only, don't run
python update.py 3.7.4 --push      # Push versioned tag to Docker Hub
python update.py 3.7.4 --push --tag-latest  # Also tag as latest
```

### Docker Compose (ARM64/Mac)

```bash
docker compose up -d --build
```

## Architecture Notes

- **x86_64**: Pre-built image on Docker Hub (`laromicas/romvault`). Uses `mono-complete`.
- **ARM64/Mac**: Must build locally. Mono runs in interpreter mode — requires `MONO_ENV_OPTIONS=--interpreter`.
- The `Dockerfile` detects architecture at build time (`dpkg --print-architecture`) and installs the appropriate Mono packages.

## Key Ports & Volumes

| Port | Use |
|------|-----|
| 5800 | Web UI (HTTP) |
| 5900 | VNC |

| Container Path | Description |
|----------------|-------------|
| `/config` | App config, state, logs |
| `/config/DatRoot` | DAT files |
| `/config/RomRoot` | Organized ROM files |
| `/config/ToSort` | Unsorted ROMs |

## Environment Variables

Key vars passed to the container:

| Variable | Default | Notes |
|----------|---------|-------|
| `USER_ID` / `GROUP_ID` | `1000` | File ownership in volumes |
| `TZ` | `Etc/UTC` | Timezone |
| `MONO_ENV_OPTIONS` | — | Set to `--interpreter` on ARM64 |
| `IONICE_CLASS` / `IONICE_LEVEL` | — | Disk I/O priority for ROMVault process |
| `KEEP_APP_RUNNING` | `0` | Auto-restart on crash |
| `DARK_MODE` | `0` | Enable dark theme |

## Dockerfile Template

`Dockerfile.template` uses `{{version_lower}}`, `{{version_upper}}`, and `{{docker_version}}` placeholders. `update.py` substitutes these and writes `Dockerfile` before building.

## CI/CD

GitHub Actions workflow at `.github/workflows/build-and-push.yml` builds and pushes the image to Docker Hub automatically on push.
