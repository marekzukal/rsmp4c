# RSMP v4 Design Rationale

---

## Status

> **Draft**
> This document explains why selected architectural choices were made. It is non-normative and should not repeat requirements from the business or architecture documents.

---

# 1. Transport and communication model

## 1.1 MQTT as the transport

MQTT is mature, widely implemented, and available in both commercial and open-source broker implementations. Using it keeps RSMP focused on protocol semantics rather than transport mechanics.

Its publish/subscribe model also fits field-device telemetry: new consumers can subscribe to existing streams without device-side changes.

## 1.2 Standard broker as the integration point

Using a standard broker avoids coupling RSMP v4 to a vendor or custom server implementation. Operators can reuse existing infrastructure and MQTT operational practices.

The broker is the communication hub because it already provides routing, connection handling, authentication, authorization, retained messages, and Last Will publication.

---

# 2. Data ownership and persistence

## 2.1 Server-side data as the primary source

Devices are not treated as queryable databases. They may buffer data for resilience, but long-term storage belongs in server-side services.

This avoids pushing historical query behavior, retention policy, indexing, and access control into field devices.

## 2.2 Single historian responsibility

MQTT is efficient for event streaming but not a natural request/response protocol for historical queries. A dedicated historian is the right boundary for persistence and query APIs.

Other services either consume live data directly or query historical data through the historian outside this standard.

---

# 3. Message structure and consistency

## 3.1 Topic granularity

Data should be split into pieces small enough for efficient subscription and updates, but large enough to remain internally consistent.

There is no explicit consistency boundary across topic paths. If consistency is needed across paths, it must be possible to reconstruct it from the data itself.

## 3.2 Optional members and partial updates

Messages may use optional members to avoid sending irrelevant fields. Partial updates that require consumers to reconstruct state from previous messages are discouraged.

Each message should preferably be meaningful on its own within the consistency boundary of its topic.

---

# 4. Device interaction model

## 4.1 No direct device-to-device publishing

Devices do not publish into each other's topic subtrees. Coordination is based on subscriptions to authorized topics.

This keeps subtree ownership clear, prevents accidental cross-device writes, and keeps authorization rules simple.

## 4.2 Supervisors as command authorities

Commands use controlled command topics rather than peer-to-peer device links. This preserves a clear chain of responsibility and keeps permissions auditable at the broker.

Client applications may receive narrowly scoped command rights, but the default model separates observation from control.

## 4.3 Command levels

Where multiple sources can issue the same command, they should be separated into priority levels.

Commands from higher levels take precedence. Commands at the same level overwrite each other, making the latest command at that level authoritative.

---

# 5. Security and access control

## 5.1 Broker-managed authorization

Centralizing access control in the broker keeps authorization close to routing and avoids scattering checks across devices and applications.

Broker ACLs map directly to the major trust boundaries: device-owned subtrees, command topics, supervisor privileges, and read-only application access.

## 5.2 Certificate-based identities

Certificate-based client identities give the broker a stable basis for authentication and ACL matching, while reusing standard renewal and revocation practices.

Once a participant is authenticated, authorization is enforced through topic permissions.

---

# 6. Extensibility and operational simplicity

## 6.1 Payload-level evolution

Keeping versioning and device type information in payloads preserves stable subscription patterns.

It also keeps topic namespaces compact and lowers migration cost when payload schemas evolve.

## 6.2 Minimal required infrastructure

The design minimizes mandatory infrastructure. A broker is sufficient for communication; storage, analytics, dashboards, and query APIs remain separate services.

This keeps the core standard deployable in small installations while allowing larger systems to add specialized services.

---
