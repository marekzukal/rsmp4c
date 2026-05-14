
# RSMP v4 business requirements

---

# 1. Core capability requirements (MUST)

## 1.1 Unified device interoperability

RSMP shall:

* Provide a **common communication framework** for heterogeneous field devices
* Support a **shared baseline capability set** across all device types:

    * Connection state (online/offline, lifecycle)
    * Operational status and health monitoring
    * Basic remote control / command execution

**Intent:** vendor-neutral integration baseline

---

## 1.2 Device-type standardization

RSMP shall:

* Enable **standardized functional models per device type**
* Ensure **cross-manufacturer interoperability** within a device type:

    * Extended telemetry/data exposure
    * Common maintenance-related operations (where applicable)
* Support:

    * Standard parameterization (if defined)
    * Standard programming interfaces (if defined)

**Scope examples:**

* Traffic signal controllers
* Detectors / counters
* Incident detection systems
* Access control systems

---

## 1.3 Data exchange and coordination

RSMP shall:

* Support **device-to-device communication** for:

    * State awareness
    * Data exchange
    * Coordination and synchronization
* Enable **low-latency real-time data distribution**

**Intent:** support cooperative traffic control / distributed logic

---

## 1.4 Data persistence and access

RSMP shall:

* Support **centralized data collection and storage**
* Enable **historical data access** for analysis
* Define:

    * Data retention policies
    * Controlled data access mechanisms

---

## 1.5 Reliability and resilience

RSMP shall:

* Tolerate:

    * Intermittent connectivity
    * Low-quality network conditions
* Ensure:

    * **No data loss** over extended disconnection periods (target: ≥ 30 days)

---

## 1.6 Security and access control

RSMP shall:

* Ensure **confidentiality and integrity of communication**
* Provide:

    * Authentication (device and client level)
    * Authorization (coarse, functionality block level)
* Support:

    * Per-device and per-application identity
    * Per-data / per-operation access policies
* Allow:

    * Centralized security management
    * Third-party and customer-owned infrastructure

---

## 1.7 Extensibility model

RSMP shall:

* Support **extensible data and functionality models**
* Enable:

    * Manufacturer-specific extensions
    * Project-specific extensions
* Ensure:

    * Controlled interoperability
    * No impact on core protocol behavior

---

## 1.8 Time management

RSMP shall:

* Define consistent handling of:

    * Time synchronization
    * Time zones and DST
    * Leap seconds
* Provide mechanisms to:

    * Operate under unreliable or manually configured time sources

---

## 1.9 Metadata and self-description

RSMP shall:

* Enable **device self-description**
* Reduce dependency on central configuration
* Provide standardized metadata describing:

    * Capabilities
    * Supported profiles
    * Data structures

---

## 1.10 Versioning and compatibility

RSMP shall:

* Provide a **versioning model**
* Ensure:

    * Backward compatibility where possible
    * Controlled evolution of features

---

## 1.11 Testing and conformity

RSMP shall:

* Define:

    * Conformance requirements
    * Test scenarios
    * Acceptance criteria
* Enable certification or validation processes

---

## 1.12 Communication infrastructure assumptions

RSMP shall:

* Define **requirements and boundaries** for:

    * Networking environment
    * Responsibility split (device vs infrastructure)
* Address:

    * IP-based communication
    * Interaction with network components (routers, firewalls)

---

# 2. Allowed capabilities (SHOULD / MAY)

## 2.1 Manufacturer-specific access

RSMP shall:

* Allow **direct vendor-specific access paths** for:

    * Troubleshooting
    * Diagnostics
    * Logs and internal states

**Constraint:** outside standardized core

---

## 2.2 Partial functionality support

RSMP shall:

* Allow devices to:

    * Implement only a **subset of standardized functionality**
* Support:

    * Market-specific or cost-driven feature sets

---

## 2.3 Multi-context operation

RSMP shall:

* Support differing:

    * Regulatory environments
    * Operational modes
    * Customer requirements

---

# 3. Explicit non-scope (SHALL NOT)

RSMP shall NOT:

* Define or standardize:

    * Manufacturer-specific configuration
    * Device troubleshooting procedures
    * Debugging mechanisms

**Interpretation:**
Protocol = interoperability layer, not device lifecycle tooling

---

# 4. Cross-cutting constraints

## 4.1 Separation of concerns

* Protocol defines **interfaces and data exchange**, not implementation
* Storage, analytics, and tooling remain external

## 4.2 Vendor neutrality

* No dependency on specific manufacturers or proprietary stacks

## 4.3 Scalability

* Must support:

    * Large device fleets (>= 1000 devices in the network)
    * Constant monitoring of all connected devices
    * Multiple consuming applications

---
