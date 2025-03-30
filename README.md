# highpeaks-devops-agent

The **High Peaks DevOps Agent** is an AI-powered assistant designed to help with DevOps and GitOps tasks. This service is envisioned as a chatbot/agent that can analyze repository changes, CI/CD pipeline states, and assist engineers by providing insights or automating tasks. It leverages large language models and the Flowise-managed flows to interpret commands and carry out actions in the DevSecOps platform. 

At this scaffold stage, the DevOps agent is a minimal FastAPI application that will serve as the foundation for these capabilities.

## Repository Structure

```text
highpeaks-devops-agent/
├── README.md               # Overview of the DevOps AI agent service
├── Dockerfile              # Containerization instructions for the FastAPI service
├── requirements.txt        # Python dependencies for FastAPI and related libraries
├── main.py                 # FastAPI app implementing basic endpoints
├── k8s/
│   ├── base/
│   │   ├── deployment.yaml # Base Kubernetes Deployment for the agent
│   │   ├── service.yaml    # Service to expose the agent (internal service)
│   │   └── kustomization.yaml # Kustomize base configuration listing resources
│   └── overlays/
│       └── dev/
│           ├── kustomization.yaml # Overlay for development (e.g., enable debug mode)
│           └── patch-debug.yaml   # Kustomize patch example (adds DEBUG env variable)
└── .github/
    └── workflows/
        └── ci.yml         # GitHub Actions workflow for CI (linting, tests, manifest validation)
```

## Development Setup

**Prerequisites:** Python 3.x and pip (similar to the ML service requirements).

1. **Create a virtual environment (optional):**  
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   ```
2. **Install dependencies:**  
   ```bash
   pip install -r requirements.txt
   ```
   This will install FastAPI, Uvicorn, and any other required libraries.
3. **Run the service locally:**  
   ```bash
   uvicorn main:app --host 0.0.0.0 --port 8000
   ``` 
   This launches the FastAPI server on port 8000. Test it by accessing `http://localhost:8000/health` (should return a status message). You can also test the placeholder assistant endpoint by sending a POST request to `http://localhost:8000/assist` with a JSON body, e.g., `{"query": "How is the system status?"}`. It will return a dummy response.
4. **Build the Docker image:**  
   ```bash
   docker build -t highpeaks-devops-agent:latest .
   ```
   Then run it locally if desired:  
   ```bash
   docker run -p 8000:8000 highpeaks-devops-agent:latest
   ``` 
   and test the endpoints as above (the service inside the container also listens on port 8000).

## Kubernetes Deployment (GitOps Agent Service)

Kustomize templates are provided to deploy the DevOps agent:
- **Base manifests (`k8s/base/`):** Define a Deployment (with one replica of the agent by default) and a ClusterIP Service on port 80 (targeting the agent's port 8000). The deployment uses the image `highpeaks-devops-agent:latest` by default.
- **Dev overlay (`k8s/overlays/dev`):** An example overlay that patches the base deployment to add a debug environment variable (`DEBUG=true`). This shows how one might adjust settings for different environments (development, staging, production) without duplicating entire manifests.

To deploy to a cluster (e.g., Kind):
1. Build and load the Docker image into the cluster (see step 4 above).
2. Apply the base manifests (or the dev overlay for a development environment):
   ```bash
   # Using base manifests directly:
   kubectl apply -f k8s/base/deployment.yaml
   kubectl apply -f k8s/base/service.yaml
   # OR using Kustomize (recommended):
   kubectl apply -k k8s/overlays/dev
   ``` 
   If using the overlay, Kustomize will automatically include the base and apply the patch.
3. Once deployed, the agent service will be accessible within the cluster (e.g., other services can call `http://highpeaks-devops-agent:80/health`). For local testing, you can port-forward:
   ```bash
   kubectl port-forward svc/highpeaks-devops-agent 8000:80
   ```
   and then access `http://localhost:8000/health` to verify it's running.

## CI/CD

The CI workflow for the DevOps agent service performs the following on each push/PR:
- Installs dependencies and runs linting (flake8) and any simple tests (in this scaffold, no real tests are included yet).
- Checks that the Kubernetes manifests are valid by running a dry-run `kubectl apply` via Kustomize.
- Attempts to build the Docker image to ensure the Dockerfile is correct.

In a full implementation, this service would have unit tests for its logic (e.g., parsing Git events, interacting with Flowise or the identity service), and possibly integration tests to simulate agent behavior. Security scans (for vulnerabilities in Python packages or container image) would also be included.

## Future Enhancements

As development progresses, the DevOps agent will integrate more deeply with the platform:
- **Flowise Integration:** The agent will use flows designed in the Flowise service to decide how to handle certain requests or incidents. For instance, a Flowise flow could define the steps to troubleshoot a failing deployment, and the agent would execute those steps.
- **Identity and Authorization:** The agent might verify the identity of users asking it to perform actions (ensuring that, for example, only authorized users can trigger a production deployment via the agent).
- **Observability:** The agent will emit logs and metrics (e.g., via OpenTelemetry) for its actions, which can be monitored by the platform's logging and monitoring tools.
