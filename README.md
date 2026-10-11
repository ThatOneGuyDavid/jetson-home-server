# Jetson home server

Docker Compose configuration for services hosted on the Jetson Orin Nano.

## Home Assistant

Home Assistant uses host networking for local-device discovery and stores its
persistent configuration under `homeassistant/config` on the Jetson.

Start or update it from this directory with:

```bash
docker compose pull homeassistant
docker compose up -d homeassistant
```

The web interface is available at `http://<JETSON-IP>:8123`.

## Local chat

Open WebUI uses the native Ollama service on the Jetson. Ollama is kept native
because its current Jetson build is confirmed to use the GPU. Open WebUI stores
its persistent data in the external Docker volume named `open-webui`.

The chat interface is available at `http://<JETSON-IP>:3000`.

Telemetry is explicitly disabled for Open WebUI in `compose.yaml`.

## Local voice

The Wyoming services provide fully local speech processing for Home Assistant:

- Whisper on `127.0.0.1:10300` performs open-ended English speech-to-text.
- Piper on `127.0.0.1:10200` generates US English spoken replies.

Both services listen only on the Jetson loopback interface. Voice hardware
connects to Home Assistant, which then uses these private local services.

## Local settings

Copy `.env.example` to `.env` and replace its example values. The `.env` file
is ignored by Git so local addresses and regional settings are not published.

The full build and security guide is in
[`docs/jetson-local-home-assistant.md`](docs/jetson-local-home-assistant.md).
