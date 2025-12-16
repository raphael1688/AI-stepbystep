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

## What you will build

- A local Ollama container (example target) to scan
- Trivy installed and verified
- Dependency-Track (bundled edition) running in Docker with persistent storage
- A Dependency-Track project and API key
- A CycloneDX SBOM JSON file generated from your AI container image
- A Python script to POST the SBOM to Dependency-Track

## Step 1: Prepare the target AI application container (example: Ollama)

We will run a containerized AI application to have a concrete target for SBOM generation. Run the AI container (Ollama example; adjust image and ports if needed):

```bash
docker run -d --name ollama -p 11434:11434 ollama/ollama:latest

```

Verify the container is running:

```bash
docker ps
curl -s http://localhost:11434
```

Optional: use Docker Compose instead of docker run:

```bash
mkdir -p ollama-lab && cd ollama-lab
cat > compose.yaml <<'EOF'
services:
  ollama:
    image: ollama/ollama
    container_name: ollama
    ports:
      - "0.0.0.0:11434:11434"
    volumes:
      - model_data:/root/.ollama
    environment:
      - OLLAMA_KEEP_ALIVE=-1
      - OLLAMA_MAX_LOADED_MODELS=5
      - OLLAMA_NUM_PARALLEL=4
    networks:
      - llmserver-labnet
    restart: unless-stopped

volumes:
  model_data:

networks:
  llmserver-labnet:
    external: true
EOF

docker compose up -d
```

Notes:

- If you use a different AI application, substitute its image name and health ports.
- In a lab environment, localhost may be replaced with the host IP or a shared ingress.

## Step 2: Install Trivy

We will install Trivy, an open-source scanner that can generate SBOMs in CycloneDX format. Our example was run on Ubuntu.

Option A (Official install script, works widely):

```bash
# Download and install the trivy binary to /usr/local/bin
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin
sudo snap install trivy

# Verify
trivy --version
```

Option B (APT repo, Ubuntu/Debian):

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

```bash
# Pull the bundled image
docker pull dependencytrack/bundled

# Create a persistent volume
docker volume create --name dependency-track

# Run Dependency-Track
docker run -d -m 8192m -p 8080:8080 --name dependency-track -v dependency-track:/data dependencytrack/bundled

```

---

## Step 4: Initial Dependency-Track configuration (admin and API key)

Log in, change the admin password, and generate an API key for uploads.

Steps in the UI:

1. Open http://HOST_IP:8080 and log in with admin / admin; set a new password.
2. Go to Administration -> Access Management -> Teams -> Administrators.
3. Click the plus icon to create a new API Key for the Administrators team.
4. Copy the API key and store it securely (you will only see it once).

![Copy the key! You only see it once!](images/01-dt_key.png)

You will also create a project for your AI application:

- Navigate to Projects -> New Project
- Name: Ollama Lab (or your preferred name)
- Classifier: Container
- Version: 1.0.1 (example)
- Team: Administrators (or another team you prefer)
- Mark Latest if appropriate
- Save the project

![Define your project.](images/02-project.png)

---

## Step 5: Generate a CycloneDX SBOM for your AI container image

We will use Trivy to produce a CycloneDX JSON SBOM for the container image you are running.

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
trivy image --format cyclonedx --scanners vuln ollama/ollama --o ollama_lab.cdx.json

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

Create a working directory and the script:

```bash
mkdir -p sbom_upload
cd sbom_upload

# Create and edit the Python script
cat > dt_sbom_upload.py <<'EOF'
import argparse
import sys
import requests

parser = argparse.ArgumentParser(prog = 'dt_sbom_uploader',description = 'Upload a sbom to Dependency Track')
parser.add_argument('-s', '--sbom', help='Path to the SBOM file to upload') 
parser.add_argument('-n', '--name', help='Dependency Track project name') 
parser.add_argument('-v', '--version', help='Dependency Track project version')
parser.add_argument('-u', '--url', help='Dependency Track base URL like http://localhost:8080')
parser.add_argument('-k', '--key', help='Dependency Track API Key')
args = parser.parse_args()

if (args.sbom is None or args.name is None or args.version is None or args.url is None or args.key is None):
    parser.print_help()
    sys.exit(1)

try:
    data = open(args.sbom, 'rb').read()
except:
    print ('Error: could not open file '+args.sbom+' for reading')
    sys.exit(1)

# Build the POST request
post_data = {}
post_data['projectName'] = args.name
post_data['projectVersion'] = args.version
post_data['autoCreate'] = 'true'
bomFile = {'bom': data}
api_key_header = {'X-Api-Key': args.key}

r = requests.post(url=args.url+'/api/v1/bom', files=bomFile, data=post_data, headers=api_key_header)

print (r)
print (r.text)
EOF
```

Install dependencies and run:

```bash
# Execute the upload script - your IP and key will be different. This is what we ran.
python3 dt_sbom_upload.py -s ./ollama_lab.cdx.json -n OllamaLab -v 1.0.1 -u http://10.1.1.4:8080 -k odt_lPW0NbQx_v9C3hwGxEpSQeqfdGAqK0kD3s2YyvxBW
```

Expected result:

- Status code: 200
- Response includes a token (used internally by Dependency-Track to process the BOM)

---

## Step 7: Review results in Dependency-Track

Reload your project page and confirm components are populated. Newly installed software often shows few or no vulnerabilities, but the components list should be present.

In the UI:

- Go to Projects -> select "Ollama Lab" (version 1.0.1)
- Check Overview, Components, Services, Dependency Graph, Audit, Vulnerabilities, Exploit Predictions
- Verify the score and component list reflect the uploaded SBOM
- Expect multiple pages of components (base image layers, OS packages, and application dependencies)

![Verify your SBOM data](images/03-verify.png)

---

## Optional Enhancements

- Automate SBOM generation and upload in CI or CD (for example, GitHub Actions, GitLab CI)
- Scan multiple services and versions and organize them by team and projects
- Schedule periodic re-scans and uploads to catch newly disclosed CVEs
- Enable notifications or webhooks in Dependency-Track when risk changes
- Add license policy checks and governance workflows
- Try adding repositories and files

---

## Troubleshooting

- Dependency-Track not accessible
  - Ensure the container is running: docker ps
  - Verify port 8080 is open and not blocked by a firewall
  - Check logs: docker logs dtrack
- Login or API key issues
  - Default admin/admin works only on first login, then you must change the password
  - API keys are shown once; if lost, generate a new key
- Trivy install problems
  - Try the install script if apt is unavailable
  - Check trivy --version to ensure it is installed
- Image naming confusion
  - Use docker ps and docker images to confirm the exact image and tag
  - For a named container, you can resolve its image with:
    docker inspect --format='{{.Config.Image}}' <container_name>

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
