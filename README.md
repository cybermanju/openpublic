# openpublic

Run [opencode](https://opencode.ai) inside an always-fresh Docker container with
your **host GitHub credentials passed through automatically** — so git push,
`gh`, and private repos just work inside the container, every time it's recreated.

> ⚠️ Currently the container image expected is `opencode-warp` (opencode + warp CLI).
> A generic `opencode` image also works — set `OPENPUBLIC_IMAGE`.

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/cybermanju/openpublic/main/openpublic -o ~/.local/bin/openpublic
chmod +x ~/.local/bin/openpublic
```

Optional wrappers:

```bash
ln -sf ~/.local/bin/openpublic ~/.local/bin/opencode-d   # attach = plain call
printf '#!/usr/bin/env bash\nexec "$HOME/.local/bin/openpublic" -r "$@"\n' > ~/.local/bin/opencode-r  # -r = reset
chmod +x ~/.local/bin/opencode-r
```

## Usage

```bash
openpublic        # attach, creating the container if missing
openpublic -r     # recreate the container fresh (container wiped, data dirs kept)
```

## How auth works (no credentials leave your machine)

1. The script reads the **active** `gh` token from your host keyring at launch
   time (`gh auth token`). Nothing is copied from dotfiles or env.
2. It writes that token to `/tmp/opencode-gh-token` with mode `0600` and mounts
   it **read-only** into the container at `/run/gh-token`.
3. Inside the container:
   - `git` uses a [credential helper](https://git-scm.com/docs/gitcredentials)
     that reads `/run/gh-token` on demand (used only for github.com pushes/pulls).
   - `opencode` is started with `GH_TOKEN` exported; `gh` (installed
     automatically if missing) also picks that up.
4. No token is stored inside the container's config files, in the repo, or in
   any script. Rotating `gh auth login` on the host? Next attach uses the new
   token automatically.

The mount is read-only: processes in the container can read the token file but
not modify the host's copy. Delete `/tmp/opencode-gh-token` on the host at any
time to revoke instantly.

## Requirements

- Docker
- `gh` (host) authenticated: `gh auth status`
- Image `opencode-warp` (or your own — override with `OPENPUBLIC_IMAGE`) with
  `opencode` in PATH. If `git`/`gh` are missing inside the image, the script
  installs them best-effort (apt for git; official binary for gh).

## Environment variables

| Var                  | Default                  | Meaning                      |
|----------------------|--------------------------|------------------------------|
| `OPENPUBLIC_IMAGE`   | `opencode-warp`          | Container image to run       |
| `OPENPUBLIC_WORKSPACE` | parent of current dir  | Host path mounted at `/workspace` |

Ports forwarded: 13000→3000, 13213→3213, 14096→4096, 17654→7654 (opencode/warp).

## Notes / limitations

- Marks the first available IP in the docker network `opencode-net` (172.20.0.0/16).
- Requires `--device /dev/net/tun` + `NET_ADMIN` (warp needs a tun device).
- If the image lacks `git`, the credential helper is skipped (git auth then
  still needs tokens to be set manually).
- MTU / VPN quirks: if the WARP tunnel misbehaves in a custom network, check
  `--cap-add NET_ADMIN` and the tun device are present (the script adds both).

## License

MIT
