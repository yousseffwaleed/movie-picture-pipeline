# Movie Picture Pipeline

A containerized full-stack movie catalog that demonstrates how to build, test, package, and deploy independent frontend and backend services to Amazon EKS.

The project combines a React interface with a Flask REST API. GitHub Actions runs linting and tests, Docker packages each service, Amazon ECR stores versioned images, and Kubernetes deploys the services to EKS with Kustomize.

## Project Highlights

- Browse a small movie catalog from a React frontend.
- Select a movie to view its details.
- Serve movie data through a Flask API.
- Keep frontend and backend as independently deployable services.
- Run automated lint and test checks in GitHub Actions.
- Build Docker images tagged with the Git commit SHA.
- Publish images to Amazon ECR.
- Deploy Kubernetes workloads and LoadBalancer Services to Amazon EKS.
- Provision supporting AWS infrastructure with Terraform.

## Architecture

```text
                         +----------------------+
                         |       Browser        |
                         +----------+-----------+
                                    |
                         +----------v-----------+
                         | Frontend LoadBalancer|
                         +----------+-----------+
                                    |
                         +----------v-----------+
                         | React container :3000|
                         +----------+-----------+
                                    |
                         HTTP API requests
                                    |
                         +----------v-----------+
                         | Backend LoadBalancer |
                         +----------+-----------+
                                    |
                         +----------v-----------+
                         | Flask container :5000|
                         +----------------------+

                 Docker images -> Amazon ECR -> Amazon EKS
```

The frontend and backend are deployed separately. The frontend must use the backend's reachable URL in `REACT_APP_MOVIE_API_URL`; `localhost` only works when both services are accessed locally.

## Technology Stack

| Layer | Tools |
| --- | --- |
| Frontend | React 18, Axios, Jest, React Testing Library |
| Backend | Python 3.10, Flask, pytest, Flake8 |
| Containers | Docker |
| Orchestration | Kubernetes, Kustomize, Amazon EKS |
| Registry | Amazon ECR |
| Infrastructure | Terraform, AWS VPC, IAM, EKS, EC2 |
| Automation | GitHub Actions |

## Repository Structure

```text
.
├── .github/workflows/
│   ├── backend-ci.yaml       # Backend lint, test, and image build checks
│   ├── backend-cd.yaml       # Backend image publication and EKS deployment
│   ├── frontend-ci.yaml      # Frontend lint, test, and image build checks
│   └── frontend-cd.yaml      # Frontend image publication and EKS deployment
├── setup/
│   ├── init.sh               # AWS/GitHub Actions setup helper
│   └── terraform/            # AWS infrastructure definitions
└── starter/
    ├── backend/
    │   ├── movies/            # Flask API and movie resources
    │   ├── k8s/               # Backend Deployment and Service
    │   ├── Dockerfile
    │   └── Pipfile
    └── frontend/
        ├── src/               # React application and components
        ├── k8s/               # Frontend Deployment and Service
        ├── Dockerfile
        └── package.json
```

## Backend API

The API exposes movie resources under `/movies`.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/movies` | Return the movie catalog |
| `GET` | `/movies/<id>` | Return one movie's details |
| `POST` | `/movies` | Registered route for creating a movie |
| `PUT` | `/movies/<id>` | Registered route for updating a movie |
| `DELETE` | `/movies/<id>` | Registered route for deleting a movie |

The current implementation uses an in-memory sample dataset, so data is not persisted between application restarts.

Example response from `GET /movies`:

```json
{
  "movies": [
    { "id": "123", "title": "Top Gun: Maverick" },
    { "id": "456", "title": "Sonic the Hedgehog" },
    { "id": "789", "title": "A Quiet Place" }
  ]
}
```

## Run Locally

### Backend

Requirements: Python 3.10 and Pipenv.

```bash
cd starter/backend
pipenv install --dev
pipenv run serve
```

The API runs at `http://localhost:5000`.

Run backend checks:

```bash
pipenv run lint
pipenv run test
```

### Frontend

Requirements: Node.js 18 and npm.

```bash
cd starter/frontend
npm ci
```

Start the React development server:

```bash
# macOS/Linux
REACT_APP_MOVIE_API_URL=http://localhost:5000 npm start

# Windows PowerShell
$env:REACT_APP_MOVIE_API_URL = "http://localhost:5000"
npm start
```

The frontend runs at `http://localhost:3000`.

Run frontend checks:

```bash
npm run lint
npm test -- --watchAll=false
npm run build
```

`REACT_APP_MOVIE_API_URL` is embedded into the React bundle during the build. Set it before starting or building the frontend.

## Run with Docker

Build and run the backend:

```bash
cd starter/backend
docker build -t movie-backend:local .
docker run --rm --name movie-backend -p 5000:5000 movie-backend:local
```

In another terminal, build and run the frontend:

```bash
cd starter/frontend
docker build \
  --build-arg REACT_APP_MOVIE_API_URL=http://localhost:5000 \
  -t movie-frontend:local .
docker run --rm --name movie-frontend -p 3000:3000 movie-frontend:local
```

Open `http://localhost:3000` after both containers are running.

## Kubernetes Deployment

The Kubernetes manifests are in `starter/backend/k8s` and `starter/frontend/k8s`. Each service contains a Deployment and a `LoadBalancer` Service.

Inspect the cluster:

```bash
kubectl get nodes
kubectl get pods -o wide
kubectl get services -o wide
```

Apply the backend:

```bash
cd starter/backend/k8s
kustomize edit set image backend=<ECR_BACKEND_REPOSITORY>:<TAG>
kustomize build | kubectl apply -f -
```

Apply the frontend:

```bash
cd starter/frontend/k8s
kustomize edit set image frontend=<ECR_FRONTEND_REPOSITORY>:<TAG>
kustomize build | kubectl apply -f -
```

The frontend image must be built with the reachable backend URL:

```bash
docker build \
  --build-arg REACT_APP_MOVIE_API_URL=http://<backend-load-balancer-hostname> \
  -t <ECR_FRONTEND_REPOSITORY>:<TAG> .
```

For temporary local verification when a LoadBalancer address is unavailable:

```bash
kubectl port-forward service/backend 5000:80
```

Then open `http://localhost:5000/movies`. This proves the backend Service and pod are reachable through Kubernetes, but it is not public LoadBalancer proof.

## CI/CD Workflows

The workflows run on `master` according to the repository configuration. CI workflows can also be started manually.

### Continuous Integration

For each service, CI:

1. Checks out the repository.
2. Installs the service dependencies.
3. Runs linting.
4. Runs tests.
5. Builds a Docker image after lint and test jobs pass.

Frontend CI uses Node.js 18 and npm. Backend CI uses Python 3.10 and Pipenv.

### Continuous Deployment

CD workflows run after changes to the matching service are pushed to the deployment branch or when manually dispatched. They:

1. Configure AWS credentials from GitHub repository secrets.
2. Authenticate with Amazon ECR.
3. Build an image tagged with `${{ github.sha }}`.
4. Push the image to ECR.
5. Configure access to the EKS cluster.
6. Update the Kubernetes image with Kustomize.
7. Apply the rendered manifests with `kubectl`.

Required GitHub repository secrets:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

Do not commit AWS credentials or temporary session tokens. For production, use short-lived federated credentials or GitHub OIDC instead of long-lived access keys.

## AWS Infrastructure

Terraform in `setup/terraform` provisions the supporting resources, including:

- VPC networking and subnets
- Internet gateway and route tables
- Amazon EKS cluster and node group
- Amazon ECR repositories
- IAM roles and policies

Typical commands:

```bash
cd setup/terraform
terraform init
terraform plan
terraform apply
terraform output
```

Configure local EKS access after the cluster is available:

```bash
aws eks update-kubeconfig --name cluster --region us-east-1
kubectl get nodes
```

Review infrastructure changes carefully and run `terraform destroy` when temporary resources are no longer needed to avoid AWS charges.

## Troubleshooting

### `EXTERNAL-IP` remains `<pending>`

Check the Service events:

```bash
kubectl describe service backend
kubectl get events --sort-by=.lastTimestamp
```

For EKS Auto Mode, the load balancer controller needs permission to inspect the VPC and route tables, and it needs eligible subnets. An AWS Organizations Service Control Policy can override account-level IAM permissions. If events report an explicit deny for `ec2:DescribeRouteTables`, an AWS Organizations administrator must update that policy.

### `kubectl` asks for credentials

Refresh the AWS session and kubeconfig:

```bash
aws sts get-caller-identity
aws eks update-kubeconfig --name cluster --region us-east-1
```

Temporary AWS credentials expire and must not be committed or shared.

### Frontend cannot reach the API

Check the value used during the frontend build. In Kubernetes, `localhost` points to the browser's machine, not the backend pod. Use the backend's public hostname or an appropriate internal Service URL instead.

## Deployment Verification

After both LoadBalancer Services have external hostnames, verify:

```text
http://<backend-load-balancer-hostname>/movies
http://<frontend-load-balancer-hostname>
```

The backend URL should return JSON, and the frontend URL should display the movie list. For portfolio evidence, capture screenshots with the browser address bar visible and include successful Backend CD and Frontend CD workflow runs.

## License

See [LICENSE.md](LICENSE.md).
