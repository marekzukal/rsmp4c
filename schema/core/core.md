# Core specification

Mandatory part of the RSMPv4 that shall be supported by all devices.

## Content


| Topic path                | Retained | Direction | Historical | Purpose                                           |
|---------------------------|----------|-----------|------------|---------------------------------------------------|
| `static/info`             | Yes      | D -> B    | Yes        | Device information                                |
|                           |          |           |            |                                                   |
| `cmd/online-ack`          | No       | S -> D    | No         | Online buffer ACK                                 |
| `cmd/offline-ack`         | No       | S -> D    | No         | Offline buffer ACK                                |
|                           |          |           |            |                                                   |
