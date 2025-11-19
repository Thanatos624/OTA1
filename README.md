#  OTA Update Simulator

[![Python](https://img.shields.io/badge/Python-3.7%2B-blue?logo=python&logoColor=white)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-Educational-green)](#license--attribution)
[![Status](https://img.shields.io/badge/Status-Active-brightgreen)]()

> 🚧 **Under active development** — This project is a work-in-progress. Features, UI and CLI may change; issues and PRs are welcome.

> A comprehensive simulator demonstrating Over-The-Air (OTA) firmware update flows, security vulnerabilities, and mitigation strategies for embedded vehicle systems.

---

##  Overview

This project simulates a realistic OTA update architecture with:
- **Legitimate OEM Server** (port 5000) serving trusted firmware with SHA256 checksums
- **Malicious Server** (port 5001) demonstrating attack vectors with fake checksums
- **Telematics Control Unit (TCU)** that vulnerably selects the highest version
- **Electronic Control Unit (ECU)** that applies firmware updates

Perfect for **security researchers, embedded systems students, and automotive engineers** learning about OTA security.

---

##   Screenshots


![WhatsApp Image 2025-11-19 at 19 58 32_cae1bf8c](https://github.com/user-attachments/assets/13823d94-96ec-4c68-aa5b-b9cf65ce41e6)
![WhatsApp Image 2025-11-19 at 19 58 34_64e42985](https://github.com/user-attachments/assets/a64cf22f-c559-4d3d-b154-14ed6b4e9d9c)
![WhatsApp Image 2025-11-19 at 19 58 33_769747ec](https://github.com/user-attachments/assets/bae970f3-317a-457b-bbb9-4110913d792a)
![WhatsApp Image 2025-11-19 at 19 58 33_81ede256](https://github.com/user-attachments/assets/14e2a2b6-ad2e-4dae-a19c-a5c4e5c08c2b)
![WhatsApp Image 2025-11-19 at 19 58 33_4929bb4d](https://github.com/user-attachments/assets/4b38408a-849d-4216-9733-7d90cd42d9d6)

---

##  Key Components

| Component | Purpose |
|-----------|---------|
| **`gui_app.py`** | Tkinter GUI orchestrating the full simulation with real-time logs |
| **`oem_server.py`** | Flask server (port 5000) serving legitimate firmware + SHA256 checksums |
| **`malicious_server.py`** | Flask server (port 5001) serving malicious payloads with fake checksums |
| **`tcu_client.py`** | Simulated TCU checking both servers, choosing highest version, verifying checksums |
| **`ecu_receiver.py`** | Simulated ECU receiving, applying, and acknowledging firmware updates |
| **`shared_utils.py`** | Shared utilities: version parsing, SHA256 calculation, file handling |
| **`config.ini`** | Configuration file (auto-generated) with server URLs, security settings, folder paths |

---

## Architecture & Data Flow

This section describes how the simulator's components interact and the exact sequence of events during an OTA update cycle.

High-level architecture:

```
[OEM Server:5000]        [Malicious Server:5001]
        \                      /
         \                    /
          \                  /
           --> [TCU Client] --> [Shared folder -> ECU Receiver]
                          |
                          v
                    [ECU Receiver]
                          |
                          v
                     [TCU ACK Folder]
```

Sequence (detailed data flow):

1. OEM / Malicious servers host files in `updates/` and `malicious_updates/` respectively and expose two endpoints:
   - `GET /check-update` — returns metadata: `{version, filename, checksum, source}`
   - `GET /download/<filename>` — serves the binary/content

2. The TCU Client polls both servers (OEM & Malicious) by calling `/check-update` on each.

3. Each server responds with the latest available metadata (version string, filename, checksum).

4. The TCU chooses the candidate with the highest semantic version (major.minor). In this simulator the TCU is intentionally simplistic and selects purely by version number.

5. If a candidate update is chosen, the TCU downloads it from the corresponding `/download/<filename>` endpoint to `tcu_downloads/`.

6. TCU computes the SHA256 checksum locally and compares it to the server-provided checksum if `checksum_verification_enabled` is true in `config.ini`.
   - If checksums match (or verification is disabled), TCU moves the firmware file to the `ecu_shared_folder`.
   - If checksums do not match and verification is enabled, the file is deleted and the update is aborted.

7. The ECU Receiver monitors the `ecu_shared_folder`. When it detects a firmware file, it simulates applying the update, removes the firmware file, and writes an `.ack` file into `tcu_ack_folder` to signal success.

8. The TCU watches the ack folder; upon receiving the ack it updates `config.ini` to the new version and reports success in the GUI logs.

Vulnerabilities demonstrated by this flow:
- Version-only selection: a malicious server can publish a higher version number to outrank OEM.
- Fake checksums (malicious server) or disabled checksum verification allow malicious payloads to be accepted.
- No server authentication or digital signatures — checksum alone is insufficient for authenticity.

Suggested mitigations:
- Use cryptographic signatures (signed metadata + signed firmware blobs) and validate signatures against a trusted public key.
- Enforce server authentication (TLS + certificate pinning) and restrict allowed update sources.
- Add policy checks (allowed signers, rollback protection, version constraints) before applying updates.

---

##  Security Highlights

This simulator is an **educational demo tool** showcasing real OTA vulnerabilities:
-  **No digital signatures** � easy to spoof checksums
-  **No server authentication** � any server can claim to be OEM
-  **Version-only selection** � highest version wins, regardless of source
-  **Checksum verification** (optional) � can be toggled ON/OFF to test impact

**Learn how to fix these issues:**
- Implement RSA/ECDSA digital signature verification
- Use TLS certificates and server authentication
- Add trusted signer allowlists
- Implement secure boot and measured boot

---

##  Quick Start

### 1 Setup (Windows PowerShell)

```powershell
# Create and activate virtual environment
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# Install dependencies
pip install -r requirements.txt
```

### 2 Run the Simulator

```powershell
python gui_app.py
```

This launches the GUI which automatically:
- Creates `config.ini` with default settings
- Creates working folders (`updates/`, `malicious_updates/`, `shared_for_ecu/`, `tcu_acks/`, `tcu_downloads/`)
- Spawns OEM Server, Malicious Server, TCU Client, and ECU Receiver as subprocesses

### 3 Use the GUI

| Button | Action |
|--------|--------|
| **Start Simulation** | Launch all servers and clients |
| **Deploy OEM Update** | Generate a new legitimate firmware version |
| **Deploy Malicious Update** | Load a custom file as malicious payload |
| **Request update** | Manually trigger TCU to check all servers |
| **Checksum Verification: ON/OFF** | Toggle security verification |
| **Clear All Logs** | Reset log panels |

---

##  Workflow Example

1. Click **Start Simulation**
2. Click **Deploy OEM Update**  creates `firmware_v1.1.bin`
3. Click **Request update**  TCU checks OEM server, finds v1.1, downloads, verifies, applies 
4. Click **Deploy Malicious Update**  select a `.txt` file  becomes `malicious_firmware_v1.2.bin`
5. Click **Request update**  TCU sees v1.2 > v1.1, chooses malicious, downloads...
   - **If checksum ON**: Detects fake checksum, rejects malicious payload 
   - **If checksum OFF**: Accepts malicious payload despite warning 

---

##  Running Components Manually

For debugging or custom testing:

```powershell
# Terminal 1: OEM Server
python oem_server.py

# Terminal 2: Malicious Server
python malicious_server.py

# Terminal 3: TCU Client (waits for keyboard input)
python tcu_client.py
# Press Enter and type: CHECK

# Terminal 4: ECU Receiver
python ecu_receiver.py
```

---

##  Configuration

`config.ini` (auto-generated by GUI):

```ini
[TCU]
current_version = 1.0
poll_interval_seconds = 10

[Server]
oem_url = http://127.0.0.1:5000
malicious_url = http://127.0.0.1:5001

[Security]
checksum_verification_enabled = true

[Folders]
ecu_shared_folder = shared_for_ecu
tcu_download_folder = tcu_downloads
tcu_ack_folder = tcu_acks
```

**Customize as needed** (e.g., change ports, disable checksums, etc.)

---

##  Runtime Folders

| Folder | Purpose |
|--------|---------|
| `updates/` | OEM server''s legitimate firmware |
| `malicious_updates/` | Malicious server''s payloads |
| `shared_for_ecu/` | Drop zone for firmware (ECU picks up from here) |
| `tcu_downloads/` | Temporary download staging area |
| `tcu_acks/` | ACK files from ECU back to TCU |

---

##  Known Limitations & Future Improvements

### Current Limitations
-  No real cryptographic signatures � only SHA256 hashes
-  No TLS/HTTPS � all traffic unencrypted
-  No persistent update history � logs cleared on restart
-  TCU logic is simplistic (always picks highest version)

### Recommended Enhancements
-  **Digital Signatures**: Implement RSA-PSS or ECDSA signing/verification
-  **TLS Support**: Serve Flask apps over HTTPS with certificate pinning
-  **Logging Database**: Store all transactions for audit trails
-  **Unit Tests**: Add pytest coverage for all modules
-  **CI/CD Pipeline**: GitHub Actions for automated testing
-  **Web UI**: Replace Tkinter with Flask/React dashboard
-  **CLI Options**: Add argparse for headless simulation runs

---

##  Educational Use Cases

- **Security Training**: Demonstrate why version-only comparison is insecure
- **Automotive Engineering**: Learn OTA architecture for connected vehicles
- **Penetration Testing**: Practice identifying and exploiting firmware update vulnerabilities
- **Compliance**: Show SOTAEVER, ASIL-D, and automotive security best practices
- **Demo Projects**: Use as a foundation for more advanced OTA simulators

---

##  License & Attribution

This repository is provided for **educational and demonstration purposes only**.

**No formal license is included.** Consult the project owner ([Thanatos624](https://github.com/Thanatos624)) ([shoob-cyber](https://github.com/shoob-cyber)) for licensing inquiries.

---

##  Support & Contributing

- **Issues/Bugs**: Open a GitHub issue with reproduction steps
- **Questions**: See the code comments or reach out to the project owner
- **Contributions**: Fork, improve, and submit a pull request!

---

**Happy learning and stay secure! **
