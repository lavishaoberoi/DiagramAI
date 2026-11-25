# DiagramAI Deployment Guide

**IBM Watson Studio / IBM Cloud — Step-by-Step Deployment**

This guide explains how to configure, run, containerize, and deploy DiagramAI using IBM Cloud, Watsonx.ai, IBM Code Engine, and Docker.

---

## Table of Contents

1. [Phase 1 — IBM Cloud Setup](#phase-1--ibm-cloud-setup)
2. [Phase 2 — Local Development Setup](#phase-2--local-development-setup)
3. [Phase 3 — IBM Code Engine Deployment](#phase-3--ibm-code-engine-deployment)
4. [Phase 4 — Docker Setup](#phase-4--docker-setup)
5. [Phase 5 — Production Checklist](#phase-5--production-checklist)

---

# Phase 1 — IBM Cloud Setup

## Step 1: Create an IBM Cloud Account

1. Visit [IBM Cloud Registration](https://cloud.ibm.com/registration).
2. Complete the registration process.
3. Verify your email address.
4. Log in to your IBM Cloud account.

## Step 2: Create an IBM Cloud API Key

1. Log in to [IBM Cloud](https://cloud.ibm.com).
2. Navigate to **Manage → Access (IAM) → API Keys**.
3. Click **Create an IBM Cloud API key**.
4. Enter the following name:

   `diagramai-key`

5. Generate and copy the API key.

   **Important:** The API key cannot be viewed again after leaving the creation screen. Store it securely.

6. Add the key to your `.env` file:

```env
IBM_CLOUD_API_KEY=<your_key>
```

## Step 3: Create a Watson Studio / watsonx.ai Project

1. Visit [IBM Cloud Pak for Data](https://dataplatform.cloud.ibm.com).
2. Click **Create a project → Create an empty project**.
3. Enter the project name:

   `DiagramAI`

4. Select an IBM Cloud Object Storage instance.
   - Create one if required.
   - The Lite tier may be available at no cost.
5. Click **Create**.
6. Open **Project Settings → General**.
7. Copy the Project ID.

Add it to your `.env` file:

```env
WATSONX_PROJECT_ID=<your_project_id>
```

## Step 4: Enable watsonx.ai and Granite Models

1. Open your Watson Studio project.
2. Click **+ Add to project → Foundation model**, where available.
3. Locate the Granite model:

   `ibm/granite-13b-instruct-v2`

4. Ensure that your IBM Cloud account has access to watsonx.ai.
5. Activate watsonx.ai if required:

   [IBM watsonx.ai](https://dataplatform.cloud.ibm.com/wx/home)

6. The `ibm-watsonx-ai` SDK manages SaaS foundation model endpoints. A separate manual model deployment is not required for supported SaaS models.

Add the model configuration to `.env`:

```env
GRANITE_MODEL_ID=ibm/granite-13b-instruct-v2
```

---

# Phase 2 — Local Development Setup

## Step 1: Create a Python Virtual Environment

```bash
python -m venv .venv
```

Activate the environment.

**Windows:**

```bash
.venv\Scripts\activate
```

**macOS / Linux:**

```bash
source .venv/bin/activate
```

## Step 2: Install Dependencies

Install all required Python packages:

```bash
pip install -r requirements.txt
```

## Step 3: Install Tectonic (LaTeX Compiler)

DiagramAI requires a LaTeX compiler to compile generated TikZ code into PDF diagrams.

**Windows (using Chocolatey):**

```bash
choco install tectonic
```

**macOS:**

```bash
brew install tectonic
```

**Linux:**

```bash
curl --proto '=https' --tlsv1.2 -fsSL https://drop.rust-lang.org/tectonic-installer.sh | sh
```

Alternatively, use `pdflatex` by configuring:

```env
LATEX_COMPILER=pdflatex
```

## Step 4: Install Poppler

Poppler is required for converting generated PDFs into PNG images.

**Windows:**

Download Poppler from:

https://github.com/oschwartz10612/poppler-windows/releases

**macOS:**

```bash
brew install poppler
```

**Ubuntu / Debian:**

```bash
sudo apt-get install poppler-utils
```

## Step 5: Configure Environment Variables

Copy the example environment file:

```bash
cp .env.example .env
```

On Windows Command Prompt, use:

```cmd
copy .env.example .env
```

Edit `.env` and configure the required credentials and settings:

```env
IBM_CLOUD_API_KEY=<your_key>
WATSONX_PROJECT_ID=<your_project_id>
GRANITE_MODEL_ID=ibm/granite-13b-instruct-v2
FLASK_SECRET_KEY=<your_secret_key>
LATEX_COMPILER=tectonic
```

**Security:** Never commit your `.env` file or expose API keys in a public repository.

## Step 6: Run DiagramAI Locally

Start the Flask application:

```bash
python app.py
```

Open the application in your browser:

http://localhost:5000

---

# Phase 3 — IBM Code Engine Deployment (Recommended)

IBM Code Engine provides a managed platform for deploying containerized applications without managing the underlying infrastructure.

## Step 1: Install IBM Cloud CLI

Download and install the IBM Cloud CLI:

https://cloud.ibm.com/docs/cli

Verify installation:

```bash
ibmcloud --version
```

## Step 2: Log In to IBM Cloud

Authenticate using single sign-on:

```bash
ibmcloud login --sso
```

If required, select the appropriate account and region.

## Step 3: Install the Code Engine Plugin

```bash
ibmcloud plugin install code-engine
```

Verify the installation:

```bash
ibmcloud plugin list
```

## Step 4: Create a Code Engine Project

Create the project:

```bash
ibmcloud ce project create --name diagramai-project
```

Select the project:

```bash
ibmcloud ce project select --name diagramai-project
```

## Step 5: Create a Container Registry Namespace

Create a namespace in IBM Cloud Container Registry:

```bash
ibmcloud cr namespace-add diagramai-ns
```

This step is optional if you are using another supported image registry, but is recommended for IBM Cloud deployments.

## Step 6: Build and Push the Docker Image

Build the Docker image:

```bash
docker build -t us.icr.io/diagramai-ns/diagramai:latest .
```

Authenticate with IBM Container Registry:

```bash
ibmcloud cr login
```

Push the image:

```bash
docker push us.icr.io/diagramai-ns/diagramai:latest
```

## Step 7: Create Code Engine Secrets

Create a secret containing sensitive environment variables:

```bash
ibmcloud ce secret create --name diagramai-env \
  --from-literal IBM_CLOUD_API_KEY=<your_key> \
  --from-literal WATSONX_PROJECT_ID=<your_project_id> \
  --from-literal FLASK_SECRET_KEY=<secure_random_string>
```

**Important:** Do not hardcode credentials in your Dockerfile or application source code.

## Step 8: Deploy the Application

Deploy DiagramAI to IBM Code Engine:

```bash
ibmcloud ce application create \
  --name diagramai \
  --image us.icr.io/diagramai-ns/diagramai:latest \
  --registry-secret ce-auto-icr-private-us-south \
  --env-from-secret diagramai-env \
  --env GRANITE_MODEL_ID=ibm/granite-13b-instruct-v2 \
  --env LATEX_COMPILER=tectonic \
  --env FLASK_ENV=production \
  --port 5000 \
  --min-scale 1 \
  --max-scale 3 \
  --cpu 1 \
  --memory 4G
```

This configuration specifies:

| Configuration | Value |
|---|---|
| Application Name | `diagramai` |
| Container Port | `5000` |
| Minimum Instances | `1` |
| Maximum Instances | `3` |
| CPU Allocation | `1 vCPU` |
| Memory Allocation | `4 GB` |
| Environment | Production |
| LaTeX Compiler | Tectonic |

**Note:** Ensure the registry secret name matches the registry configuration and region associated with your deployment.

## Step 9: Retrieve the Public Application URL

Run:

```bash
ibmcloud ce application get --name diagramai --output url
```

The command returns the public URL of your deployed application.

Open the URL in your browser to access DiagramAI.

---

# Phase 4 — Docker Setup

DiagramAI supports containerized deployment using Docker.

The Dockerfile in the project root contains the container configuration and defines the dependencies required to run the application.

Build the image:

```bash
docker build -t diagramai:latest .
```

Run the container locally:

```bash
docker run -p 5000:5000 diagramai:latest
```

Access the application:

http://localhost:5000

For additional Docker configuration, refer to the `Dockerfile` included in the project repository.

---

# Phase 5 — Production Deployment Checklist

Before deploying DiagramAI to production, verify the following:

### Security

- [ ] Set `FLASK_SECRET_KEY` to a cryptographically secure random string of at least 32 characters.
- [ ] Set `FLASK_ENV=production`.
- [ ] Set `FLASK_DEBUG=0`.
- [ ] Never expose IBM Cloud API keys in source code.
- [ ] Ensure `.env` is included in `.gitignore`.
- [ ] Restrict CORS origins in `app.py` by replacing `*` with your application domain.
- [ ] Review IBM Cloud IAM permissions and follow the principle of least privilege.

### Infrastructure

- [ ] Enable HTTPS. IBM Code Engine provides HTTPS for application endpoints.
- [ ] Configure IBM Cloud Object Storage for persistent diagram and file storage.
- [ ] Configure IBM Cloud Monitoring for application usage and resource tracking.
- [ ] Verify container resource allocations.
- [ ] Confirm that Tectonic and Poppler are correctly installed in the production container.

### Testing and Validation

- [ ] Verify IBM Cloud API authentication.
- [ ] Verify watsonx.ai project configuration.
- [ ] Test Granite model connectivity.
- [ ] Test natural-language-to-TikZ generation.
- [ ] Verify LaTeX compilation.
- [ ] Test PDF-to-PNG conversion.
- [ ] Test all application API endpoints.
- [ ] Verify the deployed application through its public URL.

---

## Deployment Summary

| Phase | Description | Status |
|---|---|---|
| Phase 1 | IBM Cloud and watsonx.ai configuration | Setup Required |
| Phase 2 | Local development environment | Setup Required |
| Phase 3 | IBM Code Engine deployment | Deployment Required |
| Phase 4 | Docker containerization | Configuration Required |
| Phase 5 | Production validation | Pending |

---

## Additional Resources

- [IBM Cloud](https://cloud.ibm.com/)
- [IBM Cloud CLI Documentation](https://cloud.ibm.com/docs/cli)
- [IBM watsonx.ai](https://www.ibm.com/products/watsonx-ai)
- [IBM Cloud Code Engine](https://cloud.ibm.com/docs/codeengine)
- [IBM Cloud Container Registry](https://cloud.ibm.com/docs/Registry)
- [Tectonic Documentation](https://tectonic-typesetting.github.io/)
- [Docker Documentation](https://docs.docker.com/)

---

**Project:** DiagramAI  
**Platform:** IBM Cloud / watsonx.ai  
**Deployment Target:** IBM Code Engine  
**Application Framework:** Python / Flask  
**AI Model:** IBM Granite  
**Containerization:** Docker  

