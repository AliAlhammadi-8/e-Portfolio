# Digital Forensics Agent System (DFAS)

A modular, agent‑based pipeline for digital forensics: discover artefacts by signature, hash with SHA‑256, normalise metadata, package with AES‑256, and transfer securely—while preserving an auditable chain of custody.

---

## 1) Project Summary
DFAS automates common forensic workflows across mixed corpora (images, documents, mailboxes, disk images). Four specialised agents coordinate via a **blackboard** repository:

- **Search Agent** – signature‑first identification using magic numbers.
- **Processing Agent** – SHA‑256 hashing, timestamp/metadata normalisation (PDF/EXIF).
- **Archiving Agent** – AES‑256 ZIP packaging with per‑archive **manifest**.
- **Communication Agent** – resumable SFTP or TLS 1.3 upload with checksum verification.

**Target runtime:** Python 3.8+ (or Docker). **Reproducibility:** containerised builds and pinned dependencies.

---

## 2) Key Features
- Signature‑based discovery (avoids spoofed extensions).
- SHA‑256 hashing with built‑in **NIST** test‑vector check.
- PDF and EXIF/GPS metadata extraction.
- AES‑256‑encrypted ZIP archives + manifest (path, size, times, digest).
- SFTP (SSHv2) and TLS 1.3 transfer; resume + post‑transfer checksum verify.
- Append‑only audit log; **RBAC** guard rails for sensitive operations.

---

## 3) System Requirements
- **OS:** Windows 10/11, Ubuntu 22.04+, macOS 13+
- **Python:** 3.8+ (or **Docker** 24+)
- **Tools:** Git, Docker, `make` (optional)
- **Optional:** The Sleuth Kit/Autopsy (image parsing); **ExifTool** installed on PATH

---

## 4) Quick Start (Docker)
```bash
git clone <REPO_URL> dfas && cd dfas
docker build -t dfas:latest .
# case_data is your working volume with logs/manifests/archives
docker run --rm -it -v "$PWD/case_data:/case_data" dfas:latest --help
```
- The container prints its **image digest** on start for audit/reproducibility.

---

## 5) Quick Start (Python venv)
```bash
python -m venv .venv && source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
python -m dfas.cli --help
```

---

## 6) Directory Layout
```
dfas/
  agents/
    search.py
    processing.py
    archiving.py
    comms.py
  core/
    blackboard.py
    rbac.py
    logging_appendonly.py
  io/
    sigs.py          # magic numbers
    exif.py          # ExifTool bridge
    pdfmeta.py       # PyPDF2 wrapper
  tests/
  scripts/
case_data/           # created at runtime
  logs/              # append-only ledger
  manifests/
  archives/
  reports/
docs/
  figures/           # UML & activity diagrams (for slides)
```

---

## 7) Configuration
Main configuration is `config.yaml` (see example below). CLI flags and environment variables can override most options.

```yaml
# config.yaml (example)
case_root: "./case_data"
threads:
  io_workers: 8
search:
  roots: ["./samples", "/mnt/images"]
  exclude_globs: ["*/tmp/*", "*/.cache/*"]
processing:
  hash: "sha256"
  verify_sample_pct: 1
archiving:
  encrypt: true
  cipher: "AES256"
  out_dir: "./case_data/archives"
comms:
  mode: "sftp"        # or "tls"
  sftp:
    host: "sftp.example"
    port: 22
    user: "evidence_bot"
    key_path: "~/.ssh/id_ed25519"
  tls:
    url: "https://portal.example/upload"
    client_cert: "./certs/client.pem"
rbac:
  require_supervisor_token_for_raw_paths: true
```

**Environment overrides (examples):**
- `DFAS_THREADS=12` (sets `threads.io_workers`)
- `DFAS_CASE_ROOT=/evidence/CASE-2025-10-001`
- `DFAS_COMMS_MODE=tls`
- `DFAS_SFTP_HOST=sftp.internal`

---

## 8) Typical Workflows

### 8.1 Initialise case & scan
```bash
python -m dfas.cli init --case "CASE-2025-10-001"
python -m dfas.cli scan  --roots "./samples" --sig-only
```

### 8.2 Hash & metadata
```bash
python -m dfas.cli process --batch-size 500 --threads 8
```

### 8.3 Package & protect
```bash
python -m dfas.cli archive --encrypt --cipher AES256 --out ./case_data/archives
```

### 8.4 Transfer with resume
```bash
python -m dfas.cli send --mode sftp --target sftp://evidence_bot@sftp.example/CASE-2025-10-001
```

### 8.5 Audit trail & reports
```bash
python -m dfas.cli report --manifest latest --open
python -m dfas.cli log --filter sha256:<digest>
```

---

## 9) Demo Script (20 minutes)

1. **Show container hash** and `config.yaml` (reproducibility & settings).
2. **Scan** mixed corpus; highlight signature/extension mismatches.
3. **Hash correctness** via NIST vector:
   ```bash
   echo -n "abc" | sha256sum
   # expect ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad
   ```
4. **Metadata** extraction (EXIF GPS; PDF properties).
5. **Archive** and open **manifest** (path, size, times, SHA-256).
6. **Send** to SFTP; simulate drop, show **resume** and checksum verify.
7. **Audit**: query append‑only log for a single artefact.

---

## 10) Testing & Quality

### 10.1 Unit tests
```bash
pytest -q
```
Covers hashing (NIST vectors), signature detection, and metadata parsers.

### 10.2 Property‑based tests
Randomised path encodings, long names, odd bytes; ensures parser robustness.

### 10.3 Integration tests
```bash
pytest -q tests/integration --maxfiles 10000 --threads 8
```
Runs over mixed corpora (PDF, JPEG, PST, DD); emits timing to `case_data/reports/perf.json`.

### 10.4 Lint & type checks
```bash
ruff check .
mypy dfas
```

---

## 11) Performance Benchmarks
**Scenario:** 100k small files (4–32 KB) on SSD.  
**Compare:** serial vs. 8 worker threads (I/O bound).  
**Metrics:** wall‑clock, CPU %, throughput (files/s).  
**Output:** `reports/bench_<timestamp>.md` with charts.

---

## 12) Security, Compliance, and Chain of Custody
- Enforce **read‑only** acquisition; deny write handles to sources.
- Compute **SHA‑256** for all artefacts; optional second‑hash sample (1%).
- Package as **AES‑256** ZIP; bind path/size/times/digest in **manifest**.
- Transfer via **TLS 1.3** or **SFTP**; verify checksums post‑transfer.
- **RBAC**: Comms agent cannot access raw paths without supervisor token.
- Aligns with GDPR Art. 32 (encryption, access control, auditability).
- **Warning:** Never run as Admin/Root against evidence sources; mount images read‑only.

---

## 13) Reproducibility
- Container prints **image digest** at start.  
- Export SBOM:
```bash
syft dfas:latest -o json > sbom.json
```
- Pinned `requirements.txt` (and `poetry.lock` if used).

---

## 14) Architecture Overview (Figures)
- **Figure 1 – UML Class Diagram:** agent classes + blackboard interfaces.
- **Figure 2 – Sequence Diagram:** init → scan → process → archive → send → confirm.
- **Figure 3 – Activity (Processing Agent):** happy path + error branches.
Diagrams live under `docs/figures/` for inclusion in slides.

---

## 15) Configuration Reference
**Exit codes (common):**
| Code | Meaning                           |
|-----:|-----------------------------------|
| 0    | Success                           |
| 2    | Config/arg error                  |
| 3    | Scan/IO error (partial results)   |
| 4    | Archive/encrypt error             |
| 5    | Transfer error                    |

**Common messages:**
- `RBAC: supervisor token required` – elevate with `--supervisor-token <file>`
- `ExifTool not found` – install and expose on PATH

---

## 16) Troubleshooting
- **Slow scans** → increase `threads.io_workers`; exclude temp/cache dirs.
- **ExifTool missing** → install and confirm `exiftool -ver` works.
- **SFTP flaps** → check keys, server logs, and MTU; enable `--resume`.
- **Hash mismatch on transfer** → re‑send; verify local archive; check disk/S.M.A.R.T.

---

## 17) FAQ
- **Why SHA‑256 (not MD5/SHA‑1)?** Stronger collision resistance; court acceptance.
- **Why AES‑256 ZIP?** Strong crypto with broad tool support; 7z pluggable if needed.
- **Post‑quantum crypto?** Swap Archiving/Comms adapters when policies require.
- **DB server needed?** No—SQLite is **serverless** by default.

---

## 18) Contribution Guide
- Branches: `main` (stable), `dev` (active).
- PR checklist: tests green, `ruff` + `mypy` clean, docs updated.
- Comments explain **why** the code exists; docstrings link to standards where relevant.

---

## 19) License
Add your project license here (e.g., MIT/Apache‑2.0).

---

## 20) References
- ENISA (2024), IETF (2018), NIST (2015), GDPR (2016/2018), IDC (2019; 2025), Carrier (2005; 2010), Casey (2011), Luttgens et al. (2014), Merkel (2014), SQLite (2018; 2025), Paramiko (2025), ExifTool (2025), Python Software Foundation (2020), Cloudflare (2018), Wooldridge (2009), Corkill (1991).
- Full bibliographic entries are in the **transcript** and can be mirrored here if required by assessment rules.
