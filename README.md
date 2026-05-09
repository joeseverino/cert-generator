# cert-generator

Issues TLS certificates signed by a private root CA running on an offline VM. Connects via SSH, generates the key and CSR on the CA host, signs the cert, retrieves the output, and cleans up. The private key never leaves the CA VM during the process.

Stack: Bash + OpenSSL + SSH

---

## Overview

Designed for homelabs using a private root CA on an air-gapped or offline VM. The script handles the full issuance workflow in a single command:

1. SSH into the CA VM
2. Generate a 4096-bit RSA key and CSR
3. Sign the cert against the CA key (passphrase prompted interactively)
4. Build a fullchain PEM and PKCS#12 bundle
5. Retrieve all output files to a local directory
6. Remove all temporary files from the CA host

The CA key is only ever used on the CA host and is never transferred. The signed service key is chmod 600 on retrieval.

---

## Prerequisites

- A private root CA with key and cert accessible on a remote host via SSH
- SSH access to the CA host configured in `~/.ssh/config` (key auth recommended)
- `openssl` available on the CA host
- `openssl` and `scp` available locally

---

## Setup

    cp config.env.example config.env

Edit `config.env`:

    ORG="Your Org"
    CA_HOST="offline-ca"          # SSH hostname or alias for the CA VM
    CA_CERT="~/ca.pem"            # Path to CA cert on the remote host
    CA_KEY="~/ca.key"             # Path to CA key on the remote host
    LOCAL_OUTPUT="${HOME}/certs"  # Local output directory

---

## Usage

    chmod +x cert-gen
    ./cert-gen <hostname> [dns:<name>|ip:<addr> ...]

Examples:

    ./cert-gen myserver
    ./cert-gen myserver dns:myserver.local
    ./cert-gen myserver dns:myserver.local ip:192.168.1.10

If no hostname is given, the script prompts for one.

Add `--debug` for verbose SSH output and line-level error tracing:

    ./cert-gen --debug myserver

---

## Output

Each run creates a subdirectory under `LOCAL_OUTPUT/<hostname>/`:

    hostname.crt      — signed certificate
    hostname.key      — private key (chmod 600)
    hostname.csr      — certificate signing request
    hostname.pfx      — PKCS#12 bundle (no password)
    fullchain.pem     — cert + CA cert concatenated

---

## Certificate Details

- RSA 4096-bit key
- 825-day validity (macOS and browser maximum for private certs)
- SAN extension included for all specified DNS names and IPs
- Signed with `-CAcreateserial` — CA serial file managed automatically on the CA host

---

## Notes

- The `config.env` file is gitignored. Never commit it.
- `config.env.example` is the template for new setups.
- The CA passphrase is prompted interactively via `ssh -t` — it is never stored or passed as an argument.
- The PKCS#12 bundle is exported with no password for easier import into system keychains. Adjust the `-passout` flag in the script if you want one.
