# Levoit Core 300S - Local Control via MQTT Redirect

Control a Levoit Core 300S air purifier locally, with no cloud and no firmware flashing. The device talks MQTT-over-TLS to a Vesync server; you redirect that traffic to your own Mosquitto broker, accept its connection with a self-signed cert, and bridge it into Home Assistant.

This is a 300S supplement to the original 200S work by [epicRE](#credits). The [200S guide](README-200S.md) covers the general approach; this doc covers what's different on the 300S: the self-signed cert test, the corrected `setSwitch` command shape, and the Home Assistant discovery configs.

## Will this work for me?

The one thing that decides it: **does your firmware accept a self-signed cert?** Some 300S/400S firmware pins the CA and rejects it. There's no way to know from the model number - you have to test (Step 2, ~5 minutes, non-destructive).

If your firmware accepts the cert, everything else here works. If it doesn't, your only local-control option is an ESPHome hardware flash ([tuct/levoit](#credits)).

Note that once the redirect is live, the device can't reach Vesync at all: no app control, no OTA updates, no telemetry leaving your network. Most people want exactly that, but it does mean the Vesync app will show the device offline.

## What you need

- A firewall that can do NAT port-forwarding (pfSense, OPNsense, OpenWrt, etc.)
- A local DNS resolver you can add overrides to (Pi-hole, AdGuard, dnsmasq)
- A Linux host for Mosquitto (an LXC/VM with 1 GB RAM is plenty)
- Home Assistant with the MQTT integration
- `tcpdump`, `openssl`, `python3`, and `mosquitto-clients` on a workstation

## How it works

```
[Levoit 300S]  --TLS:1883-->  [firewall NAT redirect]  -->  [Mosquitto broker]  --bridge-->  [Home Assistant]
```

Things worth knowing up front:

- The device uses **TLS on port 1883** (not 8883).
- The TLS ClientHello has **no SNI** - the cert is matched by content, not hostname.
- The device **ignores DHCP DNS for some lookups** and queries `8.8.8.8` / `8.8.4.4` directly. A DNS override alone won't reliably catch it; the NAT port-forward on `:1883` is what actually works.

---

## Step 1 - Sniff the real traffic

Confirm your device behaves as described before building anything:

```sh
tcpdump -i <purifier-iface> -nn 'host <purifier-ip> and (port 1883 or port 8883)'
```

You should see a persistent TCP connection to port **1883**. If you see 8883, multiple hostnames, or UDP, the protocol has changed and this guide may not apply.

---

## Step 2 - Cert acceptance test (the critical one)

This tells you in a few minutes whether your firmware will work at all.

Generate a throwaway cert:

```sh
mkdir -p /tmp/cert-test && cd /tmp/cert-test
openssl req -new -x509 -days 3650 -nodes \
  -keyout broker.key -out broker.crt \
  -subj "/CN=vdmpmqtt.vesync.com" \
  -addext "subjectAltName=DNS:vdmpmqtt.vesync.com,DNS:*.vesync.com,DNS:*.vesyncapi.com"
```

Run a minimal TLS server that logs whether the handshake succeeds:

```python
#!/usr/bin/env python3
import socket, ssl, threading, time

CERT, KEY, PORT = "/tmp/cert-test/broker.crt", "/tmp/cert-test/broker.key", 1883
ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ctx.load_cert_chain(CERT, KEY)
ctx.verify_mode = ssl.CERT_NONE

def handle(conn, addr):
    try:
        s = ctx.wrap_socket(conn, server_side=True)
        print(f"{addr} handshake OK, cipher={s.cipher()[0]}")
        s.settimeout(45)
        data = s.recv(4096)
        if data and data[0] == 0x10:
            print(f"{addr} -> MQTT CONNECT received ({len(data)} bytes)")
        s.close()
    except ssl.SSLError as e:
        print(f"{addr} HANDSHAKE FAILED: {e}")

srv = socket.socket()
srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind(("0.0.0.0", PORT)); srv.listen(8)
print(f"listening on :{PORT}")
while True:
    conn, addr = srv.accept()
    threading.Thread(target=handle, args=(conn, addr), daemon=True).start()
```

Point one purifier at this server (DNS override or a temporary NAT rule), power-cycle it, and watch:

- `handshake OK` + `MQTT CONNECT received` -> your firmware accepts self-signed certs. Continue.
- `HANDSHAKE FAILED: certificate verify failed` -> your firmware pins the CA. Stop; this approach won't work. Use the ESPHome flash instead.
- Nothing at all -> your redirect isn't catching the traffic. Fix Step 4 and retry.

---

## Step 3 - Mosquitto broker

Install on a Linux host with a static IP reachable from both the purifier network and Home Assistant:

```sh
apt update && apt install -y mosquitto mosquitto-clients openssl
```

Generate the production cert (same SANs as the test cert):

```sh
mkdir -p /etc/mosquitto/certs && cd /etc/mosquitto/certs
openssl req -new -x509 -days 3650 -nodes -keyout ca.key -out ca.crt -subj "/CN=Vesync-Local-CA"
openssl req -new -nodes -keyout broker.key -out broker.csr \
  -subj "/CN=vdmpmqtt.vesync.com" \
  -addext "subjectAltName=DNS:vdmpmqtt.vesync.com,DNS:*.vesync.com,DNS:*.vesyncapi.com"
openssl x509 -req -in broker.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out broker.crt -days 3650 \
  -extfile <(printf "subjectAltName=DNS:vdmpmqtt.vesync.com,DNS:*.vesync.com,DNS:*.vesyncapi.com")
chown mosquitto:mosquitto *.key *.crt && chmod 640 *.key
```

`/etc/mosquitto/conf.d/local.conf` - note the TLS listener is on **1883**:

```conf
listener 1883
protocol mqtt
allow_anonymous true
certfile /etc/mosquitto/certs/broker.crt
keyfile /etc/mosquitto/certs/broker.key
require_certificate false
log_type all
connection_messages true
```

Anonymous + verbose logs are right for bring-up. Tighten with users/ACLs once it's working.

Bridge to Home Assistant's broker - `/etc/mosquitto/conf.d/bridge.conf`:

```conf
connection ha_bridge
address <HA_HOST_IP>:1883
remote_username <ha-mqtt-user>
remote_password <ha-mqtt-pw>
cleansession true
start_type automatic
notifications true
notification_topic mqtt/bridge/state

# Use the SAME prefix on both sides - HA's add-on drops prefix-substituted publishes.
topic mqtt/+/v2/req out 0
topic mqtt/+/v2/bypass in 0
topic mqtt/+/v2/rsp in 0
```

```sh
systemctl enable --now mosquitto && journalctl -u mosquitto -f
```

If you use Home Assistant's Mosquitto add-on for the HA side, the bridge user must be a real HA user (**Settings > People > Users**, "Local only" is fine) - the add-on authenticates against HA's user database.

---

## Step 4 - Firewall redirect

Menu paths vary by vendor; the concepts don't. First make an alias listing your Levoit IPs so the rules survive lease changes.

**NAT port-forward (the rule that does the work)** - catches any MQTT a Levoit sends to a non-local address and redirects it to your broker. Using an inverted-subnet destination instead of a hardcoded Vesync IP keeps it working when Vesync's endpoint IPs rotate:

```
Source:               LevoitPurifiers (alias)
Destination:          NOT <purifier_subnet>   (any non-local)
Destination port:     1883
Redirect target:      <broker_ip>:1883
```

**Block rule** - force all Levoit MQTT through the redirect so nothing leaks to Vesync:

```
PASS  TCP from LevoitPurifiers to <broker_ip>:1883
BLOCK TCP from <purifier_subnet> to any :1883 and :8883
```

**DNS override** - in your resolver:

```
vdmpmqtt.vesync.com -> <broker_ip>
```

Override OR block this domain, never both: a Pi-hole that returns `0.0.0.0` for a blocked domain will make the Levoit dial `0.0.0.0:1883` forever.

**Catch the hardcoded DNS** - because the firmware queries `8.8.8.8`/`8.8.4.4` directly, add a second port-forward redirecting the Levoits' outbound `:53` to your resolver:

```
Source:           LevoitPurifiers
Destination:      NOT <resolver_ip>   (inverted)
Destination port: 53
Redirect target:  <resolver_ip>:53
```

If your purifier network policy-routes through a VPN, that route preempts the NAT redirect - add a pass rule using the default gateway, above the VPN route, scoped to the Levoit IPs.

---

## Step 5 - Capture device client IDs

Each device has a unique Vesync client ID (`vs` + 32 hex chars). Subscribe and watch:

```sh
mosquitto_sub -h <broker_ip> -p 1883 --cafile /etc/mosquitto/certs/ca.crt --insecure -t '#' -v
```

```
mqtt/vs<32-hex>/v2/req {"context":{"method":"statusChangeNtyV2",...}, "data":{...}}
```

With multiple purifiers, power-cycle one at a time and note which client ID reconnects - that's how you map IDs to physical units. You'll need these for the discovery script.

---

## Step 6 - 300S command schema

This is the part that differs from the 200S.

All commands use the `bypassV2` envelope, published to `mqtt/<client_id>/v2/bypass`:

```json
{
  "traceId": "<unix_timestamp>",
  "method": "bypassV2",
  "debugMode": false,
  "payload": { "data": <inner_data>, "method": "<inner_method>", "source": "APP" }
}
```

The device replies on `.../v2/bypass/rsp` with `{"traceId":"<echo>","code":<int>}`. `code: 0` is success; `11000000` usually means a wrong method/data shape.

| Action | Inner method | Data |
|---|---|---|
| Power on/off | `setSwitch` | `{"enabled": true\|false, "id": 0}` |
| Fan speed (1-3) | `setLevel` | `{"id": 0, "level": 1\|2\|3, "type": "wind"}` |
| Mode | `setPurifierMode` | `{"mode": "auto"\|"sleep"\|"manual"}` |

**The one that took digging:** the upstream 200S README covers fan level, mode, timer, and display, but not a discrete power on/off. On the 300S, power is `setSwitch` with `{"enabled": true, "id": 0}` - the shape used by the `pyvesync` library (`vesyncpurifier.py`). Other shapes floating around (e.g. `{"powerSwitch": 1}`) return error `11000000`.

Devices publish telemetry on `mqtt/<client_id>/v2/req`. The useful method is `statusChangeNtyV2`, emitted **on state changes only**, carrying `switch1`, `mode`, `fanSpeedLevel`, `airQualityValue`, `filterLife`, `childLock`, `displayPower`. Its payload has both `unchangedStatus` (full last state) and `changedStatus` (delta) - merge with the delta winning. PM2.5 history arrives separately in periodic `uploadAirPurifierData` messages.

Test control by hand before touching Home Assistant:

```sh
mosquitto_pub -h <broker_ip> -p 1883 --cafile /etc/mosquitto/certs/ca.crt --insecure \
  -t "mqtt/<client_id>/v2/bypass" \
  -m '{"traceId":"1","method":"bypassV2","debugMode":false,"payload":{"data":{"enabled":true,"id":0},"method":"setSwitch","source":"APP"}}'
```

If the device powers on and the rsp topic returns `code: 0`, the protocol layer is done.

---

## Step 7 - Home Assistant entities

Rather than hand-writing YAML for every entity, publish retained MQTT Discovery configs and let HA auto-create them. The script below publishes a `fan` entity plus sensors per device.

```python
#!/usr/bin/env python3
"""Publish HA MQTT discovery configs for Levoit Core 300S purifiers.
Run once on any host that can reach HA's MQTT broker.
Empty payload to a config topic (retain=true) removes that entity."""
import json, os
import paho.mqtt.publish as publish

HA_BROKER = "<HA_MQTT_HOST>"
HA_PORT = 1883
HA_USER = "<HA_MQTT_USER>"
HA_PASS = os.environ["HA_MQTT_PW"]
PREFIX = "homeassistant"

DEVICES = [
    {"room": "office", "friendly": "Office", "client_id": "<vs...>", "mac": "<aa:bb:cc:dd:ee:ff>"},
    # add more
]

HEAD = ("{%- if value_json.context.method == 'statusChangeNtyV2' -%}"
        "{%- set u = value_json.data.unchangedStatus | default({}) -%}"
        "{%- set c = value_json.data.changedStatus | default({}) -%}"
        "{%- set m = dict(u, **c) -%}")
TAIL = "{%- endif -%}"

def field(name, default="None"):
    return HEAD + f"{{{{ m.get('{name}', '{default}') }}}}" + TAIL

PM25 = ("{%- if value_json.context.method == 'uploadAirPurifierData' -%}"
        "{%- set q = value_json.data.airQuality | default([]) -%}"
        "{%- if q | length > 0 -%}{{ q[-1].pm2p5 | default(0) }}{%- endif -%}{%- endif -%}")

def device_block(d):
    return {"identifiers": [f"levoit_300s_{d['client_id']}"],
            "name": f"Air Purifier {d['friendly']}", "manufacturer": "Levoit",
            "model": "Core 300S", "connections": [["mac", d["mac"]]]}

def main():
    auth = {"username": HA_USER, "password": HA_PASS}
    msgs = []
    for d in DEVICES:
        cid, room = d["client_id"], d["room"]
        state = f"mqtt/{cid}/v2/req"
        cmd = f"mqtt/{cid}/v2/bypass"
        dev = device_block(d)
        slug = f"levoit_{room}"

        fan = {
            "name": None, "unique_id": f"{slug}_fan", "object_id": f"air_purifier_{room}",
            "state_topic": state,
            "state_value_template": HEAD + "{%- if 'switch1' in m -%}{%- if m.switch1 == 'on' -%}ON{%- else -%}OFF{%- endif -%}{%- endif -%}" + TAIL,
            "payload_on": "ON", "payload_off": "OFF", "command_topic": cmd,
            "command_template": ('{"traceId":"{{ now().timestamp() | int }}","method":"bypassV2","debugMode":false,'
                '"payload":{"data":{"enabled":{% if value == \'ON\' %}true{% else %}false{% endif %},"id":0},'
                '"method":"setSwitch","source":"APP"}}'),
            "preset_modes": ["auto", "sleep", "manual"], "preset_mode_state_topic": state,
            "preset_mode_value_template": HEAD + "{%- if 'mode' in m -%}{{ m.mode }}{%- endif -%}" + TAIL,
            "preset_mode_command_topic": cmd,
            "preset_mode_command_template": ('{"traceId":"{{ now().timestamp() | int }}","method":"bypassV2","debugMode":false,'
                '"payload":{"data":{"mode":"{{ value }}"},"method":"setPurifierMode","source":"APP"}}'),
            "percentage_state_topic": state,
            "percentage_value_template": HEAD + "{%- set lvl = m.get('fanSpeedLevel', 255) -%}"
                "{%- if lvl == 1 -%}33{%- elif lvl == 2 -%}66{%- elif lvl == 3 -%}100{%- else -%}0{%- endif -%}" + TAIL,
            "percentage_command_topic": cmd,
            "percentage_command_template": ('{"traceId":"{{ now().timestamp() | int }}","method":"bypassV2","debugMode":false,'
                '"payload":{"data":{"id":0,"level":{% if value <= 33 %}1{% elif value <= 66 %}2{% else %}3{% endif %},'
                '"type":"wind"},"method":"setLevel","source":"APP"}}'),
            "speed_range_min": 1, "speed_range_max": 100, "device": dev,
            "availability_topic": "mqtt/bridge/state", "payload_available": "1", "payload_not_available": "0",
        }
        msgs.append({"topic": f"{PREFIX}/fan/{slug}_fan/config", "payload": json.dumps(fan), "retain": True})

        for key, name, unit, tpl, dev_class in [
            ("filter_life", "Filter Life", "%", field("filterLife", "0"), None),
            ("aqv", "Air Quality", None, field("airQualityValue", "0"), None),
            ("pm25", "PM2.5", "µg/m³", PM25, "pm25"),
        ]:
            cfg = {"name": name, "unique_id": f"{slug}_{key}", "object_id": f"air_purifier_{room}_{key}",
                   "state_topic": state, "value_template": tpl, "device": dev}
            if unit: cfg["unit_of_measurement"] = unit
            if dev_class: cfg["device_class"] = dev_class
            msgs.append({"topic": f"{PREFIX}/sensor/{slug}_{key}/config", "payload": json.dumps(cfg), "retain": True})

    print(f"Publishing {len(msgs)} configs to {HA_BROKER}:{HA_PORT}")
    publish.multiple(msgs, hostname=HA_BROKER, port=HA_PORT, auth=auth)
    print("Done.")

if __name__ == "__main__":
    main()
```

```sh
pip install paho-mqtt
HA_MQTT_PW='<password>' python3 discovery.py
```

You'll get `fan.air_purifier_office`, `sensor.air_purifier_office_filter_life`, and so on. Extend the sensor loop for child lock, display, RSSI, etc. as needed.

---

## Keeping state across HA restarts (optional)

Because the device only emits `statusChangeNtyV2` on a state change, entities can read `unknown` after an HA restart until the next change happens. A small daemon that re-publishes those messages with `retain=true` fixes it - any new subscriber gets the last-known state immediately.

```python
#!/usr/bin/env python3
import json, hashlib
from collections import deque, defaultdict
import paho.mqtt.client as mqtt

RECENT = defaultdict(lambda: deque(maxlen=5))  # dedup to prevent loops

def on_message(c, u, msg):
    try:
        payload = msg.payload.decode()
        if json.loads(payload).get("context", {}).get("method") != "statusChangeNtyV2":
            return
        cid = msg.topic.split("/")[1]
        h = hashlib.md5(payload.encode()).hexdigest()
        if h in RECENT[cid]:
            return
        RECENT[cid].append(h)
        c.publish(msg.topic, payload, retain=True)
    except Exception as e:
        print(f"error: {e}")

def on_connect(c, u, f, rc):
    c.subscribe("mqtt/+/v2/req")

c = mqtt.Client()
c.on_connect, c.on_message = on_connect, on_message
c.connect("127.0.0.1", 1883)
c.loop_forever()
```

Run it as a systemd service on the broker host. (paho-mqtt v2 needs `callback_api_version=mqtt.CallbackAPIVersion.VERSION2` and adjusted callback signatures; pin `paho-mqtt<2` to use the code above as-is.)

---

## Gotchas worth knowing

- **Test cert with a real TLS server, not `openssl s_server`** - it exits per connection and can hide a successful handshake. The Python server in Step 2 is reliable.
- **Same topic prefix on both sides of the bridge.** HA's Mosquitto add-on silently drops prefix-substituted publishes.
- **`cleansession true` during bring-up.** `false` carries stale subscription state across restarts.
- **Filter reset is device-side only.** Hold the reset button; the device pushes `filterLife: 100` on its own. If it sticks at 0%, the press didn't register.
- **Hard-refresh the HA frontend** (Cmd/Ctrl+Shift+R) after entity renames - Lovelace caches aggressively.

---

## Not yet working

- `setDisplay` - the upstream 200S `{"state": false}` shape returns `11018000` on the 300S. Display state reads fine via `statusChangeNtyV2.displayPower`; only write control is missing.
- Timers / scheduling - not investigated. The 200S doc shows `addTimer` with `{"action":"off","total":<seconds>}`; untested on 300S.

Contributions welcome, especially: firmware versions where self-signed is accepted vs rejected, 400S validation, and the correct `setDisplay` shape.

---

## Credits

- **[epicRE/levoit-local-control](https://github.com/epicRE/levoit-local-control)** - the original 200S protocol work: the `bypassV2` envelope, `setLevel`, `setPurifierMode`, and the MQTT topic schema all come from here. This doc is a 300S supplement, not a replacement.
- **[pyvesync](https://github.com/webdjoe/pyvesync)** - source of the `{"enabled": bool, "id": 0}` `setSwitch` shape that the 300S needs.
- **[tuct/levoit](https://github.com/tuct/levoit)** - the ESPHome flash, your fallback if your firmware rejects self-signed certs.
