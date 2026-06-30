# SXL-TLC specification

Mandatory part of the RSMPv4 that shall be supported by all TLC devices.

## Content


| Topic path                 | Retained | QoS | Direction | Historical | Purpose                                           |
|----------------------------|----------|-----|-----------|------------|---------------------------------------------------|
| `static/tlc-configuration` | Yes      | 1   | D -> B    | Yes        | Configuration description                         |
| `dynamic/tlc-status`       | Yes      | 1   | D -> B    | Yes        | Low frequency updates                             |
| `dynamic/tlc-alarms`       | Yes      | 1   | D -> B    | Yes        | Current alarm state                               |
| `dynamic/tlc-traffic`      | No       | 1   | D -> B    | Yes        | High frequency updates                            |
| `dynamic/tlc-coordination` | No       | 1   | D -> B    | No         | Coordination messages                             |
|                            |          |     |           |            |                                                   |
| `cmd/set-program`          | No       | 1   | S -> D    | No         | Set signal program                                |
| `cmd/set-schedule`         | No       | 1   | S -> D    | No         | Set signal program schedule                       |
| `cmd/set-emergency`        | No       | 1   | S -> D    | No         | Set emergency route                               |
| `cmd/set-input`            | No       | 1   | S -> D    | No         | Set input                                         |
| `cmd/clear-alarms`         | No       | 1   | S -> D    | No         | Clear alarms                                      |
| `cmd/operation/L<level>`   | No  `     | 1   | S -> D    | No         | Operation, signal program and run-time parameters |