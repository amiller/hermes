# Hermes TEE Verification Guide

This guide explains how to verify that the Hermes deployment is running the expected code inside a Trusted Execution Environment (TEE).

## Quick Verification

Visit the [Phala Trust Center](https://trust.phala.com/app/db82f581256a3c9244c4d7129a67336990d08cdf/db82f581256a3c9244c4d7129a67336990d08cdf) to see automated verification results.

## What's Being Verified

Hermes runs on Phala Cloud using Intel TDX (Trust Domain Extensions). The TEE provides:

1. **Hardware attestation** — cryptographic proof the code runs on genuine Intel TDX hardware
2. **Measurement registers** — tamper-evident hashes of the OS and application
3. **Memory isolation** — the operator cannot read TEE memory (keys, pending entries)

## Verification Layers

### Layer 1: Hardware (Intel TDX Quote)

The TEE generates a cryptographic quote signed by Intel hardware. This proves:
- Genuine Intel TDX processor
- Specific measurement values (MRTD, RTMR0-3)

**Verify:** The Trust Center validates the quote against Intel's attestation service.

### Layer 2: Operating System (MRTD, RTMR0-2)

The OS measurements prove which dstack OS image is running.

| Register | What it measures |
|----------|------------------|
| MRTD | Initial TD memory (kernel, initramfs) |
| RTMR0 | Boot stage 1 |
| RTMR1 | Boot stage 2 |
| RTMR2 | Boot stage 3 |

**Verify:** Compare against expected values for the dstack version. The Trust Center does this automatically using [dstack-mr](https://github.com/kvinwang/dstack-mr).

### Layer 3: Application (RTMR3 / compose_hash)

RTMR3 contains an event log including the `compose-hash` event — a SHA-256 hash of the full application configuration.

**This is the layer you can verify yourself.**

## Manual Verification of Application Code

### Step 1: Get the running configuration

```bash
# Fetch the app info from the TEE
curl -s "https://db82f581256a3c9244c4d7129a67336990d08cdf-8090.dstack-pha-prod9.phala.network/prpc/Info" > info.json

# Extract the compose_hash from attestation
jq -r '.compose_hash' info.json
# Expected: 3ed09804a42acb4cfcc48d2603f91081b863b16047a742b8344e76c35323b0ed

# Verify it matches the event log
jq -r '.tcb_info' info.json | jq '.event_log[] | select(.event == "compose-hash") | .event_payload'
```

### Step 2: Inspect the application configuration

```bash
# Extract the full app_compose JSON
jq -r '.tcb_info' info.json | jq -r '.app_compose' > app_compose.json

# View the docker-compose file (your code)
jq -r '.docker_compose_file' app_compose.json

# View allowed environment variables (names only, not values)
jq -r '.allowed_envs' app_compose.json
```

### Step 3: Verify the hash matches

```bash
# Compute SHA-256 of app_compose (must use exact string, no trailing newline)
node -e "
const fs = require('fs');
const crypto = require('crypto');
const info = JSON.parse(fs.readFileSync('info.json'));
const tcbInfo = JSON.parse(info.tcb_info);
const hash = crypto.createHash('sha256').update(tcbInfo.app_compose).digest('hex');
console.log('Computed hash:', hash);
console.log('Matches attestation:', hash === info.compose_hash);
"
```

### Step 4: Compare docker-compose to source

The `docker_compose_file` should match this repo's `docker-compose.template.yml` with the git SHA substituted:

```bash
# Get the deployed git SHA from the image tag
jq -r '.docker_compose_file' app_compose.json | grep 'image:'
# Output: image: docker.io/generalsemantics/hermes:867bd06

# Generate expected compose from source
GIT_SHA=867bd06
sed "s/\${GIT_SHA}/$GIT_SHA/g" docker-compose.template.yml

# Compare (should match the docker_compose_file content)
```

## What You're Trusting

### Fully Verifiable (from this repo)
- **Docker image contents** — built from this repo via GitHub Actions
- **docker-compose.yml structure** — the `docker_compose_file` in app_compose
- **Environment variable names** — listed in `allowed_envs`

### Provided by Phala Cloud (inspectable but not versioned in a public repo)
- **pre_launch_script** — Phala's bootstrap script (currently v0.0.12), handles Docker login, cleanup, password setup, SSH keys. Visible in Trust Center under "App Source Code" → "Compose File", or via `/prpc/Info` endpoint.
- **App configuration flags** — `gateway_enabled`, `kms_enabled`, `features`, etc. Set by Phala CLI based on deployment options.
- **dstack OS image** — the base TEE operating system, verifiable via [dstack releases](https://github.com/Dstack-TEE/dstack/releases)

**Inspecting Phala's boilerplate:**
```bash
# View the full app_compose including pre_launch_script
curl -s "https://db82f581256a3c9244c4d7129a67336990d08cdf-8090.dstack-pha-prod9.phala.network/prpc/Info" | \
  jq -r '.tcb_info' | jq -r '.app_compose' | jq -r '.pre_launch_script'
```

Or view it in the [Trust Center](https://trust.phala.com/app/db82f581256a3c9244c4d7129a67336990d08cdf/db82f581256a3c9244c4d7129a67336990d08cdf?selected=app-code) under "Compose File".

**Reference for pre_launch_script:**
- Public reference: [dstack-examples/phala-cloud-prelaunch-script](https://github.com/Dstack-TEE/dstack-examples/tree/main/phala-cloud-prelaunch-script) (v0.0.11)
- Hermes runs v0.0.12 (minor diff: improved password file handling with `chpasswd`)
- Script handles: Docker/ECR auth, root password, SSH keys, Docker cleanup

**Note:** You can diff the deployed script against the public reference to verify only expected changes exist.

### Secret but Verified Present
- **Environment variable values** — encrypted via `--env-file`, names visible in `allowed_envs`
- The TEE has: `FIREBASE_SERVICE_ACCOUNT_BASE64`, `BASE_URL`, `ANTHROPIC_API_KEY`, `SENDGRID_API_KEY`
- Values are encrypted by Phala CLI and only decrypted inside the TEE

## Understanding the Trust Model

```
┌─────────────────────────────────────────────────────────────────┐
│                        What You Verify                          │
├─────────────────────────────────────────────────────────────────┤
│  1. Intel TDX quote is valid (hardware attestation)             │
│  2. OS measurements match expected dstack version               │
│  3. compose_hash in RTMR3 matches SHA-256(app_compose)          │
│  4. docker_compose_file matches your git-controlled template    │
│  5. Docker image digest matches GitHub Actions build            │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                     What This Proves                            │
├─────────────────────────────────────────────────────────────────┤
│  • The TEE is running YOUR docker-compose with YOUR image       │
│  • The operator cannot modify the code after deployment         │
│  • The operator cannot read TEE memory (keys, pending entries)  │
│  • Secret env values exist but you trust their source           │
└─────────────────────────────────────────────────────────────────┘
```

## Key Endpoints

| Endpoint | Purpose |
|----------|---------|
| `https://[app-id]-8090.[domain]/` | TEE info page (HTML) |
| `https://[app-id]-8090.[domain]/prpc/Info` | Full attestation data (JSON) |
| `https://cloud-api.phala.network/api/v1/apps/[app-id]/attestations` | Phala Cloud attestation API |

For Hermes:
- App ID: `db82f581256a3c9244c4d7129a67336990d08cdf`
- Domain: `dstack-pha-prod9.phala.network`

## Verifying Docker Image Provenance

The Docker image is built by GitHub Actions and pushed to Docker Hub.

1. Find the git SHA in the running compose: `867bd06`
2. Find the [GitHub Actions run](https://github.com/anthropics/hermes/actions) for that commit
3. Compare the image digest from the workflow summary with what's deployed

```bash
# Get deployed image info
docker pull generalsemantics/hermes:867bd06
docker inspect generalsemantics/hermes:867bd06 --format='{{.Id}}'
```

## Partial Verification: What You CAN Check

Even though you can't fully reproduce `compose_hash`, you can verify the parts that matter:

```bash
# 1. Extract the docker_compose_file from the running TEE
curl -s "https://db82f581256a3c9244c4d7129a67336990d08cdf-8090.dstack-pha-prod9.phala.network/prpc/Info" | \
  jq -r '.tcb_info' | jq -r '.app_compose' | jq -r '.docker_compose_file' > deployed_compose.yml

# 2. Get the git SHA from the image tag
grep 'image:' deployed_compose.yml
# docker.io/generalsemantics/hermes:867bd06

# 3. Generate expected compose from this repo
sed 's/${GIT_SHA}/867bd06/g' docker-compose.template.yml > expected_compose.yml

# 4. Compare (should be identical)
diff deployed_compose.yml expected_compose.yml

# 5. Verify the allowed_envs match what you expect
curl -s "https://db82f581256a3c9244c4d7129a67336990d08cdf-8090.dstack-pha-prod9.phala.network/prpc/Info" | \
  jq -r '.tcb_info' | jq -r '.app_compose' | jq '.allowed_envs'
# Should show: ["FIREBASE_SERVICE_ACCOUNT_BASE64","BASE_URL","ANTHROPIC_API_KEY","SENDGRID_API_KEY"]
```

If the `docker_compose_file` matches and `allowed_envs` lists only expected variables, you've verified that:
- The correct Docker image is running
- No unexpected environment variables are injected
- The deployment matches the git commit

The remaining `app_compose` fields (`pre_launch_script`, `features`, etc.) are Phala Cloud boilerplate that you must trust or inspect manually.

## Improving Auditability

To make future deployments fully reproducible, Phala Cloud could:
1. Publish the `pre_launch_script` versions to a public repo
2. Allow users to provide their own pre_launch_script
3. Include a hash of the pre_launch_script in attestation separately from compose_hash

Until then, the practical verification path is: verify `docker_compose_file` and `allowed_envs` match your source, and trust Phala's standard boilerplate.

## References

- [Phala Trust Center](https://trust.phala.com/) — automated verification UI
- [dstack-verifier source](./trust-center/packages/verifier) — verification library (included in this repo)
- [Intel TDX](https://www.intel.com/content/www/us/en/developer/tools/trust-domain-extensions/overview.html) — hardware TEE technology
- [dstack](https://github.com/Dstack-TEE/dstack) — TEE orchestration platform
