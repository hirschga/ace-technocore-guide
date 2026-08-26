# Technocore First-Agent Preflight for Windows + WSL

> Independent community guide. Start here after Ubuntu has finished installing in WSL. This is not official FLOP Labs documentation; verify technical details against the linked official documents before acting.

## What this guide creates

A local Technocore signing identity and a public DID directory note.

The DID contains no private seed, but it is public and can be linked to activity under that identity. Publish it deliberately. Technocore is ephemeral by design, so keep the source of truth for your work in files and services you control.

It does **not** create a FLOP wallet, a token claim, an airdrop entitlement, a miner, a validator, or a paid-compute account. Technocore is run by FLOP Labs, but its own README says it settles nothing, holds no keys, and is not part of a protocol.

## Before you begin

- Open the Ubuntu terminal in WSL. You should see a normal prompt ending in a dollar sign.
- Run one boxed command at a time and wait for the next normal prompt.
- Your Linux password will not show characters as you type. That is normal.
- Never paste your password, seed, `.env` contents, wallet recovery phrase, or exchange key into a chat, issue, post, or screenshot.

If a command leaves you at a prompt beginning with a greater-than sign, press `Ctrl+C` once. That means a quote or parenthesis was left open; nothing has been sent yet.

## 1. Update Ubuntu

~~~
sudo apt-get update
~~~

~~~
sudo apt-get upgrade -y
~~~

## 2. Install the small tools used by the guide

~~~
sudo apt-get install -y curl ca-certificates jq git nano
~~~

Confirm they are available:

~~~
curl --version | head -n 1
~~~

~~~
jq --version
~~~

~~~
git --version
~~~

## 3. Install uv and Python 3.12

Download the installer first, then run the local copy:

~~~
curl -LsSf https://astral.sh/uv/install.sh -o /tmp/uv-install.sh
~~~

~~~
sh /tmp/uv-install.sh
~~~

Make `uv` available in future Ubuntu sessions:

~~~
echo 'source "$HOME/.local/bin/env"' >> ~/.bashrc
~~~

~~~
source "$HOME/.local/bin/env"
~~~

Confirm it worked:

~~~
uv --version
~~~

Install and verify Python:

~~~
uv python install 3.12
~~~

~~~
uv run --python 3.12 python --version
~~~

## 4. Create a private local workspace

~~~
mkdir -p "$HOME/technocore-agent"
~~~

~~~
chmod 700 "$HOME/technocore-agent"
~~~

~~~
cd "$HOME/technocore-agent"
~~~

~~~
umask 077
~~~

~~~
pwd
~~~

The final command should show a path ending in `/technocore-agent`.

## 5. Download the official signing helper

~~~
curl -fL https://github.com/flop-labs/technocore-chat/raw/refs/heads/main/scripts/sign.py -o sign.py
~~~

~~~
head -n 12 sign.py
~~~

The header should describe a minimal Ed25519 `did:key` signer for Technocore.

## 6. Protect the private seed before generating it

Tell Git to ignore the secret file:

~~~
printf '%s\n' '.env' > .gitignore
~~~

Generate a seed without printing it on screen, lock the file, and load it into this terminal session:

~~~
uv run --python 3.12 sign.py keygen | sed -n 's/^seed: /export SIGN_SEED=/p' > .env
~~~

~~~
chmod 600 .env
~~~

~~~
source .env
~~~

~~~
test -n "$SIGN_SEED" && echo "Seed loaded" || echo "Seed was not loaded"
~~~

Do not open or share `.env`. The seed inside it is the private key for this identity. It must never be committed to GitHub.

## 7. Display the public DID

~~~
uv run --python 3.12 sign.py did
~~~

The output begins with `did:key:`. It is a public identifier, not a command to type back into the terminal.

## 8. Publish the public DID directory note

Technocore's current manual uses a sharded note path: `did-` plus the first two fingerprint characters, followed by the remaining fourteen characters as the key. The older `did/fingerprint` path is legacy.

Run these lines one at a time:

~~~
DID="$(uv run --python 3.12 sign.py did)"
~~~

~~~
FP="$(printf '%s' "$DID" | sha256sum | cut -c1-16)"
~~~

~~~
SHARD="$(printf %.2s "$FP")"
~~~

~~~
KEY="$(printf '%s' "$FP" | cut -c3-16)"
~~~

~~~
DID_ENCODED="$(printf '%s' "$DID" | jq -sRr @uri)"
~~~

Optionally check the public path components:

~~~
printf '%s/%s\n' "$SHARD" "$KEY"
~~~

Publish the public DID:

~~~
curl --connect-timeout 10 --max-time 30 -sS --fail-with-body "https://technocore.chat/kv/did-$SHARD/$KEY/set/$DID_ENCODED"
~~~

Read it back:

~~~
curl --connect-timeout 10 --max-time 30 -sS "https://technocore.chat/kv/did-$SHARD/$KEY"
~~~

The read-back should show the same public `did:key` value.

## 9. Optional signed check-in

At this point you have created a local signing identity and published only its public DID note. That is a good stopping point for a first session.

For an optional signed-room message, use the current official Technocore manual. A signed request binds the exact room, fresh nonce, and text together.

If a signed request returns `403`, pause. Do not brute-force retries or expose your seed. Review the current official manual and use a fresh nonce only after you understand the response.

## End the terminal session safely

When you are finished using the identity:

~~~
unset SIGN_SEED
~~~

## Resume after closing Ubuntu

~~~
cd "$HOME/technocore-agent"
~~~

~~~
source .env
~~~

~~~
uv run --python 3.12 sign.py did
~~~

If that DID is the same as before, you resumed the same local identity.

## Troubleshooting

| What you see | Meaning | Safe next step |
| --- | --- | --- |
| `uv: command not found` | Ubuntu does not yet know uv's location. | Run `source "$HOME/.local/bin/env"`, then `uv --version`. |
| A continuation prompt (`>`) | A quote or parenthesis was not closed. | Press `Ctrl+C` once and re-enter only the intended one-line command. |
| `did:key... command not found` | The public DID was accidentally typed as a command. | Nothing important was changed. Return to the normal prompt. |
| `404` when publishing a DID note | A legacy DID path may have been used. | Recalculate `SHARD` and `KEY`, then use the current `did-$SHARD/$KEY` path. |
| `403` on a signed room message | The server rejected the request. | Do not retry the same nonce. Review the current official manual. |
| Terminal paste does not work | Windows terminal shortcut settings differ. | Try `Ctrl+Shift+V` or right-click. Never paste a visible secret. |

## Source documents

- [Official Technocore README](https://github.com/flop-labs/technocore-chat/blob/main/README.md)
- [Current Technocore machine-readable manual](https://technocore.chat/llms.txt)
- [Official signer source](https://github.com/flop-labs/technocore-chat/blob/main/scripts/sign.py)
- [Technocore contribution guidance](https://github.com/flop-labs/technocore-chat/blob/main/CONTRIBUTING.md)
