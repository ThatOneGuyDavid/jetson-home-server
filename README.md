# A private, local Home Assistant and voice AI server on a Jetson Orin Nano

I built a single, compact home server that runs Home Assistant, a local chat
interface, and an entirely local voice pipeline. Spoken audio stays inside the
home network: a Home Assistant Voice device captures it, Whisper transcribes
it, a local Qwen model answers it, and Piper speaks the response.

This guide recreates that build without exposing any addresses, device IDs,
account names, or other details from the original installation.

## What the finished system does

- Runs Home Assistant in Docker with persistent configuration.
- Hosts Open WebUI for private chat from trusted devices on the LAN.
- Runs Qwen locally through Ollama with NVIDIA GPU acceleration.
- Uses Whisper for local speech-to-text and Piper for local text-to-speech.
- Connects Home Assistant Voice Preview Edition as the microphone and speaker.
- Keeps the Voice device on an isolated IoT network with only two narrowly
  scoped firewall paths to the Home Assistant server.
- Leaves the existing smart-home platform untouched until devices can be
  migrated deliberately, one at a time.

## Hardware used

- NVIDIA Jetson Orin Nano Developer Kit, 8 GB
- Active cooling
- 1 TB NVMe SSD
- Home Assistant Voice Preview Edition
- An OpenWrt router capable of separate Trusted and IoT networks
- An IoT wireless access point or mesh in access-point mode

The original network used a GL.iNet Flint 2 running OpenWrt and a separate Wi-Fi
mesh for IoT clients. Those exact products are optional; VLAN-capable equipment
with equivalent firewall controls will work.

## Software layout

```text
Home Assistant Voice (IoT network)
        |
        | ESPHome native API, TCP 6053
        v
Home Assistant on Jetson (Trusted network)
        |
        +--> Whisper, 127.0.0.1:10300
        +--> Ollama,  127.0.0.1:11434
        +--> Piper,   127.0.0.1:10200
        |
        +--> TTS audio returned to the Voice device

Trusted browser --> Open WebUI on Jetson, TCP 3000
```

Home Assistant and Open WebUI use host networking. The Wyoming endpoints are
published only on loopback, so other LAN clients cannot connect to Whisper or
Piper directly.

## 1. Prepare the Jetson

Use a current NVIDIA JetPack release, boot from the NVMe SSD, and install Docker
Engine with the Compose plugin. Confirm that the NVIDIA container runtime is
available before continuing.

Ollama runs natively on the Jetson in this build. That choice was intentional:
GPU use was verified with the native Jetson installation, while Home Assistant,
Open WebUI, Whisper, and Piper remain reproducible Docker services.

Before changing an older installation, inventory it:

```bash
docker ps -a
docker volume ls
docker info | sed -n '/Runtimes/,+3p'
systemctl status ollama --no-pager
```

Do not delete an existing Open WebUI volume until it has been backed up. Also
check for duplicate Docker installations, such as both Snap Docker and the
normal Docker Engine, because two daemons can make containers appear to vanish.

## 2. Create the Compose project

Clone this repository, copy the environment template, and edit only the local
copy:

```bash
cp .env.example .env
chmod 600 .env
```

Set `TZ` to the local IANA timezone and set `OPEN_WEBUI_ORIGIN` to the exact URL
used to open Open WebUI. The real `.env` file is ignored by Git.

If this is a new Open WebUI installation, create its external volume:

```bash
docker volume create open-webui
```

Then start the services:

```bash
docker compose pull
docker compose up -d
docker compose ps
```

The included Compose file starts:

- Home Assistant on TCP 8123
- Open WebUI on TCP 3000
- Wyoming Whisper on loopback TCP 10300
- Wyoming Piper on loopback TCP 10200

Verify that the speech services are not exposed on all interfaces:

```bash
ss -ltn | grep -E ':(10200|10300)'
```

Both listeners should show `127.0.0.1`, not `0.0.0.0`.

## 3. Install and test the local model

Install Ollama using the appropriate Jetson/NVIDIA instructions, then pull a
model that fits within the Orin Nano's shared 8 GB memory:

```bash
ollama pull qwen3:4b-instruct
ollama run qwen3:4b-instruct
```

Qwen 3 4B Instruct provided a useful quality and speed balance in this build.
The smaller reasoning model tested first frequently misunderstood simple
instructions, so it was removed after Qwen passed basic response and tool-call
tests.

Open Open WebUI at `http://<JETSON-IP>:3000`. The first local account becomes
the administrator, so use a unique password. Disable public signup unless the
server is intentionally shared with other household users.

## 4. Secure Home Assistant before adding devices

Open `http://<JETSON-IP>:8123` and complete onboarding.

- Use a unique administrator username and password.
- Enable authenticator-app MFA.
- Disable analytics and diagnostics if the goal is minimal data sharing.
- Do not forward TCP 8123 or TCP 3000 from the internet.
- Do not migrate existing smart-home devices yet. First make the platform,
  backups, and voice path reliable.

Plain HTTP is acceptable only on a trusted local network as an initial state.
Add locally trusted HTTPS later, before expanding access.

## 5. Connect Ollama to Home Assistant

In Home Assistant, go to **Settings → Devices & services → Add integration** and
add **Ollama**.

Because Home Assistant uses host networking, use:

```text
http://127.0.0.1:11434
```

No API key is required for this loopback-only connection. Add a conversation
agent with these practical settings:

- Name: `Local Qwen`
- Model: `qwen3:4b-instruct`
- Context window: `4096`
- Maximum history messages: `5`
- Keep alive: `1800` seconds
- Think before responding: off
- Home-control tools: off initially

Keeping home-control tools off is deliberate. Review which entities Home
Assistant exposes to Assist before allowing an LLM to invoke device actions.

A concise voice-oriented instruction template is:

```text
You are a private local voice assistant for Home Assistant.
Answer accurately and directly in one or two short sentences unless the user asks for more detail.
Use natural spoken language without markdown, headings, lists, emoji, or descriptions of your reasoning.
If you are uncertain, say that you do not know rather than inventing an answer.
The current local date and time is {{ now() }}.
When asked for the time, answer in natural 12-hour spoken form with AM or PM.
```

The timestamp is injected by Home Assistant at request time. Without it, a
local model may guess the current time.

## 6. Add local speech services

Add **Wyoming Protocol** twice under **Settings → Devices & services**:

| Purpose | Host | Port |
|---|---|---:|
| Whisper speech-to-text | `127.0.0.1` | `10300` |
| Piper text-to-speech | `127.0.0.1` | `10200` |

Create or edit a voice assistant pipeline with:

- Conversation agent: `Local Qwen`
- Speech-to-text: `faster-whisper`
- Language: English
- Text-to-speech: `piper`
- Voice: `en_US-lessac-medium`

The Jetson generates speech, but its own speakers are not used. Home Assistant
sends the resulting audio to the Voice hardware, which plays it in the room.

## 7. Provision Home Assistant Voice without a phone

The Voice Preview Edition can be provisioned from a Chromium browser over a USB
data cable. Use the official installer:

<https://esphome.github.io/home-assistant-voice-pe/>

1. Connect the Voice unit to the computer using a USB-C data cable.
2. Select **Connect** and choose its USB serial/JTAG device.
3. Select **Connect to Wi-Fi** and join the isolated IoT SSID.
4. Follow the link back to the local Home Assistant instance.
5. Add the ESPHome integration using the Voice device's reserved IP and default
   TCP port `6053`.
6. Allow the firmware update to finish without removing power.
7. Choose **Full local processing** and assign the `Local AI` pipeline.

The hardware mute switch physically cuts microphone power. For normal use, set
it so the red mute marking is not visible; expose the marking whenever physical
privacy is desired.

## 8. Isolate the Voice hardware with firewall rules

Place the Jetson on the Trusted network and Voice hardware on the IoT network.
Give both DHCP reservations. In the examples below, replace the documentation
addresses with the real reserved addresses:

```text
Jetson:       <JETSON-TRUSTED-IP>
Voice device: <VOICE-IOT-IP>
```

Create only these IPv4 TCP allow rules:

| Name | Source | Destination | Port |
|---|---|---|---:|
| HA-to-Voice-ESPHome | `<JETSON-TRUSTED-IP>` | `<VOICE-IOT-IP>` | 6053 |
| Voice-to-HA-Audio | `<VOICE-IOT-IP>` | `<JETSON-TRUSTED-IP>` | 8123 |

Keep the general IoT-to-Trusted policy blocked. Do not enable broad mDNS
reflection merely for discovery; entering the reserved Voice IP manually avoids
that additional cross-network exposure.

After applying the rules, test TCP 6053 from the Jetson:

```bash
timeout 5 bash -c '</dev/tcp/<VOICE-IOT-IP>/6053' \
  && echo reachable || echo blocked
```

Successful spoken playback through the Voice device also confirms the narrowly
allowed return path to Home Assistant.

Create a fresh encrypted or access-restricted router configuration backup after
adding the reservation and rules. Router archives contain credentials and must
never be committed to a public repository.

## 9. Test the full local path

Start with a deterministic text test in Home Assistant's conversation-agent
debug window, then test the physical device:

```text
Okay Nabu, what is two plus two?
```

The first request can take around 20 seconds while the 4B model loads. Requests
made while the model is warm should be substantially faster. A 30-minute keep
alive provides a reasonable compromise on an 8 GB shared-memory system.

The finished request path is:

```text
Voice microphone
  -> Whisper transcription
  -> Qwen response
  -> Piper speech generation
  -> Voice speaker
```

## 10. Before enabling smart-home control

Do not expose every entity to the LLM by default. Before enabling the Ollama
agent's Home Assistant control option:

1. Inventory entities and remove obsolete or duplicate entries.
2. Expose only devices that genuinely need voice control.
3. Exclude locks, alarms, garage doors, purchases, and other consequential
   actions until explicit safeguards are designed and tested.
4. Test harmless lights or switches first.
5. Maintain a current Home Assistant backup and an off-device copy.

Migrate devices from the previous smart-home platform in small groups. Confirm
local control and automations in Home Assistant before removing each device from
the old platform.

## Operational notes

- Pin container versions or image digests before treating the deployment as
  production infrastructure.
- Review release notes and back up configuration before upgrades.
- Keep Home Assistant, Open WebUI, and router administration inaccessible from
  the public internet.
- Prefer a VPN for remote administration rather than port forwarding.
- Back up Home Assistant data, the Compose project, Open WebUI data, and router
  configuration to a separate machine.
- Never commit `.env`, Home Assistant configuration, database files, tokens,
  router archives, Wi-Fi details, screenshots, or diagnostic bundles.

The result is not instant cloud-assistant speed, but it is private, inspectable,
and useful: one small Jetson hosts the smart-home controller, local chat, and a
complete open voice pipeline while the room device remains isolated on the IoT
network.
