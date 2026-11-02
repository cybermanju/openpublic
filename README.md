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

### One-command setup (installs everything automatically)

Paste this on a fresh Linux machine. It installs Docker, the GitHub CLI (`gh`), `curl`, downloads `openpublic` to `~/.local/bin`, configures your PATH, authenticates `gh`, pulls the `opencode-warp` image, and runs a version check:

```bash
set -e
# 1. System prerequisites
sudo apt-get update -y
sudo apt-get install -y curl ca-certificates gnupg lsb-release git

# 2. Docker (official repo, if not already installed)
if ! command -v docker >/dev/null 2>&1; then
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  sudo apt-get update -y
  sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
  sudo usermod -aG docker $USER
  newgrp docker
fi

# 3. GitHub CLI (gh)
if ! command -v gh >/dev/null 2>&1; then
  type -p curl >/dev/null || sudo apt-get install -y curl
  curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
  sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
  sudo apt-get update -y
  sudo apt-get install -y gh
fi

# 4. Install openpublic to ~/.local/bin
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/cybermanju/openpublic/main/openpublic -o ~/.local/bin/openpublic
chmod +x ~/.local/bin/openpublic
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/.local/bin:$PATH"

# 5. Authenticate with GitHub (needed for git push / private repos inside the container)
gh auth login

# 6. Pull the opencode-warp image
docker pull opencode-warp:latest

# 7. Verify opencode runs (unauthenticated, bypasses server-side cap)
openpublic -u -- --version
```

After step 5 completes (browser auth flow), run:

```bash
openpublic          # attach (authenticated, tied to your GH account cap)
openpublic -r       # recreate fresh: new IP + wiped state (resets IP-based bans)
openpublic -r -u    # recreate fresh AND unauthenticated: bypasses GH server-side cap entirely
openpublic -u -- --prompt "your message here"   # send one prompt, no rate-limit ban
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
openpublic                 # attach, creating the container if missing (authenticated via GH_TOKEN)
openpublic -r              # recreate container fresh: new IP + wiped mutable state (resets bans)
openpublic -u              # run unauthenticated (no GH_TOKEN) — bypasses GitHub server-side cap
openpublic -r -u           # full reset: new IP + fresh device ID + no GH_TOKEN
openpublic -u -- --prompt "hello"   # send one prompt to opencode (default model), no rate-limit ban
openpublic -r -u -- run "msg" --model opencode/nemotron-3.5-lightning-free   # different free model
openpublic --continue      # continue the last opencode session inside the container
```

Everything after `--` is passed through to `opencode` verbatim. Use `-r` to get a new container IP
(Cloudflare/WARP identity reset) and wipe tabs/rate-limit state; use `-u` to run with a fresh
opencode device identity (empty data dirs, no stored auth, randomized `OPENCODE_CLIENT`) and no
`GH_TOKEN`.

### Bypassing OpenCode's server-side free-tier limit

OpenCode's free tier is rate-limited **per model per day**. Each free model has a separate limit.
When the default model (`opencode/mimo-v2.6-flash-free`) hits its cap, switch models:

```bash
openpublic -r -u -- run "your message" --model opencode/nemotron-3.5-lightning-free
openpublic -r -u -- run "your message" --model opencode/big-pickle
openpublic -r -u -- run "your message" --model opencode/deepseek-v4.1-flash-free
openpublic -r -u -- run "your message" --model opencode/flint-40b-free
openpublic -r -u -- run "your message" --model opencode/glm-5.3-flash-free
openpublic -r -u -- run "your message" --model opencode/pulse-alpha-free
```

List all free models: `openpublic -u -- models`

### Full identity reset (ban evasion)

Each `-r -u` run gives:
- **New container IP** via WARP (never reuses previous IP)
- **New `/etc/machine-id`** (randomized UUID)
- **Empty opencode data dirs** (no stored sessions, auth, or device ID)
- **Randomized `OPENCODE_CLIENT`** (changes `x-opencode-client` header + User-Agent)
- **Randomized `OPENCODE_TUI_CHANNEL`** (changes User-Agent channel)
- **No `GH_TOKEN`** (not tied to your GitHub account's server-side cap)

## How auth works (no credentials leave your machine)

1. The script reads the **active** `gh` token from your host keyring at launch time (`gh auth token`).
2. It writes that token to a temp file with mode `0600` and mounts it **read-only** into the container at `/run/gh-token`.
3. Inside the container:
   - `git` uses a credential helper that reads `/run/gh-token` on demand.
   - `opencode` runs with `GH_TOKEN` exported; `gh` (installed automatically if missing) also picks that up.

## Requirements

- Docker (local daemon or reachable via `DOCKER_HOST`)
- `gh` authenticated on the host **only for authenticated mode** (`gh auth status`). Skip with `-u`/`--no-auth`.
- Image `opencode-warp` (or override with `OPENPUBLIC_IMAGE`). The opencode binary is taken from
  your host's `~/.opencode/bin/opencode` (mounted read-only); the image only needs to provide the
  WARP container runtime. If `git`/`gh` are missing inside, the script installs them best-effort
  (authenticated mode only).

## Limit-ban reset

OpenCode's free tier is rate-limited server-side, tied to your opencode device identity and IP.
To fully reset it:

```bash
openpublic -r -u -- --prompt "your message"
```

1. `-r` recreates the container: **new outbound IP** via WARP (never reuses the previous IP) and
   **wipes** `~/.local/share/opencode`, `~/.local/state/opencode`, `~/.cache/opencode`
   (device ID, tabs, bans vanish).
2. `-u` mounts **empty opencode data dirs** inside the container (no stored Google auth,
   no device ID) and passes no `GH_TOKEN`, so opencode is not bound to your host GitHub
   account's server-side cap.
3. `-- --prompt "..."` forwards the prompt to opencode.

Verified: `openpublic -r -u -- run "Say exactly: TOKEN"` returns the token from the model with
no "free usage limit reached" error.

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
