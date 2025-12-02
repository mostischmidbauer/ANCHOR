# README: Anchor

```
File: README-anchor.md
```

## Project: Anchor

**Purpose**

Anchor is the lightweight identity and integrity anchoring component for the Encounter Liveness project. Its job is to create verifiable tamper-evidence for encounter logs (JSON) produced by the client tool. Anchor produces a concise cryptographic receipt (hash + timestamp + optional signature) that can be stored locally or pushed to a verifiable backend.

**Key responsibilities**

* Compute a canonical SHA-256 hash of exported encounter JSON blobs.
* Optionally sign the hash with a private key (Ed25519 recommended) to produce a compact proof.
* Produce a small receipt object containing: `hash`, `timestamp`, `algorithm`, `signature` (optional), and `anchorId`.
* Provide a minimal CLI and REST endpoint for integration.

---

### Quick features

* Local-only mode: compute receipts on-device without network calls.
* Server mode: accept JSON via HTTP POST and return anchored receipts (for organizations that want centralized anchoring).
* Deterministic canonicalization step to avoid differences from spacing/order.

---

## Example receipt (JSON)

```json
{
  "anchorId": "anchor_2025-0001",
  "timestamp": "2025-12-02T08:12:34.567Z",
  "algorithm": "SHA-256",
  "hash": "3a7bd3c...",
  "signature": "MEUCIQ..."  
}
```

---

## Installation (CLI mode)

This component can be provided as a tiny Node.js utility or a Go single-binary for portability. Example (Node.js):

```bash
# clone repository
git clone <repo-url>
cd encounter-liveness/anchor
npm install
# create a local keypair (ed25519) if you want signing
node scripts/generate-keypair.js --out keys/anchor-key.json
```

## Usage (local CLI)

```bash
# compute receipt from a JSON file
node bin/anchor.js --input ../encounter_liveness_hr_logs.json --out receipt.json

# compute and sign
node bin/anchor.js --input ../encounter_liveness_hr_logs.json --sign keys/anchor-key.json --out receipt.json
```

## Usage (server mode)

Start the service (example Node.js express):

```bash
NODE_ENV=production node server.js --port 4001 --sign-key keys/anchor-key.json
```

POST your JSON to `/anchor` with `Content-Type: application/json`.

Example cURL:

```bash
curl -X POST https://anchor.your-domain.com/anchor \
  -H "Content-Type: application/json" \
  -d '@encounter_liveness_hr_logs.json'
```

Response: 201 with receipt JSON (see example above).

---

## API (minimal)

**POST /anchor**

* Request: application/json (the encounter log or array of logs)
* Response: 201 Created, JSON receipt

**GET /anchor/:anchorId**

* Retrieve a stored receipt for verification/audit

---

## Security

* If using signing, keep signing keys in a hardened HSM or secure vault.
* Use TLS for server mode.
* Store only hashes if privacy concerns exist; do not store full encounter JSON unless required and consented.

---

## Integration notes

* The client (Encounter HR web app) should produce the JSON export and then either:

  * call Anchor CLI locally to produce a receipt (local workflow), or
  * POST the JSON to the Anchor server endpoint (organization-managed service).

* The receipt is small and safe to include in audit logs or to store alongside the exported JSON.

---

## Contact

Maintainer: Mr. Moustafa Mohammed Elsayed Elsayed Ali

* Email: [mostischmidbauer@web.de](mailto:mostischmidbauer@web.de)
* Tel: +49 1522 5321568

---

# README: Verifier

```
File: README-verifier.md
```

## Project: Verifier

**Purpose**

Verifier receives an encounter log and an anchor receipt and evaluates the integrity and authenticity of the encounter. It verifies the hash and, if present, the signature. Verifier is intended for auditors, payors, and backend systems that need to automatically check whether an exported encounter was anchored and retained integrity.

**Key responsibilities**

* Re-compute canonical hash of the provided encounter log and compare to `receipt.hash`.
* Verify signature (if `receipt.signature` exists) using the signing public key.
* Check timestamp freshness and anchor provenance.
* Return a clear verification result with `status`, `details`, and optionally `verificationScore`.

---

## Example verification result

```json
{
  "anchorId": "anchor_2025-0001",
  "verified": true,
  "details": "Hash match and signature valid",
  "timestampChecked": true,
  "timestamp": "2025-12-02T08:12:34.567Z",
  "verificationScore": 0.95
}
```

---

## Installation

Simple Node.js script or a server component. Example (Node.js):

```bash
git clone <repo-url>
cd encounter-liveness/verifier
npm install
```

## Usage

### CLI verification

```bash
node bin/verify.js --log encounter.json --receipt receipt.json --pubkey keys/anchor-pub.pem
```

Output: JSON verification result printed to stdout and exit code 0 on success, non-zero on failure.

### HTTP API (recommended for backends)

Start the verifier server:

```bash
node server.js --port 4002 --anchor-key-store keys/
```

POST `/verify` with JSON payload:

```json
{
  "log": { ... },
  "receipt": { ... }
}
```

Response: 200 with verification result JSON (see example above).

Example cURL:

```bash
curl -X POST https://verifier.your-domain.com/verify \
  -H "Content-Type: application/json" \
  -d '{"log": ... , "receipt": ... }'
```

---

## API (minimal)

**POST /verify**

* Request JSON: `{ log: <object>, receipt: <object>, pubkey?: <string> }`
* Response JSON: verification result

---

## Trust model & recommendations

* The verifier must have access to the public keys (or a PKI) used by the Anchor service.
* Use TLS, authenticate the verifier service inside your infrastructure.
* Keep a tamper-evident audit trail of verification requests and results (store receipts or references only).

---

## Contact

Maintainer: Mr. Moustafa Mohammed Elsayed Elsayed Ali

* Email: [mostischmidbauer@web.de](mailto:mostischmidbauer@web.de)
* Tel: +49 1522 5321568

---

# README: Encounter HR (client)

```
File: README-encounter-hr.md
```

## Project: Encounter HR (web client)

**Purpose**

The Encounter HR component is the browser-based client that runs the liveness challenge and computes an optical heart-rate estimate (rPPG) from the camera feed. It also captures professional metadata and exports JSON logs for anchoring and backend ingestion.

**Key responsibilities**

* Guide a user through a short liveness sequence (challenge–response).
* Capture a camera ROI (face-following where available) and sample the green channel to estimate pulse.
* Provide signal-quality feedback (green/red dot) and debug information for developers.
* Provide fields for: `encounterId`, `professionalName`, `professionalLicense`, `issuingAuthority`.
* Save logs to `localStorage` and export logs as JSON.

---

## Quick features

* Local-only processing: camera frames are not uploaded anywhere by the client.
* FaceDetector API support with fallback ROI.
* Single-file `index.html` (drop-in) for fast deployment (GitHub Pages / static hosting).
* Export JSON matching the `EncounterLog` schema (see API spec).

---

## EncounterLog JSON schema (summary)

```json
{
  "timestamp": "2025-12-02T08:30:12.345Z",
  "encounterId": "ENC-2025-0001",
  "professionalName": "...",
  "professionalLicense": "...",
  "issuingAuthority": "...",
  "livenessCompleted": true,
  "livenessDurationSec": 12.5,
  "hrBpm": 76.2,
  "hrQuality": 0.18,
  "hrDurationSec": 20.1
}
```

This schema matches the OpenAPI `EncounterLog` schema in `docs/api-log-format.yaml`.

---

## How to run (local / GitHub Pages)

### GitHub Pages (HTTPS)

1. Add `index.html` to the repo root.
2. Enable GitHub Pages (branch: `main`, folder: `/root`).
3. Open the page via the GitHub Pages HTTPS URL.

### Local server (if `file://` camera restrictions apply)

```bash
# Python local server (simple)
python3 -m http.server 8000
# open http://localhost:8000/index.html in Chrome/Edge
```

---

## Integration with Anchor & Verifier

1. After an encounter, click **Export JSON** in the client.
2. Option A (local-only anchor): run Anchor CLI against the downloaded JSON to generate receipt.
3. Option B (centralized ingestion): POST the JSON to your backend `/encounters/logs` endpoint (see OpenAPI). Backend should store the JSON and call Anchor service (or perform anchoring itself).
4. Verifier can be used to re-check stored logs by comparing receipts to re-computed hashes.

### Example backend ingestion (curl)

```bash
curl -X POST https://api.your-domain.com/encounters/logs \
  -H 'Authorization: Bearer <JWT>' \
  -H 'Content-Type: application/json' \
  -d @encounter_liveness_hr_logs.json
```

The backend should validate the JSON against the `EncounterLog` schema and optionally call Anchor to produce a receipt and then return stored metadata.

---

## Testing & debug tips

* Use even, front-facing lighting for better HR samples.
* Keep head still during HR sampling (8–25s window recommended).
* If FaceDetector isn't available in the browser, ensure the central ROI contains the forehead area.
* Check the debug panel for `Samples`, `Duration`, `Raw BPM` and `Quality` values.

---

## Security & privacy

* All client-side processing is local; exports are explicit actions by the user.
* If you integrate with a backend, enforce TLS, authentication, RBAC, and data-retention policies.
* Follow GDPR / HIPAA guidelines when storing or transmitting encounter logs.

---

## Contact

Maintainer: Mr. Moustafa Mohammed Elsayed Elsayed Ali

* Email: [mostischmidbauer@web.de](mailto:mostischmidbauer@web.de)
* Tel: +49 1522 5321568

---

*End of READMEs.*
