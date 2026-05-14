# RSMP v4 Architecture and System Requirements

---

## Status

> **Draft** 
> Sections marked **[TBD]** require further clarification.

---

# 1. Communication model

## 1.1 Transport

* RSMP v4 uses **MQTT** as the sole transport protocol.
* A single **standard MQTT broker** (no proprietary extensions required) serves as the central message hub.
* All participants — devices, supervisors, and client applications — connect as MQTT clients.
* The broker is the only required infrastructure component for the communication layer.

## 1.2 Participants

| Role           | Description                                                                                                                                                                                                             |
|----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Device**     | A field device (e.g., traffic signal controller, detector). Publishes state and data; subscribes to command topics.                                                                                                     |
| **Supervisor** | A privileged service assigned to a device category. Responsible for device lifecycle, history, and command issuance.                                                                                                    |
| **Client**     | Any consumer application (analytics, dashboards, third-party systems). Read-only access in most cases, can use specific commands relevant to the type of the client (scenario control, public transport priority, etc.) |

* Each device has **exactly one supervisor** (determined by device category).
* Different device categories may use different supervisor instances, all sharing a common broker.

---

# 2. Device identity

* Every device is identified by a **unique device ID** (`{device-id}`).
* Device IDs conform to the following rules:
  * Maximum **31 characters**
  * Allowed characters: **lowercase letters (a–z)**, **digits (0–9)**, **dot (`.`)**, **dash (`-`)**
  * Must start with a **lowercase letter (a–z)**
  * Regex: `^[a-z][a-z0-9.\-]{0,30}$`
* Device type is **not** encoded in the topic path; it may appear in the payload (self-description).
* Each device is configured with its own **key pair and a certificate** used for broker authentication.

## 2.1 Device provisioning

Two provisioning paths are defined:

### Path A — Vendor manifest (preferred)

Vendors act as their own **Certificate Authority (CA)**. Each device is issued an **X.509 client certificate** signed by the vendor CA, with `CN = {device-id}`.

**Provisioning steps:**

1. The vendor generates one or more CA certificates (self-signed) and shares them with the broker administrator out-of-band.
2. The broker administrator imports the vendor CA certificate(s) into the broker's trusted CA store.
3. The vendor issues a device certificate per device:
   ```
   Subject:  CN={device-id}
   Issuer:   CN={Vendor} RSMP CA
   Validity: Not Before: <issue date>, Not After: <expiry date>
   Key:      EC P-256
   ```
4. The device is configured with its certificate and private key.
5. On connect, the broker validates the TLS client certificate against the trusted CA store and maps `CN` to the device's ACL entry. No per-device import by the admin is required.

### Multiple CAs per vendor

Vendors MAY operate multiple CA certificates (e.g., per product line, manufacturing site, or region). All CAs must be explicitly imported by the broker administrator. Vendors are responsible for ensuring **device IDs are globally unique** across all their CAs — the broker admin is not expected to detect collisions.

### Key validity and rotation

* Each device certificate carries a `notAfter` expiry date.
* The broker enforces expiry at the TLS handshake level — no custom logic required.
* On key rotation or certificate renewal, the vendor issues a new certificate for the same `CN`. The broker accepts any valid, non-expired, non-revoked certificate that chains to a trusted CA with the correct CN.
* Multiple valid certificates MAY coexist for the same device (e.g., during a rollover window).

### Revocation

* The vendor issues a **signed revocation manifest** (a CRL or equivalent) identifying the certificate serial number(s) to revoke.
* The broker administrator applies the revocation to the broker's CRL or blocklist.
* Revocation of one certificate does not affect other valid certificates for the same device.

### ACL interaction

The broker ACL is keyed on the TLS client certificate CN (`= device-id`). This means:

| Concern                            | Handled by                                                                        |
|------------------------------------|-----------------------------------------------------------------------------------|
| Is the key trusted?                | TLS — certificate chains to an imported vendor CA                                 |
| Is the key still valid?            | TLS — certificate `notAfter`                                                      |
| Is the key revoked?                | CRL / broker blocklist (via revocation manifest)                                  |
| What is this device allowed to do? | ACL rule on `CN = {device-id}` — unchanged regardless of which key/cert is active |

A single wildcard ACL rule per device covers all valid certificates for that device, with no ACL changes needed on key rotation.

---

# 3. Topic structure

## 3.1 Namespace

All RSMP topics are rooted under the `rsmp/` prefix:

```
rsmp/{device-id}/{category}/{...}
```

## 3.2 Topic categories

| Category     | Path pattern                              | Update frequency   | Retained | Description                                                                                                       |
|--------------|-------------------------------------------|--------------------|----------|-------------------------------------------------------------------------------------------------------------------|
| **Static**   | `rsmp/{device-id}/static/#`               | Low / on change    | **Yes**  | Device configuration, capabilities, self-description. Late-joining subscribers receive current state immediately. |
| **Dynamic**  | `rsmp/{device-id}/dynamic/#`              | High / real-time   | No       | Live operational data, measurements, status updates.                                                              |
| **History**  | `rsmp/{device-id}/history/#`              | Burst on reconnect | No       | Offline-buffered updates replayed after reconnection, mirroring `dynamic/` sub-topic structure.                   |
| **Command**  | `rsmp/{device-id}/cmd/{command}`          | Event-driven       | No       | Command delivery (supervisor/client → device).                                                                    |
| **Response** | `rsmp/{device-id}/cmd/{command}/response` | Event-driven       | No       | Command acknowledgement and result (device → supervisor).                                                         |

## 3.3 Static topics

All devices MUST publish the following static topic:

```
rsmp/{device-id}/static/info
```

Payload includes at minimum:

| Field          | Description                                                                  |
|----------------|------------------------------------------------------------------------------|
| `type`         | Device type identifier                                                       |
| `manufacturer` | Manufacturer name                                                            |
| `model`        | Model name or number                                                         |
| `firmware`     | Firmware version                                                             |
| `buffer_drops` | Cumulative count of messages dropped from the offline buffer due to overflow |

Additional fields MAY be included. The `static/info` topic MUST be published as retained. The `buffer_drops` field changes only when the device reconnects.

Device categories define which additional topic paths (beyond the mandatory baseline) or extend standard payload fields.


## 3.4 Dynamic topics (examples)

```
rsmp/{device-id}/dynamic/status          # operational status and health
rsmp/{device-id}/dynamic/measurements/#  # telemetry streams
rsmp/{device-id}/dynamic/events          # event log entries
```

## 3.5 Command topics

* Commands are issued by publishing to `rsmp/{device-id}/cmd/{cmd-or-group}`.
* Devices **subscribe to** their own command topics.
* Command responses are published by the device to the corresponding `/response` sub-topic.

### Command issuers

| Issuer                  | Scope                                                                   |
|-------------------------|-------------------------------------------------------------------------|
| **Supervisor**          | Most commands; access is per-command-group, not necessarily all groups. |
| **Client applications** | Commands within the scope of their use case, explicitly granted in ACL. |

Inter-device coordination is achieved through **topic subscriptions only** — devices subscribe to relevant dynamic topics of other devices. No device-to-device command publishing is defined at this stage.

The set of permitted command groups per issuer is enforced by the broker ACL.

## 3.6 Versioning

* Protocol version is **not** encoded in the topic path.
* Version information is carried in the **payload** (e.g., a `version` field in the message envelope).
* Backward compatibility is maintained at the payload level.

---

# 4. Access control

## 4.1 Mechanism

* Access control is enforced via standard **MQTT broker ACLs** (per-client publish/subscribe rules).
* No custom broker extensions are required.

## 4.2 Device ACL rules

Each device client (`{device-id}`) is restricted to its own topic subtree:

| Operation     | Allowed topic pattern                                          |
|---------------|----------------------------------------------------------------|
| **Publish**   | `rsmp/{device-id}/#`                                           |
| **Subscribe** | `rsmp/{device-id}/cmd/#`                                       |
| **Subscribe** | `rsmp/+/dynamic/#` *(read-only, for inter-device observation)* |

A device **cannot** publish to another device's topic path. This is enforced by the broker.

## 4.3 Supervisor ACL rules

The supervisor has wildcard permissions across all devices:

| Operation     | Allowed topic pattern |
|---------------|-----------------------|
| **Subscribe** | `rsmp/+/#`            |
| **Publish**   | `rsmp/+/cmd/#`        |


> **Note:** A supervisor may be restricted to specific command groups by narrowing the publish pattern (e.g., `rsmp/+/cmd/control`). The default assumption is wildcard access over its assigned device population.

## 4.4 Client ACL rules

Regular clients receive read-only access by default. Most data is readable within the broker's trust boundary:

| Operation     | Allowed topic pattern                                                 |
|---------------|-----------------------------------------------------------------------|
| **Subscribe** | `rsmp/{device-id}/static/#`                                           |
| **Subscribe** | `rsmp/{device-id}/dynamic/#`                                          |
| **Publish**   | `rsmp/{device-id}/cmd/{granted-group}` *(only if explicitly granted)* |

* All data topics are readable by any authenticated client — there are no restricted read topics.
* Clients authorized to issue commands receive publish rights scoped to specific command groups relevant to their use case.

## 4.5 Device-to-device coordination

Devices coordinate with each other exclusively through **topic subscriptions** — a device subscribes to the relevant `dynamic` topics of other devices it needs to observe. No device-to-device command publishing is defined.

## 4.6 Per-device and per-topic policies

* ACL rules are defined **per client identity** (device, supervisor, named application).
* Wildcard patterns (`#`, `+`) are used to express broad rules; narrower rules restrict sensitive sub-topics.
* Access policies are managed centrally in the broker configuration.

---

# 5. Connection and lifecycle

## 5.1 Device presence

* Each device MUST configure a **Last Will and Testament (LWT)** message on connect.
* The LWT is published by the broker to `rsmp/{device-id}/dynamic/status` on unexpected disconnection.
* The device also publishes to this topic on clean connect and disconnect.

### Status payload — state bits


> The exact JSON schema for the status payload is defined below.

### Status payload schema

**Topic:** `rsmp/{device-id}/dynamic/status` — retained, QoS 1

```json
{
  "v": 1,
  "flags": ["local", "disconnected", "error", "warning", "notice", "active", "idle"],
  "features": ["rsmp-4-core", "rsmp-4-tlc"]
}
```


| Field            | Type              | Description                                                                |
|------------------|-------------------|----------------------------------------------------------------------------|
| `v`              | integer           | Payload schema version                                                     |
| `flags`          | string enum array | List of standard state flags                                               |
| `features`       | string array      | List of functional features (equivalent to SXLs supported by the device)   |


| flags enum       | Description                                                                |
|------------------|----------------------------------------------------------------------------|
| `local`          | Device is under local/manual control                                       |
| `disconnected`   | Set in the LWT; indicates loss of connection                               |
| `error`          | One or more active alarms at priority 1 (high)                             |
| `warning`        | One or more active alarms at priority 2 (medium)                           |
| `notice`         | One or more active alarms at priority 3 (low)                              |
| `active`         | Device is in active/normal operating mode (mutually exclusive with `idle`) |
| `idle`           | Device is powered but not in active use (mutually exclusive with `active`) |

**LWT:** the device pre-configures the LWT at connect time with `disconnected: true` and all other state fields cleared. The broker publishes it automatically on unexpected disconnection.

## 5.2 Offline resilience and history

The device handles offline resilience through a two-buffer mechanism. The supervisor is the sole consumer of the history path. History payloads are published unmodified — the original event timestamp from the `dynamic/` message is preserved as-is.

### Sequence numbers

Every dynamic update is assigned a **monotonically increasing sequence number** (`seq`). Sequence numbers are used for deduplication and acknowledgement across both buffers.

### Online buffer

While the device is connected:

1. The device publishes updates to `rsmp/{device-id}/dynamic/...` as normal.
2. Each published update is also stored in the **online buffer** (device-side FIFO queue).
3. The online buffer capacity equals the **MQTT client's maximum in-flight message window** — it is intentionally small and sized to cover messages in transit only.
4. The supervisor acknowledges received updates by publishing to:
   ```
   rsmp/{device-id}/cmd/online-ack
   ```
   Payload: `{ "seq": <last-received-seq> }` — all updates up to and including this sequence number are removed from the online buffer.
5. The supervisor may skip directly to the latest sequence number it has received.

### Transition to offline buffer

The entire online buffer is moved to the **offline buffer** when:
- The connection is lost or times out.

Should the online buffer overflow due to ack-congestion, it is flushed into the offline buffer.

### Offline buffer

The offline buffer is a **FIFO queue** with a default capacity of **1000 messages**. Devices MAY provide a larger capacity.

When the buffer is full, the **oldest messages are dropped** to make room for new ones. The count of dropped messages MUST be tracked by the device and reported in the static info or a dedicated status field so that the supervisor can detect data loss.

When the device reconnects:

1. The device publishes buffered updates one by one to `rsmp/{device-id}/history/...`, mirroring the structure of the corresponding `dynamic/` sub-topics.
2. The supervisor acknowledges received history updates by publishing to:
   ```
   rsmp/{device-id}/cmd/offline-ack
   ```
   Payload: `{ "seq": <last-received-seq> }` — all updates up to and including this sequence number are removed from the offline buffer.
3. Live dynamic updates continue in parallel on `dynamic/` topics — history replay does not block online operation.

### Topic summary

| Topic                              | Direction           | Description                                     |
|------------------------------------|---------------------|-------------------------------------------------|
| `rsmp/{device-id}/dynamic/...`     | Device → broker     | Live operational updates                        |
| `rsmp/{device-id}/history/...`     | Device → broker     | Offline-buffered updates, replayed on reconnect |
| `rsmp/{device-id}/cmd/online-ack`  | Supervisor → device | Acknowledge live updates up to `seq`            |
| `rsmp/{device-id}/cmd/offline-ack` | Supervisor → device | Acknowledge history updates up to `seq`         |


## 5.3 Retained messages

* All **static** topics MUST be published with the `retained` flag so that late-joining subscribers receive the current state without waiting for the next publish cycle.
* Dynamic and command topics MUST NOT use retained messages unless explicitly specified.

---

# 6. Scalability constraints

* The broker and ACL infrastructure MUST support at least **1000 simultaneously connected devices**.
* Multiple client applications MAY subscribe to the same topics concurrently without impact on device behavior.
* Supervisor instances are scoped per device category, allowing for each device category to be managed independently while maintaining a single broker instance and wide interoperability between the device categories.

---

# 7. QoS requirements

* **Default QoS: 1** (at-least-once delivery) for all topics unless otherwise specified.
* **QoS 0** (fire-and-forget) MAY be specified for high-frequency telemetry topics where occasional loss is acceptable and latency is prioritized.
* **Commands and responses** MUST use QoS 1 to ensure delivery.
* QoS level is defined per topic category in the device-type specification.

---

# 8. Security boundaries

* All MQTT connections MUST use **TLS** for confidentiality and integrity in transit.
* Every client (device, supervisor, application) MUST authenticate to the broker with a unique identity using its **key pair or certificate**.
* Devices MUST be configured with the **broker's CA certificate** to verify the broker's identity at connection time, ensuring they connect only to the designated broker.
* The broker ACL is the single enforcement point for authorization.
* Device credentials and ACL entries are provisioned via vendor manifest, see section 2.1.
* Centralized management of credentials and ACLs MUST be supported.

---
