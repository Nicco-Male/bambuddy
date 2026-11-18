# XIAO TPU feeder integration — staging-only test plan

These instructions test the integration in **our personal Bambuddy fork**. No changes are made to the production Bambuddy database, container, volume or upstream repository.

Firmware branch: [Nicco-Male/TPU-Feed-Assist-ESP32 — feature/spoolbuddy-fork-poc](https://github.com/Nicco-Male/TPU-Feed-Assist-ESP32/tree/feature/spoolbuddy-fork-poc).

Original physical button remote hardware inspiration: [MacDat's project](https://makerworld.com/en/models/2704030), credited in firmware README and NOTICE.

## 1. Run isolated Bambuddy fork

On a Docker host on a **trusted test LAN** with free TCP port 18080:

```bash
git clone --branch feature/tpu-feed-assist-spoolbuddy-poc \
  https://github.com/Nicco-Male/bambuddy.git bambuddy-tpu-poc
cd bambuddy-tpu-poc
docker compose -f docker-compose.tpu-poc.yml -p bambuddy-tpu-poc up -d --build
curl -f http://127.0.0.1:18080/health
```

This compose file deliberately does **not** use the upstream `network_mode: host`, `container_name: bambuddy`, port 8000 or production data volumes. It binds a separate port and uses unique named volumes. Do NOT expose TCP 18080 on the public internet or copy secrets from live Bambuddy.

## 2. ESP32 setup

1. Flash firmware from the **fork POC branch** using PlatformIO.
2. Do not connect the Bambu feeder to the XIAO. For the network test, power via USB-C bench supply only.
3. Join hotspot `TPU-Feed-XXXX` and open `http://192.168.4.1`.
4. Select the test Wi-Fi, enter the staging URL `http://<TEST-HOST-LAN-IP>:18080` and a staging Bambuddy API key with inventory-update permission. The staging instance may require creating a user/API key first depending on auth settings.
5. Enter separate admin and HTTP API credentials for the XIAO. Do not put these into Git.

Only `http://` on a trusted test LAN is currently implemented for Bambuddy registration; HTTPS currently fails closed until trust-anchor management is available.

## 3. Verify registration and heartbeat

On the staging Bambuddy UI open **Settings → SpoolBuddy Devices**. Expected:
- A card labeled `TPU Feed Assist · XIAO ESP32-S3`, with XIAO device ID, IP and firmware version.
- Status online after registration/heartbeat; offline after about 30 seconds without heartbeats.
- Existing Raspberry Pi NFC/scale device cards unchanged.
- No SSH reboot/update buttons or scale/NFC warnings on the XIAO card.

The XIAO sends `POST /api/v1/spoolbuddy/devices/register` once and `POST /devices/{device_id}/heartbeat` periodically, using a private device API key. The user-facing Bambuddy page is **not an actuator yet**. FEED/FORCE/UNLOAD from Bambuddy are intentionally absent.

## 4. What this DOES NOT prove

- Button electrical isolation or original Bambu control board safety.
- Actual feeder response or filament movement.
- Automatic AMS ↔ TPU spool switching, retract or unload.
- Security under a public or untrusted Wi-Fi network.
- PostgreSQL migration compatibility: this POC CI currently tests SQLite only.

## 5. CI and next phase

Targeted API tests are in `backend/tests/integration/test_spoolbuddy.py`. A separate workflow in the firmware repository runs those tests and an ESP32 firmware build against these branches. After registration/heartbeat tests and bench hardware verification, add a queued, authenticated actuator command protocol with acknowledgement, expiry and printer-idle checks; do not treat contact pulses as motor movement confirmation.

**Do not merge this draft fork PR to main or propose upstream until staging/CI and hardware safety checks are complete.**
