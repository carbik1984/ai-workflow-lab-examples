# n8n Course #1 – Install n8n Locally on Windows with Docker

Video: *(link coming soon)*

## Steps

1. Install and start **Docker Desktop** – wait until it says "Engine running".
   https://docs.docker.com/desktop/setup/install/windows-install/
2. Create a folder `n8n`.
3. Inside it, create the file `compose.yaml` – copy it from [compose.yaml](compose.yaml).
   Spaces matter in YAML, so copy it exactly.
4. Open a terminal in the folder and run:
   ```
   docker compose up -d
   ```
5. Open http://localhost:5678 and create your owner account.
6. Save your n8n **encryption key** in a password manager (see below).

## compose.yaml explained

| Line | Meaning |
|---|---|
| `image: ...:2.45.6` | fixed n8n version – it updates only when you decide (not `latest`) |
| `restart: unless-stopped` | when Docker starts, n8n starts too |
| `127.0.0.1:5678:5678` | n8n is reachable only from this computer |
| `GENERIC_TIMEZONE`, `TZ` | time zone for scheduled workflows – **change it to yours** |
| `n8n_data:/home/node/.n8n` | volume – your workflows and settings survive when the container is deleted |

## Commands from the video

```
docker compose up -d        # start n8n in the background
docker compose ps           # check that it is running ("Up")
docker compose restart      # restart n8n
docker compose exec n8n cat /home/node/.n8n/config   # show the encryption key
```

## Encryption key

n8n encrypts all saved credentials (passwords, API keys) with this key.
If you lose it, you lose your saved credentials. Copy the value after `"encryptionKey"`
into your password manager. Never share it and never commit it to git.

## "This page isn't working" after a restart?

The container is running, but n8n is still starting. Wait about 10 seconds and refresh.
