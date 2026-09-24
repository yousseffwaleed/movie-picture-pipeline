# Movie Picture Pipeline

A production-minded movie catalog application built to demonstrate full-stack development and automated cloud delivery.

The project combines a React frontend with a Flask API and uses GitHub Actions to test, containerize, and deploy both services to Kubernetes on Amazon EKS. Images are stored in Amazon ECR and tagged with the commit SHA, making each deployment traceable to an exact version of the source code.

## Highlights

- Browse a movie catalog and view movie details in the React application.
- Serve movie data through a Flask REST API.
- Run frontend and backend tests and linting in GitHub Actions.
- Build reproducible Docker images for both services.
- Publish images to Amazon ECR.
- Deploy versioned images to an Amazon EKS cluster with Kustomize.

## Architecture

```text
Browser
  |
  v
React frontend :3000 -----> Flask API :5000
       |                         |
       +------ Docker -----------+
                 |
        Amazon ECR -> Amazon EKS
```

The frontend is a React 18 application. The backend is a Python 3.10 Flask application that exposes the `/movies` endpoint. Kubernetes manifests for each service are in the corresponding `k8s` directory.

## Technology Stack

| Area | Technology |
| --- | --- |
| Frontend | React, Axios, Jest, React Testing Library |
| Backend | Python, Flask, pytest, Flake8 |
| Containers | Docker |
| Cloud | AWS ECR and Amazon EKS |
| Delivery | GitHub Actions, Kubernetes, Kustomize |

## Repository Structure

```text
starter/
├── backend/
│   ├── movies/              # Flask API and movie resources
│   ├── k8s/                 # Backend Kubernetes manifests
│   ├── Dockerfile
│   └── Pipfile
└── frontend/
    ├── src/                 # React application and components
    ├── k8s/                 # Frontend Kubernetes manifests
    ├── Dockerfile
    └── package.json
```

## Run Locally

### Backend

Requirements: Python 3.10 and Pipenv.

```bash
cd starter/backend
pipenv install
pipenv run serve
```

The API is available at `http://localhost:5000/movies`.

Run the backend checks with:

```bash
pipenv run test
pipenv run lint
```

### Frontend

Requirements: Node.js and npm.

```bash
cd starter/frontend
npm ci
REACT_APP_MOVIE_API_URL=http://localhost:5000 npm start
```

The frontend is available at `http://localhost:3000`.

Run the frontend checks with:

```bash
npm test -- --watchAll=false
npm run lint
npm run build
```

`REACT_APP_MOVIE_API_URL` is embedded into the React production build. For a deployed environment, set it to the public URL of the backend rather than `localhost`.

## Run with Docker

Start the backend:

```bash
cd starter/backend
docker build -t mp-backend:latest .
docker run --name mp-backend -p 5000:5000 -d mp-backend:latest
```

Build and start the frontend:

```bash
cd starter/frontend
docker build --build-arg REACT_APP_MOVIE_API_URL=http://localhost:5000 -t mp-frontend:latest .
docker run --name mp-frontend -p 3000:3000 -d mp-frontend:latest
```

Open `http://localhost:3000` after both containers are running.

## CI/CD

The workflows are in `.github/workflows`:

- `frontend-ci.yaml` and `backend-ci.yaml` run on pull requests that change the relevant application and can also be started manually. Linting and tests run before the build.
- `frontend-cd.yaml` and `backend-cd.yaml` run on pushes to `main` that change the relevant application and can also be started manually.

The deployment pipeline:

1. Builds a Docker image tagged with `${{ github.sha }}`.
2. Pushes the image to Amazon ECR.
3. Updates the Kubernetes image reference with the same commit tag.
4. Builds the manifests with Kustomize and applies them to Amazon EKS.

Deployment requires AWS credentials and cluster access configured for GitHub Actions. The required repository secrets are `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

To apply manifests manually after configuring AWS and Kubernetes access:

```bash
cd starter/frontend/k8s
kustomize edit set image frontend=<ECR_REPOSITORY>:<TAG>
kustomize build | kubectl apply -f -
```

Use the equivalent commands in `starter/backend/k8s` for the API.

## AWS Infrastructure

The `setup/terraform` directory contains the Terraform configuration used to provision the supporting AWS resources. Review the expected changes before applying infrastructure, and destroy temporary resources when they are no longer needed to avoid unnecessary charges.

```bash
cd setup/terraform
terraform apply
terraform output
```

The `setup/init.sh` helper configures the GitHub Actions IAM user for Kubernetes access.

## License

[License](LICENSE.md)
