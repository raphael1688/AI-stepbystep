# Getting started with SBOM for your AI applications
## Goals
By the end of this lab, you will:
- Run a target AI application in a container (example: Ollama) so there is something to inspect
- Generate a Software Bill of Materials (SBOM) using Trivy in CycloneDX JSON format
- Bring up Dependency-Track (OWASP) using its bundled container
- Create a project and API key in Dependency-Track
- Upload your SBOM via a Python script using the Dependency-Track API
- Review components and vulnerabilities in the Dependency-Track UI

Architecture note: Trivy generates a CycloneDX SBOM locally and Dependency-Track ingests it for analysis. The result is a project-centric view of components, licenses, and known CVEs for your AI container and its dependencies.

## Prerequisites
1. A Linux host or VM with:
   - Docker (20.10+ recommended)
   - Python 3.9+ and pip
   - curl
   - Optionally, NVIDIA drivers and NVIDIA Container Toolkit if you want GPU access in your container
2. Internet access to pull Docker images and install tools
3. Permissions to run Docker and open port 8080 (Dependency-Track UI)
4. Target AI application image to scan (example: Ollama)
5. If you are running in a shared or lab environment (for example, UTF), note that URIs or addresses will differ from a local desktop

## What you will build
- A local AI container (example target) to scan
- Trivy installed and verified
- Dependency-Track (bundled edition) running in Docker with persistent storage
- A Dependency-Track project and API key
- A CycloneDX SBOM JSON file generated from your AI container image
- A Python script to POST the SBOM to Dependency-Track

---
## Step 1: Prepare the target AI application container (example: Ollama)
We will run a containerized AI application to have a concrete target for SBOM generation. If you want GPU acceleration and have NVIDIA hardware, enable the NVIDIA runtime first.

![Step 1](images/sbom-step01.png)

Optional (GPU): set Docker's default runtime to NVIDIA and restart Docker:
```bash
# Enable NVIDIA runtime for Docker (Ubuntu example)
# Requires NVIDIA drivers and nvidia-container-toolkit installed.
sudo mkdir -p /etc/docker
cat <<'EOF' | sudo tee /etc/docker/daemon.json
{
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  },
  "default-runtime": "nvidia"
}
EOF

# Restart Docker to apply
sudo systemctl restart docker
```

Run the AI container (Ollama example; adjust image and ports as needed):
```bash
# Pull and run Ollama (CPU-only)
docker run -d --name ollama -p 11434:11434 ollama/ollama:latest

# If using GPU (requires NVIDIA runtime)
# docker run -d --gpus all --name ollama -p 11434:11434 ollama/ollama:latest
```

Verify the container is running:
```bash
docker ps

# Quick health check (Ollama exposes a JSON API)
curl -s http://localhost:11434/api/version
```

Optional: use Docker Compose instead of docker run:
```bash
mkdir -p ollama-lab && cd ollama-lab
cat > compose.yaml <<'EOF'
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434"
    # Uncomment for GPU
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - capabilities: [gpu]
EOF

docker compose up -d
```

Notes:
- If you use a different AI application, substitute its image name and health check.
- In a lab environment, localhost may be replaced with the host IP or a shared ingress.

---
## Step 2: Install Trivy
We will install Trivy, an open-source scanner that can generate SBOMs in CycloneDX format.

![Step 2](images/sbom-step02.png)

Option A (Snap, easiest on Ubuntu):
```bash
sudo snap install trivy
trivy --version
```

Option B (Official install script, works widely):
```bash
# Download and install the trivy binary to /usr/local/bin
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
| sudo sh -s -- -b /usr/local/bin

# Verify
trivy --version
```

Option C (APT repo, Ubuntu/Debian):
```bash
sudo apt-get update
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb stable main" | \
  sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy

trivy --version
```

---
## Step 3: Bring up Dependency-Track (bundled container)
We will run the bundled edition to simplify setup (no external database required). Data will be persisted in a Docker volume.

![Step 3](images/sbom-step03.png)

```bash
# Pull the bundled image
docker pull dependencytrack/bundled:latest

# Create a persistent volume
docker volume create dtrack-data

# Run Dependency-Track
docker run -d --name dtrack \
  -p 8080:8080 \
  -v dtrack-data:/data \
  dependencytrack/bundled:latest

# Confirm it is up
docker logs -f dtrack  # Press Ctrl+C when you see it started
```

Access the UI:
- URL: http://HOST_IP:8080 (on a local desktop, http://localhost:8080)
- Default credentials: admin / admin (you will be prompted to change the password)

---
## Step 4: Initial Dependency-Track configuration (admin and API key)
Log in, change the admin password, and generate an API key for uploads.

![Step 4](images/sbom-step04.png)

Steps in the UI:
1. Open http://HOST_IP:8080 and log in with admin / admin; set a new password.
2. Go to Administration -> Access Management -> Teams -> Administrators.
3. Click the plus icon to create a new API Key for the Administrators team.
4. Copy the API key and store it securely (you will only see it once).

You will also create a project for your AI application:
- Navigate to Projects -> New Project
- Name: Ollama Lab (or your preferred name)
- Classifier: Container
- Version: 1.0.1 (example)
- Team: Administrators (or another team you prefer)
- Mark Latest if appropriate
- Save the project

---
## Step 5: Generate a CycloneDX SBOM for your AI container image
We will use Trivy to produce a CycloneDX JSON SBOM for the container image you are running.

![Step 5](images/sbom-step05.png)

Identify the image name:
```bash
# See running containers (IMAGE column shows the image name)
docker ps

# Optional: list images directly
docker images
```

Generate SBOM (CycloneDX JSON) with vulnerabilities included:
```bash
# Example targeting the Ollama image; adjust the image tag as needed
# --scanners vuln includes vulnerability data
# --format cyclonedx-json produces CycloneDX JSON
trivy image --scanners vuln --format cyclonedx-json \
  --output ollama_lab.cdx.json \
  ollama/ollama:latest

# If your container is named "ollama" and you want to resolve the exact image:
# trivy image --scanners vuln --format cyclonedx-json \
#   --output ollama_lab.cdx.json \
#   $(docker inspect --format='{{.Config.Image}}' ollama)

# Inspect the SBOM file
ls -lh ollama_lab.cdx.json

# Optional: preview contents
head -n 50 ollama_lab.cdx.json
```

Notes:
- CycloneDX JSON is widely supported and easy to parse.
- You can scan other images by changing the target in the command above.

---
## Step 6: Upload the SBOM to Dependency-Track via API (Python script)
We will use a short Python script to POST the SBOM to Dependency-Track. It should return a 200 response and a token indicating the upload has been accepted for processing.

![Step 6](images/sbom-step06.png)

Create a working directory and the script:
```bash
mkdir -p sbom_upload
cd sbom_upload

# Create and edit the Python script
cat > dt_sbom_upload.py <<'EOF'
import base64
import json
import os
import sys
import requests

# Configuration (update these to match your environment)
DT_BASE_URL = os.environ.get("DT_BASE_URL", "http://localhost:8080")
DT_API_KEY = os.environ.get("DT_API_KEY", "REPLACE_WITH_YOUR_API_KEY")
PROJECT_NAME = os.environ.get("DT_PROJECT_NAME", "Ollama Lab")
PROJECT_VERSION = os.environ.get("DT_PROJECT_VERSION", "1.0.1")
BOM_PATH = os.environ.get("DT_BOM_PATH", "../ollama_lab.cdx.json")  # path to your SBOM file

def main():
    if not DT_API_KEY or DT_API_KEY == "REPLACE_WITH_YOUR_API_KEY":
        print("ERROR: DT_API_KEY is not set. Export DT_API_KEY or edit the script.")
        sys.exit(1)

    # Read and base64-encode the SBOM file
    try:
        with open(BOM_PATH, "rb") as f:
            bom_bytes = f.read()
    except FileNotFoundError:
        print(f"ERROR: SBOM file not found at {BOM_PATH}")
        sys.exit(1)

    bom_b64 = base64.b64encode(bom_bytes).decode("utf-8")

    # Prepare JSON payload for Dependency-Track
    payload = {
        "projectName": PROJECT_NAME,
        "projectVersion": PROJECT_VERSION,
        "autoCreate": True,   # create the project if it does not exist
        "bom": bom_b64
    }

    headers = {
        "X-Api-Key": DT_API_KEY,
        "Content-Type": "application/json"
    }

    url = f"{DT_BASE_URL}/api/v1/bom"
    try:
        resp = requests.post(url, headers=headers, json=payload, timeout=30)
    except requests.exceptions.RequestException as e:
        print(f"ERROR: Failed to upload SBOM: {e}")
        sys.exit(1)

    # Expect 200 OK with a token
    print(f"Status: {resp.status_code}")
    try:
        data = resp.json()
        print("Response JSON:", json.dumps(data, indent=2))
        if "token" in data:
            print(f"Upload token: {data['token']}")
    except ValueError:
        print("No JSON in response. Raw text:", resp.text)

if __name__ == "__main__":
    main()
EOF
```

Install dependencies and run:
```bash
python3 -m pip install --upgrade pip requests

# Export environment variables for convenience
export DT_BASE_URL="http://localhost:8080"     # or http://HOST_IP:8080
export DT_API_KEY="PASTE_YOUR_API_KEY_HERE"
export DT_PROJECT_NAME="Ollama Lab"
export DT_PROJECT_VERSION="1.0.1"
export DT_BOM_PATH="../ollama_lab.cdx.json"

# Execute the upload script
python3 dt_sbom_upload.py
```

Expected result:
- Status code: 200
- Response includes a token (used internally by Dependency-Track to process the BOM)

---
## Step 7: Review results in Dependency-Track
Reload your project page and confirm components are populated. Newly installed software often shows few or no vulnerabilities, but the components list should be present.

![Step 7](images/sbom-step07.png)

In the UI:
- Go to Projects -> select "Ollama Lab" (version 1.0.1)
- Check Overview, Components, Services, Dependency Graph, Audit, Vulnerabilities, Exploit Predictions
- Verify the score and component list reflect the uploaded SBOM
- Expect multiple pages of components (base image layers, OS packages, and application dependencies)

---
## Optional Enhancements
- Automate SBOM generation and upload in CI or CD (for example, GitHub Actions, GitLab CI)
- Scan multiple services and versions and organize them by team and projects
- Schedule periodic re-scans and uploads to catch newly disclosed CVEs
- Enable notifications or webhooks in Dependency-Track when risk changes
- Add license policy checks and governance workflows

---
## Troubleshooting
- Dependency-Track not accessible
  - Ensure the container is running: docker ps
  - Verify port 8080 is open and not blocked by a firewall
  - Check logs: docker logs dtrack
- Login or API key issues
  - Default admin/admin works only on first login, then you must change the password
  - API keys are shown once; if lost, generate a new key
- SBOM upload returns errors
  - Confirm DT_BASE_URL and DT_API_KEY are correct
  - Ensure BOM_PATH points to the file and the file exists
  - The JSON payload must contain a base64-encoded "bom"
- Trivy install problems
  - Try the install script if snap or apt is unavailable
  - Check trivy --version to ensure it is installed
- Image naming confusion
  - Use docker ps and docker images to confirm the exact image and tag
  - For a named container, you can resolve its image with:
    docker inspect --format='{{.Config.Image}}' <container_name>
- GPU runtime issues
  - Verify NVIDIA drivers and nvidia-container-toolkit are installed
  - Confirm Docker's daemon.json is set correctly and Docker was restarted

---
## Cost and Performance Notes
- Trivy scans are fast and free; scanning a single image SBOM often completes in under a minute
- Dependency-Track bundled edition is lightweight but not intended for large-scale production; consider external databases for scaling

---
## Summary
You now have a repeatable process to:
- Run an AI application container as a target
- Generate a CycloneDX SBOM with Trivy
- Stand up Dependency-Track and configure a project and API key
- Upload and analyze your SBOM via API

With this in place, you can visualize components and track vulnerabilities across your AI projects as they evolve.