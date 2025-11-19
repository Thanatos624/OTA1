**OTA Update Simulator**

- **Description**: Simulates an Over-The-Air (OTA) update flow for an embedded Vehicle stack (TCU -> OEM Server / Malicious Server -> ECU). This repository provides a GUI-driven simulator demonstrating how a vulnerable TCU can be tricked by a malicious update source and why integrity checks (checksums) are important.

**Key Components**
- **`gui_app.py`**: Tkinter GUI that orchestrates the simulation, launches the other scripts as subprocesses, shows logs/status, and allows manual deployment of OEM or malicious updates.
- **`oem_server.py`**: A simple Flask server that serves legitimate update files from the `updates/` folder and reports SHA256 checksums.
- **`malicious_server.py`**: A Flask server that serves files from `malicious_updates/` and returns a fake checksum (used to demonstrate a malicious source).
- **`tcu_client.py`**: Simulated TCU (Telematics Control Unit) that checks both OEM and Malicious servers for updates, downloads a chosen update, verifies checksum (optional), and transfers firmware to the ECU.
- **`ecu_receiver.py`**: Simulated ECU that "listens" for firmware files placed in a shared folder and applies them, then writes an acknowledgement file back to the TCU ack folder.
- **`shared_utils.py`**: Utility helpers (version parsing, SHA256 calculation, etc.) used across components.
- **`requirements.txt`**: Minimal Python dependencies: `Flask` and `requests`.

**Design / Flow**
- The GUI launches four processes: OEM server, Malicious server, TCU client and ECU receiver.
- The TCU queries `/check-update` on both servers. Each server returns `{version, filename, checksum, source}`.
- TCU chooses the newest version among available servers, downloads it from the corresponding `/download/<filename>` endpoint, optionally verifies SHA256 checksum (based on `config.ini`), and then places the firmware file in the ECU shared folder.
- The ECU picks up the firmware, simulates applying it, removes the firmware file and writes an acknowledgement (e.g. `firmware_vX.Y.bin.ack`) to the TCU ack folder.

**Security Note**
- This project is intentionally a teaching/demo tool. The `malicious_server.py` demonstrates how an attacker can serve a malicious payload and (in this demo) supply a fake checksum. Use this simulator to study mitigation strategies such as strict checksum/signature verification, TLS, and server authentication.

**Setup (Windows PowerShell)**
- Create a virtual environment and install dependencies:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

- The GUI will create a `config.ini` automatically (if missing) with sane defaults and create the folders used by the simulation.

**Quick Start (recommended)**
- Start the GUI which runs everything for you:

```powershell
python gui_app.py
```

- In the GUI: click **Start Simulation**. The GUI will create the folders:
  - `updates/` (OEM updates)
  - `malicious_updates/` (malicious updates)
  - `shared_for_ecu/` (files moved to ECU)
  - `tcu_downloads/` (temporary downloads)
  - `tcu_acks/` (acknowledgements from ECU)

- Use **Deploy OEM Update** to add a legitimate firmware, or **Deploy Malicious Update** to add a user-chosen malicious file.
- Use **Request update** in the TCU panel or click the GUI button to trigger a manual check.

**Running components manually (optional)**
- You can run the servers/clients manually in separate shells if you prefer to inspect logs directly.

OEM server (port 5000):
```powershell
python oem_server.py
```

Malicious server (port 5001):
```powershell
python malicious_server.py
```

TCU client (manual trigger):
```powershell
python tcu_client.py
# then type `CHECK` and press Enter to trigger a single check
```

ECU receiver:
```powershell
python ecu_receiver.py
```

**Configuration**
- `config.ini` is created by the GUI with these default sections/keys (adjust as needed):
  - `TCU.current_version` (e.g. `1.0`)
  - `Server.oem_url` (default `http://127.0.0.1:5000`)
  - `Server.malicious_url` (default `http://127.0.0.1:5001`)
  - `Security.checksum_verification_enabled` (`true`/`false`)
  - `Folders.ecu_shared_folder`, `Folders.tcu_download_folder`, `Folders.tcu_ack_folder`

**Folders created at runtime**
- `updates/`, `malicious_updates/`, `shared_for_ecu/`, `tcu_downloads/`, `tcu_acks/`

**Known Limitations & Next Steps**
- Checksum verification uses SHA256 from `shared_utils.calculate_sha256`; however, the malicious server returns a fake checksum — use this to study real protections.
- Improvements you may add:
  - Implement digital signature verification (recommended over plain checksums).
  - Serve the Flask apps over HTTPS and add server authentication.
  - Add unit tests and CI.
  - Add CLI options and better logging configuration.

**License & Attribution**
- This repository is provided for educational/demo use. No license file is included; consult the project owner for licensing.

**Contact / Author**
- Project: OTA Update Simulator
  

**Screenshots**

Below are screenshots from the simulator. If you upload the matching image files to `docs/images/` (see filenames below), these will render in the README. I can add the files into the repository for you if you place the images in the workspace or re-attach them here.

![OEM Server (log)](docs/images/screenshot-oem.png)

**Figure 1**: OEM Server log output showing deployed firmware and serving activity.

![Malicious Server (log)](docs/images/screenshot-malicious.png)

**Figure 2**: Malicious Server log output showing a malicious payload being served.

![ECU Receiver (log)](docs/images/screenshot-ecu.png)

**Figure 3**: ECU Receiver output showing applied updates and acknowledgements.

![GUI Overview](docs/images/screenshot-gui-overview.png)

**Figure 4**: Full GUI layout (OEM, Malicious, TCU, ECU panels).

![TCU Log (demo)](docs/images/screenshot-tcu.png)

**Figure 5**: TCU log showing a manual check, checksum mismatch, and (if disabled) acceptance of a malicious payload.

Instructions to add the actual images:

- Place the image files in the repository at `docs/images/` with the filenames above (or edit the paths in this README to match your filenames).
- I recommend these filenames: `screenshot-gui-overview.png`, `screenshot-oem.png`, `screenshot-malicious.png`, `screenshot-ecu.png`, `screenshot-tcu.png`.
- After you add the files, open this README in GitHub or a Markdown viewer — the images will show inline.

If you'd like, I can add the images into the repo for you now — attach the image files here or put them into the workspace under `docs/images/` and I'll commit them.
