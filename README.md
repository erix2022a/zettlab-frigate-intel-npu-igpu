# Zettlab D6 Ultra — Frigate Intel NPU + Ollama Intel Arc iGPU + Home Assistant AI Email Alerts

Run a local camera AI pipeline on a Zettlab D6 Ultra NAS: **Intel NPU detects people**, **Intel Arc iGPU runs Ollama/Qwen3-VL to describe them**, and **Home Assistant sends the description by email through SMTP**.

Tested setup described by the project author: **Intel Core Ultra 5 125H**, Frigate **0.18 series**, Ollama **Qwen3-VL 8B**, Reolink RTSP camera, Home Assistant, and an MQTT broker. This is a community guide, not an official Zettlab or Frigate release. Other NAS firmware, Intel drivers, cameras, or Frigate versions may need adjustments.

> **Privacy:** All files in this repository are public-safe **examples**, not a direct export of private NAS configuration. Do **not** paste your real NAS, camera, MQTT or Gmail credentials into GitHub. The Frigate configuration stored here has placeholder values.

## The 4 important files

Follow the steps **in the order shown below**:

| Order | File | What it does |
|---|---|---|
| **1. Install Frigate** | [`docker-compose.frigate.yml`](docker-compose.frigate.yml) | Runs Frigate with the Intel NPU (`/dev/accel`) and Intel iGPU (`/dev/dri`) exposed |
| **2. Configure Frigate** | [`frigate/config.example.yml`](frigate/config.example.yml) | Enables OpenVINO NPU detection, Reolink RTSP, person-only AI descriptions, recordings and MQTT |
| **3. Install Ollama** | [`docker-compose.ollama.yml`](docker-compose.ollama.yml) | Runs Qwen3-VL on the Intel iGPU and connects to Frigate's Docker network |
| **4. Create Home Assistant automation** | [`home-assistant/person-ai-email.example.yaml`](home-assistant/person-ai-email.example.yaml) | Subscribes to Frigate's MQTT description events and sends an email through the SMTP integration |

**Home Assistant SMTP and MQTT setup comes between steps 3 and 4.** You also need an **MQTT broker** (for example, Mosquitto) accessible to Frigate *before Frigate can establish its MQTT connection*. Home Assistant's MQTT integration is configured later in this guide.

### System flow

```mermaid
flowchart LR
  CAMERA["RTSP camera"] --> FRIGATE["Frigate container"]
  NPU["Intel NPU / OpenVINO"] --> FRIGATE
  FRIGATE -->|"Person snapshot"| OLLAMA["Ollama / Qwen3-VL 8B"]
  GPU["Intel Arc iGPU / Vulkan"] --> OLLAMA
  OLLAMA -->|"Generated description"| FRIGATE
  FRIGATE --> MQTT["MQTT broker"]
  MQTT --> HA["Home Assistant"]
  HA --> SMTP["SMTP / email"]
```

## Prerequisites

- Zettlab D6 Ultra or compatible Intel Core Ultra Linux NAS with Docker Compose support and working Intel NPU + integrated GPU devices. Not every Zettlab model includes an NPU.
- A supported **RTSP camera** (original setup used Reolink).
- An **MQTT broker** such as Mosquitto, with a dedicated account for Frigate. Install/start the broker before expecting Frigate MQTT to connect.
- Home Assistant with administrator access and an email provider/account usable for SMTP. For Gmail, use an **app password** if your account and security settings support it; never publish the password.
- Enough storage for camera recordings, enough available system memory for Frigate and the chosen vision model, and host Intel GPU drivers/NPU firmware compatible with the containers.

On the NAS **host**, check devices:

```sh
ls -la /dev/accel /dev/dri
```

You should see an Intel NPU device under `/dev/accel` and render devices under `/dev/dri`. Device numbering varies. The actual provided Frigate and Ollama Compose files are kept unchanged in this repository.

## Step 1 — Install the Frigate Docker container

1. On the NAS, create a dedicated installation directory; for example `/volume1/docker/frigate-ai` (change this to the path supported by your Zettlab NAS).
2. Copy [`docker-compose.frigate.yml`](docker-compose.frigate.yml) into that directory.
3. Create **private**, local folders for the provided Frigate Compose mounts:

   ```sh
   mkdir -p config recordings
   ```

4. Keep the Compose project name exactly **`frigate`**, because the separate Ollama Compose file expects the Docker network named **`frigate_default`**. Launch it from the directory where the Frigate Compose file lives:

   ```sh
   docker compose -p frigate -f docker-compose.frigate.yml up -d
   ```

   If Frigate restarts because its configuration is missing, continue with **Step 2**. To configure everything before starting Frigate, complete Step 2 first and then rerun the command above.

5. Confirm the Docker network exists:

   ```sh
   docker network inspect frigate_default
   ```

6. Frigate's web interface is **`https://YOUR-NAS-IP:8971`**. Complete login/authentication setup once the container is healthy.

**Ports and paths in your Compose:** HTTPS UI 8971, WebRTC TCP/UDP 8555, `./config:/config`, `./recordings:/media/frigate`, 512 MB shared memory, 1 GB tmpfs cache. Do **not** expose Frigate's unauthenticated internal port 5000 to the internet. The Compose file uses the `stable` image tag, which may advance to a later release; pin a specific tested tag when reproducibility matters.

## Step 2 — Apply the Frigate app configuration YAML

1. Copy [`frigate/config.example.yml`](frigate/config.example.yml) into the **private local** `config` directory you created in Step 1. Name the local file **`config/config.yml`**.
2. Replace the **placeholder values only in your local private file**:

   | Placeholder | Replace with |
   |---|---|
   | `{FRIGATE_MQTT_HOST}` | Your MQTT broker's LAN IP or hostname |
   | `{FRIGATE_MQTT_USER}` | Your dedicated Frigate MQTT username |
   | `{FRIGATE_MQTT_PASSWORD}` | Your MQTT password |
   | `{FRIGATE_CAMERA_RTSP_URL}` | Your full working RTSP URL (both ffmpeg and go2rtc entries) |
   | `CHANGE_ME@example.com` | Your chosen contact email for Frigate browser push (not SMTP) |

   **Important:** Your standalone Frigate Compose includes **only `TZ`** under `environment:`, so the `FRIGATE_...` placeholders in the public YAML do **not** receive values automatically from `.env.example` with this Compose. For the provided standalone setup, **replace placeholders in the local, gitignored `config/config.yml`**. Alternatively, advanced users can explicitly add the `FRIGATE_...` container environment variables to their private deployment; do not commit them.

3. The RTSP URL must match **your camera**. Percent-encode reserved characters in the RTSP username/password where required. Test the URL locally rather than publishing it.
4. Verify that the detection stream dimensions match the example `detect.width: 1920`, `height: 720`, and `fps: 5`, or change them for your stream.
5. Restart the container and check Frigate's **System Metrics** for the `intel_npu` detector:

   ```sh
   docker compose -p frigate -f docker-compose.frigate.yml restart frigate
   docker compose -p frigate -f docker-compose.frigate.yml logs --tail=100 frigate
   ```

### What the supplied Frigate YAML enables

- OpenVINO detector with `device: NPU` and the supplied 300×300 object model.
- Tracking **person, cat, mouse, car**; **person-only Review alerts** and AI **object** descriptions.
- `objects.genai.use_snapshot: true` to analyze one person snapshot rather than many video frames.
- **Qwen3-VL 8B Instruct** through `http://ollama:11434`.
- Motion recording, 14-day alert/detection retention and snapshots, Reolink/go2rtc, face recognition, semantic search, and audio detection as in the example.
- **Full Review GenAI summaries disabled** by default: `review.genai.enabled: false`. This is intentional because the original 8B installation had multi-frame context overflow and 120-second timeout errors.

**Detection and recording can be configured before Ollama is installed.** AI descriptions cannot work until Step 3 is complete.

## Step 3 — Install the separate Ollama Docker container

1. Put [`docker-compose.ollama.yml`](docker-compose.ollama.yml) in an **Ollama-specific folder**, such as `/volume1/docker/ollama`. Its `./data` is relative to that folder and stores downloaded models.
2. Make sure the Frigate network `frigate_default` already exists from Step 1.
3. Start Ollama:

   ```sh
   mkdir -p data
   docker compose -f docker-compose.ollama.yml up -d
   ```

4. Download the exact model specified in the Frigate YAML:

   ```sh
   docker exec ollama ollama pull qwen3-vl:8b-instruct
   ```

5. After an AI request, check acceleration and logs:

   ```sh
   docker exec ollama ollama ps
   docker logs --tail=100 ollama
   ```

   When the model is loaded, check the **PROCESSOR** column in `ollama ps`. The original setup reported **100% GPU** with the Intel iGPU flags. If it reports CPU, check host `/dev/dri`, Intel Vulkan driver support, container permissions and Ollama logs.

The provided Ollama Compose uses **`mem_limit: 32g`**, the Intel iGPU `/dev/dri`, Vulkan-related environment settings, a 16,384-token context, and 5-minute keep-alive. **On a 32 GB RAM NAS, a 32 GB Ollama limit does not reserve memory for Frigate/other services**; monitor total RAM and lower the limit or switch to a smaller vision model if the NAS experiences out-of-memory failures.

### Verify Frigate can reach Ollama

Both containers must be on **`frigate_default`**. The provided Ollama file joins that external network, while the Frigate file creates it only if started with project name `frigate`.

Inspect the network members:

```sh
docker network inspect frigate_default
```

Frigate uses `http://ollama:11434` as configured in its private YAML. The Ollama hostname resolves only when both containers share an appropriate Docker network. Do not publish Ollama port 11434 to the public internet.

Trigger a real person detection. In Frigate, check that the person was detected and that Ollama produced an object description. If it is slow or times out, try `qwen3-vl:4b-instruct` as an **optional** performance alternative, and change Frigate's model tag to match.

## Step 4 — Home Assistant: install/configure SMTP first

1. In **Home Assistant → Settings → Devices & services → Add integration**, search for **SMTP**. Set up your email provider's SMTP server, port, encryption, sender email, recipient(s), and credentials using Home Assistant's UI or supported YAML configuration for your HA version.
2. For Gmail, follow its SMTP/App Password requirements; never store the app password in your public GitHub files.
3. Find the resulting notification entity (such as `notify.YOUR_SMTP_ENTITY`) under **Settings → Devices & services → Entities** or **Developer tools → Actions**, depending on your Home Assistant version.
4. Test SMTP before configuring any MQTT email automation:

   ```yaml
   action: notify.send_message
   target:
     entity_id: notify.YOUR_SMTP_ENTITY
   data:
     title: "Frigate email test"
     message: "SMTP notification is working."
   ```

Use the entity ID your Home Assistant installation actually exposes. If the SMTP setup instead provides a service-specific action, use that integration's documented action.

**Frigate's `notifications.email` setting is not an SMTP email sender.** It does not replace Home Assistant SMTP setup.

## Step 5 — Home Assistant: install/configure MQTT

1. Make sure your **MQTT broker is already running** (for example, the Mosquitto broker add-on or a separate Mosquitto container), because Frigate's config references it.
2. In **Home Assistant → Settings → Devices & services → Add integration → MQTT**, connect HA to the broker with the broker hostname/IP, port (usually 1883), and credentials. If MQTT is discovered automatically, use the discovered integration.
3. In **MQTT → Configure → Listen to a topic**, enter:

   ```text
   frigate/tracked_object_update
   ```

4. Trigger a new **person** event and let it complete. The expected type of update for a successful AI object description is:

   ```json
   {
     "type": "description",
     "id": "EXAMPLE-EVENT-ID",
     "description": "A person wearing a dark jacket walks near the entrance."
   }
   ```

The actual event includes a unique ID. Frigate may publish other MQTT message types as well. An `end` event on `frigate/reviews` with `metadata: null` is **not** proof that an object description failed; Review summaries and object descriptions use separate pipelines.

If Frigate is not publishing description messages, troubleshoot Frigate/Ollama first rather than changing SMTP.

## Step 6 — Home Assistant: create the email automation

1. Open **Home Assistant → Settings → Automations & scenes → Create automation → Edit in YAML**.
2. Copy the complete YAML from [`home-assistant/person-ai-email.example.yaml`](home-assistant/person-ai-email.example.yaml).
3. Replace `notify.YOUR_SMTP_ENTITY` with your **working SMTP notification entity** and `YOUR-NAS-LAN-IP` with the NAS address in your own **private HA automation**.
4. Save and enable the automation; then test another person event.
5. If the email does not arrive, inspect **Automations → Traces** and verify the message was received on `frigate/tracked_object_update` with `type: description`.

The example email sends the AI **text**, the derived local event timestamp, and **photo/MP4 links**, not attachments. These links require an authenticated Frigate session and LAN/VPN access; opening the clip from an unauthenticated Gmail/browser session can return **401 Unauthorized**. Recordings must exist for a clip URL to play. Use a secure VPN or properly authenticated integration for remote viewing; **do not bypass authentication**.

## Quick verification checklist

- [ ] NPU and iGPU devices exist on the NAS host.
- [ ] Frigate starts with the private `config/config.yml` and detects/tracks a person.
- [ ] Docker network **`frigate_default`** exists and contains Frigate + Ollama.
- [ ] Ollama has `qwen3-vl:8b-instruct` installed; `ollama ps` reports GPU use after a request.
- [ ] Home Assistant SMTP test email arrives.
- [ ] Home Assistant MQTT listener receives a `type: description` message.
- [ ] Home Assistant automation sends the person description by email.

## Troubleshooting

| Symptom | Check |
|---|---|
| `/dev/accel` not found or NPU detector unavailable | Host kernel, Intel NPU firmware, mapping and Frigate logs |
| Frigate YAML won't load | Replace private placeholders, check RTSP URL, validate syntax, inspect Frigate logs |
| Ollama fails: external network missing | Start Frigate with `-p frigate` or attach both containers to the same network |
| Frigate cannot resolve `ollama` | Verify both containers are on `frigate_default` and Ollama is running |
| `ollama ps` shows CPU | Host `/dev/dri`, Vulkan driver availability, container permissions and Ollama logs |
| Context-size error or 120-second timeout | Use snapshot object descriptions, disable Review GenAI, consider Qwen3-VL 4B |
| Person detection works but no email | Check `frigate/tracked_object_update`, automation traces, then SMTP |
| Video clip URL returns 401 | Authenticate to Frigate; never expose its unauthenticated API |
| Missing snapshots or recording clip | Check that the media was recorded and retained; attachments need additional HA actions |

## Privacy and publication safety

This repository contains **placeholder configuration only**. In the actual standalone layout, keep these paths strictly local and untracked: `./config/`, `./recordings/`, and Ollama's `./data/`. The provided [`.gitignore`](.gitignore) excludes them. Also read [`SECURITY.md`](SECURITY.md).

**Important before making this repository public:** earlier Git revisions may contain values that were later replaced. Changing the latest file does **not** remove them from Git history. Rotate any previously exposed credentials, and scrub the repository's Git history or recreate a clean repository before switching visibility to Public.

## Optional alternative

The root [`docker-compose.yml`](docker-compose.yml) is an **older combined Frigate + Ollama example**, **not the primary installation method**. Do not start it together with the two standalone Compose files: duplicate services/ports and differing volume paths will conflict. This guide intentionally prioritizes the author's four-file, two-container installation.

## References

- [Frigate installation](https://docs.frigate.video/frigate/installation/)
- [Frigate OpenVINO object detector](https://docs.frigate.video/configuration/object_detectors/)
- [Frigate object GenAI](https://docs.frigate.video/configuration/genai/genai_objects/)
- [Frigate MQTT](https://docs.frigate.video/integrations/mqtt/)
- [Home Assistant SMTP](https://www.home-assistant.io/integrations/smtp/)
- [Home Assistant MQTT](https://www.home-assistant.io/integrations/mqtt/)
- [Ollama Docker documentation](https://docs.ollama.com/docker)

**This project is unofficial and is not affiliated with Zettlab, Frigate, Intel, Ollama, Reolink, or Home Assistant.**
