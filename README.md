# openpublic

Run [opencode](https://opencode.ai) inside an always-fresh Docker container with
your **host GitHub credentials passed through automatically** — so git push,
`gh`, and private repos just work inside the container, every time it's recreated.

> ⚠️ Currently the container image expected is `opencode-warp` (opencode + warp CLI).
> A generic `opencode` image also works — set `OPENPUBLIC_IMAGE`.

## Quick install (paste-ready)

### For Linux/macOS with Docker

```bash
mkdir -p ~/.local/bin && curl -fsSL https://raw.githubusercontent.com/cybermanju/openpublic/main/openpublic -o ~/.local/bin/openpublic && chmod +x ~/.local/bin/openpublic && echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
```

### For proot-distro (Ubuntu in Termux/Android) — paste-ready

If you're running Ubuntu inside proot-distro on Termux, you'll need Docker. On many Android setups you can't run the full Docker daemon natively; options:

- **Use a remote Docker host** (e.g. your desktop via SSH tunnel/DOCKER_HOST) — recommended and easiest inside proot.
- **Run in a VM/container host that has Docker**.
- **Use rootless Docker only if your kernel supports it and it's available in your environment** (often not in pure proot without proper namespaces).

If you have access to Docker from inside the proot Ubuntu (via DOCKER_HOST pointing to a host/VM running dockerd), install prerequisites and run openpublic:

```bash
sudo apt update && sudo apt install -y curl git gh
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/cybermanju/openpublic/main/openpublic -o ~/.local/bin/openpublic
chmod +x ~/.local/bin/openpublic
export PATH="$HOME/.local/bin:$PATH"
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
# Point to your Docker daemon (replace with your host/VM IP/socket)
export DOCKER_HOST=tcp://YOUR_DOCKER_HOST:2375
# or: export DOCKER_HOST=unix:///var/run/docker.sock  (if mounted)
gh auth login  # authenticate on host that can reach Docker + GitHub
openpublic
```

If Docker isn't installed in the proot environment and you can't reach a daemon, install Docker inside the proot Ubuntu only if your proot setup supports it (some need additional kernel features). Otherwise use the remote Docker approach above.

Optional wrappers (muscle memory):

```bash
ln -sf ~/.local/bin/openpublic ~/.local/bin/opencode-d
printf '#!/usr/bin/env bash\nexec "$HOME/.local/bin/openpublic" -r "$@"\n' > ~/.local/bin/opencode-r
chmod +x ~/.local/bin/opencode-r
```

## Usage

```bash
openpublic        # attach, creating the container if missing
openpublic -r     # recreate the container fresh (container wiped, data dirs kept)
```

## How auth works (no credentials leave your machine)

1. The script reads the **active** `gh` token from your host keyring at launch time (`gh auth token`).
2. It writes that token to a temp file with mode `0600` and mounts it **read-only** into the container at `/run/gh-token`.
3. Inside the container:
   - `git` uses a credential helper that reads `/run/gh-token` on demand.
   - `opencode` runs with `GH_TOKEN` exported; `gh` (installed automatically if missing) also picks that up.

## Requirements

- Docker (local daemon or reachable via `DOCKER_HOST`)
- `gh` authenticated on the machine running the script: `gh auth status`
- Image `opencode-warp` (or override with `OPENPUBLIC_IMAGE`) with `opencode` in PATH. If `git`/`gh` are missing inside, the script installs them best-effort.

## Environment variables

| Var                  | Default                  | Meaning                      |
|----------------------|--------------------------|------------------------------|
| `OPENPUBLIC_IMAGE`   | `opencode-warp`          | Container image to run       |
| `OPENPUBLIC_WORKSPACE` | parent of current dir  | Host path mounted at `/workspace` |
| `DOCKER_HOST`        | (unset)                 | Use remote Docker daemon if needed |
| `OPENPUBLIC_UID_GID` | `$(id -u):$(id -g)`     | UID:GID for container user  |
| `OPENPUBLIC_EXEC_CMD` | `opencode`             | Override command to run    |

Ports forwarded: 13000→3000, 13213→3213, 14096→4096, 17654→7654.

## Notes

- The script runs as a non-root `dev` user inside the container (matching host UID/GID) so files created are owned by you, not root.
- On proot without Docker daemon, use a remote Docker host. Installing Docker inside proot varies by kernel/support.
- If mounting host system dirs is attempted, the script refuses for safety.

## License

MIT
