This is the final, comprehensive documentation for the **Advanced Hex Transmit** feature. It has been structured to separate the immediate implementation (Phase 1) from the long-term roadmap (Phase 2) to ensure a smooth, professional review by the TinyGS maintainers.

---

# Feature: Advanced Hex Transmit Tool

## 1. Overview

The **Advanced Hex Transmit** tool is a diagnostic and operational feature for the TinyGS Local Web Dashboard. It enables station operators to transmit raw binary payloads—such as compiled C-structs, encrypted blocks, or custom headers—by inputting a hexadecimal string directly through the local web interface.

## 2. The Problem Statement

TinyGS currently prioritizes text-based communication via the console and MQTT. While effective for simple messaging, these inputs use 8-bit ASCII encoding. This presents significant barriers for specialized operations:

- **Compiled Structs:** Sending binary C-structs for satellite commanding.
- **Protocol Headers:** Injecting specific headers (like AX.25 or CCSDS [^1]) where certain byte values (e.g., `0x00`, `0x1B`) are not representable as printable ASCII.
- **Encrypted Payloads:** Transmitting encrypted blocks that require byte-perfect alignment.

## 3. Implementation Roadmap

### Phase 1: Core Functionality & Compliance (Current PR)

The focus of Phase 1 is a "Fire and Forget" tool that is memory-safe and legally compliant.

- **Sanitized Frontend:** A hex-only input field with an integrated validator (even-length, valid characters) and a real-time byte counter. It includes auto-sanitization to seamlessly strip whitespace, commas, and 0x prefixes from pasted payloads.
- **Legal Gatekeeper & Identification:** A mandatory T&C checkbox requiring the operator to confirm they are operating within local regulations. Includes a Callsign-to-Hex Converter with a 1-click "+ Append" shortcut to help operators quickly comply with licensing ID rules.
- **Local Transaction Log:** A browser-side, audit-ready history table utilizing localStorage. It safely stores the last 100 timestamped transmissions and features a 1-click JSON Export for regulatory audits—requiring zero ESP32 memory overhead.
- **`RadioTxManager` Class:** A new standalone class to handle parsing and radio state checks without bloating `ConfigManager.cpp`.
- **Zero-Heap Backend:** String-to-byte conversion performed using a static buffer to prevent ESP32 memory fragmentation.
- **Operator Responsibilities:**
  - **Configuration:** The user must manually configure the Ground Station to connect to the target satellite via the existing TinyGS web app.
  - **Monitoring:** The user must use the existing TinyGS web app or console to view and verify incoming response packets.

### Phase 2: Advanced Interaction (Future Scope)

- **Response View:** Tight integration with `RadioLib` interrupts to automatically catch and display satellite acknowledgments after a TX burst.
- **LocalStorage History:** Browser-side storage of the last 10–20 transmitted packets for easy recall.
- **Airtime Calculator:** Real-time calculation of packet airtime based on current SF/BW settings to monitor duty cycles.

---

## 4. Technical Architecture

### Component Breakdown

| Component            | Responsibility                                                             |
| -------------------- | -------------------------------------------------------------------------- |
| **`RadioTxManager`** | Standalone class for parsing hex strings and checking radio "busy" states. |
| **`/tx-handler`**    | The AsyncWebServer endpoint receiving the POST payload.                    |
| **Local Dashboard**  | A new "Advanced Uplink" UI card featuring templates and validation.        |

### Constraints

- **TX-to-RX Switching:** The implementation utilizes existing Radio::startTransmit logic to switch modes, blast the payload, and return to a listening state.
- **MTU Enforcement:** Payloads are limited to **255 bytes** per standard LoRa/FSK buffer limits.
- **Endianness:** Users must manually account for the Little-Endian vs. Big-Endian nature of their target satellite processor when entering hex.
- **Header Padding:** TinyGS does not add preamble or CRC beyond the current radio configuration; the hex string is the _exact_ payload passed to the radio.

---

## 5. Usage & Examples: The CCSDS Workflow

The tool bridges the gap between high-level mission planning and low-level hardware execution. Below is a realistic example of a satellite "Health Check" request.

**1. Logical Request (Human Readable)**

- **Packet Version:** 0
- **Packet Type:** Telecommand (1)
- **APID:** 0x1A5 (Station ID)
- **Sequence Flags:** Unsegmented (11)
- **Sequence Count:** 42
- **Packet Length:** 2 bytes (The actual payload `0xDE 0xAD`)

**2. C-Type Memory Layout (Byte-Packed)**
In memory, this data aligns as a packed struct:

```cpp
struct CCSDS_Header {
    uint16_t primary_header; // Version, Type, APID
    uint16_t sequence;       // Flags and Count
    uint16_t length;         // Data length - 1
    uint8_t  payload[2];     // Command: 0xDEAD
} __attribute__((packed));

// Resulting memory bytes: 1D 01 C0 2A 00 01 DE AD

```

**3. Final Hex Input**
The operator simply enters the following into the dashboard to ensure no ASCII corruption occurs:

> `01A5C0010007DEAD`

---

## 6. Testing & Evidence

| Test Case          | Input     | Expected Result                                     |
| ------------------ | --------- | --------------------------------------------------- |
| **Valid Hex**      | `A1B2`    | Radio transmits `0xA1`, `0xB2`. UI shows "Success". |
| **Invalid Length** | `A1B`     | Rejected by frontend/backend as "Invalid Length".   |
| **Invalid Chars**  | `G1Z2`    | Rejection due to non-hexadecimal characters.        |
| **Max Payload**    | 255 Bytes | Verified transmission without buffer overflow.      |

---

[^1]:

4.1.3 PACKET PRIMARY HEADER, RECOMMENDED STANDARD CCSDS 133.0-B-2 BLUE BOOK June 2020 [Reference](https://ccsds.org/Pubs/133x0b2e2.pdf)
