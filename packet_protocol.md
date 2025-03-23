# OrbiPacket Protocol Specification - V1.1.0

## Introduction

The current document presents the technical specification for the packet-based communication protocol designed for interaction with CanSat's. The current version of this protocol (1.1.0) was developed by OrbiSat Oeiras during the 12th edition of CanSat Portugal.

![Diagram representing the protocol](protocol_diagram.svg)

## Goals

The protocol was designed with the following goals in mind:

- minimal overhead
- ease of use and ease of implementation
- support for telemetry (TM) and telecommands (TC)

## Packet Structure

Each packet consists of the following fields, which must be encoded in the specified order:

| Field                | Size        | Description                                                     |
| -------------------- | ----------- | --------------------------------------------------------------- |
| **Header**           | 11 bytes    | Contains metadata for packet processing.                        |
| - Version            | 1 byte      | Protocol version for compatibility management.                  |
| - Length             | 1 byte      | Size of the payload in bytes.                                   |
| - Control            | 1 byte      | Encodes packet type and device ID.                              |
| - Timestamp          | 8 bytes     | Time since Unix epoch in nanoseconds (64-bit unsigned integer). |
| **Payload**          | 0–255 bytes | Application-specific data.                                      |
| **CRC**              | 2 bytes     | Cyclic redundancy check for error detection.                    |
| **Termination Byte** | 1 byte      | A fixed value (`0x00`) marking the end of the packet.           |

## Packet Fields

### Header

#### Version
- **Size**: 1 byte
- **Description**: Indicates the protocol version. Breaking updates to the protocol increment this field to ensure backwards compatibility.
- **Value**: the current version of the protocol (1.1.0) specifies `0x01` as the version byte.

#### Length
- **Size**: 1 byte
- **Description**: Specifies the size of the payload in bytes, ranging from 0 to 255.

#### Control
- **Size**: 1 byte
- **Description**: Encodes metadata about the packet type and device identifier.
  - **TM/TC Flag**: 1 bit, where `0` indicates telemetry and `1` indicates a telecommand.
  - **Device ID**: 5 bits, uniquely identifying the source device for TM or target device for TC. Note that device IDs should remain consistent across TM and TC packets.
  - **Reserved Bits**: 2 bits, reserved for future use. The value of this field should be ignored.

#### Timestamp
- **Size**: 8 bytes
- **Description**: Represents the time since the Unix epoch in nanoseconds as a 64-bit unsigned integer.

### Payload

- **Size**: 0–255 bytes
- **Description**: Contains application-specific data, such as telemetry readings or command instructions. The structure of the payload is currently up to the application, but that may be subject to change in future versions.

### CRC

- **Size**: 2 bytes
- **Description**: Cyclic redundancy check value computed over all preceding fields (excluding the termination byte). The generator polynomial used by the protocol is [CRC-16/OPENSAFETY-B](https://reveng.sourceforge.io/crc-catalogue/all.htm#crc.cat.crc-16-opensafety-b), or `0xbaad` in Koopman's notation.

### Termination Byte

- **Size**: 1 byte
- **Description**: Marks the end of the packet, ensuring clear packet delimitation.
- **Value**: Fixed at `0x00`

## Field Order and Endianness

Packet fields must be encoded in the specified order, as a byte string. Multi-byte fields are encoded in **little-endian**.

## Encoding

To prevent the occurrence of the termination byte (`0x00`) within the packet, the protocol employs **COBS (Consistent Overhead Byte Stuffing)** encoding. This encoding is applied to all fields except the termination byte itself, simplifying parsing and improving error resilience.

## Packet Operations

### Creation
- Packet fields, excluding the CRC and termination bytes, are converted into binary and chained together in the specified order.
- A CRC is computed over the resulting binary string, and appended to it.
- COBS encoding is applied to the entire binary string, and the termination byte is appended to it, concluding packet creation.

### Transmission
- Packets are transmitted in binary, over an application-specific digital or physical medium (e.g. radio).

### Reception
- Packets are decoded using COBS
- The packet structure is recovered from the decoded binary string.

### Validation
- Packets which don't start at `0x00` (the termination byte of the previous packet) are discarded. This allows for re-synchronization in case of a broken or lossy connection.
- Packets which cannot be properly decoded with COBS are discarded.
- Packets which fail CRC validation are discarded.
- Packets with erroneous length fields are discarded.

### Acknowledgment and Retransmission
Acknowledgment of packets, and their retransmission in case of packet loss, is not currently required by the protocol. It is up to implementations to decide if, and how, this is handled.

### Version Negotiation
No version negotiation strategy is enforced by the packed in the event of communicators having mismatched versions.


## Example Packet

| Field           | Example Value                | Notes                     |
| --------------- | ---------------------------- | ------------------------- |
| **Version**     | `0x01`                       | Protocol version 1.1.0    |
| **Length**      | `0x04`                       | Payload size: 4 bytes     |
| **Control**     | `0x81`                       | Telecommand, Device ID 1  |
| **Timestamp**   | `0x000001787ABCEF0123456789` | Example timestamp         |
| **Payload**     | `0xDEADBEEF`                 | Application-specific data |
| **CRC**         | `0x5A`                       | Example CRC value         |
| **Termination** | `0x00`                       | Packet terminator         |

## Packet Overhead

The fixed fields (header and CRC) result in a total, unencoded overhead of 13 bytes, so packet sizes range from 13 to 268 bytes, depending on payload length. COBS encoding has a maximum overhead of 1 byte per 254 bytes of unencoded data. Thus, accounting for the termination byte, the maximum overhead is 15 or 16 bytes per packet.

## Implementation Guidelines

If available, use a reference implementation of the protocol for the chosen language. Otherwise, make sure to conform to the specifications of this document.

### Versioning

The protocol is versioned using [Semantic Versioning](https://semver.org/). Breaking changes always increment the version byte. Implementations of the protocol must clearly specify to the end user the supported protocol versions.

### CRC Computation

The generator polynomial used by the protocol is [CRC-16/OPENSAFETY-B](https://reveng.sourceforge.io/crc-catalogue/all.htm#crc.cat.crc-16-opensafety-b), or `0xbaad` in Koopman's notation. The maximum packet length (excluding the CRC and termination bytes) is 255 + 11 = 266 bytes or 2128 bits. According to [Koopman's research](https://users.ece.cmu.edu/~koopman/crc/c16/0xbaad_len.txt), this polynomial can protect 7985 bits at a Hamming distance of 4 and 108 bits (equivalent to 2 bytes of payload data) at a Hamming distance of 5, with negligible protected lengths for higher Hamming distances. This is deemed sufficient for this protocol. The parameters of this polynomial are:

- *width*: `16`
- *poly*: `0x755b`
- *init*: `0x0000`
- *refin*: `false`
- *refout*: `false`
- *xorout*: `0x0000`
- *check*: `0x20fe`
- *residue*: `0x0000`

### COBS
Reference implementations of the COBS algorithm should be used whenever possible. Otherwise, refer to the literature for details on the encoding and decoding process. The packet data should be stuffed to remove the termination byte, that is, `0x00`.
